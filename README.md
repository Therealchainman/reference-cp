# reference-cp

A structured, topic-organized **reference library** of algorithms, data structures, and
mathematics for competitive programming.

Each entry is a canonical explainer — *"this is how X works"* — usually written in Markdown
with C++ (and some legacy Python) implementations. These notes are meant to be stable and
edited in place as understanding improves, not appended chronologically.

## How this fits with my other repos

- **[archive-cp](../archive-cp)** — solutions to actual contest problems (AtCoder, Codeforces, CSES, etc.).
- **reference-cp** (this repo) — *canonical reference* for the techniques those solutions use.
- **[brain-cp](../brain-cp)** — *personal learnings*: free-form intuition, patterns, and "aha"
  moments accumulated while solving problems.

If a note is impersonal and reusable, it belongs here. If it's a personal insight or
cross-cutting pattern you discovered, it belongs in `brain-cp`.

## Layout

Topics are grouped into top-level directories:

| Directory | Contents |
|-----------|----------|
| `bitwise_and_subsets/` | Bit manipulation, submask enumeration, SOS DP, XOR basis |
| `combinatorics/` | Binomial coefficients, inclusion-exclusion, generating functions, Catalan numbers |
| `data_structures/` | Union-find, treaps, sparse tables, ordered/sorted sets, caches |
| `dynamic_programming/` | Knapsack, LIS, bitmask/digit/subset DP, Lagrangian relaxation |
| `game_theory/` | Sprague-Grundy and related |
| `geometry/` | Convex hull, prefix sums, transformations, Pick's theorem |
| `graph_theory/` | Flows, matching, bridges, centroid decomposition, segment/Fenwick trees |
| `mathematics/` | Matrix exponentiation, polynomials, series, fractions |
| `misc/` | FFT/FWHT, binary lifting, 2-SAT, coordinate compression, and more |
| `number_theory/` | Modular arithmetic, sieves, factorization, Lucas/Fermat theorems |
| `order_theory/` | Dilworth's theorem and related |
| `probability/` | Expected value, random sampling |
| `scripts/` | Binary search/exponentiation, sorting, ternary search, quickselect |
| `strings/` | KMP, Z-algorithm, suffix arrays, hashing, Aho-Corasick, Manacher |
| `languages/` | Language-specific reference: `cpp/` (custom comparators, bitset, overflow, reading inputs, ...) and `python/` |
| `images/` | Supporting diagrams referenced by the notes |

## Compiling C++ examples

```sh
g++ -std=c++20 main.cpp -o main
```
