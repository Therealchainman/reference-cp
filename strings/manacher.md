# Manacher's Algorithm

Finds the longest palindrome centered at *every* position of a string in `O(n)` total time.

## The transformed string

The trick is to insert a separator (`#`) between every character (and at both ends). This turns
even-length palindromes into odd-length ones, so a single "odd palindrome" routine handles both cases.

```
s = "abaaba"
t = "#a#b#a#a#b#a#"     (length 2*len(s) + 1 = 13)

index:  0 1 2 3 4 5 6 7 8 9 10 11 12
   t:   # a # b # a # a # b  #  a  #
```

- **Odd indices of `t`** (`1,3,5,…`) are the real characters — they are the centers of **odd-length** palindromes in `s`.
- **Even indices of `t`** (`0,2,4,…`) are the `#` separators — they are the centers of **even-length** palindromes in `s`.

## What `manacher` returns

`manacher(s)` returns an array `parr` of length `len(t) = 2*len(s) + 1`, one entry per index of `t`.

`parr[i]` is the **radius** of the longest palindrome in `t` centered at index `i` (the number of positions
it reaches out on each side, including the center). The key fact you use in practice:

> The longest palindrome of `s` centered at position `i` of `t` has length **`parr[i] - 1`**.

### Worked example

For `s = "abaaba"`:

```
index:   0  1  2  3  4  5  6  7  8  9 10 11 12
   t:    #  a  #  b  #  a  #  a  #  b  #  a  #
parr:    1  2  1  4  1  2  7  2  1  4  1  2  1
```

Reading a few entries (`length in s = parr[i] - 1`):

| `i` | `t[i]` | `parr[i]` | palindrome length in `s` | the palindrome |
|-----|--------|-----------|--------------------------|----------------|
| 1   | `a`    | 2         | 1                        | `a`            |
| 3   | `b`    | 4         | 3                        | `aba`          |
| 6   | `#`    | 7         | 6                        | `abaaba` (whole string) |
| 9   | `b`    | 4         | 3                        | `aba`          |

The largest value, `parr[6] = 7`, sits on the `#` between the two middle characters — the center of the
even-length palindrome `abaaba`.

```py
def manacher(s):
    t = "#".join(s)
    t = "#" + t + "#"
    res = manacher_odd(t)
    return res

def manacher_odd(s):
    n = len(s)
    s = "$" + s + "^"          # sentinels so the while-loop never runs off the ends
    p = [0] * (n + 2)
    l, r = 1, 1
    for i in range(1, n + 1):
        p[i] = max(0, min(r - i, p[l + (r - i)]))
        while s[i - p[i]] == s[i + p[i]]:
            p[i] += 1
        if i + p[i] > r:
            l, r = i - p[i], i + p[i]
    return p[1:-1]

parr = manacher(s)            # length 2*len(s) + 1
```

### Optional: per-character half-lengths

If instead you want, for each center, how far the palindrome extends to one side, post-process `parr`:

```py
for i in range(len(parr)):
    if i % 2 == 0:
        parr[i] -= 1
    parr[i] //= 2
```

For `s = "abaaba"` this yields `[0, 1, 0, 2, 0, 1, 3, 1, 0, 2, 0, 1, 0]`. Here odd indices give the
half-length of the longest odd palindrome centered on that character, and even indices give the half-length
of the longest even palindrome centered between characters.

```cpp
vector<int> manacher(const string& s) {
    string t = "#";
    for (char ch : s) {
        t += ch;
        t += "#";
    }
    vector<int> parr = manacher_odd(t);
    return parr;
}
vector<int> manacher_odd(string& s) {
    int N = s.size();
    s = "$" + s + "^";
    vector<int> P(N + 2, 0);
    int l = 1, r = 1;
    for (int i = 1; i <= N; i++) {
        P[i] = max(0, min(r - i, P[l + (r - i)]));
        while (s[i - P[i]] == s[i + P[i]]) {
            P[i]++;
        }
        if (i + P[i] > r) {
            l = i - P[i];
            r = i + P[i];
        }
    }
    return vector<int>(P.begin() + 1, P.end() - 1);
}
```

## Manacher for static range queries

Once you have `parr`, you can answer in `O(1)` whether **any** substring `s[l..r-1]` is a palindrome.

The query is over the half-open range `[l, r)`, so the substring has length `r - l`.

**Why it works:** the center of `s[l..r-1]` lands at index `l + r` in `t`. The substring is a palindrome
exactly when the longest palindrome centered there is at least as long as the substring, i.e.
`parr[l + r] - 1 >= r - l`, which simplifies to `parr[l + r] > r - l`.

Using `parr = [1,2,1,4,1,2,7,2,1,4,1,2,1]` for `"abaaba"`:

- `query(0, 3)` → `s[0:3] = "aba"`, center `l+r = 3`, `parr[3] = 4 > 3` → **true**
- `query(2, 4)` → `s[2:4] = "aa"`, center `l+r = 6`, `parr[6] = 7 > 2` → **true**
- `query(0, 4)` → `s[0:4] = "abaa"`, center `l+r = 4`, `parr[4] = 1 > 4` → **false**
- `query(0, 6)` → whole `"abaaba"`, center `l+r = 6`, `parr[6] = 7 > 6` → **true**

```cpp
vector<int> marr;

vector<int> manacher_odd(string& s) {
    int N = s.size();
    s = "$" + s + "^";
    vector<int> P(N + 2, 0);
    int l = 1, r = 1;
    for (int i = 1; i <= N; i++) {
        P[i] = max(0, min(r - i, P[l + (r - i)]));
        while (s[i - P[i]] == s[i + P[i]]) {
            P[i]++;
        }
        if (i + P[i] > r) {
            l = i - P[i];
            r = i + P[i];
        }
    }
    return vector<int>(P.begin() + 1, P.end() - 1);
}
vector<int> manacher(const string& s) {
    string t = "#";
    for (char ch : s) {
        t += ch;
        t += "#";
    }
    vector<int> parr = manacher_odd(t);
    return parr;
}
// palindrome check over the half-open range [l, r)
bool query(int l, int r) {
    return marr[l + r] > r - l;
}
```

Build the array once (here `t` is the string you want to query):

```cpp
marr = manacher(t);
```

## Manacher on `vector<int>`

When the input is a sequence of integers instead of a string, the algorithm is identical — only the
separator and boundary characters change from `char` to reserved `int` values. Everything else
(the returned array, the `parr[i] - 1` length rule, and the `query(l, r)` function) behaves exactly as above.

The separator and the two boundary guards must be values that **never appear in the input**. Below they
use the bottom of the `int` range; if your input can contain those values, pick other reserved values or
offset the input first.

```cpp
const int SEP   = INT_MIN;      // separator between elements  (the "#")
const int LEFT  = INT_MIN + 1;  // left boundary guard         (the "$")
const int RIGHT = INT_MIN + 2;  // right boundary guard        (the "^")

vector<int> manacher_odd(vector<int> s) {
    int N = s.size();
    s.insert(s.begin(), LEFT);
    s.push_back(RIGHT);
    vector<int> P(N + 2, 0);
    int l = 1, r = 1;
    for (int i = 1; i <= N; i++) {
        P[i] = max(0, min(r - i, P[l + (r - i)]));
        while (s[i - P[i]] == s[i + P[i]]) {
            P[i]++;
        }
        if (i + P[i] > r) {
            l = i - P[i];
            r = i + P[i];
        }
    }
    return vector<int>(P.begin() + 1, P.end() - 1);
}
vector<int> manacher(const vector<int>& s) {
    vector<int> t;
    t.push_back(SEP);
    for (int x : s) {
        t.push_back(x);
        t.push_back(SEP);
    }
    return manacher_odd(t);
}
```

For example, encoding `"abaaba"` as `{1, 2, 1, 1, 2, 1}` returns the same
`parr = [1, 2, 1, 4, 1, 2, 7, 2, 1, 4, 1, 2, 1]`, so `parr[i] - 1` is again the longest palindrome length
centered at index `i` of the transformed sequence.
