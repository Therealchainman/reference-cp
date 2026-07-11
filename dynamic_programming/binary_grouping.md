# Binary Grouping (Powers-of-Two Decomposition)

A representation trick: **every integer count from `0` to `m` can be written as a subset
sum of only `O(log m)` groups.** Instead of listing `m` copies of a `1`, you bundle them
into a handful of blocks whose subset sums cover `[0, m]` exactly.

## The idea

Take the powers of two `1, 2, 4, 8, ...` while they fit, then use a single leftover block
for the remainder. For `m = 13`:

```
1 + 2 + 4 = 7      (powers of two that fit)
13 - 7    = 6      (leftover)

groups: 1, 2, 4, 6
```

Every value `0..13` is reachable as a subset sum of `{1, 2, 4, 6}`:

- `0` = {}
- `5` = 1 + 4
- `10` = 4 + 6
- `13` = 1 + 2 + 4 + 6

So thirteen `1`s collapse into **4** blocks instead of 13.

### Why it works

The powers `1, 2, 4, ..., 2^(k-1)` already cover every value in `[0, 2^k - 1]` (this is just
binary representation). Once the remaining count `r = m - (2^k - 1)` is smaller than the next
power, adding one final block of size `r` extends the reachable range to `[0, m]` without
introducing any gap: anything in `[r, m]` is `r` plus something in `[0, m - r] ⊇ [0, 2^k - 1]`,
and anything in `[0, r)` is already covered by the powers. The number of blocks is
`⌊log2(m)⌋ + 1 = O(log m)`.

## Decomposition snippet

```cpp
// split a count `m` into powers-of-two blocks + leftover
vector<int> groups;
int c = 1;
while (m > c) {
    m -= c;
    groups.emplace_back(c);
    c <<= 1;
}
if (m) groups.emplace_back(m); // leftover
```

## Where this is used

The canonical application is the **Bounded Knapsack Problem**: you have `count[i]` copies of
item `i`, and taking `t` copies contributes `t * weight[i]` weight and `t * value[i]` value.
Enumerating each copy individually is `O(count)` items; binary grouping turns each item into
`O(log count)` virtual items, each of which is then handled by an ordinary **0/1 knapsack**.
Choosing any subset of the blocks reproduces taking any number `0..count[i]` of copies.

See the Bounded Knapsack section in [knapsack.md](knapsack.md) for the full
knapsack code that consumes these blocks.

```cpp
for (int p = 1; c > 0; p <<= 1) {
    int take = min(p, c);
    int weight = take * x;
    dp |= (dp << weight);
    c -= take;
}
```

The same decomposition helps anywhere you need to represent "any quantity up to `m`" with a
small number of additive building blocks — e.g. reachable-sum DP with multiplicities, or
compressing a multiset of identical elements before running a subset-sum style DP.

## Bitset reachability variant (each shift = one 0/1 item)

For counting/reachability (not optimization), `dp` is a `bitset` where bit `v` means
"sum `v` is reachable." The key mental model:

- `dp << weight` sets bit `v + weight` for every reachable `v` — "take this block once."
- `dp |= (...)` unions it with the old `dp` — "**or** don't take it."

Because `dp << weight` is computed from the *old* `dp` and OR'd in exactly once, a single
statement can never add `weight` twice: **each shift is one 0/1 item of value `take * x`.**
The loop stacks `O(log m)` such 0/1 items whose sizes are `1, 2, 4, …, leftover`, so the
count of copies you land on = the sum of the block-sizes you chose to include — exactly like
writing that number in binary. Every multiplicity in `[0, m]` is therefore reachable.

Concrete example, `freq[x] = 3` so `m = 2*freq[x] = 6` gives blocks `{1, 2, 3}`; their subset
sums hit every count `0..6` (e.g. take 3 copies = block `1 + 2`, take 5 = block `2 + 3`).

### Offset trick for signed sums

A common companion move: when each element may be **added, subtracted, or skipped**
(coefficient `d_i ∈ {-1, 0, +1}`), signed sums are negative and awkward to index into a
bitset. Substitute `c_i = d_i + 1 ∈ {0, 1, 2}` and offset the whole target by `sum`:

```
∑ d_i A_i + sum  =  ∑ c_i A_i  ∈  [0, 2*sum]
```

Now every element is used `0`, `1`, or `2` times, so value `x` (with `freq[x]` copies)
contributes a multiplicity in `[0, 2*freq[x]]` — precisely the bounded-knapsack range the
binary grouping above handles. Reachable offset value `v ≥ sum` corresponds to a nonnegative
signed result, so counting reachable `v` in `[sum, 2*sum]` counts the distinct nonnegative
(equivalently, distinct absolute) outcomes.

```cpp
bitset<MAXN> dp;
dp[0] = 1;
for (int x = 0; x < MAXN; ++x) {
    if (!freq[x]) continue;
    int c = 2 * freq[x];                 // each element usable 0,1,2 times
    for (int p = 1; c > 0; p <<= 1) {    // binary-grouped 0/1 items
        int take = min(p, c);
        dp |= (dp << (take * x));
        c -= take;
    }
}
int ans = 0;
for (int v = sum; v <= 2 * sum; ++v) ans += dp[v];  // nonnegative half
```
