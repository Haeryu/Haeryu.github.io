---
title: A pointer-free Red-Black Tree in Zig
subtitle: Zig Red-Black Tree
layout: default
date: 2025-04-24
---

I made a Red-Black Tree in Zig because I couldn’t find one on the internet.
[repo](https://github.com/Haeryu/rbtree/tree/master)

 - No pointers.
 - Freelist-based reuse. No per-node allocation.
 - Flat, cache-friendly layout.
 - From scratch, just std.
 - Small simple, minimal, and efficient.


Why pointer-free?
---
I didn’t start with the idea of making it pointer-free.
I just didn’t want to allocate memory per node.
So I tried to minimize heap usage — and this structure came out.
It turned out to have some nice side-effects:

 - Easy serialization – everything’s flat, can dump/load as-is
 - Cache-friendly – tight memory layout, no indirection
 - Debuggable – indices are easier to reason about than pointers
 - No GC / lifetime tracking – simple memory management

How I made it pointer-free
--- 
I used two key Zig features:
 - Non-exhaustive `enum`

```zig
    pub const Idx = enum(usize) {
        sentinel = std.math.maxInt(usize),
        _,
    }
```

    This gives me a type-safe way to represent indices — like `Idx` instead of plain `usize`.
    I originally used it just for type safety, but Zig lets me add a sentinel (`Idx.sentinel`) with zero runtime overhead.
    I think I saw a similar pattern in Zig’s own compiler code.

 - `std.MultiArrayList`
  
    Perfect for flattening tree structures into a tightly packed memory layout.
    All node fields (keys, values, colors...) are stored in contiguous arrays.
    That means I can query individual fields like a columnar database: access just what I need, and iteration is cache-friendly.
    Serialization becomes trivial — just dump the whole buffer.


Memory reuse (and a problem I ran into)
---
At first, I didn’t worry too much about memory reuse.
But once deletion worked, I realized freeing individual nodes would mean updating every index pointing to them.
That’s messy, error-prone, and slow.

So I went with a freelist strategy.
Instead of actually freeing nodes, I keep them in a linked list of reusable indices.
In pointer-based implementations, you’d usually `reinterpret` the memory of a freed node to store the next pointer.
But since I’m not using pointers, I reused an existing field to store the next index.

```zig

    pub const Node = struct {
        // other fields
        parent: Idx = .sentinel,

        pub const Idx = enum(usize) {
            // reuse .parent to store the next index in the freelist
            pub fn getNext(self: Idx, tree: *const Tree) Idx {
                const slice = tree.nodes.slice();
                return slice.items(.parent)[@intFromEnum(self)];
            }

            pub fn setNext(self: Idx, next: Idx, tree: *const Tree) void {
                const slice = tree.nodes.slice();
                slice.items(.parent)[@intFromEnum(self)] = next;
            }

            pub fn isFreeSpace(self: Idx) bool {
                return self != .sentinel;
            }
        };
    };

    fn recycleNode(tree: *Self, node: Node.Idx) void {
        node.setNext(tree.free_list, tree);
        tree.free_list = node;
    }
```

Final thoughts
---
It was a great way to apply some techniques I'd always wanted to try.

Data structures are hard. 
Unless you're using Zig.