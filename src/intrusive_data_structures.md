# Intrusive data structures

What if our program needs to allocate more that just numbers and sequences of numbers? In my library I need at least:

1. single linked lists (for append-only lists of AST nodes)
2. double linked lists (for some internal data structures that can be extended both ways via `{push,pop}_{back,front}`)
3. hashmaps (to handle things like a set of local variables better-than-in-`O(N)`)
4. dynamic consecutive arrays (for all kinds of strings)

This is the place where things get complicated.

Let's say we have a single linked list that is (somehow?) allocated on arena and we want to push existing arena-allocated value to this list. If it's written in a traditional way that would be something like

```rust
struct List<T> {
    head: *mut ListNode<T>,
    tail: *mut ListNode<T>,
}

struct ListNode<T> {
    value: T,
    next: *mut ListNode<T>,
}
```

where `T` can be literally anything, and `*mut T` points to some place on arena where `T` is located. There's one problem though: `T` must be a part of the `ListNode` and so when we append it to a list we are force to either:

1. allocate a new `ListNode` and copy `T` into it
2. embed a pointer to `*const T` in `ListNode` instead of the `T` itself

The former is not only slow in some cases (if `T` is big enough) but also requires singnificantly more memory, because every time we push `T` we consume `sizeof(T)` memory of arena.

The latter introduces an unnecessary level of indirection that makes lookups slower (because we need to go through another pointer every time we read a list item).

Here comes a solution: **intrusive data structures**.

> Elements of intrusive data structures know that they belongs to a certain structure.

Yes, when I explained this concept to my friend his first impression was "wait, but it strongly violates encapsulation". And yes, he was absolutely correct, but that's the point.

Not every `T` can become an element of a single-linked intrusive list, only those that have some extra data fields that `List<T>` expects.
