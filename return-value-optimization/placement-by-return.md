# `placement-by-return`

This is a collection of thoughts/ideas on the [`placement-by-return` RFC][pbr-rfc].

[pbr-rfc]: https://github.com/rust-lang/rfcs/pull/2884

## Applications

These are the applications that I think this feature can be used for if extended
appropriately:

1. creating (allocating + initializing) large objects,
2. reliably ensuring RVO,
3. allow creation without introducing new APIs
4. allow creation to be fallible,
5. allow creation of pinned data,
6. allow creation of self-referential data.

Some of these points are already implemented (1.) and some are addressed in the
[RFC][pbr-rfc] as problems (2., 3. and 4.).

Now I will go over each point and present motivation for that feature along with
a possible solution:

## Point 2: Lack of explicity

### Motivation

Users of this feature want to reliably know when RVO kicks in and when not. When
working in an embedded/kernel/systems programming environment, allocations on
the stack might be tightly constrained [^1].


[^1]: View [this](https://github.com/Rust-for-Linux/linux/issues/879) issue for more.


### [RFC][pbr-rfc]'s solution

[RFC][pbr-rfc] names lints as the solution to this problem, but I do not think
that this will be a particularly good solution:

- either everyone will see these lints even if they do not care about if their
  functions create 1k sized arrays on the stack, or users have to opt-in to see
  the lints. This presents an issue, if a user discovers this feature on e.g.
  stackoverflow and does not read/get the information about having to enable the
  lint,
- not every pattern can be caught from the start. While rustc's lints and error
  messages are amazing, they had to be specifically designed this way and are a
  *lot* of effort, so it is going to take time until every pattern is detected
  and gives a reasonable lint message.
- lints do not enable `unsafe` code to rely on additional properties. When
  trying to create pinned data ([point 5][point-5-pinned-data]) it would be
  great to be able to rely on certain data to be pinned after successful creation.

### New solution

I would propose adding a way to tell the compiler that one would like to enable
RVO on a certain function. It could be an attribute on the function itself, a
keyword or some other kind of marker on the return type:

```rust
#[in_place]
pub fn new() -> MyStruct {
    MyStruct { ... }
}
```

This way the compiler

- could still lint for functions returning big types and suggest to use
  `#[in_place]`,
- ensure that for functions marked by `#[in_place]` RVO does indeed kick in,
  otherwise it would issue a compile error, now `unsafe` code can rely on RVO,

Additionally, this allows the feature to be

- better documented, every function needs to explicitly opt in and it will be
  reflected in the documentation
- a SemVer sensitive (in that adding `#[in_place]` would be ok, but removing it
  would be a breaking change),
- theoretically it could be expanded to take an argument like `#[inline]` does,
  some people might want to turn off RVO (I have no idea if this is desirable,
  but we would have the option to easily add it).

The optimizer would of course still be allowed to do RVO on other functions, but
it would not be guaranteed.

#### Disadvantages

Users will have to manually specify the attribute, new users could end up
confused (without the attribute, they would be in the dark, this can be viewed
as better or worse, depending on how you want to see it).
This would also require existing code which wants RVO to be augmented with the
attribute.

## Point 3: No new APIs

This point has already been addressed by the RFC in [this section](https://github.com/PoignardAzur/placement-by-return/blob/placement-by-return/text/0000-placement-by-return.md#lazy-parameters). Illustrating example from the [RFC][pbr-rfc]:
```rust
impl<T> Vec<T> {
    fn push(&mut self, lazy value: T);
}

some_vec.push(create_large_data(params));
```
I believe the need for new APIs to be a huge disadvantage. Most users will not
need to care, if their functions do emplacement or not. The people who *do* care
will be able to look up the documentation of `create_large_data`/`Vec::push`
and see `#[in_place]`/`lazy` and thus conclude that RVO is applied.
There could be a new lint that would warn when a value returned by a `#[in_place]`
function is copied and if the place it is copied to is the parameter of a
crate-local function the compiler could suggest adding `lazy` to that parameter.

`lazy` would of course not be allowed on all parameters, because we should
prevent the following footgun:
```rust
impl<T> Vec<T> {
    pub fn push(&mut self, lazy value: T) {
        self.inner_push(value)
        //              ^^^^^ error this lazy value is being evaluated on the
        //                    stack, defeating the purpose of lazy.
        //                    help: mark `value` of `inner_push` as lazy
    }

    fn inner_push(&mut self, value: T);
}
```
`lazy` would also allow the following pattern together with `#[in_place]`:
```rust
pub struct WithBuf<T> {
    data: T,
    buf: [u8; 1000_000_000],
}

impl<T> WithBuf<T> {
    #[in_place]
    pub fn new(lazy data: T) -> Self {
        Self {
            data,
            buf: [0; 1000_000_000]
        }
    }
}

let buf_buf = Box::new(WithBuf::new([1; 1000_000_000]));
```

## Point 4: Fallible creation

[point-4-fallible-creation]: #point-4-fallible-creation

### Motivation

Systems programming must be able to handle the rarest and most obscure errors.
Usersland programs most often choose to panic in these situations, but in the
kernel this is not an option. One such error is the out of memory error when
trying to allocate e.g. driver state. This state could live on the heap, but
might also store data that is itself on the heap at a different location.
It is necessary to allocate in the initializer and it might thus fail.

### [RFC][pbr-rfc]'s discussion

The [RFC][pbr-rfc] mentions [splitting the tag from the union][split-disc] and
achieving that via different ways:

| solution                                                                        | advantages   | disadvantages                                              |
|:--------------------------------------------------------------------------------|:------------:|------------------------------------------------------------|
| change the ABI of all `enum`s, storing the tag in a register/in pointer metadata|    -         | breaking change, this almost seems like a non-starter      |
|   only change the ABI of returning `enum`s to store the tag separately          | less breakage|  still a breaking change, RFC postpones solving this issue |


[split-disc]: https://github.com/PoignardAzur/placement-by-return/blob/placement-by-return/text/0000-placement-by-return.md#split-the-discriminant-from-the-payload

### New solution

Add a new function call ABI that is used when returning an `enum` from a `#[in_place]`
function. This ABI returns the tag in a register and places the union part
as usual in the caller supplied slot. An example:
```rust
pub struct Data<T> {
    data: T,
    buf: Box<[u8; 1000_000_000]>
}
impl<T> Data<T> {
    #[in_place]
    pub fn new(lazy data: T) -> Result<Self, AllocError> {
        Ok(Self {
            data,
            buf: Box::try_new([0; 1000_000_000])?,
        })
    }
}

fn main() -> Result<(), AllocError> {
    let data = Box::try_new(Data::new(fetch_data())?)?;
    handle_data(&*data);
    Ok(())
}
fn handle_data(data: &Data<[u32; 1000_000]>);
#[in_place]
fn fetch_data() -> [u32; 1000_000];
```
The compiler would allow exiting early from functions with `lazy` parameters
when the expression supplied as the parameter consists only of functions that
are `#[in_place]`. `Result` would need to be changed to allow `T, E: ?Sized` and
all unwrap functions be marked `#[in_place]` (as well as pattern matching
adjusted).

I am not really satisfied with this solution. It is relying too much on compiler
magic to make things work. We could also only allow this optimization for
`Result` and immediate `unwrap*`/`?` calls. As that will be the main usage.


## Point 5: Pinned data

[point-5-pinned-data]: #point-5-pinned-data

### Motivation

In the linux kernel the synchronization primitives (e.g. `mutex`) need to be
pinned, because they contain self-referential data structures (this section is
about initializing pinned data, not self-referential data structures, see the
[next][point-6-self-referential-data] section for that).
Because these need to be pinned, any type containing `Mutex<T>` (the safe
wrapper) needs to be also be pinned. One could solve this problem by using a
rust specific mutex without pinning, adding an additional indirection through
`Pin<Box<Mutex<T>>>`. But these solutions are not fitting for the strict
performance requirements the kernel has.

### The solution

Introduce a new attribute/keyword similar to `#[in_place]` named `#[pin_in_place]`.
It ensures that values yielded by the marked function will always be pinned
(during and after initialization). This can only be applied to functions with a
`!Unpin` return type. There also needs to be a way to mark parameters as
compatible with this attribute. For this post I am going to use the same
attribute, but on the parameter. These parameters of course also need to be
`lazy`. So `Box::try_pin` would look like this:
```rust
impl<T> Box<T> {
    pub fn try_new(#[pin_in_place] lazy data: T) -> Result<Pin<Self>, AllocErr>;
}
```
The implementation for `Box` will probably need to be special. Other uses of
`#[pin_in_place]` on a parameter will need to be delegated to functions with
also `#[pin_in_place]` on that parameter. Also `core::pin::Pin::pin!` should
allow the creation on the stack. Additionally some `unsafe` way to circumvent
the compiler check would need to be made available.

## Point 6: Self referential data

[point-6-self-referential-data]: #point-6-self-referential-data

### Motivation

As already mentioned in the last section, the kernel has self referential types
almost everywhere. It would be great to be able to initialize the `Mutex` purely
via rust (at the moment a C function is being called, I do not think this will
change any time soon, but the ability to do this in rust would be useful
elsewhere).


### The solution

In a function without a receiver parameter and that is marked `#[pin_in_place]`
users are able to use `self`. It will have `*mut $ret` as the type where `$ret`
is the return type of the function. This can be combined with [general field projection](https://internals.rust-lang.org/t/pre-rfc-field-projection/17383)
[^2] to easily point to fields from `self`.
I think that at some point we could change its type to `Pin<UninitPtr<$ret>>`,
but that does not exist yet (and nice ergonomics for a type like it also do
not). Functions that have a receiver type are also excluded for the time being,
but additional syntax could alleviate this (or one would write a helper function
with an explicit `this: &Self` or similar).

[^2]:
	I believe that such a feature will be much easier to implement than this, so we
	could rely on it, and if not users will just have to use `unsafe` in the meantime.

A bigger problem would be the integration with returning an `enum` and the
feature from [section 4][point-4-fallible-creation]. In this function:
```rust
impl MyStruct {
    #[pin_in_place]
    pub fn new() -> Result<Self, AllocError> {
        Ok(Self {
            data: Box::try_new(...)?,
            buf: [0; 1000],
            buf_ptr: &raw const self.buf,
            pin: PhantomPinned,
        })
    }
}
```
`self` should have type `*mut Self`, but with the current behavior it would have
`*mut Result<Self, AllocError>`, but this pointer would actually point to a
`union` with `Self` and `AllocError` as fields due to the `enum` optimization
from [section 4][point-4-fallible-creation].
I do not have a good idea of how to fix this, maybe
- always use `Self` if available (bad default?),
- let the user specify it with `#[pin_in_place($variant)]` where `$variant` is
  the variant that will be selected for choosing the type of `self`. But this is
  produces the problem of choosing a layout-equivalent (iirc we do not have that
  yet) type. We could require `#[repr(C)]` or `#[repr(Int)]` on such an enum,
  because [their layout is defined](https://rust-lang.github.io/rfcs/2195-really-tagged-unions.html).


## Smaller problem collection

### Unnecessarily strict "bad" example

From the guide level explanation there is this example labeled as "bad":
```rust
fn bad() -> Struct2 {
    let q = Struct { ... }
    Struct2 { member: q }
}
```
I believe that it stands in direct conflict with the section
[absolutely minimum viable copy elision](https://github.com/PoignardAzur/placement-by-return/blob/placement-by-return/text/0000-placement-by-return.md#absolutely-minimum-viable-copy-elision). It should be possible to use
`slot.member` as the memory location of `q`. Instead the following
pattern should be labeled as "bad":
```rust
fn bad(uncontrolled_param: bool) -> Struct2 {
    let mut p = Struct { ... };
    let mut q = Struct { ... };
    foo(&mut p, &mut q);
    Struct2 {
        member: if uncontrolled_param { p } else { q },
    }
}
```
Because the compiler would not be able to assign both `p` and `q` to the same
memory location.



## Conclusion

This was a bit of a random collection of ideas that I had to improve placement
by return, as I am working on making pinned initialization in the kernel free of
`unsafe`. Most of these ideas need more fleshing out before they can be added to
the RFC. I hope that we can achieve safe pinned initialization together with
this, because I think that these problems are connected.

If you are interested in learning more about kernel initialization, you can participate in the [zulip
discussion](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/safe.20initialization).
Or take a look at the [repository](https://github.com/Rust-for-Linux/linux).

[back](../)

*[RVO]: return value optimization
