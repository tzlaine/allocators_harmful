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

references:
  - id: BoostIface
    citation-label: Boost.STLInterfaces
    title: "Boost.STLInterfaces"
    author:
      - family: Laine
        given: Zach
    URL: https://github.com/tzlaine/stl_interfaces

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

## Allocators Violate the "Don't Pay For What You Don't Use" Principle

Adding allocator-aware constructors roughly doubles the number of constructors
for a type or template.  There is a compile-time cost to this, even for users
that never use allocators.  Moreover, there is a teachability to doubling the
number of constructors of a type or template.  Though both of these costs are
probably not large, users who do not care about allocators must still pay
them.

## Allocators Are a Bad Use of Committee Time

Counted by the number of template instantiations of allocator-aware standard
library types and templates, allocator use is probably less than 1%.  There
are also large numbers of C++ programmers who have never used a non-default
allocator.

As such, the benefits of allocator use, even if profound, are not realized by
most C++ programmers.  The amount of implementation time and API review time
associated with allocators are substantial.  Deduction guides are needed far
more often when allocators are in play, and so far deduction guides have
proved to be particularly time-consuming for committee members to review.

Standardizing allocator-aware APIs is poor use of our limited committee time.

## An Alternative to Allocators

The entire value proposition of using allocators is that you can effectively
create a new container with a different storage policy by writing relatively
less code than it would take to rewrite the entire container.  For instance,
writing a fixed-size allocator may be easier than writing a container like
`boost::container::static_vector`.

Writing a standard-library-compliant allocator is hard.  Writing a container
that conforms to the standard library's sequence container requirements is
also hard.

If we were to standardize a CRTP template `sequence_container_interface`,
analogous to `ranges::view_interface`, we could make writing conforming
sequence containers a straightforward process -- more straightforward than
either writing an allocator or writing a sequence container is in the status
quo.  The proposed [@BoostIface] library is an existence proof that this
approach is viable.
