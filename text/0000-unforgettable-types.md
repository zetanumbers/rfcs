# `unforgettable_types`

- Feature Name: `unforgettable_types`
- Start Date: 2024-06-27
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Add an auto trait `Forget` to indicate which types are safe to forget and special `Unforget` wrapper type to constrain `Forget`'s implementation.
Unforgettableness establishes a safety contract between objects/entities, usually between borrows and an operational object/entity,
i.e. a borrow from the stack and the thread manager in case of now removed [`std::thread::JoinGuard`].

As such unforgtettable type should be safe to forget if it outlives every lifetime that aforementioned operational object/entity is able to outlive, which is enforced with the `Unforget` wrapper.

Because currently every type is assumed to be safely forgettable, this feature nessecitates implicit `Forget` bounds on every used type and object in, so called, old code.
New and old code should be distinguished by the new code containg syntax indicating explicit support for unforgettable types,
this potentially being the crate's edition or some other syntax possibly similar albeit orthogonal to the edition system.
Tools like `cargo fix` may assist migration from old to new code.

This feature would allow now removed safe [`std::thread::scoped`] API while actually fixing the unsoundness issue [#24292](https://github.com/rust-lang/rust/issues/24292).
While to some extend the new [`std::thread::scope`] allows to circumvent the `'static` closure requirement, it is not generally applicable to the async/await code.

# Motivation
[motivation]: #motivation

In cases where an API requires some cleanup code to run for it to safely operate on non-static values, no universal solution have yet been found.
For example, as described in the [summary], there is a way to send non-static closures to other threads using stable [`std::thread::scope`],
but it disallows `.await` points while working with `std::thread::ScopedJoinHandle`s and is generally a bit unflexible.

```rust
let a = vec![1, 2, 3];

// cannot send borrows since spawned thread can outlive borrowed lifetime if we forget its `JoinHandle`
let thrd = std::thread::spawn(|| {
    dbg!(&a);
});

// this function takes closure, it is impossible to `.await` within its body
std::thread::scope(|s| {
    s.spawn(|| {
        dbg!(&a);
    });
});
```

In particular there can hypothetically be a function or a method that returns some object which mutably (exclusively) borrows one of the function's arguments.
While returned object is alive user is unable to refer to borrowed value, which can be a useful property to exploit.
Library author can invalidate some public safety invariant of a borrowed type but then restore it inside of the object's drop.
However given the fact that forgetting such object is would be safe, it is impossible to make such sound safe API.
There is one known example of this as once mentioned planned feature [`Vec::drain_range`](https://github.com/rust-lang/rust/issues/24292#issuecomment-93513451).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `Forget` trait is at the center of this proposal, used to distinguish whether a type is allowed to be forgotten or not via `T: Forget` bound.
It's marked unsafe so that the unsafe code would be able to safely rely on it.
Since forgetting or leaking an object is usually considered to be a logic bug anyway, there's little to no utility in the safe variantion.

The `T: Forget` bounds usually appear where drop on values of type `T` may not be called, that includes:

- Obviously the `forget` function;
- `ManuallyDrop` and `MaybeUninit` types as they disable drop;
- `Rc` and `Arc` types as they allow creation of an ownership cycle ([The Rust Programming Language - Chapter 15.6](https://doc.rust-lang.org/book/ch15-06-reference-cycles.html#reference-cycles-can-leak-memory));
- `std::sync::mpsc::(sync_)channel` and other channel types as they allow creation of an ownership cycle;
- etc.

To mark type as unforgettable there's a special wrapper type called `Unforget<'a, T>`, with interface similar to the `ManuallyDrop<T>`.
Overall interface is similar to `ManuallyDrop`, except for a lifetime parameter `'a` which purpose is explained in the [reference-level-explanation] section.
For now the `'static` lifetime should be considered its default lifetime,

To implement safe [`std::thread::JoinGuard`] once again, such type must be marked as unforgettable:

```rust
// NOTE: This code may be opinionated and is only an example implementation of the `JoinGuard` type.

use std::mem::ManuallyDrop;
use std::thread;

/// Handle to a thread, which joins on drop.
pub struct JoinGuard<'a, T> {
    child: ManuallyDrop<thread::JoinHandle<T>>,
    _unforget: Unforget<'static, PhantomData<&'a ()>>,
    // ...
}

impl<'a, T> Drop for JoinGuard<'a, T> {
    fn drop(&mut self) {
        // SAFETY: We are in the drop, so the `self.child` field won't be used again.
        let _ = unsafe { ManuallyDrop::take(&mut self.child) }.join();
    }
}

pub fn spawn_scoped<'a, F, T>(f: F) -> JoinGuard<'a, T>
where
    F: FnOnce() -> T + Send + 'a,
    T: Send + 'a,
{
    JoinGuard {
        // SAFETY: Thread is joined in the drop implementation, `JoinGuard` is an
        // unforgettable type because of the `_unforget` field's type
        child: unsafe {
            ManuallyDrop::new_unchecked(thread::Builder::new().spawn_unchecked(f).unwrap())
        },
        _unforget: Unforget::new(PhantomData),
        // ...
    }
}
```

However there's one last aspect that should be accounted as it is possible to send `JoinGuard` to the thread it "holds", essentially creating an ownership loop and forgetting it.
Thus it must be `!Send` to disallow sending between threads, unless forgetting is allowed:

```rust
pub struct JoinGuard<'a, T> {
    // ...
    _unsend: PhantomData<*mut ()>,
}

unsafe impl<T> Send for JoinGuard<'_, T> where Self: Forget {}
unsafe impl<T> Sync for JoinGuard<'_, T> {}
```

Some perculiar properties can be observed there when `T: 'static`, forgetting and sending `JoinGuard` to other threads becomes allowed.
Generally for any type `T: 'static` the `T: Forget` also holds true.
It would be possible to implement the old `std::thread::JoinHandle` using only `JoinGuard`, assuming sensible methods like `detach` and `join` were added to it.
The documentation of one such example implementation can be seen at [zetanumbers/leak-playground](https://zetanumbers.github.io/leak-playground/leak_playground_std).

On the other hand to support unforgettable types library author must explicitly allow unforgettable types and objects in their code.
This RFC doesn't weight on any specific syntax nor its locality characteristics, so these are some possible approaches:

- Trait bounds with a question mark `?Forget`;
- Some next rust edition `edition = "20XX"` no longer assumes every type is forgettable and explicit `Forget` bounds are placed wherever forgetness is required;
- Attribute over some code `#![support(unforgettable_types)]` is placed and explicit `Forget` bounds are used wherever forgetness is required.

The reason for such requirement is that the old code may otherwise assume any type is forgettable.
Adding support for unforgettable types in a package may happen in different ways depending on the present code:

- In case approach with explicit `Forget` bounds will be taken, author of any crate may switch from assumed forgetness to explicit `Forget` bounds without updating dependencies or any breaking changes to the API:
  - Assuming each package is sound by itself, any package without unsafe can be automatically migrated by tools like `cargo fix`, by specifying explicit `Forget` bounds anywhere it's required;
  - Any package containing internal unsafe code should conservativelly work only on forgettable types until code is checked to ensure no unforgettable object can be forgotten or leaked;
  - Any package containing public unsafe interfaces should conservativelly work only on forgettable types until API is checked to ensure no unforgettable object can be forgotten or leaked without adding new safety requirements to the documentation as that would be a breaking change;
- In case approach with explicit `?Forget` bounds will be taken, some crates should consider adding a support for unforgettable types first:
  1. Crates operating on futures like various async executors or traits [`StreamExt`](https://docs.rs/futures-util/0.3.30/futures_util/stream/trait.StreamExt.html), [`Sink`](https://docs.rs/futures-util/0.3.30/futures_util/sink/trait.SinkExt.html) and similar, as using unforgettable types would mostly affect async blocks making them also unforgettable;
  2. Crates operating on closures like for methods of `Iterator` trait, `Option`, `Result` types or [some client types], as passing unforgettable types into closures would make them also unforgettable.

<!--
Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.
- Discuss how this impacts the ability to read, understand, and maintain Rust code. Code is read and modified far more often than written; will the proposed feature make code easier to maintain?

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.
-->

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

<!--
```rust
unsafe auto trait Forget {}
```

The unforgetness works in harmony with the borrow checker.

```rust
#[repr(transparent)]
pub struct Unforget<'a, T: ?Sized> {
    // ...
}

unsafe impl<'a, T: ?Sized + 'a> Forget for Unforget<'a, T> {}
```
-->


----

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

The main issue with this proposal is old code's incompatibility with new hypothetical types like `JoinGuard` whenever it wasn't spawned with a `'static` closure.
It would harm rust-lang's ergonomics as everything that might be able take unforgettable objects should be able to, otherwise introducing the redundant forgetness constraint.
Such constrain would contradict addition of unforgettable types as these two facts would discourage each other's use when usecases intersect.
However this intersection might be negligible as only `JoinGuard`, [some client types] and `ScopedAsyncTask` will gain from this feature.

Old macros may expand into a code which would forget/leak a type without violating any `T: Forget` bound:

```rust
macro_rules! forget {
    ($b:expr) => {{
        // $b must be sized as we pass it to the `Box::new` by value
        let mut b = Some(Box::new($b));
        let byte_size = std::mem::size_of_val(&b);
        let p = (std::ptr::addr_of_mut!(b) as *mut u8);
        for i in 0..byte_size {
            unsafe { *p.add(i) = 0 };
        }
    }};
}
```

As such, macros with unsafe code should carefully take individual measures to ensure no unforgettable object is forgotten or leaked, possibly somehow conservativelly marking macro for compiler to know it's lack of support of unforgettable types in the maner of the old code.
However it might be better to just break such niche code from the compiler side anyway, but cost of this move is hard to estimate.

By implicitly disabling `T: Forget` default bound in next edition, some old documentation and learning materials might similarly become outdated and start containing instructions to or samples of unsound unsafe code when unforgettable types are taken into the account.
<!-- TODO: There should be some examples of that -->

<!-- Enabling placing default implicit `T: Forget` bound might significantly increase compilation times due to increasing complexity of the proof conducted by the type system.
Given this dificult situation it would be useful to correlate compilation time to the amount of syntax in the program.
To achieve that it would require instead of implicitly creating `T: Forget` predicates unless `T: ?Forget` is written, maybe introduce `T: ?Forget` predicates for that syntax instead similarly to coinduction. -->
<!-- No measurements were taken yet due to introduced deadlocks -->

---

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.

[`std::thread::scoped`]: https://doc.rust-lang.org/1.0.0/std/thread/fn.scoped.html
[`std::thread::JoinGuard`]: https://doc.rust-lang.org/1.0.0/std/thread/struct.JoinGuard.html
[`std::thread::scope`]: https://doc.rust-lang.org/1.79.0/std/thread/fn.scope.html
[some client types]: https://docs.rs/tigerbeetle-unofficial/0.4.0+0.15.3/tigerbeetle_unofficial/struct.Client.html