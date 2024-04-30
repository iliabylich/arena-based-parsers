# Allocator implementation

This is the heart of the whole story.

We want to have some data structure that:

1. encapsulates a raw pointer + size behind it
2. keeps track of where we have stopped writing
3. has interior mutability for the write pointer
4. allows to allocate (via const ref) a chunk of `N` bytes and moves the write pointer

Also we need to be careful with alignment, we do support byte arrays that can reserve `2 ^ N` bytes, but most of our data structures have alignment 8 (because most of them encpsulate pointers of some kind).

Here to simplify things we can introduce a simple (but reasonable) requirement: the region of memory that we take in control must be 8-byte-aligned (i.e. it should be a `*mut usize`), and there should be no way for us to access it as blob of bytes.

```rust
struct Blob<'b> {
    ptr: *mut usize,
    len: Cell<usize>,
    capacity: usize,
    _marker: core::marker::PhantomData<&'b ()>,
}

impl<'b> From<&'b mut [usize]> for Blob<'b> {
    fn from(slice: &'b mut [usize]) -> Self {
        Self {
            ptr: slice.as_mut_ptr(),
            len: Cell::new(0),
            capacity: slice.len(),
            _marker: core::marker::PhantomData,
        }
    }
}
```

The blob can only be constructed from a slice of `usize` (e.g. from a stack allocated array `[usize; N]` or from heap-allocated `vec.as_mut_slice()`). Then allocations becomes much simpler:

```rust
use core::mem::{size_of, align_of};

fn write_ptr(&self) -> *mut usize {
    unsafe { self.ptr.add(self.len.get()) }
}

fn alloc_uninitialized<T>(&self) -> NonNull<T> {
    assert_eq!(size_of::<T>() % size_of::<usize>(), 0);
    assert_eq!(align_of::<T>(), size_of::<usize>());

    let ptr = self.write_ptr();
    let count = size_of::<T>() / size_of::<usize>();
    self.assert_has_space_for_extra_words(count);

    self.len.set(self.len.get() + count);
    unsafe { NonNull::new_unchecked(ptr as *mut _) }
}

fn assert_has_space_for_extra_words(&self, required_words: usize) {
    let left = self.capacity - self.len.get();
    assert!(
        required_words <= left,
        "OOM error: can't allocate {} words, only {} has left",
        required_words,
        left
    );
}
```

So instead of operating on `*const u8` we use `*const usize` and thus we can always be sure that our pointer is properly aligned.

Bytes allocation (for `ByteArray`) requires some rounding though:

```rust
pub fn push_bytes(&self, bytes: &[u8]) -> &'b [u8] {
    let len_in_words = bytes.len().next_multiple_of(size_of::<usize>()) / size_of::<usize>();
    self.assert_has_space_for_extra_words(len_in_words);
    let ptr = self.write_ptr();

    unsafe {
        core::ptr::copy_nonoverlapping::<u8>(
            bytes.as_ptr(),
            self.write_ptr().cast(),
            bytes.len(),
        )
    };

    self.len.set(self.len.get() + len_in_words);

    unsafe { core::slice::from_raw_parts(ptr.cast(), bytes.len()) }
}
```

So if we need to write 10 bytes we write 16. The pointer is 8-byte-aligned before and after writing.

Once we have these primitives we need to make decision on how our data structures must be initialized:

1. we can enforce the underlying memory of the blob to be always zero-initialized (and `panic!` if it's not true), then those data structures that support zero initialization can be allocated as is, via `blob.alloc_uninitialized::<T>()`
2. we can add an explicit constructor to each structure that allocates on a blob and initializes acquired region with the default state

I went with the second option, this way the array can be re-used between parser runs with no extra computations:

```ruby
let mut mem = [0; 1000];

for file in files {
    let blob = Blob::from(&mut mem);
    parse(file.content, &blob);
}
```

However it requires some extra code in each struct, and `Blob::alloc_uninitialized` must be explicitly marked as a private function so that no code accidentally calls it:

```rust
impl ArenaAllocatedStruct {
    // Having a dedicated method allows other structs
    // to "embed" it and still have ability to
    // initialize it with default fields.
    // 
    // This could also be `impl Default` of course.
    fn new_in_place() -> Self {
        Self {
            // ...
        }
    }

    fn new_in(blob: &Blob<'b>) -> &'b Self {
        let this: &'b mut Self = blob.alloc_uninitialized();
        *this = Self::new_in_place();
        this
    }
}
```
