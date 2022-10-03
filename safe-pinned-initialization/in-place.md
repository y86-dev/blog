# Language feature: in-place construction

{% raw %}

In this blog post, I am going to go over how in-place constructors could be added to Rust. Doing
this will enable better ergonomics for embedded and kernel-space programs. There are two distinct
problems that this feature will cater to:

- safe pinned initialization,
- ergonomic in place initialization of big structs.

I explained in my [overview] blog post why the first point is needed. The second point is also
wanted by Linux kernel developers, because they need exact control over where something gets
initialized. As the kernel has very limited stack size -- compared to userland programs -- and
thus cannot allocate big structs (multiple KB in size) on the stack.

# What are in-place constructors?

In the [in-place constructor] section of my overview blog post, I already introduced the concept of
in-place constructors. In-place constructors are a way to initialize data in-place. Instead of
returning a value, they write it to a caller provided location. They are very common in C and C++,
in C they are just plain functions with a pointer argument:
```c
struct Foo {
    struct Bar bar;
    int info;
};
int init_foo(struct Foo* foo) {
    if (init_bar(&foo->bar) != 0) {
        return -1;
    }
    foo->info = 42;
    return 0;
}
```
At the call site one just creates some location that can hold a `Foo` and calls `init_foo` on that:
```c
int main() {
    struct Foo* foo = (struct Foo*) malloc(sizeof(struct Foo));
    init_foo(foo);
    // use foo
}
```

C++ also has in-place constructors in the form of the placement new operator (I am not familiar with
C++, so I cannot elaborate).

These implementations of in-place constructors are not directly transferable to Rust. They are
`unsafe` in nature, because they require valid memory. We need to extend them in order to make them
safe.

# Designing an in-place constructors library for Rust

I will explain my design journey of my [`pinned-init`][pinned-init] library. Because there this is a
complex issue where mistakes are common, I cannot guarantee the soundness of the library as
presented here.

Afterwards I will try to find ways to create a language feature that achieves the same thing as my
library.

## An unsafe start

Let's create something `unsafe` at first and see how we can make it safe later! Because we are
handling a pointer to uninitialized data, we cannot use a reference `&mut Foo`. Instead we will use
a raw pointer `*mut Foo`. While we could use `&mut MaybeUninit<Foo>`, it actually has no benefit for
us at the moment (this would change when we have [field projection] for `MaybeUninit`). As we still need
to access the memory using raw pointers. And we will need to cast from and to `&mut MaybeUninit<T>`
all the time if we were to use it.

Now we have two options:
```rust
impl Foo {
    pub unsafe fn init(slot: *mut Foo) {
        todo!()
    }
}
```
```rust
impl Foo {
    pub unsafe fn new() -> impl FnOnce(*mut Foo) {
        |slot| todo!()
    }
}
```

The first option is a direct translation from C. We would use it almost identically:
```rust
let mut foo = Box::new_uninit();
unsafe { Foo::init(foo.as_mut_ptr()) };
let foo = unsafe { foo.assume_init() };
```
The other approach looks like this:
```rust
let mut foo = Box::new_uninit();
unsafe { Foo::new()(foo.as_mut_ptr()) };
let foo = unsafe { foo.assume_init() };
```

I chose the second option, because it allows us to write an abstraction that removes `unsafe`.

## Removing unsafe

`new` is marked as `unsafe` because the returned closure needs to be called correctly:

1. the pointer must be valid (dereferenceable, aligned) but may point to uninitialized memory
2. the caller should `assume_init` after using the closure.

This is the same safety contract as `init` with the only difference being that all of this must be
upheld when calling the closure.

So let's first move the unsafe to the closure, because then `new` can drop the `unsafe` keyword.
```rust
pub unsafe trait Init<T> {
    unsafe fn init(self, slot: *mut T);
}
```
Instead of returning `impl FnOnce(*mut T)`, we will return `impl Init<T>` (we will worry later about
the concrete type that we return). This now allows us to change `new` to be safe. The trait needs to
be `unsafe`, because we want to assume that the `init` function actually initializes the given slot.

This does not really change our call site:
```rust
let mut foo = Box::new_uninit();
let init = Foo::new(); // note that creation is now safe!
unsafe { init.init(foo.as_mut_ptr()) };
let foo = unsafe { foo.assume_init() };
```

But it now allows us to remove `unsafe` from it! How? By combining the action of calling `init`
with `assume_init`:
```rust
impl<T> Box<T> {
    pub fn new_in_place(init: impl Init<T>) -> Box<T> {
        let mut this = Box::new_uninit();
        // SAFETY: the pointer is valid and we `assume_init` below
        unsafe { init.init(this.as_mut_ptr()) };
        // SAFETY: we initialized the value via `init`
        unsafe { this.assume_init() }
    }
}
```
Our new call site is also a lot cleaner:
```rust
let foo = Box::new_in_place(Foo::new());
```

This way of abstracting unsafety away and consolidating it is one of the principles used to make code
safer. If there is a bug in the code, we only have to fix it once. It also let's us write more safe
code instead of having to call the initializer manually.

## Refining the solution

The solution mostly works, but we still need to iron out some details and support some situations:

1. fallible allocation
2. fallible initialization
3. pinned initialization
4. how do we create an `impl Init<T>`?

The first one is relatively easy:
```rust
impl<T> Box<T> {
    pub fn try_new_in_place(init: impl Init<T>) -> Result<Box<T>, AllocError> {
        let mut this = Box::try_new_uninit()?;
        // SAFETY: the pointer is valid and we `assume_init` below
        unsafe { init.init(this.as_mut_ptr()) };
        // SAFETY: we initialized the value via `init`
        Ok(unsafe { this.assume_init() })
    }
}
```

### 2. Fallible initialization

Here we need to change our `Init` trait:
```rust
pub unsafe trait Init<T, E> {
    unsafe fn init(slot: *mut T) -> Result<(), E>;
}
```

We also change the safety contract of the trait and the function:

Everything from before only applies if `Ok` is returned. When `Err` is returned, then the memory
must be deallocatable (this is important later when we visit pinning).

We also need to change the `Box` function:

```rust
pub enum AllocOrInitError<E> {
    Init(E),
    Alloc,
}

impl<E> From<AllocError> for AllocOrInitError<E> {
    fn from(e: AllocError) -> Self {
        Self::Alloc
    }
}

impl<T> Box<T> {
    pub fn try_new_in_place<E>(init: impl Init<T, E>) -> Result<Box<T>, AllocOrInitError<E>> {
        let mut this = Box::try_new_uninit()?;
        // SAFETY: the pointer is valid and we `assume_init` below
        unsafe { init.init(this.as_mut_ptr()).map_err(AllocOrInitError::Init)? };
        // SAFETY: we initialized the value via `init`
        Ok(unsafe { this.assume_init() })
    }
}
```

For types that have no error when they are created we can just use the [never type].

### 3. Pinned initialization

We need to add the guarantee that the data behind `slot` will not be moved. We can just add this to
the list of invariants on the `Init` trait. But because we do not want to force everyone to stay
pinned after initialization, we create a new trait instead:
```rust
pub unsafe trait PinInit<T, E> {
    unsafe fn pinned_init(slot: *mut T) -> Result<(), E>;
}
```
with the same invariants as `Init`, with the addition of `slot` is pinned on `pinned_init`.

We also need to add the following function to `Box`:
```rust
impl<T> Box<T> {
    pub fn try_pin_in_place<E>(init: impl PinInit<T, E>) -> Result<Pin<Box<T>>, AllocOrInitError<E>> {
        let mut this = Box::try_new_uninit()?;
        // SAFETY: the pointer is valid and pinned and we `assume_init` below
        unsafe { init.init(this.as_mut_ptr()).map_err(AllocOrInitError::Init)? };
        // SAFETY: we initialized the value via `init`
        Ok(Box::into_pin(unsafe { this.assume_init() }))
    }
}
```

Here the previous decision to make the memory deallocatable after returning `Err` helps us keep the
[`Pin` `Drop` guarantee].

We introduced a small inconvenience: we cannot call `try_pin_in_place` on `impl Init<T, E>`. But
this should be possible! The solution: make `PinInit` a supertrait of `Init`. We will add the
invariant that `Init::pinned_init` should just call `Init::init`.

### 4. Creating an impl Init

Essentially we want the user to be able to just write a closure that takes a `*mut T` and returns
a `Result<(), E>`. But that needs to fulfill the `Init` invariants. So we simply create a newtype
around a closure and make constructing the type `unsafe`:
```rust
pub struct InitClosure<F, T, E>(F, PhantomData<fn(T, E) -> (T, E)>);

impl<F: FnOnce(*mut T) -> Result<(), E>, T, E> InitClosure<F, T, E> {
    pub unsafe fn from_closure(f: F) -> Self {
        Self(f)
    }
}

impl<F: FnOnce(*mut T) -> Result<(), E>, T, E> Init<T, E> for InitClosure<F, T, E> {
    unsafe fn init(self, slot: *mut T) -> Result<(), E> {
        (self.0)(slot)
    }
}

impl<F: FnOnce(*mut T) -> Result<(), E>, T, E> PinInit<T, E> for InitClosure<F, T, E> {
    unsafe fn pinned_init(self, slot: *mut T) -> Result<(), E> {
        Self::init(self, slot)
    }
}
```

Similarly we also create `PinInitClosure` which only implements `PinInit`.

### Invariants
[invariants]: #invariants

Now might be a good time to list all of the invariants we enforce for `Init`:
- `Init::init` returns `Ok(())` if and only if it initialized every field of `slot`
- `Init::init` returns `Err(err)` if and only if it encountered an error. Callers can rely on the
  following guarantees:
  - `slot` is fully uninitialized, it can be freely repurposed (e.g. used by some other initializer,
    deallocated etc.)
  - in particular `slot` does not need to be dropped
- callers of `init` need to supply a valid pointer to uninitialized memory

The same invariants apply to `PinInit` with the addition of:
- callers of `PinInit::pinned_init` need to ensure that `slot` stays pinned
- if a type implements `Init` its `pinned_init` function needs to call `Init::init`, as it has given
  up on the pinning guarantee

### Making creation of constructors ergonomic

At the moment our init function for `Foo` looks like this:
```rust
struct Foo {
    bar: Bar,
    info: i32,
}
impl Foo {
    pub fn new() -> impl Init<Foo, BarError> {
        let init = |slot: *mut Foo| {
            unsafe { Bar::new().init(addr_of_mut!((*slot).bar))? };
            unsafe { addr_of_mut!((*slot).info).write(42) };
            Ok(())
        };
        unsafe { InitClosure::from_closure(init) }
    }
}
```
So we still need to use `unsafe`. Except, not really. We can write a macro that combines all of the
`unsafe` guarantees and requirements:
```rust
impl Foo {
    pub fn new() -> impl Init<Foo, BarError> {
        init! { Foo {
            bar: Bar::new(),
            info: 42,
        }}
    }
}
```
The syntax is identical to a struct initializer. It has no `unsafe` and it magically uses
`Bar::new()` as an initializer and `42` as a value to write. This macro needs to be carefully
crafted, because we want to guarantee soundness. Here are a couple of gotchas that we need to keep
in mind:

- after we initialize a field, we then need to drop it, if another field fails to initialize
  (otherwise we are violating the drop guarantee)
- initializing a field multiple times or not at all **must** be a compile error

So here is the macro:
```rust
macro_rules! init {
    ($t:ident $(<$($generics:ty),* $(,)?>)? {
        $($field:ident $(: $val:expr)?),*
        $(,)?
    }) => {{
        let init = move |slot: *mut $t $(<$($generics),*>)?| -> ::core::result::Result<(), _> {
            $(
                $(let $field = $val;)?
                // call the initializer
                // SAFETY: place is valid, because we are inside of an initializer closure, we return
                //         when an error/panic occurs.
                unsafe { $crate::Init::init($field, ::core::ptr::addr_of_mut!((*slot).$field))? };
                // create the drop guard
                // SAFETY: we forget the guard later when initialization has succeeded.
                let $field = unsafe { $crate::DropGuard::new(::core::ptr::addr_of_mut!((*slot).$field)) };
            )*
            #[allow(unreachable_code, clippy::diverging_sub_expression)]
            if false {
                let _: $t $(<$($generics),*>)? = $t {
                    $($field: ::core::todo!()),*
                };
            }
            $(
                // forget each guard
                ::core::mem::forget($field);
            )*
            Ok(())
        };
        let init: $crate::InitClosure<_, $t $(<$($generics),*>)?, _> = unsafe { $crate::InitClosure::from_closure(init) };
        init
    }}
}
```

1. Create the init closure. With the [`move`] keyword we ensure that any parameters used
   within the closure are moved into it (and cannot be used outside any longer).
2. Within that closure we iterate over every field and value pair
    1. evaluate the given value if present, if not, a variable with the same field name should be
      present. We bind the value to a variable with the name of the field, in order to support the
      short form syntax.
    2. we call `Init::init` on the produced value and the projected field.
    3. we use `?` to propagate any errors, so we exit the closure here on error/panic (everything
      that has been initialized up until now will be dropped)
    4. if init succeeded, we create a drop guard. When this guard gets dropped, it calls
      [`drop_in_place`] on the provided pointer.
3. After initializing all fields, we ensure that all fields have been mentioned exactly once. We
   "abuse" a struct initializer for this purpose: the `if false` body will only be type checked,
   never executed. For each field that was mentioned we add it to the initializer.
4. after initialization was successful we need to `forget` each guard, because they would free the
   just initialized object.
5. In the last line we add the workaround for [#99793]

The macro upholds all of our [invariants]. Similarly we also create a `pin_init!` macro.

There is a bit of magic going on in the `$crate::Init::init(...)` invocation. If you look at the
actual source of my library, you will see a different function call: `$crate::__private::__InitImpl::__init`.
This is some trait magic that enables the user to specify either an `impl Init<T>` or a direct value
`T` without needing extra syntax. I explain how this is implemented in [appendix A].

#### Small addition for self referential types

Because many pinned types are often self referential, we would like to support accessing `slot`
inside of the macro. We can just give access to the raw pointer, as doing anything with it requires
`unsafe`:

```rust
macro_rules! pin_init {
    ($(&$this:ident in)? $t:ident $(<$($generics:ty),* $(,)?>)? {
        $($field:ident $(: $val:expr)?),*
        $(,)?
    }) => {{
        let init = move |slot: *mut $t $(<$($generics),*>)?| -> ::core::result::Result<(), _> {
            $(let $this = unsafe { ::core::ptr::NonNull::new_unchecked(slot) };)?
            $(
                $(let $field = $val;)?
                // call the initializer
                // SAFETY: place is valid, because we are inside of an initializer closure, we return
                //         when an error/panic occurs.
                unsafe { $crate::PinInit::init_pinned($field, ::core::ptr::addr_of_mut!((*slot).$field))? };
                // create the drop guard
                // SAFETY: we forget the guard later when initialization has succeeded.
                let $field = unsafe { $crate::DropGuard::new(::core::ptr::addr_of_mut!((*slot).$field)) };
            )*
            #[allow(unreachable_code, clippy::diverging_sub_expression)]
            if false {
                let _: $t $(<$($generics),*>)? = $t {
                    $($field: ::core::todo!()),*
                };
            }
            $(
                ::core::mem::forget($field);
            )*
            Ok(())
        };
        let init: $crate::PinInitClosure<_, $t $(<$($generics),*>)?, _> = unsafe { $crate::PinInitClosure::from_closure(init) };
        init
    }}
}
```
So we allow prepending `&this in` before the initializer and in the closure body we create a
`NonNull` from `slot`.

## Fixing the unsoundness

Sadly there is a way to exploit the current library design that I completely overlooked. Luckily
Gary made me aware of this problem. Here is an example that exhibits UB:
```rust
pub struct ListHead {
    next: *mut ListHead,
    prev: *mut ListHead,
    pin: PhantomPinned,
}

impl ListHead {
    pub fn new() -> impl PinInit<Self> {
        pin_init!(&this in Self {
            next: this.as_ptr(),
            prev: this.as_ptr(),
            pin: PhantomPinned,
        })
    }
}

impl fmt::Debug for ListHead {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        let mut cur = self;
        write!(f, "{{ ")?;
        loop {
            write!(f, "{cur:p}, ")?;
            // SAFETY: we only construct ListHead pointing to itself
            cur = unsafe { &*cur.next };
            if ptr::eq(cur, self) {
                break;
            }
        }
        write!(f, "}}")
    }
}

#[pin_project::pin_project]
#[derive(Debug)]
pub struct Container<T> {
    t: T,
}

impl<T> Container<T> {
    pub fn new(init_t: impl PinInit<T>) -> impl PinInit<Self> {
        pin_init!(Self { t: init_t })
    }

    pub fn swap(self: Pin<&mut Self>, other: Pin<&mut Self>) {
        let this = self.project();
        let other = other.project();
        core::mem::swap(this.t, other.t);
    }
}

fn evil(mut c1: Pin<&mut Container<ListHead>>) -> Result<(), !> {
    stack_init!(let c2 = Container::new(ListHead::new()));
    let c2 = c2?;
    c1.as_mut().swap(c2);
    println!("{c1:?}");
    Ok(())
}

fn main() -> Result<(), !> {
    stack_init!(let c1 = Container::new(ListHead::new()));
    let mut c1 = c1?;
    evil(c1.as_mut())?;
    println!("{c1:?}");
    println!("{c1:?}");
    Ok(())
}
```
If we run this in debug mode we get:
```text
Container { t: { 0x7fffb00ee288, 0x7fffb00ee188, } }
Segmentation fault (core dumped)
```
The problem is that `ListHead` thinks it cannot be moved, because it can only be created via
`PinInit`, but in reality we can move it, because of `pin-project`.

The solution is to only allow initialization via `PinInit` when the field is actually pinned. So we
need to add a new macro `pin_data!` it is essentially doing the same thing that `pin-project-lite`
is doing. It allows us to specify pinned fields:
```rust
pin_data! {
    pub struct ListHead {
        next: *mut ListHead,
        prev: *mut ListHead,
        #pin
        pin: PhantomPinned,
    }
}
```
When we would now try to write the above program, we would get the following compile error:
```text
error[E0277]: the trait bound `impl PinInit<T>: pinned_init::Init<T, _>` is not satisfied
  --> src/main.rs:48:9
   |
48 |         pin_init!(Self { t: init_t })
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |         |
   |         the trait `pinned_init::Init<T, _>` is not implemented for `impl PinInit<T>`
   |         required by a bound introduced by this call
   |
   = help: the trait `pinned_init::Init<T, E>` is implemented for `InitClosure<F, T, E>`
   = note: required for `impl PinInit<T>` to implement `__InitImpl<T, _, Closure>`
note: required by a bound in `_::__ThePinData::<T>::t`
  --> src/main.rs:38:1
   |
38 | / pin_data! {
39 | |     #[pin_project::pin_project]
40 | |     #[derive(Debug)]
41 | |     pub struct Container<T> {
42 | |         t: T,
   | |         - required by a bound in this
43 | |     }
44 | | }
   | |_^ required by this bound in `_::__ThePinData::<T>::t`
   = note: this error originates in the macro `$crate::pin_init` which comes from the expansion of the macro `pin_data` (in Nightly builds, run with -Z macro-backtrace for more info)

For more information about this error, try `rustc --explain E0277`.
error: could not compile `tmp` due to previous error
```
So we are required to use an `impl Init` instead of the `impl PinInit`.

It is inconvenient having to write `pin_data!` for every struct. We will not need that once we have
[safe pin projections][field projection].

# How could and should a language solution look like?

We now have a library solution that fulfills almost all requirements that I set out in my [overview]
blog post. We could improve ergonomics and ecosystem integration with a language feature.
Additionally this way of initializing would become officially blessed by the language and would be
considered in future changes to the memory model as well as other changes.

The domain of a language solution also gives us access to new syntax and other goodies. For example
we could prevent the need for `Box::try_pin_in_place` and its cousins (see later).


## Syntax

### `<-` (reverse arrow)

First a bit of syntax. scottmcm on [zulip][zulip-rev-arrow] gave me the idea to use `<-`. We could
introduce a couple of new expressions:

```rust
let expr: Type = ...;
// `<- expr` evaluates to `impl Init`/`impl PinInit`, so binding it to a variable does nothing:
let init: impl Init<Type, !> = <- expr;
// to safely call an initializer we can initialize a variable on the stack. this is only
// possible if Error = !
let val: Type <- init;
// we can combine this with if let:
let init: impl Init<Type, Err> = ...;
if let val <- init {
    // can access val
} else {
    // it failed
}
// and with let else
let val <- init else { panic!() };
// we should also provide a way to handle the error:
let val <-? init;
// maybe there should also exist match let?
match let val <- init {
    Ok(()) => {
        // can use val here
    }
    Err(err) => {
        panic!("got error: {err:?}");
    }
}
// or without the let?
match val <- init {
    Ok(val) => {
        // val is of type `Pin<&mut Type>`
    }
    Err(err) => {
        panic!("got error: {err:?}");
    }
}

impl<T> Box<T> {
    pub fn try_new(init: impl Init<T, E>) -> Result<Box<T>, AllocOrInitError<E>> {
        let mut loc = Box::try_new_uninit()?;
        // we can call `init` on `impl Init`, but that function is `unsafe`, because we still have
        // to uphold the invariants that the library also needs to uphold.
        unsafe { init.init(&mut *loc)? };
        Ok(loc.assume_init())
    }
}

// `<- $StructLiteral` also evaluates to `impl Init`/`impl PinInit`
let init: impl Init<Struct, !> = <- Struct {
    a: Foo { x: [0; 1024 * 1024] },
    b: 42,
};
// the struct literal initializes each value in place, so it desugars to the following closure:
|slot| unsafe {
    addr_of_mut!((*slot).a.x).write_bytes(0, 1);
    addr_of_mut!((*slot).b).write(42);
};

// when inside such a struct literal we also support initializers:
let init: impl Init<Struct, !> = <- Struct {
    a: Foo::new(),
    b: 42,
};
// it will desugar to
|slot| unsafe {
    Foo::new().init(addr_of_mut!((*slot).a))?;
    addr_of_mut!((*slot).b).write(42);
}

// in an initializer that evaluates to impl PinInit we also have access to `self`, its type is some
// pointer type, either `*mut Self`, or `&mut MaybeUninit<Self>` or something else
let init: impl PinInit<Struct, !> = <- Struct {
    a: Foo::new(),
    b: self.addr(),
};

// using `<-` on PinInit only results in Pin<&mut Struct>, as accessing the backing storage is unsound
let value: Pin<&mut Struct> <- init;

// We also need field pin projection from RFC-3318 for this. One is only allowed to use an
// impl PinInit on fields that are actually pinned:
struct Struct {
    #[pin]
    a: Foo,
    b: usize,
}
struct Foo {
    x: [u8; 1024 * 1024],
}
impl Foo {
    fn new() -> impl PinInit<Self, !> {
        todo!()
    }
}
let init: impl PinInit<Struct, !> = <- Struct {
    // only allowed, because a is marked with `#[pin]`
    a: Foo::new(),
    b: self.addr(),
};

```
Using all of that would let our example look like this:
```rust
struct Foo {
    bar: Bar,
    info: i32,
}

impl Foo {
    pub fn new() -> impl Init<Foo, BarError> {
        <- Foo {
            bar: Bar::new(),
            info: 42,
        }
    }
}

fn main() {
    // init on the heap still looks like this
    let foo = Box::init(Foo::new());
    // on the stack we could add extra syntax:
    let foo <- Foo::new();
}
```
We would also allow `self` in such an initializer:
```rust
pub struct ListHead {
    // using *const for convenience
    next: *const Self,
    prev: *const Self,
    pin: PhantomPinned,
}

impl ListHead {
    pub fn new() -> impl PinInit<Self> {
        <- Self {
            next: self,
            prev: self,
            pin: PhantomPinned,
        }
    }
}

let list_head <- ListHead::new();
```

We could also make the following decision:
```rust
impl Foo {
    pub fn new() -> impl Init<Foo, BarError> {
        <- Foo {
            bar <- Bar::new(),
            info: 42,
        }
    }
}
```
So to use an initializer inside of a struct literal one needs to use `<-`. This would be more
explicit and could help with inference, but it also is new syntax...

How would we differentiate between creating `PinInit` and `Init`?

#### 1. Default to Init and use PinInit only if necessary

This could just be normal type inference. When `self` is used, we automatically choose `PinInit`.
Defaulting to `Init` could be a potential footgun though:
```rust
#[derive(Debug)]
pub struct ListHead {
    val: u8,
    ptr: *const u8,
    pin: PhantomPinned,
}
// we forget that we can use `self`
let mut list_head <- ListHead { val: 42, ptr: ptr::null(), pin: PhantomPinned };
list_head.ptr = &list_head.ptr;
let mut other <- ListHead { val: 42, ptr: ptr::null(), pin: PhantomPinned };
other.ptr = &other.ptr;
mem::swap(&mut list_head, &mut other);
drop(other);
println!("{:?}", unsafe { &*list_head.ptr }); // ptr is now dangling
```

#### 2. Default to PinInit and require explicit type annotation for Init

This avoids the afore mentioned footgun but is mildly annoying when one actually wants an `Init`
directly on the stack:
```rust
let list_head = (<- ListHead { val: 42, ptr: ptr::null(), pin: PhantomPinned }) as impl Init<ListHead>;
let list_head <- list_head;
```

#### 3. Add an extra keyword

The user explicitly states that they want the pinned version via `pin <- $expr` or `<- pin $expr`.

---

I like option 2 the most and would *really* like to avoid option 3.

### A new keyword

Instead of `<-` we could introduce a keyword. I have not thought about how to
transfer `let a <- b;` to the keyword idea, but I wanted to at least mention it.

I have grown rather fond of the `<-` syntax...

### Nothing?

This might be a bit radical, but I think we could add only a single language magic thing: coerce a
struct initializer to `impl Init`/`impl PinInit`. Calling it will be `unsafe`, but we would not need
to change the syntax.

Not sure if we want this, maybe as an experiment at the beginning, but in the long term, I think we
want to support using initializers safely.

## General design

I think in general most of the library design can be carried over to the language solution. We
should probably make `Init` and `PinInit` not implementable traits. They should have a blanket impl,
so we can still use `Box::try_new(data)` with direct data. And of course `<- $expr` would need to
implement `PinInit`/`Init`.

With this trait based approach we force all initialization functions to be generic. There might be
some situations where this is a problem when using `dyn` trait objects.

### Not general enough return type

Another problem that I am concerned with is this: what if the initializer can actually return
more than two states? So something between success and fail:
```rust
enum InitResult {
    Success,
    PartialSuccess,
    Fail,
}

struct Thing {
    buf: Option<Box<Big>>,
    some_other_data: Foo,
}

fn init(slot: *mut Thing) -> InitResult {
    match Foo::new().init(addr_of_mut!((*slot).some_other_data)) {
        Ok(()) => {}
        Err(e) => return InitResult::Fail,
    }
    match Box::try_new_in_place(Big::new()) {
        Ok(buf) => {
            addr_of_mut!((*slot).buf).write(Some(buf));
            InitResult::Success
        }
        Err(_) => {
            addr_of_mut!((*slot).buf).write(None);
            InitResult::PartialSuccess
        }
    }
}
```
There probably exists a workaround. In this example it also is not really a problem, because one can
just check if `buf` is inhabited or not, but in general this might not be possible.


### Errors

The macro currently has some rather unhelpful errors:
```text
error[E0308]: mismatched types
  --> src/main.rs:20:9
   |
20 |         init!(Self { bar: Bar::new() })
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |         |
   |         expected opaque type, found struct `Bar`
   |         arguments to this function are incorrect
...
25 |     pub fn new() -> impl PinInit<Self> {
   |                     ------------------ the expected opaque type
   |
   = note: expected raw pointer `*mut impl pinned_init::PinInit<Bar>`
              found raw pointer `*mut Bar`
note: associated function defined here
  --> pinned-init-0.0.2/src/__private.rs:24:15
   |
24 |     unsafe fn __init(self, slot: *mut T) -> Result<(), E>;
   |               ^^^^^^
   = note: this error originates in the macro `init` (in Nightly builds, run with -Z macro-backtrace for more info)

For more information about this error, try `rustc --explain E0308`.
error: could not compile `tmp` due to previous error
```
Here is the code:
```rust
pin_data! {pub struct Foo {
    #pin
    bar: Bar,
}}

pin_data! {pub struct Bar {
    #pin
    pin: PhantomPinned,
}}

impl Foo {
    pub fn new() -> impl Init<Self> {
        init!(Self { bar: Bar::new() })
    }
}

impl Bar {
    pub fn new() -> impl PinInit<Self> {
        pin_init!(Self { pin: PhantomPinned })
    }
}
```
The error occurs because `Bar::new` returns `PinInit`, but we actually try to use it as if it were a
`Init`.

So error handling would be improved by a lot when using a language solution (because we would not
need to implement the trait magic that I have).

# Conclusion

In this blog post I described the design rationale behind my library [`pinned-init`][pinned-init] in
detail. I also presented some syntax that could make this library the basis for a language feature.

The requirements for a solutions are still the same when I wrote my [overview] post:

1. Pinning guarantees (ensures that the pointee stays pinned if it is `!Unpin`)
2. Safety
3. Fallible initialization,
4. Performance (performs *almost* as well as the `unsafe` version would)
5. Ergonomics,
6. Ecosystem migration ergonomics (has low friction of migration, both users and library authors
  should only have to apply minimal changes to the call/definition sites)

The language solution presented here would provide excellent ergonomics and ecosystem migration.
While it also excels at the other points.

Overall in-place constructors also seem like a good fit for Rust. They are explicit and
flexible enough to solve the problems currently present. In-place constructors do not have the
issues that `&uninit` and other new pointer types have. They similarly avoid the complexity that is
required to make typestates work.

I will continue to improve my library and will write an RFC based on it soon.

# Prior art

There are quiet a few earlier proposals on how to handle initialization:

- [`&out T` and `&uninit T`][rusty-yato] and its subsequent [RFC][rfc-uninit]
- [my typestate proposal][typestate]
- [`&out`][repax-out] by repax
- [`&out` `&in` `&uninit`][canndrew] by canndrew
- [`&move`][tema2] by tema2
- [partially initialized types][soni] by Soni


# Appendix A
[appendix A]: #appendix-a

We want to simultaneously allow supplying an `impl Init<T>` or a bare `T` to initialize a field. I
achieved this by creating a helper trait:
```rust
pub unsafe trait __InitImpl<T, E, W: InitWay>: __PinInitImpl<T, E, W> {
    unsafe fn __init(self, slot: *mut T) -> Result<(), E>;
}
```
The extra type parameter `W` is used to differentiate between a bare `T` and an initializer.
`InitWay` is a sealed trait that is only implemented for two types `Direct` and `Closure`.
We then `impl<T> __PinInitImpl<T, !, Direct> for T` and `impl<I, T, E> __InitImpl<T, E, Closure> for I where I: Init<T, E>`.
Then we rely on type inference to always resolve to the correct function (because it is rather
uncommon to value initialize an initializer, this works very well).

We of course need to mirror the same thing for `PinInit`.

[back](../)

[overview]: ../overview.html
[in-place constructor]: ../overview.html#in-place-constructor
[never type]: https://doc.rust-lang.org/reference/types/never.html
[`Pin` `Drop` guarantee]: https://doc.rust-lang.org/std/pin/index.html#drop-guarantee
[pinned-init]: https://crates.io/crates/pinned-init
[`move`]: https://doc.rust-lang.org/std/keyword.move.html
[`drop_in_place`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.drop_in_place
[#99793]: https://github.com/rust-lang/rust/issues/99793
[zulip-rev-arrow]: https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/safe.20initialization/near/298893680
[field projection]: https://github.com/rust-lang/rfcs/pull/3318
[rusty-yato]: https://internals.rust-lang.org/t/pre-rfc-partial-initialization-and-write-pointers/8310
[rfc-uninit]: https://github.com/rust-lang/rfcs/pull/2534
[typestate]: https://internals.rust-lang.org/t/safe-partial-initialization/16443
[repax-out]: https://internals.rust-lang.org/t/out-argument-for-efficient-initialisation/7254
[canndrew]: https://internals.rust-lang.org/t/in-out-uninit-references-is-it-time-yet/6792
[tema2]: https://internals.rust-lang.org/t/pre-rfc-move-references/14511
[soni]: https://internals.rust-lang.org/t/pre-rfc-partially-initialized-types/8402

{% endraw %}
