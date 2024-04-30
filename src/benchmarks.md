# Benchmarks

For benchmarks I've made a script that downloads top 300 Ruby libraries (by total downloads) and unpacks them.

The corpus contains:

1. `4176379` LOC
2. `170114575` bytes

Results on my local MBP on M2:

| Parser                  | Total time | Bytes per second | Lines per second |
| ----------------------- | ---------- | ---------------- | ---------------- |
| lib-ruby-parser (arena) | ~1.93s     | ~88,000,000      | ~2,160,000       |
| lib-ruby-parser (heap)  | ~4.4s      | ~38,000,000      | ~950,000         |
| ripper (Ruby 3.3.1)     | ~24s       | ~7,000,000       | ~175,000         |
| whitequark/parser       | ~245s      | ~700,000         | ~17,000          |

`ripper` and `whitequark/parser` are here just to show how fast things can get:

1. `ripper` is a built-in Ruby parser
2. `whitequark/parser` is a popular Ruby parser written in Ruby (yep, that's not a fare comparison at all)

For both of them I've disabled garbage collection and they both have no IO while running tests (i.e. becnhmark script reads all the files, then stops GC, then parses each file in a loop)

Results on my local Intel i5-11400 on Debian Sid:

| Parser                  | Total time | Bytes per second | Lines per second |
| ----------------------- | ---------- | ---------------- | ---------------- |
| lib-ruby-parser (arena) | ~3.18s     | ~TODO            | ~TODO            |
| lib-ruby-parser (heap)  | ~7.2s      | ~TODO            | ~TODO            |
| ripper (Ruby 3.3.1)     | ~18.2s     | ~TODO            | ~TODO            |

Ripper is doing better on Linux, I assume there's something wrong with `aarch64` builds of Ruby (or `glibc` allocator is significantly better than whatever comes with MacOS).
