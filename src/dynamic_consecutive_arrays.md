# Dynamic consecutive arrays

This is an easy one, it has the same API as the built-in `std::vec::Vec<T>`:

```rust
struct ByteArray<'b> {
    ptr: Cell<*mut u8>,
    len_in_bytes: Cell<usize>,
    capacity_in_bytes: Cell<usize>,
    marker: core::marker::PhantomData<&'b ()>,
}
```

Implementing methods like `.grow()`, `.push()` and `.pop()` is easy, you can use standard `vec` as a reference implementation.
