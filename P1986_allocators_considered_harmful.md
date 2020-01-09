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
  - name: David Stone
    email: <david.stone@uber.com>, <david@doublewise.net>
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

TODO: Jonathan Wakely to fill this in with his recent experience implementing
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
are very unlikely to do so.  Specifically, you are unlikely to consider these
valid but improbable possibilities:

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
etc.).  However, when using `pmr::vector`, the problem is even worse than the
explicit allocator template parameter case since you cannot know what allocator
will be used at runtime from the type alone.  In the case of `pmr::vector` you
must reason about the correctness of `use_vec()` for all possible allocators
that may ever exist.

While type erasure and genericity are very useful in some contexts, when the
variation in behavior is closely tied to the semantics of another type or
template, you must be careful.  For instance, if I create a type-erased
`widget` type, I probably created it precisely because the widgets in my
program are supposed to vary widely in their behavior -- perhaps some are meant
to be buttons, others scrollbars, etc.  If I apply the same arbitrariness to how I
define a `pmr::vector`'s allocator's behavior, I have not created a useful
abstraction, just an unsettlingly vague one.  Remember that a `vector` is only
intended to be a template that manages a heap-allocated buffer.

## Allocators are Viral

Many C++ users, including committee members. are surprised to find out that
the container adaptors all have allocator-aware constructors, even though none
of them allocate anything.  Many users are also surprised to find that
`tuple` and `pair` have constructors that take allocators, even though neither
allocate anything.

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
probably not large, users who do not care about allocators must still pay for
them.

## Allocators Break Expected Equivalences

For almost every type, there are certain operations that users can count on
being equivalent, except for possibly performance.

<table>
<tr>
  <th style="width:9em">This operation:</th>
  <th style="width:7em">is equivalent to this operation:</th>
  <th style="width:20em">except if:</th>
</tr>

<tr>
<td><pre><code>T a = f();
T b;
b = a;</code></pre>
<td><pre><code>T a = f();
T b = a;
&nbsp;</code></pre>

<td>T is an allocator-aware type. Then it is equivalent only if
select_on_container_copy_construction returns a copy of the source allocator
and propagate_on_container_copy_assignment is true, or if
select_on_container_copy_construction returns a default constructed allocator
and propagate_on_container_copy_assignment is false.

<tr>
<td><pre><code>a.~T();
new(&a) T(b);</code></pre>
<td><pre><code>a = b;
&nbsp;</code></pre>

<td>T is an allocator-aware type. Then it is equivalent only if
select_on_container_copy_construction returns a copy of the source allocator
and propagate_on_container_copy_assignment is true, or if
select_on_container_copy_construction returns a default constructed allocator
and propagate_on_container_copy_assignment is false.

<tr>
<td><pre><code>T a = f();
T b = a;
a.~T();</code></pre>
<td><pre><code>T a = f();
T b = move(a);
a.~T();</code></pre>

<td>T is an allocator-aware type. Then it is equivalent only if
select_on_container_copy_construction returns a copy of the source
allocator. The move constructor always moves from the source allocator.

<tr>
<td><pre><code>T a = f();
T b = g();
T c = move(a);
a = move(b);
b = move(c);</code></pre>
<td><pre><code>T a = f();
T b = g();
swap(a, b);
&nbsp;
&nbsp;</code></pre>

<td>T is an allocator-aware type. Then it is equivalent only if
propagate_on_container_move_assignment is equal to
propagate_on_container_swap.

<tr>
<td><pre><code>noexcept(swap(a, b))</code></pre>
<td><pre><code>true</code></pre>

<td>T is an allocator-aware type. Then it is equivalent only if
propagate_on_container_swap is true or is_always_equal is true.

<tr>
<td><pre><code>swap(a, b)</code></pre>
<td><pre><code>// Swapping</code></pre>

<td>T is an allocator-aware type. Then it is well-defined only if
propagate_on_container_swap is true or a.get_allocator() == b.get_allocator().
</table>

In English, allocators violate the following equivalences:

- Default construction + copy assignment should be equivalent to copy
  construction, except copy construction might be faster.

- Default construction + move assignment should be equivalent to move
  construction, except move construction might be faster.

- Destruction + construction should be equivalent to assignment, except
  assignment might be faster or have stronger exception guarantees.

- Copy + destroy source should be equivalent to move + destroy source, except
  move might be faster and might be noexcept.

- Manually swapping with move construction + two move assignments should be
  equivalent to calling `swap`, except `swap` might be faster and should
  always be `noexcept`.

- `swap` should be well-defined if both inputs are readable.

## Allocators defeat return-by-value and move semantics

The advent of move semantics in C++11 enabled large objects to be returned
without additional allocation on a consistent basis:

```c++
std::vector<int> f() { /*...*/ }
//...
void g() {
    const std::vector<int> val = f(); // An additional allocation
                                      // is not required for return-by-value.
}
```

Say, however, that the author of `f()` desires to support users which make use
of `std::pmr` dynamic allocators. Naively they might revise the signature of
the function as follows:

```c++
std::pmr::vector<int> f() { /*...*/ }
```

Clients unfortunately cannot use this function efficiently if they want to use
custom allocators.

```c++
//...
void g() {
    const std::vector<int> val1 = f(); // An additional allocation/copy is
                                       // required when mixing with std::vector.

    const std::pmr::vector<int> val2 = f(); // This, alone, is performant.

    std::pmr::vector<int> val3(my_allocator);
    val3 = f(); // An additional allocation/copy is
                // required with a custom allocator.
}
```

Allocator advocates frequently suggest working around this issue by avoiding
return-by-value entirely.

```c++
void f(std::pmr::vector<int>* result) { /*...*/ }
//...
void g() {
    std::pmr::vector<int> temp;
    f(&temp);
    const std::vector<int> val1 = temp; // An additional allocation/copy is
                                        // still required when mixing with std::vector.

    std::pmr::vector<int> val2;
    f(&val2); // Still performant

    std::pmr::vector<int> val3(my_allocator);
    f(&val3); // Now performant
}
```

Note that this solution implies that all allocating return values should be
replaced with out pointer arguments! This, in the opinion of the authors, is
undoing much of progress made in the past 15 years to improve the readability
(return-by-value) and safety (extensive usage of `const`) of C++.

## Allocators Turn Value Types into Something in Between Value and Reference Types

TODO: David Stone to fill this in

## Allocators Have Existing Compelling Alternatives

Allocators allegedly provide a quick and easy way to improve performance of an
application by simply changing the allocator used by an object. Usually this
situation involves a large number of small objects that would otherwise be
expensive to allocate on the heap on an individual basis. In this situation,
however, a compelling alternative is making usage of an object pool. Not only
does this avoid the large number of small allocations, but, if configured to do
so, avoids execution of a possibly expensive destructor when objects are
released back to the pool.

Allocators also allegedly provide a means to track allocation hot spots in
applications, allowing for identification of memory hogs. In order for this to
work properly, the *entire* application (including all its dependencies) must
be made allocator aware. This is a huge up-front cost in additional code,
testing, and stability seems hardly worth it, but it is even less so given the
functionality existing tools provide. Consider
[tcmalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html) which
provides similar functionality but requires only a change in an application's
link line.


## Allocators Are a Bad Use of Committee Time

Counted by the number of template instantiations of allocator-aware standard
library types and templates, allocator use is probably less than 1%.  There
are also large numbers of C++ programmers who have never used a non-default
allocator.

Consider the relatively unknown fact that the frequently used standard library
templates `std::function` and `std::variant` are *not* allocator aware. And
yet, this being the case, there hasn't been any public outcry or proposals
coming forward suggesting that allocator awareness be added. This is a strong
indicator that very few engineers are making use of the standard library's
allocator awareness model.

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
quo.  The recently-accepted [@BoostIface] library is an existence proof that
this approach is viable.
