# #![no_std]

To make sure that there's not even a single allocation anywhere in the code the easiest solution is to enable a `#![no_std]` crate-level flag.

Luckily there are multiple "modules" in the parser library that can be migrated independently:

1. AST library. This is a separate crate that contains types of AST nodes and all kinds of parsing errors. Now it also comes with all of the arena-related data structures that I mentioned before, and I was able to add **both the new and the old** versions of this library to the root parser crate.
2. Lexer (or tokenizer if you prefer). An iterator-like object that returns tokens, one by one. I wished it shared less state with the parser, but it's Ruby. I was able to gradually make small parts `no_std`-compatible, one step at a time.
3. Diagnostic messages, this is the module that reports errors/warnings, and it's pretty self-contained. I slowly migrated it to use the version of the AST library and allocate everything on arena.
4. The parser itself. This was the biggest all-or-nothing change, took me almost a week.
5. Unit tests. I'm using file-based fixtures, for each of them there's a unit test that reads it from the disk, parses it to split input vs expected output and runs the parser. That was also a relatively easy change. `std::fs::read` is now `include_bytes!`, so no FS logic is needed.

Low-level parts related to arena have been fully tested with `miri`, and their interfaces are safe, so migration overall was easy. If you have safe and stable primitives as your building bricks migration becomes mostly mechanical work.

From some POV `no_std` is an insanely strict requirement for the code that probably is going to run on a full `std` environment anyway. However I don't see any other way to keep track of allocations other that providing a custom global `Allocator` that `panic!`-s at runtime if you try to allocate. Honestly, I prefer a compile-time error here.
