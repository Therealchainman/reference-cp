# Lagrangian Relaxation (Alien's Trick)

Count-based constraint + additive objective + efficiently solvable penalized problem. The query may
fix the count, or it may fix a required value and ask for the minimum count.

Lagrangian relaxation is an optimization technique used to solve complex constrained problems by moving difficult constraints into the objective function with a penalty. It simplifies the problem into an "easy" subproblem that is efficiently solvable, generating tight mathematical bounds for integer or linear programming

You replace "must use exactly K" with "using one more costs λ", then tune λ until the unconstrained optimum behaves like the constrained optimum.


## Make λ always a penalty (max vs min)

Maximizing and minimizing problems are the **same shape**, just for opposite optimization
directions. The key to keeping the binary search on count simple and consistent is to always
make `λ` mean **"how much I punish taking another item."** Do that and `cnt(λ)` is
non-increasing in *both* cases, so the exact same binary search works either way.

The trick:

- **Max problem → subtract λ:** solve `max(rawValue - λ·cnt)`
- **Min problem → add λ:** solve `min(rawCost + λ·cnt)`

In both, larger `λ` makes taking an item less attractive, so the optimal count drops.

### Same monotonicity

| Problem type | Penalized objective       | Meaning of larger `λ`          | `cnt(λ)`       |
| ------------ | ------------------------- | ------------------------------ | -------------- |
| Max DP       | `rawValue - λ·cnt`        | taking items gives less profit | non-increasing |
| Min DP       | `rawCost  + λ·cnt`        | taking items costs more        | non-increasing |

```cpp
λ goes up -> items are less attractive -> cnt goes down
```

So both share the identical binary search. Because `cnt(λ)` is non-increasing, search for the
smallest λ whose count drops to `K` (or below). This is the preferred form — it has a clean
`lo < hi` / `hi = mid` invariant and no off-by-one ambiguity:

```cpp
// cnt(lambda) is non-increasing; find the smallest lambda with cnt <= K.
// hi is an upper bound on the useful penalty (e.g. max marginal cost, here N + 1).
int64 lo = 0, hi = N + 1;
while (lo < hi) {
    int64 mid = lo + (hi - lo) / 2;
    auto [penalizedCost, cnt] = solve_penalty(mid);
    if (cnt > K) {
        lo = mid + 1;   // too many items -> raise the penalty
    } else {
        hi = mid;       // few enough -> this lambda is a candidate
    }
}
// lo is the chosen penalty; re-run solve_penalty(lo) to recover the answer.
```

### Tie-break: always prefer the larger count

So that `cnt(λ)` is well-defined and the recovery hits exactly K, break ties toward more items:

```cpp
// Max: maximize value, on tie prefer larger count
bool better(Res a, Res b) {
    if (a.val != b.val) return a.val > b.val;
    return a.cnt > b.cnt;
}

// Min: minimize cost, on tie prefer larger count
bool better(Res a, Res b) {
    if (a.cost != b.cost) return a.cost < b.cost;
    return a.cnt > b.cnt;
}
```

### The only real difference: the final correction

The DP applied `λ` once per chosen item, so undo it for exactly `K` items at the end —
in the *opposite* direction of how you applied it:

```cpp
// if you SUBTRACTED λ in the DP, ADD it back at the end
answer = penalizedValue + λ * K;   // max problem

// if you ADDED λ in the DP, SUBTRACT it back at the end
answer = penalizedCost  - λ * K;   // min problem
```

### Compact rule

```cpp
// Maximize exactly K items
solve(λ) = max(rawValue - λ·cnt)   // cnt(λ) non-increasing
answer   = solve(λ).val + λ * K

// Minimize exactly K items
solve(λ) = min(rawCost + λ·cnt)    // cnt(λ) non-increasing
answer   = solve(λ).cost - λ * K
```

They are mirror images. Both make `λ` mean "the penalty for taking another item," which is
what keeps the count monotonic and the binary search uniform.


## Value-threshold / inverse-frontier variant

The exact-count form is not the only useful query on the same Lagrangian frontier. Suppose

```text
F(c) = maximum raw value obtainable with c items
```

is non-decreasing and discretely concave. The classic query gives the horizontal coordinate and
asks for the vertical coordinate:

```text
given M, compute F(M).
```

The threshold variant gives a vertical requirement and asks for the first horizontal coordinate:

```text
given K, compute min {c : F(c) >= K}.
```

Examples include the fewest paths needed to collect at least `K` reward, the fewest facilities
needed to provide at least `K` capacity, or the fewest operations needed to achieve a required
benefit. This is still Lagrangian relaxation; it is an **inverse query on the same concave
count-value frontier**.

### The relaxed oracle is unchanged

Put a penalty `λ` on each selected item:

```text
solve(λ) = max_c (F(c) - λ * c).
```

Have the oracle return both fields

```text
res.penalized = F(res.cnt) - λ * res.cnt
res.cnt       = selected count.
```

Recover the raw value of that selected point with

```cpp
raw(res, lambda) = res.penalized + lambda * res.cnt;
```

For this variant, break equal penalized values toward the **smaller count**. This chooses the left
endpoint of every exposed face of the frontier and makes the adjacent-penalty recovery convention
below unambiguous.

As `λ` increases, selected items become more expensive. Both `res.cnt` and the selected raw value
are non-increasing. Unlike exact-count Alien's trick, however, the binary-search predicate uses the
raw value:

```cpp
// Assume lambda = 0 reaches K and max_penalty is a known upper bound.
// Find the largest lambda whose selected frontier point still reaches K.
int64 lo = 0, hi = max_penalty;
while (lo < hi) {
    int64 mid = lo + (hi - lo + 1) / 2;
    Res r = solve_penalty(mid);
    if (raw(r, mid) >= K) {
        lo = mid;
    } else {
        hi = mid - 1;
    }
}
```

The contrast is:

| Query | Known constraint | Unknown answer | Search predicate | Recovery |
| --- | --- | --- | --- | --- |
| Exact count | `c = M` | best value `F(M)` | compare `cnt(λ)` with `M` | `penalized + λ * M` |
| Value threshold | `F(c) >= K` | minimum count `c` | compare selected raw value with `K` | interpolate count |

In geometric terms, exact-count Alien's trick asks for the frontier height at a known `x`; the
threshold form asks where the frontier first crosses a known `y`.

### Why two neighboring penalties are needed

In the exact-count query, the final count `M` is already known, so adding back `λ * M` is enough.
Here the count is the answer, so after the binary search evaluate both `λ = lo` and `λ + 1`:

```text
(c1, v1) from solve(lo),     where v1 >= K
(c2, v2) from solve(lo + 1), where v2 < K.
```

Normally `c2 < c1`. The counts between `c2` and `c1` were skipped because WQS exposes a whole face
of the discrete concave frontier at once. All points on this face have the same marginal value, so
the frontier is exactly linear throughout this block. Therefore the first count reaching `K` is

```cpp
answer = c2 + ceil_div((K - v2) * (c1 - c2), v1 - v2);
```

with integer ceiling division

```cpp
ceil_div(a, b) = (a + b - 1) / b;  // a >= 0, b > 0
```

This is exact discrete interpolation, not an approximation. The formula is often written with the
general ratio above; when every item in the block has a known common marginal gain, it can be
simplified to “how many more equal-gain items are required?”

### Required conditions and edge cases

This version needs the same structural assumptions as ordinary WQS:

- the relaxed problem `max(rawValue - λ * count)` is efficiently solvable;
- the optimal count is monotone in `λ` under a consistent tie-break;
- `F(c)` has the required discrete-concave frontier, so skipped counts lie on exposed linear faces.

Also handle these boundaries explicitly:

- If even the maximum attainable raw value is below `K`, the threshold is impossible.
- Define whether count `0` is allowed in the relaxed oracle; it is often useful to set `F(0) = 0`.
- Use wide enough integers for `(K - v2) * (c1 - c2)`.
- If the problem asks for transitions between chosen objects rather than the number of objects, do
  the final conversion afterward. For example, `p` driving paths require `p - 1` teleports.

### Compact rule

```text
Exact-count WQS:
    input count M -> recover frontier value F(M)

Value-threshold WQS:
    input value K -> recover min count c with F(c) >= K
    search λ by raw(solve(λ)) >= K
    recover c from solve(λ) and solve(λ + 1)
```


## Implementation example

Just note you usually tune λ using binary search, and the unconstrained optimum is often found using a greedy or dynamic programming algorithm.

```cpp
using int64 = long long;
const int64 INF = numeric_limits<int64>::max();
class Solution {
private:
    vector<int64> pref;
    pair<int64, int> solve_penalty(int l, int r, int64 lambda) {
        int N = pref.size() - 1;
        vector<int64> dp(N + 1, -INF);
        vector<int> cnt(N + 1, 0);
        deque<tuple<int, int64, int>> dq;
        for (int i = 1; i <= N; ++i) {
            dp[i] = dp[i - 1];
            cnt[i] = cnt[i - 1];
            int j = i - l;
            if (j >= 0) {
                int64 cand = dp[j];
                int count = cnt[j];
                if (cand < 0) {
                    cand = 0;
                    count = 0;
                }
                cand -= pref[j];
                while (!dq.empty()) {
                    auto [x, y, z] = dq.back();
                    if (make_pair(cand, -count) < make_pair(y, -z)) break;
                    dq.pop_back();
                }
                dq.emplace_back(j, cand, count);
            }
            // remove those that are no longer valid from front of dq
            while (!dq.empty() && get<0>(dq.front()) < i - r) {
                dq.pop_front();
            }
            if (!dq.empty()) { // choose subarray ending at i - 1
                auto [j, ncost, ncount] = dq.front();
                ncost += pref[i] - lambda;
                ncount++;
                if (ncost > dp[i] || (ncost == dp[i] && ncount < cnt[i])) {
                    dp[i] = ncost;
                    cnt[i] = ncount;
                }
            }
        }
        return {dp[N], cnt[N]};
    }
public:
    int64 maximumSum(vector<int>& nums, int m, int l, int r) {
        int N = nums.size();
        pref.assign(N + 1, 0);
        for (int i = 0; i < N; ++i) {
            pref[i + 1] += pref[i] + nums[i];
        }
        int64 lo = 0, hi = 1e18;
        while (lo < hi) {
            int64 mid = lo + (hi - lo) / 2;
            auto [penalizedCost, cnt] = solve_penalty(l, r, mid);
            if (cnt > m) {
                lo = mid + 1;
            } else {
                hi = mid;
            }
        }
        auto [penalizedCost, cnt] = solve_penalty(l, r, lo);
        int64 ans = penalizedCost + 1LL * lo * m;
        return ans;
    }
};
```
