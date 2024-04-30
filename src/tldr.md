# TL;DR

This micro-book is about parsers, memory allocations and arenas.

+ The parser is written in Rust, it parses code in Ruby. You don't need to know both though.
+ The change that I'm describing here made a parser that I created in the past to go from 37MB/s to 80MB/s (which is roughly a 2x improvement)
+ I was able to completely eliminate all heap allocations (the parser is written in Rust, so it runs in `#[no_std]` mode)
+ Result of the parsing is located exclusively on arena, so the whole blob of underlying memory can be written on disk and later `mmap`-ed to quickly get it back if needed, so AST caching is significantly easier.

The code is a bit "experimental" but it's available in the [`arena-fixes` branch](https://github.com/lib-ruby-parser/lib-ruby-parser/tree/arena-fixes) (make sure to enable `--features=development` if you run it from a Git repo).

**I am not going to release it. If you need a Ruby parser better try [`prism`](https://github.com/ruby/prism) (it has Rust bindings that are available on crates.io). This whole story is more like an experiment to see if it's worth doing this type of work.**
