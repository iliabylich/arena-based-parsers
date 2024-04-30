# AST nodes

Before and after the change AST nodes were represented as a sum type of all possible variants. [There's quite a lot of them](https://github.com/lib-ruby-parser/nodes/blob/master/NODES.md).

For example, an `ArrayNode` that represents the following code

```ruby
[1, "foo", 42]
```

previously had roughly the following structure:

```rust
pub struct ArrayNode {
    pub elements: Vec<Node>,
    
    pub begin_l: Option<Loc>,
    pub end_l: Option<Loc>,
    pub expression_l: Loc,
}

struct Loc {
    pub begin: usize,
    pub end: usize
}

enum Node {
    ArrayNode(ArrayNode),
    // ... other 100+ variants
}
```

Now it's fully arena-allocated:

```rust
pub struct ArrayNode<'b> {
    pub elements: &'b List<'b, Node<'b>>,
    
    pub begin_l: Option<Loc>,
    pub end_l: Option<Loc>,
    pub expression_l: Loc,

    next: Cell<Option<NonNull<Node<'b>>>>,
}

enum Node<'b> {
    ArrayNode(ArrayNode<'b>),
    // ... other 100+ variants
}
```

It's still possible to access fields and match on a node, however you can no longer pattern match on it without specify the `..` pattern in the fields list (as if it's been marked as `#[non_exhaustive]`)

Also, to simplify instantiation of node I had to change builder functions. Previously I was able to construct any node from any place, but now there's a `next` pointer to support embedding a node in a `List<Node>`.

I came up with the following solution:

```rust
impl<'b> ArrayNode<'b> {
    fn new_in<F>(blob: &'b Blob<'b>, f: F) -> &'b Node<'b>
    where
        F: FnOnce(&mut Self),
    {
        let mut uninit = MaybeUninit::<Self>::zeroed();
        let mut_ref = unsafe { &mut *uninit.as_mut_ptr() };

        // .. initialize required fields on mut_ref
        // of types like List<T> or ByteArray
        mut_ref.elements = List::new_in(blob);

        let mut variant = unsafe { uninit.assume_init() };

        f(&mut variant);

        let node = blob.alloc_uninitialized_mut::<Node>();
        *node = Node::ArrayNode(variant);

        node
    }
}
```

This way I can temporarily get access to the inner variant in a mutable fashion and get back at the end a wrapped `const Node` reference, which doesn't violate the rule of Rust of having no overlapping references in a single context:

```rust
let node: &Node = ArrayNode::new_in(blob, |array| {
    array.elements.push(child1, blob);
    array.elements.push(child2, blob);
    array.elements.push(child3, blob);

    array.begin_l = Some(Loc::new(...));
    array.end_l = Some(Loc::new(...));
    array.expression_l = Loc::new(...);
});
```
