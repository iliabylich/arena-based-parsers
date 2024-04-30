# Double linked list

It's almost the same as our single linked lists, except that we need to have another constraint for `T`:

```rust
trait DoubleLinkedIntrusiveListItem {
    fn prev(&self) -> Option<NonNull<Self>>;
    fn set_prev(&self, new_prev: Option<NonNull<Self>>);
    fn next(&self) -> Option<NonNull<Self>>;
    fn set_next(&self, new_next: Option<NonNull<Self>>);
}
```

and so to have a list of numbers our supplementary data structure must look like this:

```rust
struct IntrusiveNumber {
    n: u64,
    prev: Cell<Option<NonNull<Self>>>,
    next: Cell<Option<NonNull<Self>>>,
}

impl DoubleLinkedIntrusiveListItem for IntrusiveNumber {
    fn prev(&self) -> Option<NonNull<Self>> {
        self.prev.get()
    }

    fn set_prev(&self, new_prev: Option<NonNull<Self>>) {
        self.prev.set(new_prev)
    }

    fn next(&self) -> Option<NonNull<Self>> {
        self.next.get()
    }

    fn set_next(&self, new_next: Option<NonNull<Self>>) {
        self.next.set(new_next)
    }
}
```

It might sound like it requires too much memory and in the case of a number that's probably true. However I only need it for some internal data structures that are allocated [on the scratch arena](/memory_usage_optimisations.md).

It's possible to slightly reduce memory usage [by using relative offsets](/memory_usage_optimisations.md).
