---
title: "Allocators Considered Harmful"
document: D1986R0
date: 2019-11-10
audience:
  - LEWG-I
  - LEWG
author:
  - name: Zach Laine
    email: <whatwasthataddress@gmail.com>
toc: false

---

# Abstract

TODO: Fill in later, based on the content of the other sections.

# Allocators Have Numerous Technical Issues

TODO: Jonathan Wakeley to fill this in with his recent experience implementing
all the allocator machinery for libstdc++

# Allocators Are a Poor Abstraction

Even were one to address all the technical issues with allocators, allocators
would still be inappropriate for standard C++, because they are a poor
abstraction.

Consider `vector`, the default standard container, as recommended by the
standard itself and experts alike.  Its invariant is that, within the limits
of available memory, it manages the allocations, copies, and deallocations of
storage for a contiguous sequence of objects of its element template parameter
`T`.

Now consider this function:

```c++
template<typename T, typename Alloc>
auto use_vec(std::vector<T, A> & vec)
{
    if (some_condition())
        vec.push_back(get_new_t());
}
```

Let's assume that out-of-memory is a very uncommon condition for the
environment in which this code is run.  What are you as a code reviewer to
make of this function?  You will probably think about the semantics of how
`vec` is used outside of the function, the semantics of `some_condition()`,
and the semantics of `get_new_t()`.

What you must always also think about is the behavior of `Alloc`, though you
is very unlikely to do so.  Specifically, you are unlikely to consider these
valid but unlikely possibilities:

- `Alloc` has a fixed capacity of 8 `T`s, and so throws when pushing the 9th
  element;

- `Alloc` produces logging or other bookkeeping as a side-effect, with some
  performance impact; or

- `Alloc` throws on each call to `allocate()` on its author's birthday, just
  for funsies.

While the first two are more reasonable, the possibility of unreasonable or
incorrect code from the author of `Alloc` means that you _must_ go into the
definition of `Alloc` to make an informed decision as to whether the code in
`use_vec()` is correct.  That kind of non-local reasoning is harmful to
correctness.

In short, the valid types that can be supplied for `Alloc` are allowed to
arbitrarily, and possibly dramatically, change the user's expectation of what
it means to use a `vector`.

A `vector` that throws when main memory is exhausted, a `vector` that throws
when its allocation pool is exhausted, and a `vector` that throws when its
storage for 8 `T`s is exhausted should probably not all be called `vector`.
To use the same name for all three types is a violation of the principle of
least surprise, and a violation of the single-responsibility principle of type
design.

## Enter `pmr`

Thankfully, most uses of `vector` in real code do not look like `use_vec()`,
because most uses imply the use of the default allocator (`vector<int>`,
etc.).  However, when using `pmr::vector`, the problem is even worse, since
you cannot know what allocator will be used at runtime from the type alone.
In the case of `pmr::vector` you must reason about the correctness of
`use_vec()` for all possible allocators that may ever exist.

While type erasure and genericity are very useful in some contexts, when the
variation in behavior is closely tied to the semantics of another type or
template, you must be careful.  For instance, if I create a type erased
`widget` type, I probably created it precisely because the widgets in my
program are supposed to vary widely in their behavior -- perhaps some are mean
to be buttons, others scrollbars.  If I apply the same arbitrariness to how I
define a `pmr::vector`'s allocator's behavior, I have not created a useful
abstraction, just an unsettlingly vague one.  Remember that a `vector` is only
intended to be a template that manages a heap-allocated buffer.

## Allocators are Viral

Many C++ users, including committee members. are surprised to find out that
the container adaptors all have allocator-aware constructors, even though none
of them allocates anything.  Many users are also surprised to find that
`tuple` and `pair` have constructors that take allocators, even though neither
allocates anything.

These allocator-aware constructors are there so that allocating members can be
given a particular allocator that the members may be constructed with.  These
constructors use special "uses-allocator construction" wording in the library
wording.  This is another violation of the principle of least surprise and the
single responsibility principle.

## Allocators Violate the "Do Not Pay For What You Do Not Use" Principle

TODO: There is a real user cost to allocator-aware interfaces: runtime
overheads; overload resolution is very expensive at runtime.  Pmr costs at
runtime even when you do not use it; even though it is opt-in. if you want to
make your code allocator-future-proof, that has a cost.

## Allocators Are a Bad Use of Committee Time

TODO: They are almost never used, especially with respect to template
instantiations.  They are bad bang-for-buck for our limited time.

## A Possible Alternative

TODO: Section about making containers easier to write.
