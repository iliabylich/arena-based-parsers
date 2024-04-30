# AST caching

As I mentioned before it's guaranteed that produced AST has references only to other arena-allocated objects, which means that it's possible

1. to write the content of arena straight on disk
2. read it later and "re-interpret" back to AST

The code could look roughly like this:

```rust
fn main() {
    dump();

    // This code below is meant to be executed later
    let data = read();
    let node = load(&data);
}

// Parses code, dumps it to a temp file, returns nothing
fn dump() {
    let code = b"10 + 20";

    let mut mem = [0; 1000];
    let base_ptr = &mem as *const usize;
    let blob = Blob::from(&mut mem);

    let ast: &Node = Parser::new(code, &blob).parse().ast.unwrap();

    let mut f = File::create("/tmp/ast").unwrap();

    let root_node_offset = unsafe {
        let node_ptr = ast as *const Node;
        let offset = node_ptr.byte_offset_from(base_ptr);
        assert!(offset > 0);
        offset
    };
    f.write_all(&usize_to_bytes(root_node_offset as usize)).unwrap();
    f.write_all(blob.data()).unwrap();
}

fn read() -> Vec<u8> {
    let mut f = File::open("tmp/ast").unwrap();
    let mut data = vec![];
    f.read_to_end(&mut data).unwrap();
    data
}

fn load(data: &[u8]) -> &Node {
    let root_node_offset = bytes_to_usize(*data.first_chunk().unwrap());
    let data = &data[8..];

    unsafe { data.as_ptr().add(root_node_offset).cast::<Node>().as_ref().unwrap() }
}

fn usize_to_bytes(n: usize) -> [u8; 8] {
    n.to_ne_bytes()
}
fn bytes_to_usize(bytes: [u8; 8]) -> usize {
    unsafe { core::mem::transmute(bytes) }
}
```

This could be a huge advantage for static analysis tools, caching becomes just

1. read `mtime` of the source file
2. check if cached AST is newer, read + return if so
3. otherwise, parse it and write on disk

AST of multiple source files could probably be packed together in a single binary file (e.g. based on hierarchy of files, a bin file per directory)
