# Overview: Safe Pinned Initialization

In this blog post I will try to explain the problem of Safe Pinned Initialization. I am going to tackle the
issue from two perspectives:

- Rust language design: how can we make Safe Pinned Initialization ergonomic for as many people as possible?
- Linux kernel design: how can we satisfy the kernel's needs with Rust?

In order to aid the second perspective, I will try to cater specifically to C-only kernel
developers. While this document is written with C developers in mind, it does not explain the basic
concepts of Rust. So if you are not at all familiar with Rust, I would recommend reading at least
chapters 3 and 4 of the [Rust book].

I want to remove `unsafe` from the `Mutex<T>` initializing API [in the Linux kernel][kernel-mutex].
From a Rust perspective, this API design is *terrible*. Sadly, it *is* warranted and results from the poor
[`Pin<P>`] ergonomics present in the language.

First I am going to explain the origin of the unsafety. After that I will present the solutions that
exist to get rid of it.

# What does [`Pin<P>`] have to do with `Mutex<T>`?

## What is [`Pin<P>`]?

There are some [great resources on pinning][pin-and-suffering]. It is a rather
long article and I would only suggest it to you if you already know a little bit about futures.
In the next section I will explain how the Linux mutex works and also why it needs to be pinned. For
a C programmer it might be enough to just read this part to understand what pinning achieves here.

## `mutex` in the kernel

The `Mutex<T>` from the [rust binding of the kernel][kernel-mutex] is a wrapper of the
[`mutex`][mutex-docs] struct from the kernel. Here is its definition[^1]:
```c
struct mutex {
    atomic_long_t       owner;
    raw_spinlock_t      wait_lock;
    struct list_head    wait_list;
};
```

The spinlock protects the `wait_list`, so we can only access it while holding the
lock. Locking roughly works like this[^2]:

check if `owner` is `null` (this is atomic, so no data races can occur)
- yes => set `owner`. The mutex is now locked and owned by us.
- no => busy wait and see if the owner releases the lock soon. If not, then acquire the spinlock and insert a new entry
       into the `wait_list` and then go to sleep (of course release the spinlock before).

The crux of the problem lies in the type of list used as the `wait_list`. The list is

- intrusive (you have to embed it in order to add an element to the structure),
- circular (there is no "first"/"last" element, just a list),
- doubly linked (each element has both prev and next elements).

It is very common in the kernel, but Rust has its share of difficulties with it. 
Using such a list has many benefits for the kernel:

- adding and removing elements can be done without branches (i.e. no pipeline stalls),
- the individual elements can live wherever and do not need to be allocated in advance (they can
  even live on the stack frame of a function!).

There are probably more advantages, but for this post these are the most important ones. The Rust binding
*could* use a different, more Rust compatible list (implementing a doubly linked list in Rust is a *pain*).
But that would not be a great solution, as now a majority of Linux structures could not be used in
Rust.

This list is a problem for Rust, because it is self-referential and that is poorly compatible with
the ownership model in Rust.

### What role does [`Pin<P>`] play?

Let's look at the following example that produces a segfault:
```c
#include<stdio.h>

struct list_head {
    struct list_head* next;
    struct list_head* prev;
};

void init_list_head(struct list_head* head) {
    // a list with one element points to itself
    head->next = head;
    head->prev = head;
}

void add_list(struct list_head* head, struct list_head* element) {
    // we want to insert element as the next element of head.
    // for that the current next elment needs to become
    // the successor of element.
    element->next = head->next;
    head->next->prev = element;
    head->next = element;
    element->prev = head;
}

void debug_iter_list(struct list_head* head) {
    struct list_head* start = head;
    do {
        printf("%p\n", head);
        head = head->next;
    } while(head != start);
}

struct list_head add_element(struct list_head* head) {
    struct list_head element;
    init_list_head(&element);
    add_list(head, &element);
    return element;
}

int main() {
    struct list_head head;
    init_list_head(&head);
    struct list_head element = add_element(&head);
    printf("iterating list...\n");
    debug_iter_list(&head);
}
```
When executing it segfaults:
```text
$ ./a.out
iterating list...
0x7ffd841be180
0x7ffd841be160
0x7ffd841be2a8
0x7ffd841bf184
0x74756f2e612f2e
Segmentation fault (core dumped)
```
Most C developers will immediately see the problem with this snippet. The `add_element` function
allocates a `list_head` on the stack and then inserts it into the list. The compiler then moves
the element from the stack frame of `add_element` into the stack frame of `main`. But the
pointers in `head` have not been updated. It still points into the former stack frame of
`add_element`. That memory is repurposed by `printf` and now contains garbage.

So in order for this type of list to work, each element needs to have a *stable address* for the
duration it stays in the list. It would be completely fine to remove the element at the end of
`add_element`.

Because this problem is a memory error, Rust will need to *somehow* ensure
the invariant at compile time (after all that is what Rust does). The solution is [`Pin<P>`]
wrapping a pointer `P`. For certain types, (types that are [`!Unpin`]) it prevents users from
moving the pointee. This is achieved through not giving access to `&mut T` (`unsafe` code
can still do what it wants, but we will have to ensure the invariant manually!).

In order for our API to work in Rust, each function that would normally take a `&mut ListHead` now
needs to use `Pin<&mut ListHead>` instead. At first glance, this is not a problem. But how do we
access a mutex from an outer struct? Here is an artificial example:
```rust
struct DriverData {
    command_queue: Mutex<Queue<Command>>,
    // ...
}
impl DriverData {
    /// # Safety
    /// caller has to call `init`
    unsafe fn new_driver_data() -> Self {
        Self {
            // SAFETY: caller will call init()
            command_queue: unsafe { Mutex::new(Queue::new()) },
        }
    }

    fn init(self: Pin<&mut Self>) {
        Mutex::init(&mut self.mutex);
        //          ^^^^^^^^^^^^^^^ error: cannot borrow as mutable
    }
}
```
Because `Pin` prohibits access to `&mut Self`, we cannot get access to the `command_queue`.
But how do we access `Pin<&mut Mutex>` from `Pin<&mut Self>`? This is called a pin-projection.
Fundamentally, it is a tricky operation, because we have to ensure that only `Pin<&mut Mutex>` is
ever produced (otherwise we would be violating the pin invariant). Currently this operation is
`unsafe`.

The [`pin-project`] crate provides a proc-macro that allows safe pin-projections, but it requires
the [`syn`] crate as a dependency. That is not an option for the kernel at the moment[^6]. There is an
alternative without proc macros: [`pin-project-lite`], but it is implemented as a *very* convoluted
macro. So the kernel has been using `unsafe` for projections.

I am currently working on a solution to the problem of projection. You can find the
[RFC here][fp-rfc]. I am relatively confident that we can solve the projection issue soon (at least
faster than the issue presented here).

### So everything is fine?

Well this section would not exist if it was... How do we initialize a self-referential struct like
`list_head`? In Rust we would normally write an initializer like this:
```rust
impl<T> Mutex<T> {
    fn new(data: T) -> Self {
        Self {
            data,
            // ...
        }
    }
}
```
The caller will move the returned value to a suitable location. This could for example be the heap
via `Box::new(Mutex::new())` [^3].

When handling `list_head`, we would like to set `next` and `prev` to the *address* of `self`.
However, that address is only known after the construction of our value...

The current solution in the kernel binding is to have two functions:
```rust
impl<T> Mutex<T> {
    /// # Safety
    /// the caller is required to call `init`
    unsafe fn new() -> Self;
    
    fn init(self: Pin<&mut Self>);
}
```
In `new` the `list_head` is not actually initialized. The field is stored in `MaybeUninit` so it is
left uninitialized until `init` is called. `init` then can then set `next` and `prev`, because it
knows the location is not going to change. `unsafe` is used to make sure `init` is called before the
mutex is used. If it were used without calling `init`, the mutex would be accessing uninitialized
memory.

It is this `unsafe` that I want to get rid off. As a consequence, the API will only provide a single
function to create and initialize a mutex, so it will also become more ergonomic in the process.

# What are the requirements for a solution?

1. Pinning guarantees (ensures that the pointee stays pinned if it is [`!Unpin`])
2. Safety
3. Fallible initialization,
4. Performance (performs *almost* [^4] as well as the [`unsafe` version][unsafe-solution] would)
5. Ergonomics,
6. Ecosystem migration ergonomics (has low friction of migration, both users and library authors
  should only have to apply minimal changes to the call/definition sites)

The two perspectives (Rust design and kernel design) have different priorities. Rust design would
order the requirements like this:

1. Pinning guarantees
2. Safety
3. Ecosystem migration ergonomics, Performance, Ergonomics
4. Fallible initialization

Kernel design has different priorities:

1. Pinning guarantees
2. Fallible initialization
3. Performance
4. Ecosystem migration ergonomics, Ergonomics, Safety

In an ideal world we would be able to have a solution that fulfills every requirement. That still
*might* be possible, but also difficult to find. The total order I am going to use to rate the
solutions is going to be:

1. Pinning guarantees
2. Safety
3. Fallible initialization
4. Performance
5. Ergonomics
6. Ecosystem migration ergonomics

Lets go over each point and discuss why we need it.

## 1. Pinning guarantees

This is quintessential for any self-referential struct to work.

## 2. Safety

This essentially boils down to why we want Rust in system programming in the first place:

- marking code regions with `unsafe` makes reviewing for memory safety a *lot* easier,
- better entry point for new developers. They can use the safe abstractions and do not have to touch
  dangerous functions,
- consolidating `unsafe` code results in easier to maintain code, less code results in less bugs.

## 3. Fallible initialization

In the kernel, memory allocations can fail. When this happens in a userspace program, then the system
is indeed in a dire situation. But in the kernel the very code we are writing is in charge of the
physical memory. We could for example move some pages to swap space if we run out of memory. So
graceful handling of errors is of utmost importance.

## 4. Performance

The kernel is -- like many system programs -- performance critical. One solution is to wrap every `Mutex`
inside a `Box`. This would introduce a heap allocation for **every single mutex**, which is
unacceptable for the kernel.

While it is probably unrealistic to expect machinecode parity with the [`unsafe` solution][unsafe-solution],
optimizing this is still a high priority.

## 5. Ergonomics

Users should not need to adhere to unnecessarily convoluted syntax and there should only be one
function call/statement that they need to write in order to initialize something pinned.

Simpler code is easier to read, write and thus maintain.

## 6. Ecosystem migration ergonomics

Making it easy for existing codebases to migrate to the newest feature is an important
consideration. If we are going to add a new feature then we should make it as accessible as
possible.

Ideally nobody would have to change anything. But because Rust needs to be transparent, this is
impossible. The bare minimum is going to be changing the definition site of initializers and smart
pointer initializers. But I am not sure if that is enough to fulfill the other requirements.

# Solutions

## `unsafe` solution
[unsafe-solution]: #unsafe-solution

I already introduced it earlier, but here is the complete code to initialize a mutex in the Rust
binding of the Linux kernel[^5]:

```rust
pub struct Mutex<T: ?Sized> {
    /// The kernel `struct mutex` object.
    mutex: Opaque<bindings::mutex>,

    /// A mutex needs to be pinned because it contains a [`struct list_head`] that is
    /// self-referential, so it cannot be safely moved once it is initialised.
    _pin: PhantomPinned,

    /// The data protected by the mutex.
    data: UnsafeCell<T>,
}

impl<T> Mutex<T> {
    /// Constructs a new mutex.
    ///
    /// # Safety
    ///
    /// The caller must call [`Mutex::init_lock`] before using the mutex.
    pub const unsafe fn new(t: T) -> Self {
        Self {
            mutex: Opaque::uninit(),
            data: UnsafeCell::new(t),
            _pin: PhantomPinned,
        }
    }

    pub fn init(self: Pin<&mut Self>, name: &'static CStr, key: &'static LockClassKey) {
        unsafe { bindings::__mutex_init(self.mutex.get(), name.as_char_ptr(), key.get()) };
    }
}
```

The `name` and `key` parameters in the `init` function are from [lockdep] and not needed for the
functionality of the mutex. `Opaque<bindings::mutex>` essentially is `MaybeUninit<bindings::mutex>`
(it does some more things that are not important for this post).

Now if we want to embed a `Mutex<T>` in a custom struct, all we need to do is:
```rust
struct DriverData {
    command_queue: Mutex<Queue<Command>>,
    // we also have a big buffer that might fail to allocate
    buffer: Box<[u8; 1000_000]>,
}
impl DriverData {
    /// # Safety
    /// caller has to call init()
    unsafe fn new() -> Result<Self, AllocError> {
        Ok(Self {
            // SAFETY caller will call init()
            command_queue: unsafe { Mutex::new(Queue::new()) },
            buffer: Box::try_new([0; 1000_000])?,
        })
    }

    fn init(self: Pin<&mut Self>) {
        // SAFETY: we never move out of the closure variable, we need this to do projections
        let command_queue = unsafe { Pin::map_unchecked_mut(self, |this| &mut this.command_queue) };
        // this macro creates the lockclasskey and initializes the mutex
        mutex_init!(command_queue, "DriverData::command_queue");
    }
}
```

Let's see how many criteria are met:

1. [x] Pinning guarantees
2. [ ] Safety
3. [x] Fallible initialization
4. [x] Performance
5. [ ] Ergonomics
6. [ ] Ecosystem migration ergonomics

Point 1 is only checked because we heavily rely on `unsafe` though...

A big problem is that we can still forget to initialize the mutex.

All in all not a great solution. However, it checks all of the boxes a solution requires to be used
in the kernel (which is why it is currently being used).



## `&uninit T`

There already has been an [RFC][rustyyato-rfc] trying to introduce this concept.
The proposal has some drawbacks and it was postponed because it wanted to introduce a new reference
type. That is something the lang team is very careful with.
There exists an even [older RFC][old-ptr-rfc] that tried to create something similar.

I do not think a new proposal will go any different. But regardless, lets apply our
checklist:

1. [ ] Pinning guarantees
2. [x] Safety
3. [ ] Fallible initialization
4. [x] Performance
5. [x] Ergonomics
6. [ ] Ecosystem migration ergonomics

The biggest problems are the inability to handle fallible initialization and failure to uphold the
pinning guarantees. It would need to be *somehow* extended to account for that. There is no obvious
solution and I view the alternatives as more promising, so I think we can rule this approach out as
impractical.

Because of these limitations and the lack of specification of this approach, I will not try to
recreate the example.

## `placement by return`

I have already written a [post][placement-by-return] about this. In that post I go over how the [RFC
for placement by return][pbr-rfc] would need to be modified in order to support our use case.

My view has changed a bit since writing that post. One of the bigger blockers are spurious
copies inserted for correctness. This was brought up in the [zulip discussion][pbr-discussion]
by bjorn. There are a lot of strings attached to solving that, so I think that placement by return
is not the best way forward.

Here is the example from before with all of my extensions from the blog post:
```rust
struct DriverData {
    command_queue: Mutex<Queue<Command>>,
    // we also have a big buffer that might fail to allocate
    buffer: Box<[u8; 1000_000]>,
}

impl DriverData {
    #[pin_in_place]
    fn new() -> Result<Self, AllocError> {
        Ok(Self {
            command_queue: Mutex::new(Queue::new()),
            buffer: Box::try_new([0; 1000_000])?,
        })
    }
}
```

Ergonomically, this is really nice. But it relies *a lot* on compiler magic doing the correct thing.
Nonetheless lets go over our checklist:

1. [x] Pinning guarantees
2. [x] Safety
3. [x] Fallible initialization
4. [x] Performance
5. [x] Ergonomics
6. [x] Ecosystem migration ergonomics

well isn't that great! However there are a couple of asterisks attached to those boxes. Here is the
version without all of the *definitely not controversial/incomplete* changes that I proposed in my
blog post:

1. [ ] Pinning guarantees
2. [x] Safety
3. [ ] Fallible initialization
4. [x] Performance
5. [ ] Ergonomics
6. [ ] Ecosystem migration ergonomics

Without those it doesn't work. These extensions would also make the RFC a *lot* more complex
than it already is. Altogether I do not think that we can merge these two features into one.


## In-place constructor

[Here][mcyoung-ctors] is a blog post going over interoperability with C++'s move constructors.
This pattern can already be turned into a succinct library that hides all of the `unsafe` stuff
behind a declarative macro. I have created an implementation [here][simple-safe-init] and an
integration into the [kernel here][ssi-kernel].

The design is as follows:

```rust
struct DriverData {
    command_queue: Mutex<Queue<Command>>,
    // we also have a big buffer that might fail to allocate
    buffer: Box<[u8; 1000_000]>,
}
impl DriverData {
    fn new() -> impl PinInitializer<Self, AllocError> {
        pin_init!(Self {
            command_queue: Mutex::new(Queue::new()),
            buffer: Box::try_new([0; 1000_000])?,
        })
    }
}
```
No `unsafe` needed!

The mutex is a bit more complicated, but it also has to dip its toes into ffi:
```rust
impl<T> Mutex<T>
    pub const fn new(
        t: T,
        name: &'static CStr,
        key: &'static LockClassKey,
    ) -> impl PinInitializer<Self, !> {
        fn init_mutex(
            name: &'static CStr,
            key: &'static LockClassKey,
        ) -> impl PinInitializer<Opaque<bindings::mutex>, !> {
            let init = move |place: *mut Opaque<bindings::mutex>| {
                unsafe {
                    bindings::__mutex_init(Opaque::raw_get(place), name.as_char_ptr(), key.get())
                };
                Ok(())
            };
            // SAFETY: mutex has been initialized
            unsafe { PinInit::from_closure(init) }
        }
        pin_init!(Self {
            mutex: init_mutex(name, key),
            data: UnsafeCell::new(t),
            _pin: PhantomPinned,
        })
    }
}
```

To create a `Pin<Box<DriverData>>` we have to call a new function on `Box`:
`Box::pin_init(DriverData::new())`.

Here is the checklist:

1. [x] Pinning guarantees
2. [x] Safety
3. [x] Fallible initialization
4. [x] Performance
5. [x] Ergonomics
6. [ ] Ecosystem migration ergonomics

Additionally, this library solution is the smallest I have been able to come up yet (I had two prior
solutions with a *lot* more complexity. Like 2000+ lines of complexity). All of the code fits into
less than 400 lines!

The drawbacks of this solution are:

- users have to call a function different from `Box::new` to put the value inside of the `Box`,
- in the initializer it is not clear which fields are initialized in-place and which are initialized
  via another initializer,
- should I choose `pin_init!` or `init!`?

I addressed the last two problems in a [new variation][simple-safe-init-experiment]. The first
problem is inherent to this solution and it will probably not be fixable, unless we want to extend
`Box::new`[^3].

Lang support *could* lead to better ergonomics, but I am not sure if we want to lose the
transparency. In practice it could be done by changing the function
signatures to:
```rust
impl<T> Box<T> {
    fn new(data: k#init T) -> Self;
    fn new_in(data: k#init T) -> Self;
    fn pin(data: k#init T) -> Self;
    fn try_new(data: k#init T) -> Result<Self, AllocError>;
    // etc.
}
```

And then one could continue to use `Box::pin(Mutex::new())`[^3].
However, user initializers would still have to be rewritten (just not the call sites).

### Additional usage

These in-place constructors could also be used in other places. They would also permit the
efficient initialization of large structures/arrays, because there would be no construction on the
stack.

The issue with that would be, most initializers are currently written in the return-and-then-copy
fashion. They would need to be rewritten and Rust's standard way of initialization would become
in-place. I am not sure if we want to start such a large migration without exploring every
alternative at hand.

 With a bit of work, unsized values could also be supported. It could work similarly to how the
 current [placement by return RFC][pbr-rfc] handles them: the initializer is a state machine that

1. computes the layout based on the given parameters,
2. lets the caller allocate some memory,
3. populate the memory location as before.

I have not implemented that in the macro yet. So I do not know of any hurdles that might be present.

# Conclusion

I am not aware of any other solutions and would love to hear something new on this issue. At
the moment, I think that in-place constructors seem like the best way forward. With a small
lang addition (keyword/attribute/magic type) parameters and return types could be marked as an
initializer for the given value. There are still some more details that need to be nailed down,
(going to write a pre-rfc soon) but in principal I think this could work.

With the macro I have written this can already be tried with a bit more ergonomic overhead (I plan
to release it as a crate, but I am waiting for a little bit more feedback).

If you have feedback/questions feel free to post them in the [internals thread].




[^1]:
    I have stripped debug information and other pre-processor information, see the original code
    [here](https://elixir.bootlin.com/linux/v5.19.9/source/include/linux/mutex.h#L63).

[^2]: I have simplified the process in order to aid understanding. Visit the [documentation][mutex-docs] for details.

[^3]:
    When I am writing `Box::new` I am also referring to all other `Box` creation functions,
    especially those returning a `Result` instead of panicking.

[^4]: In some contexts it is fine to have worse performance, if we can guarantee correctness.

[^5]: Again, some part have been omitted. For the exact details review [the code here](https://github.com/Rust-for-Linux/linux/blob/459035ab65c0ebb8d7054b24b6c00de907819eb2/rust/kernel/sync/mutex.rs).

[^6]: The `syn` crate has about 55k+ loc. The rust binding for the Linux kernel has about 22k+ loc.


[back](../)

[placement-by-return]: ../return-value-optimization/placement-by-return.md
[`Pin<P>`]: https://doc.rust-lang.org/core/pin/struct.Pin.html
[`Unpin`]: https://doc.rust-lang.org/core/marker/trait.Unpin.html
[`!Unpin`]: https://doc.rust-lang.org/core/marker/trait.Unpin.html
[`pin-project`]: https://crates.io/crates/pin-project
[`pin-project-lite`]: https://crates.io/crates/pin-project-lite
[lockdep]: https://www.kernel.org/doc/html/latest/locking/lockdep-design.html
[`syn`]: https://crates.io/crates/syn
[kernel-mutex]: https://github.com/Rust-for-Linux/linux/blob/459035ab65c0ebb8d7054b24b6c00de907819eb2/rust/kernel/sync/mutex.rs#L56
[mutex-docs]: https://www.kernel.org/doc/html/latest/locking/mutex-design.html
[pin-and-suffering]: https://fasterthanli.me/articles/pin-and-suffering
[pbr-rfc]: https://github.com/rust-lang/rfcs/pull/2884
[mcyoung-ctors]: https://mcyoung.xyz/2021/04/26/move-ctors/
[simple-safe-init]: https://github.com/y86-dev/simple-safe-init/tree/experiment
[simple-safe-init-experiment]: https://github.com/y86-dev/simple-safe-init/tree/experiment-syntax
[ssi-kernel]: https://github.com/y86-dev/linux/tree/simple-init-experiment
[pbr-discussion]: https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/placement.20by.20return/near/299535580
[rustyyato-rfc]: https://github.com/rust-lang/rfcs/pull/2534
[old-ptr-rfc]: https://github.com/rust-lang/rfcs/pull/98
[Rust book]: https://doc.rust-lang.org/stable/book/
[fp-rfc]: https://github.com/rust-lang/rfcs/pull/3318
[internals thread]: https://internals.rust-lang.org/t/safe-pinned-initialization/17404
