- Feature Name: Panic safe methods and their documentation
- Start Date: 2021-07-14
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary

[summary]: #summary

With Rust &mdash; our goal is to empower developers to build reliable software, and the way we do it is by trying to enforce correctness. Now this correctness can have multiple meanings, but here we're talking about methods that can cause runtime panics which isn't essentially _bad_, but is something that shouldn't come as a surprise.

For example, say you want to `remove` the element at the i-th position in a vector and that index that doesn't exist &mdash; this would cause a runtime panic with the current implementation. The plausible solution would be to provide a `try_remove` method. One might argue that then why not have having `try_swap_remove` or `try_swap` in `VecDeque`? You are right then. Also having so many `try_*` methods would make documentation harder to read (and maintain). This RFC proposes to simplify this entire _chain of thought_.

# Motivation

[motivation]: #motivation

This RFC proposes to add:

- A new form of the `#[doc]` attribute that helps link _panicky_ and _non-panicky_ methods together

As underlined in the [summary](#summary), these methods will help reduce the surprises when a method causes a runtime panic. At the same time:

- it makes it more convenient for users to find out _panicky_ and _non-panicky_ methods
- for library developers, it simplifies intra-doc links and duplication of documentation

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

## For the library developer

Let's say you're a developer building a new kind of _magical vector_ that reduces the number of reallocations by using some sort of chaining between blocks of memory. You've just finished writing the library and realize that you have methods like `remove` and `swap` which can panic if the provided index is out of bounds. The best way to mitigate this is by providing `try_remove` and `try_swap` methods. Instead of duplicating the documentation between `swap` and `remove`, you'll have to use the `#[doc(nopanic="method")]` attribute.

Say your remove function is defined like:

```rust
/// This function will remove the element at the provided index and will panic if it doesn't exist
pub fn remove(&mut self, index: usize) {
    // ...
}
```

then you will need to add:

```rust
#[doc(nopanic="try_remove")]
/// This function will remove the element at the provided index and will panic if it doesn't exist
pub fn remove(...) { ... }
```

And that would be it. `rustdoc` will link both your functions together and will render the documentation for your original remove method and then show `try_remove` to be the non-pancicking version.

## For the user

You just came across the `magicvec` crate and want to use it. In your application, you allow users to remove items from their list of favorite books by the book number which is essentially a `magicvec.remove(i)` function call.
But you're surprised that there is a runtime panic when you do a `remove` for a non-existent index. You head over the `rustdoc` for the `magicvec` crate and go to the remove method and find `try_remove` as the non-panicking method. You proceed to use the function and catch any possible errors while removing.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

This is another case of intra-doc linking where two `item`s in a crate can be linked by the new `nopanic` form of the doc attribute. This should be only applied
on member functions and trait methods, i.e just functions. This means that when running a doctest, we should error if a `nopanic` attribute is used on anything that isn't a function.

On seeing a function marked with the `nopanic` attribute, we will attempt to find the function that it is linked with. The functions are then displayed _close to each other_ in the final render with the panicking-one linking to the non-panicking version.

# Drawbacks

[drawbacks]: #drawbacks

- It adds more complexity to the way `rustdoc` builds documentation
- Increased build times for documentation
- More complex HTML templates
- Once this is added we will have to take up the task of grouping such methods together

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

- This design provides future expandibility for `try_*` like methods and reduces the _clutter_ in documentation
- This design also reduces duplication of documentation because now `rustdoc` does the job of having the documentation linked together
- Users will have lesser trouble looking for _panic safe_ methods in the documentation as not all methods can logically have names starting with '_try_'

# Prior art

[prior-art]: #prior-art

- Panic safe methods like [`try_reserve`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.try_reserve) and [`try_reserve_exact`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.try_reserve_exact) exist which involve a lot of duplication and also implicates the fact that we might see more `try_*` methods in implementations
- Intra-doc links were already implemented as a part of [RFC 1946](https://rust-lang.github.io/rfcs/1946-intra-rustdoc-links.html) which greatly simplified intra-doc linking

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- How exactly do we want to render the panicking and non-panicking methods?
- How will this be handled in trait methods?
- Can we now say that there are _panicking structs_ or objects in general?

# Future possibilities

[future-possibilities]: #future-possibilities

There are several future possibilites and `nopanic` might just be the start. For example, we could allow some kind of _grouping_ in the documentation to show all related methods (at the discretion of the crate developer) in one place instead of the usual _render where it is_ approach taken by `rustdoc`. This would complicate some things, but at the end of the day has the ability to simplify a lot of things for the end-user.
