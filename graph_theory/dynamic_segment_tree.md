# DYNAMIC SEGMENT TREE

Also called an **implicit** or **sparse** segment tree.

Use this when the index/coordinate space is huge (say up to `1e18`) but the number of
updates/queries is small (say up to `2e5`), and you either cannot or do not want to
coordinate-compress ahead of time (for example the coordinates arrive online).

Instead of preallocating `2 * size` nodes over the whole range, nodes are created
**lazily** the first time a position is touched. Each point update creates at most
`O(log C)` nodes, where `C` is the size of the coordinate range, so total memory is
`O(Q log C)` for `Q` operations.

Key differences from the classic array-backed segment tree:
- The tree is defined over a fixed logical range `[0, C)` (or `[lo, hi]`), not over `n` leaves.
- Children are referenced by node index into a growable pool; `0` means "does not exist yet".
- A missing child contributes the neutral element, so you never have to build the whole tree.

## When to use this vs coordinate compression

If you can collect all the queries/updates offline first, just compress coordinates and use
the normal fast segment tree in [segment_trees.md](segment_trees.md) — it is simpler and faster.

Reach for the dynamic segment tree when:
- Coordinates are online (you must answer queries before seeing later updates), or
- You are merging/persisting trees (segment tree merging, persistent segment tree), or
- It is just more convenient to index directly by value without a compression step.

## Point update, Range query (C++)

Node pool grows on demand. `merge` here is sum; swap it for min/max/gcd/etc. and change
`neutral` accordingly. Range is `[0, C)`, queries are inclusive `[l, r]`.

```cpp
using int64 = long long;

struct DynamicSegmentTree {
    struct Node {
        int64 val = 0;          // aggregate over this node's range
        int left = 0, right = 0; // child indices into `pool`; 0 == no child
    };

    int64 C;                    // coordinate range covered is [0, C)
    int64 neutral = 0;
    vector<Node> pool;

    DynamicSegmentTree(int64 range) : C(range) {
        pool.reserve(1 << 20); // reserve to avoid reallocation invalidating references
        pool.emplace_back();   // index 0 is the shared "null" node (stays neutral)
        pool.emplace_back();   // index 1 is the root
    }

    int64 merge(int64 a, int64 b) { return a + b; }

    // set position pos to val (use += in the leaf line for point-add instead of assign)
    void update(int64 pos, int64 val) { update(1, 0, C - 1, pos, val); }

    void update(int node, int64 lo, int64 hi, int64 pos, int64 val) {
        if (lo == hi) {
            pool[node].val = val; // pool[node].val += val; for point-add
            return;
        }
        int64 mid = lo + (hi - lo) / 2;
        if (pos <= mid) {
            if (!pool[node].left) { pool[node].left = pool.size(); pool.emplace_back(); }
            update(pool[node].left, lo, mid, pos, val);
        } else {
            if (!pool[node].right) { pool[node].right = pool.size(); pool.emplace_back(); }
            update(pool[node].right, mid + 1, hi, pos, val);
        }
        pool[node].val = merge(pool[pool[node].left].val, pool[pool[node].right].val);
    }

    // inclusive query over [l, r]
    int64 query(int64 l, int64 r) { return query(1, 0, C - 1, l, r); }

    int64 query(int node, int64 lo, int64 hi, int64 l, int64 r) {
        if (!node || r < lo || hi < l) return neutral; // missing node or no overlap
        if (l <= lo && hi <= r) return pool[node].val;  // full overlap
        int64 mid = lo + (hi - lo) / 2;
        return merge(query(pool[node].left, lo, mid, l, r),
                     query(pool[node].right, mid + 1, hi, l, r));
    }
};
```

Usage:

```cpp
DynamicSegmentTree seg(1e9);   // logical range [0, 1e9)
seg.update(500000000, 3);      // O(log C) time and up to O(log C) new nodes
seg.update(7, 10);
int64 s = seg.query(0, 500000000); // inclusive range sum
```

Notes on the pool:
- Index `0` is a permanent null/neutral node; a `0` child means the subtree is all neutral, so
  reading `pool[0].val` during `merge` is safe and returns the neutral value.
- `pool.reserve(...)` up front matters: if the vector reallocates while you hold a reference
  like `pool[node]`, that reference dangles. Writing `pool[node].left = ...` by index (as above)
  avoids holding a reference across an `emplace_back`, but reserving is still the safe habit.
  Size the reserve to about `Q * log2(C)` nodes plus a margin.

## Range update, Range query with lazy propagation (C++)

Same idea, but each node also carries a lazy tag. This supports range-add + range-sum over a
huge coordinate space. Children are materialized on push-down.

```cpp
using int64 = long long;

struct LazyDynamicSegmentTree {
    struct Node {
        int64 val = 0, lazy = 0;
        int left = 0, right = 0;
    };
    int64 C;
    vector<Node> pool;

    LazyDynamicSegmentTree(int64 range) : C(range) {
        pool.reserve(1 << 20);
        pool.emplace_back(); // null node 0
        pool.emplace_back(); // root 1
    }

    int child(int node, bool right) {
        int &c = right ? pool[node].right : pool[node].left;
        if (!c) { c = pool.size(); pool.emplace_back(); }
        return c;
    }

    void applyAdd(int node, int64 lo, int64 hi, int64 add) {
        pool[node].val += (hi - lo + 1) * add;
        pool[node].lazy += add;
    }

    void pushDown(int node, int64 lo, int64 hi) {
        if (pool[node].lazy == 0) return;
        int64 mid = lo + (hi - lo) / 2;
        applyAdd(child(node, false), lo, mid, pool[node].lazy);
        applyAdd(child(node, true), mid + 1, hi, pool[node].lazy);
        pool[node].lazy = 0;
    }

    void update(int64 l, int64 r, int64 add) { update(1, 0, C - 1, l, r, add); }

    void update(int node, int64 lo, int64 hi, int64 l, int64 r, int64 add) {
        if (r < lo || hi < l) return;
        if (l <= lo && hi <= r) { applyAdd(node, lo, hi, add); return; }
        pushDown(node, lo, hi);
        int64 mid = lo + (hi - lo) / 2;
        update(child(node, false), lo, mid, l, r, add);
        update(child(node, true), mid + 1, hi, l, r, add);
        pool[node].val = pool[pool[node].left].val + pool[pool[node].right].val;
    }

    int64 query(int64 l, int64 r) { return query(1, 0, C - 1, l, r); }

    int64 query(int node, int64 lo, int64 hi, int64 l, int64 r) {
        if (!node || r < lo || hi < l) return 0;
        if (l <= lo && hi <= r) return pool[node].val;
        pushDown(node, lo, hi);
        int64 mid = lo + (hi - lo) / 2;
        return query(pool[node].left, lo, mid, l, r) +
               query(pool[node].right, mid + 1, hi, l, r);
    }
};
```

The lazy version allocates children eagerly on `pushDown`, so worst-case node count is a little
higher than the point-update version, but still `O(Q log C)`.

## Python

Dictionary-backed variant. Slower constant factor but avoids the reallocation pitfalls entirely.
Point update, inclusive range-sum query over `[0, C)`.

```py
import sys
input = sys.stdin.readline

class DynamicSegmentTree:
    def __init__(self, C: int):
        self.C = C
        self.val = {}    # node id -> aggregate
        self.left = {}   # node id -> left child id
        self.right = {}  # node id -> right child id
        self.cnt = 1     # next free node id; node 1 is the root
        self.root = 1

    def _val(self, node):
        return self.val.get(node, 0)

    def update(self, pos: int, v: int) -> None:
        self._update(self.root, 0, self.C - 1, pos, v)

    def _update(self, node, lo, hi, pos, v):
        if lo == hi:
            self.val[node] = v   # += v for point-add
            return
        mid = (lo + hi) // 2
        if pos <= mid:
            if node not in self.left:
                self.cnt += 1
                self.left[node] = self.cnt
            self._update(self.left[node], lo, mid, pos, v)
        else:
            if node not in self.right:
                self.cnt += 1
                self.right[node] = self.cnt
            self._update(self.right[node], mid + 1, hi, pos, v)
        self.val[node] = self._val(self.left.get(node)) + self._val(self.right.get(node))

    def query(self, l: int, r: int) -> int:
        return self._query(self.root, 0, self.C - 1, l, r)

    def _query(self, node, lo, hi, l, r):
        if node is None or r < lo or hi < l:
            return 0
        if l <= lo and hi <= r:
            return self._val(node)
        mid = (lo + hi) // 2
        return self._query(self.left.get(node), lo, mid, l, r) + \
               self._query(self.right.get(node), mid + 1, hi, l, r)
```

## Complexity

- Time: `O(log C)` per update and per query, where `C` is the coordinate range.
- Space: `O(Q log C)` for `Q` update operations (only touched nodes exist).

For very large `C` and many operations, watch memory: reserve/estimate `Q * ceil(log2 C)` nodes.
If you hit memory limits, prefer offline coordinate compression + the array segment tree.
