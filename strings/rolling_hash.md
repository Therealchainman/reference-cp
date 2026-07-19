# Polynomial Rolling Hash

## Rolling hash 

Sometimes you need different M value, and P needs to be adjusted so there are no collisions.  
Sometimes you don't have the test cases offline, and it may be hard to avoid collisions, you can always try a double hash
to reduce the chance of a collision.

```cpp
const int C = 4, B = 95, P = 96;
int M = pow(B, C);

int polynomial_hash(const string &s) {
    int h = 0;
    for (char c : s) {
        h = (h * P + c) % M;
    }
    return h;
}
```

Check for collisions in part 1:

```cpp
// try some p until it works. 
int polynomial_hash(const string &s, int p) {
    int h = 0;
    for (char c : s) {
        h = (h * p + c) % M;
    }
    return h;
}

void solve() {
    while (getline(cin, text)) titles.push_back(text);
    for (int p = 97; p < 3'000; p++) {
        bool ok = true;
        set<int> hashes;
        for (const string &text : titles) {
            int h = polynomial_hash(text, p);
            if (hashes.find(h) != hashes.end()) { // found collision
                ok = false;
                break;
            }
            hashes.insert(h);
        }
        if (ok) {
            cout << p << endl;
            break;
        }
    }
}
```

## Rolling hash to find pattern in string

This works for when you have only lowercase english letters, 
This is an implementation of a rolling hash to find pattern in 
string

This can lead to collision sometimes, so it is not 100% guaranteed to work

```py
def find_pattern_rolling_hash(string: str, pattern: str) -> int:
    p, MOD = 31, int(1e9)+7
    coefficient = lambda x: ord(x) - ord('a') + 1
    pat_hash = 0
    for ch in pattern:
        pat_hash = ((pat_hash*p)%MOD + coefficient(ch))%MOD
    POW = 1
    for _ in range(len(pattern)-1):
        POW = (POW*p)%MOD
    cur_hash = 0
    for i, ch in enumerate(string):
        cur_hash = ((cur_hash*p)%MOD + coefficient(ch))%MOD
        if i>=len(pattern)-1:
            if cur_hash == pat_hash: return i-len(pattern)+1
            cur_hash = (cur_hash - (POW*coefficient(string[i-len(pattern)+1]))%MOD + MOD)%MOD
    return -1
```

## Reusable rolling hash (tunable single hash)

A general-purpose version of the pattern-finding rolling hash above.  Precomputes prefix hashes and powers once in O(n), then answers the hash of any substring s[l:r) in O(1), so the same object works for pattern matching, comparing substrings, palindrome checks (build one on the reversed string), binary searching the longest common prefix, counting distinct substrings, etc.

**Specialized for lowercase english letters**: the coefficient maps 'a'..'z' to 1..26.  The mapping must start at 1, not 0, otherwise leading 'a's are invisible to the hash ("a", "aa", "aaa" would all hash to 0).  For a different alphabet, change `coefficient` (and make sure `p` stays larger than the alphabet size).

Tunable parameters:
- `p`: the base, a prime larger than the alphabet size, e.g. 31 or 53 for lowercase letters.
- `mod`: the modulus.  Default is the Mersenne prime 2^61 - 1, which makes collisions far less likely than 1e9+7.  Intermediate products overflow 64 bits, so `mulmod` goes through `__int128`; that also means any `mod` up to ~2^62 is safe to plug in.

This is a single hash, so collisions are still possible, just unlikely.  If it fails, switch to a double hash.

```cpp
struct RollingHash {
    long long p, mod;
    vector<long long> pref, pw; // pref[i] = hash of s[0, i), pw[i] = p^i % mod
    RollingHash(const string &s, long long p = 31, long long mod = (1LL << 61) - 1) : p(p), mod(mod) {
        int n = s.size();
        pref.assign(n + 1, 0);
        pw.assign(n + 1, 1);
        for (int i = 0; i < n; i++) {
            pref[i + 1] = (mulmod(pref[i], p) + coefficient(s[i])) % mod;
            pw[i + 1] = mulmod(pw[i], p);
        }
    }
    long long mulmod(long long a, long long b) const {
        return (long long)((__int128)a * b % mod);
    }
    long long coefficient(char c) const { // lowercase english letters -> 1..26
        return c - 'a' + 1;
    }
    long long query(int l, int r) const { // hash of s[l, r), 0-indexed, r exclusive, O(1)
        long long h = (pref[r] - mulmod(pref[l], pw[r - l])) % mod;
        return h < 0 ? h + mod : h;
    }
    long long hashOf(const string &t) const { // hash of a whole other string (e.g. a pattern), comparable with query
        long long h = 0;
        for (char c : t) h = (mulmod(h, p) + coefficient(c)) % mod;
        return h;
    }
};
```

Example, find first occurrence of pattern in string:

```cpp
int findPattern(const string &s, const string &pattern) {
    RollingHash rh(s);
    long long patHash = rh.hashOf(pattern);
    int m = pattern.size();
    for (int i = 0; i + m <= (int)s.size(); i++) {
        if (rh.query(i, i + m) == patHash) return i;
    }
    return -1;
}
```

Example, check if s[l1, r1) == s[l2, r2) in O(1):

```cpp
RollingHash rh(s);
bool equal = r1 - l1 == r2 - l2 && rh.query(l1, r1) == rh.query(l2, r2);
```

## Double hash

if rolling hash fails, you could get accepted with a double hash, here is an example of it used in a problem. 

### Reusable double hash (tunable)

Drop-in companion to the `RollingHash` class above (requires it): runs two independent single hashes and returns their results as a tuple, so two substrings are considered equal only if both hashes match.  Same O(n) build / O(1) `query(l, r)` interface, and inherits the same **lowercase english letters** specialization from `RollingHash.coefficient` (1..26); change that method for other alphabets.

Tunable parameters: two bases and two moduli, defaults chosen to be independent (different primes for both).  Use this when the single hash gets hacked or collides; the chance both hashes collide simultaneously is negligible.

```cpp
struct DoubleRollingHash {
    RollingHash rh1, rh2;
    DoubleRollingHash(const string &s, long long p1 = 31, long long p2 = 53,
                      long long mod1 = 1e9 + 7, long long mod2 = 1e9 + 9)
        : rh1(s, p1, mod1), rh2(s, p2, mod2) {}
    pair<long long, long long> query(int l, int r) const { // hashes of s[l, r), 0-indexed, r exclusive, O(1)
        return {rh1.query(l, r), rh2.query(l, r)};
    }
    pair<long long, long long> hashOf(const string &t) const { // hashes of a whole other string, comparable with query
        return {rh1.hashOf(t), rh2.hashOf(t)};
    }
};
```

Usage is identical to `RollingHash`, the hashes are just pairs now.  They work directly as keys in `map`/`set`; for `unordered_map`/`unordered_set` combine them into one value first, e.g. `h.first * (mod2) + h.second`, since `std::pair` has no default hasher.

```cpp
int findPattern(const string &s, const string &pattern) {
    DoubleRollingHash rh(s);
    auto patHash = rh.hashOf(pattern);
    int m = pattern.size();
    for (int i = 0; i + m <= (int)s.size(); i++) {
        if (rh.query(i, i + m) == patHash) return i;
    }
    return -1;
}
```

### Example from a problem

```py
p, MOD1, MOD2 = 31, int(1e9) + 7, int(1e9) + 9
coefficient = lambda x: ord(x) - ord('a') + 1
shashes = {}
add = lambda h, mod, ch: ((h * p) % mod + coefficient(ch)) % mod
n = len(wordsContainer)
for i in reversed(range(n)):
    word = wordsContainer[i]
    hash1 = hash2 = 0
    if len(word) <= shashes.get((hash1, hash2), (0, math.inf))[1]:
        shashes[(hash1, hash2)] = (i, len(word))
    for ch in reversed(word):
        hash1 = add(hash1, MOD1, ch)
        hash2 = add(hash2, MOD2, ch)
        if len(word) <= shashes.get((hash1, hash2), (0, math.inf))[1]:
            shashes[(hash1, hash2)] = (i, len(word))
```

## Rolling Hash when you have -1 in array

Example of very similar rolling hash implementation but for an array containing [-1, 0, 1] elements.  So you encode the coefficient by adding 2,  cause you can't have a 0 I believe. 

```py
n, m = len(nums), len(pattern)
p, MOD = 31, int(1e9)+7
coefficient = lambda x: x + 2
pat_hash = 0
for v in pattern:
    pat_hash = (pat_hash * p + coefficient(v)) % MOD
diff = [0] * (n - 1)
for i in range(n - 1):
    if nums[i + 1] > nums[i]: diff[i] = 1
    elif nums[i + 1] < nums[i]: diff[i] = -1
POW = 1
for _ in range(m - 1):
    POW = (POW * p) % MOD
ans = cur_hash = 0
for i, v in enumerate(diff):
    cur_hash = (cur_hash * p + coefficient(v)) % MOD
    if i >= m - 1:
        if cur_hash == pat_hash: ans += 1
        cur_hash = (cur_hash - coefficient(diff[i - m + 1]) * POW) % MOD
return ans
```

## Rolling Hash on Segment Tree

using segment tree with queries of [l, r)

```py
class SegmentTree:
    def __init__(self, n, neutral, func, initial_arr):
        self.func = func
        self.neutral = neutral
        self.size = 1
        self.n = n
        while self.size<n:
            self.size*=2
        self.nodes = [neutral for _ in range(self.size*2)] 
        self.build(initial_arr)

    def build(self, initial_arr):
        for i, segment_idx in enumerate(range(self.n)):
            segment_idx += self.size - 1
            val = initial_arr[i]
            self.nodes[segment_idx] = val
            self.ascend(segment_idx)

    def ascend(self, segment_idx):
        while segment_idx > 0:
            segment_idx -= 1
            segment_idx >>= 1
            left_segment_idx, right_segment_idx = 2*segment_idx + 1, 2*segment_idx + 2
            self.nodes[segment_idx] = self.func(self.nodes[left_segment_idx], self.nodes[right_segment_idx])
        
    def update(self, segment_idx, val):
        segment_idx += self.size - 1
        self.nodes[segment_idx] = val
        self.ascend(segment_idx)
            
    def query(self, left, right):
        stack = [(0, self.size, 0)]
        result = self.neutral
        while stack:
            # BOUNDS FOR CURRENT INTERVAL and idx for tree
            segment_left_bound, segment_right_bound, segment_idx = stack.pop()
            # NO OVERLAP
            if segment_left_bound >= right or segment_right_bound <= left: continue
            # COMPLETE OVERLAP
            if segment_left_bound >= left and segment_right_bound <= right:
                result = self.func(result, self.nodes[segment_idx])
                continue
            # PARTIAL OVERLAP
            mid_point = (segment_left_bound + segment_right_bound) >> 1
            left_segment_idx, right_segment_idx = 2*segment_idx + 1, 2*segment_idx + 2
            stack.extend([(mid_point, segment_right_bound, right_segment_idx), (segment_left_bound, mid_point, left_segment_idx)])
        return result
    
    def __repr__(self):
        return f"nodes array: {self.nodes}, next array: {self.nodes}"

import random
import math    

mod = 2**61 - 1
base0 = 3

while True:
    k = random.randint(1, mod - 1)
    base = pow(base0, k, mod)
    if base <= ord("z"): continue
    if math.gcd(base, mod - 1) != 1: continue
    break

def main():
    N, Q = map(int, input().split())
    S = input()
    pw = [1] * (N + 1)
    for i in range(N):
        pw[i + 1] = (pw[i] * base) % mod
    segfunc = lambda a, b: ((a[0] + (b[0] * pw[a[1]]) % mod) % mod, a[1] + b[1])
    seg = SegmentTree(N, (0, 0), segfunc, [(ord(ch), 1) for ch in S])
    seg_rev = SegmentTree(N, (0, 0), segfunc, [(ord(ch), 1) for ch in reversed(S)])
    for _ in range(Q):
        t, l, r = input().split()
        if t == "1":
            x, c = int(l) - 1, ord(r)
            seg.update(x, (c, 1))
            seg_rev.update(N - x - 1, (c, 1))
        else:
            l, r = int(l) - 1, int(r) - 1
            h = seg.query(l, r + 1)
            l_rev, r_rev = N - r - 1, N - l - 1
            h_rev = seg_rev.query(l_rev, r_rev + 1)
            if h == h_rev: print("Yes")
            else: print("No")
```

```cpp

```