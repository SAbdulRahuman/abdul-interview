# Phase 4: Hash Tables — Hash Sets

## What is a Hash Set?

A **hash set** is an unordered collection of **unique** elements backed by a hash table. It provides:

- **O(1)** average insert, delete, lookup
- **No duplicates** — adding an existing element is a no-op
- **No key-value pairs** — only keys (membership)

In Go, sets are typically implemented using `map[T]struct{}` (zero-cost value).

---

## Example 1: Basic Hash Set with map[string]struct{}

```go
package main

import "fmt"

type StringSet map[string]struct{}

func NewStringSet() StringSet {
    return make(StringSet)
}

func (s StringSet) Add(val string) {
    s[val] = struct{}{}
}

func (s StringSet) Contains(val string) bool {
    _, ok := s[val]
    return ok
}

func (s StringSet) Remove(val string) {
    delete(s, val)
}

func (s StringSet) Size() int {
    return len(s)
}

func (s StringSet) Values() []string {
    result := make([]string, 0, len(s))
    for k := range s {
        result = append(result, k)
    }
    return result
}

func main() {
    s := NewStringSet()
    s.Add("go")
    s.Add("rust")
    s.Add("go") // duplicate, ignored

    fmt.Println("Contains 'go':", s.Contains("go"))
    fmt.Println("Contains 'java':", s.Contains("java"))
    fmt.Println("Size:", s.Size())

    s.Remove("rust")
    fmt.Println("After remove 'rust':", s.Values())
}
```

**Textual Figure — Basic Hash Set Operations:**

```
  Operation          │ Set State (map[string]struct{})    │ Notes
 ════════════════════╪════════════════════════════════════╪══════════════════════════
  NewStringSet()     │ { }                                │ empty set created
  Add("go")          │ { "go" }                           │ "go" inserted
  Add("rust")        │ { "go", "rust" }                   │ "rust" inserted
  Add("go")          │ { "go", "rust" }                   │ duplicate → no-op
                     │                                    │
  Contains("go")     │         ┌───────────────┐          │
                     │         │  "go"  → ✓    │          │ returns true
                     │         └───────────────┘          │
  Contains("java")   │         ┌───────────────┐          │
                     │         │  "java" → ✗   │          │ returns false
                     │         └───────────────┘          │
  Size()             │ len = 2                             │ returns 2
                     │                                    │
  Remove("rust")     │ { "go" }                           │ "rust" deleted
                     │                                    │
  Values()           │ ["go"]                             │ remaining elements

  Internal map layout (map[string]struct{}):
  ┌────────────┬──────────────┐
  │    Key     │    Value     │
  ├────────────┼──────────────┤
  │   "go"     │  struct{}{}  │   ← zero-size value
  └────────────┴──────────────┘
```

---

## Example 2: Set Operations — Union, Intersection, Difference

```go
package main

import "fmt"

type Set map[int]struct{}

func NewSet(vals ...int) Set {
    s := make(Set)
    for _, v := range vals {
        s[v] = struct{}{}
    }
    return s
}

func (s Set) Union(other Set) Set {
    result := NewSet()
    for k := range s {
        result[k] = struct{}{}
    }
    for k := range other {
        result[k] = struct{}{}
    }
    return result
}

func (s Set) Intersection(other Set) Set {
    result := NewSet()
    // Iterate over smaller set for efficiency
    small, big := s, other
    if len(small) > len(big) {
        small, big = big, small
    }
    for k := range small {
        if _, ok := big[k]; ok {
            result[k] = struct{}{}
        }
    }
    return result
}

func (s Set) Difference(other Set) Set {
    result := NewSet()
    for k := range s {
        if _, ok := other[k]; !ok {
            result[k] = struct{}{}
        }
    }
    return result
}

func (s Set) SymmetricDifference(other Set) Set {
    result := NewSet()
    for k := range s {
        if _, ok := other[k]; !ok {
            result[k] = struct{}{}
        }
    }
    for k := range other {
        if _, ok := s[k]; !ok {
            result[k] = struct{}{}
        }
    }
    return result
}

func (s Set) Slice() []int {
    r := make([]int, 0, len(s))
    for k := range s {
        r = append(r, k)
    }
    return r
}

func main() {
    a := NewSet(1, 2, 3, 4, 5)
    b := NewSet(4, 5, 6, 7, 8)

    fmt.Println("A ∪ B:", a.Union(b).Slice())
    fmt.Println("A ∩ B:", a.Intersection(b).Slice())
    fmt.Println("A - B:", a.Difference(b).Slice())
    fmt.Println("A △ B:", a.SymmetricDifference(b).Slice())
}
```

**Textual Figure — Set Operations on A and B:**

```
  A = {1, 2, 3, 4, 5}          B = {4, 5, 6, 7, 8}

          ┌─────────────────────────────────────┐
          │           Venn Diagram               │
          │                                     │
          │    ┌─────────┐   ┌─────────┐        │
          │    │ A only  │   │ B only  │        │
          │    │ 1 2 3   │   │ 6 7 8   │        │
          │    │    ┌────┼───┼────┐    │        │
          │    │    │ A∩B│   │    │    │        │
          │    │    │4  5│   │    │    │        │
          │    └────┼────┘   └────┼────┘        │
          │         └─────────────┘              │
          └─────────────────────────────────────┘

  ┌──────────────────────┬──────────────────────────────┐
  │ Operation            │ Result                       │
  ├──────────────────────┼──────────────────────────────┤
  │ A ∪ B  (Union)       │ {1, 2, 3, 4, 5, 6, 7, 8}    │
  │ A ∩ B  (Intersect)   │ {4, 5}                       │
  │ A − B  (Difference)  │ {1, 2, 3}                    │
  │ A △ B  (Sym. Diff.)  │ {1, 2, 3, 6, 7, 8}           │
  └──────────────────────┴──────────────────────────────┘

  Intersection optimization: iterate smaller set
    len(A)=5, len(B)=5 → pick either
    for k in small: if k in big → keep
      4 ∈ B? ✓   5 ∈ B? ✓   → result = {4, 5}
```

---

## Example 3: Remove Duplicates from Slice

```go
package main

import "fmt"

func removeDuplicates(arr []int) []int {
    seen := make(map[int]struct{})
    result := make([]int, 0, len(arr))
    for _, v := range arr {
        if _, ok := seen[v]; !ok {
            seen[v] = struct{}{}
            result = append(result, v)
        }
    }
    return result
}

func main() {
    input := []int{4, 2, 7, 2, 4, 8, 7, 1, 4, 8}
    fmt.Println("Input: ", input)
    fmt.Println("Unique:", removeDuplicates(input))
    // Preserves order of first appearance
}
```

**Textual Figure — Remove Duplicates Step-by-Step:**

```
  Input: [4, 2, 7, 2, 4, 8, 7, 1, 4, 8]

  Step │ Element │ In seen? │ seen (after)             │ result (after)
  ═════╪═════════╪══════════╪══════════════════════════╪══════════════════════
    0  │    4    │   No     │ {4}                      │ [4]
    1  │    2    │   No     │ {4,2}                    │ [4,2]
    2  │    7    │   No     │ {4,2,7}                  │ [4,2,7]
    3  │    2    │   Yes ✗  │ {4,2,7}                  │ [4,2,7]        ← skip
    4  │    4    │   Yes ✗  │ {4,2,7}                  │ [4,2,7]        ← skip
    5  │    8    │   No     │ {4,2,7,8}                │ [4,2,7,8]
    6  │    7    │   Yes ✗  │ {4,2,7,8}                │ [4,2,7,8]      ← skip
    7  │    1    │   No     │ {4,2,7,8,1}              │ [4,2,7,8,1]
    8  │    4    │   Yes ✗  │ {4,2,7,8,1}              │ [4,2,7,8,1]    ← skip
    9  │    8    │   Yes ✗  │ {4,2,7,8,1}              │ [4,2,7,8,1]    ← skip

  Output: [4, 2, 7, 8, 1]    (order of first appearance preserved)

  Before: [4, 2, 7, 2, 4, 8, 7, 1, 4, 8]   len=10
           ─  ─  ─  ×  ×  ─  ×  ─  ×  ×
  After:  [4, 2, 7, 8, 1]                   len=5
```

---

## Example 4: Two-Sum Using a Set

```go
package main

import "fmt"

func twoSumExists(nums []int, target int) (int, int, bool) {
    seen := make(map[int]int) // value → index
    for i, n := range nums {
        complement := target - n
        if j, ok := seen[complement]; ok {
            return j, i, true
        }
        seen[n] = i
    }
    return 0, 0, false
}

func hasPairWithSum(nums []int, target int) bool {
    seen := make(map[int]struct{})
    for _, n := range nums {
        if _, ok := seen[target-n]; ok {
            return true
        }
        seen[n] = struct{}{}
    }
    return false
}

func main() {
    nums := []int{2, 7, 11, 15}
    target := 9
    i, j, found := twoSumExists(nums, target)
    fmt.Printf("Two-sum(%d): indices (%d,%d), found=%v\n", target, i, j, found)

    fmt.Println("Has pair summing to 26:", hasPairWithSum(nums, 26))
    fmt.Println("Has pair summing to 50:", hasPairWithSum(nums, 50))
}
```

**Textual Figure — Two-Sum Lookup with Hash Set:**

```
  nums = [2, 7, 11, 15],  target = 9

  twoSumExists (returns indices):
  ┌──────┬──────┬────────────┬────────────────┬─────────────────────┐
  │  i   │ n    │ complement │ seen map       │ Action              │
  │      │      │ (target-n) │ (val → idx)    │                     │
  ├──────┼──────┼────────────┼────────────────┼─────────────────────┤
  │  0   │  2   │  9-2 = 7   │ {}             │ 7 not in seen       │
  │      │      │            │ {2:0}          │ add 2→0             │
  ├──────┼──────┼────────────┼────────────────┼─────────────────────┤
  │  1   │  7   │  9-7 = 2   │ {2:0}          │ 2 in seen! j=0  ✓  │
  │      │      │            │                │ return (0, 1, true) │
  └──────┴──────┴────────────┴────────────────┴─────────────────────┘

  hasPairWithSum (target=26):
  ┌──────┬──────┬────────────┬────────────────┬────────────────────┐
  │  i   │  n   │  26 - n    │ seen set       │ Action             │
  ├──────┼──────┼────────────┼────────────────┼────────────────────┤
  │  0   │  2   │    24      │ {}             │ 24 ∉ seen → add 2  │
  │  1   │  7   │    19      │ {2}            │ 19 ∉ seen → add 7  │
  │  2   │ 11   │    15      │ {2,7}          │ 15 ∉ seen → add 11 │
  │  3   │ 15   │    11      │ {2,7,11}       │ 11 ∈ seen → true ✓ │
  └──────┴──────┴────────────┴────────────────┴────────────────────┘
       11 + 15 = 26  ✓
```

---

## Example 5: Subset and Superset Check

```go
package main

import "fmt"

type Set map[int]struct{}

func NewSet(vals ...int) Set {
    s := make(Set)
    for _, v := range vals {
        s[v] = struct{}{}
    }
    return s
}

func (s Set) IsSubsetOf(other Set) bool {
    for k := range s {
        if _, ok := other[k]; !ok {
            return false
        }
    }
    return true
}

func (s Set) IsSupersetOf(other Set) bool {
    return other.IsSubsetOf(s)
}

func (s Set) Equals(other Set) bool {
    return len(s) == len(other) && s.IsSubsetOf(other)
}

func (s Set) IsDisjoint(other Set) bool {
    small, big := s, other
    if len(small) > len(big) {
        small, big = big, small
    }
    for k := range small {
        if _, ok := big[k]; ok {
            return false
        }
    }
    return true
}

func main() {
    a := NewSet(1, 2, 3, 4, 5)
    b := NewSet(2, 3, 4)
    c := NewSet(1, 2, 3, 4, 5)
    d := NewSet(10, 20)

    fmt.Println("B ⊂ A:", b.IsSubsetOf(a))   // true
    fmt.Println("A ⊃ B:", a.IsSupersetOf(b))  // true
    fmt.Println("A == C:", a.Equals(c))         // true
    fmt.Println("A ∩ D = ∅:", a.IsDisjoint(d)) // true
}
```

**Textual Figure — Subset, Superset, Equality, and Disjoint Checks:**

```
  A = {1, 2, 3, 4, 5}    B = {2, 3, 4}    C = {1, 2, 3, 4, 5}    D = {10, 20}

  ┌─────────────────────────────────────────────────────────────────────┐
  │  B ⊂ A (IsSubsetOf)?                                              │
  │                                                                    │
  │    A: [ 1   2   3   4   5 ]                                        │
  │    B: [     2   3   4     ]                                        │
  │              ✓   ✓   ✓        All B elements found in A → true     │
  ├─────────────────────────────────────────────────────────────────────┤
  │  A ⊃ B (IsSupersetOf)?                                            │
  │                                                                    │
  │    B.IsSubsetOf(A) → true     A contains all of B → true          │
  ├─────────────────────────────────────────────────────────────────────┤
  │  A == C (Equals)?                                                  │
  │                                                                    │
  │    A: {1, 2, 3, 4, 5}                                              │
  │    C: {1, 2, 3, 4, 5}                                              │
  │    len(A)==len(C)==5  AND  A ⊂ C → true                            │
  ├─────────────────────────────────────────────────────────────────────┤
  │  A ∩ D = ∅ (IsDisjoint)?                                           │
  │                                                                    │
  │    A: {1, 2, 3, 4, 5}                                              │
  │    D: {10, 20}                                                     │
  │    Iterate smaller (D):  10 ∈ A? No   20 ∈ A? No  → true          │
  └─────────────────────────────────────────────────────────────────────┘

  Relationship diagram:
       ┌─────────────────┐
       │   A = C          │
       │  ┌───────────┐   │
       │  │     B     │   │          D = {10, 20}
       │  │  {2,3,4}  │   │          completely separate
       │  └───────────┘   │              (disjoint)
       │  {1, 2, 3, 4, 5} │
       └─────────────────┘
```

---

## Example 6: Find First Duplicate

```go
package main

import "fmt"

func firstDuplicate(arr []int) (int, int) {
    seen := make(map[int]int) // value → first index
    for i, v := range arr {
        if firstIdx, ok := seen[v]; ok {
            return v, firstIdx
        }
        seen[v] = i
    }
    return -1, -1
}

func countDistinct(arr []int) int {
    s := make(map[int]struct{})
    for _, v := range arr {
        s[v] = struct{}{}
    }
    return len(s)
}

func main() {
    arr := []int{3, 5, 1, 4, 2, 5, 7, 1}
    val, idx := firstDuplicate(arr)
    fmt.Printf("First duplicate: %d (first seen at index %d)\n", val, idx)
    fmt.Println("Distinct count:", countDistinct(arr))
}
```

**Textual Figure — First Duplicate Detection and Distinct Count:**

```
  arr = [3, 5, 1, 4, 2, 5, 7, 1]

  firstDuplicate trace (seen = map[int]int, value → first index):
  ┌─────┬─────────┬──────────┬───────────────────────────────┐
  │  i  │  arr[i] │ In seen? │ seen (after)                  │
  ├─────┼─────────┼──────────┼───────────────────────────────┤
  │  0  │    3    │   No     │ {3:0}                         │
  │  1  │    5    │   No     │ {3:0, 5:1}                    │
  │  2  │    1    │   No     │ {3:0, 5:1, 1:2}               │
  │  3  │    4    │   No     │ {3:0, 5:1, 1:2, 4:3}          │
  │  4  │    2    │   No     │ {3:0, 5:1, 1:2, 4:3, 2:4}     │
  │  5  │    5    │  Yes ✓   │ ←── FOUND! first seen at idx 1  │
  └─────┴─────────┴──────────┴───────────────────────────────┘
  Result: first duplicate = 5, first seen at index 1

  Index:  0   1   2   3   4   5   6   7
         [3] [5] [1] [4] [2] [5] [7] [1]
               ↑               ↑
               └─── duplicate ──┘

  countDistinct: put all into set → {3,5,1,4,2,7} → len = 6
```

---

## Example 7: Hash Set from Scratch (No Built-in Map)

```go
package main

import "fmt"

type HashSet struct {
    buckets [][]int
    cap     int
    size    int
}

func NewHashSet(cap int) *HashSet {
    return &HashSet{
        buckets: make([][]int, cap),
        cap:     cap,
    }
}

func (s *HashSet) hash(val int) int {
    h := val % s.cap
    if h < 0 {
        h += s.cap
    }
    return h
}

func (s *HashSet) Add(val int) bool {
    if s.Contains(val) {
        return false
    }
    if float64(s.size+1)/float64(s.cap) > 0.75 {
        s.resize()
    }
    idx := s.hash(val)
    s.buckets[idx] = append(s.buckets[idx], val)
    s.size++
    return true
}

func (s *HashSet) Contains(val int) bool {
    idx := s.hash(val)
    for _, v := range s.buckets[idx] {
        if v == val {
            return true
        }
    }
    return false
}

func (s *HashSet) Remove(val int) bool {
    idx := s.hash(val)
    for i, v := range s.buckets[idx] {
        if v == val {
            s.buckets[idx] = append(s.buckets[idx][:i], s.buckets[idx][i+1:]...)
            s.size--
            return true
        }
    }
    return false
}

func (s *HashSet) resize() {
    old := s.buckets
    s.cap *= 2
    s.buckets = make([][]int, s.cap)
    s.size = 0
    for _, bucket := range old {
        for _, v := range bucket {
            idx := s.hash(v)
            s.buckets[idx] = append(s.buckets[idx], v)
            s.size++
        }
    }
}

func main() {
    set := NewHashSet(4)
    for _, v := range []int{10, 20, 30, 10, 40, 20, 50} {
        added := set.Add(v)
        fmt.Printf("Add(%d) → added=%v, size=%d\n", v, added, set.size)
    }

    fmt.Println("\nContains(30):", set.Contains(30))
    set.Remove(30)
    fmt.Println("After Remove(30), Contains(30):", set.Contains(30))
}
```

**Textual Figure — Hash Set from Scratch (Bucket Structure):**

```
  Initial capacity = 4,  hash(val) = val % cap

  Add(10): hash(10)=10%4=2   Add(20): hash(20)=20%4=0   Add(30): hash(30)=30%4=2
  Add(10): duplicate, skip    Add(40): hash(40)=40%4=0

  Before resize (load=4/4=1.0 > 0.75 triggers at Add(40)):
  ┌─────────┬───────────────────┐
  │ Bucket  │ Contents          │
  ├─────────┼───────────────────┤
  │   [0]   │ [20]              │
  │   [1]   │ []                │
  │   [2]   │ [10, 30]          │  ← collision (chaining)
  │   [3]   │ []                │
  └─────────┴───────────────────┘

  Resize to cap=8 (rehash all elements):
  ┌─────────┬───────────────────┐
  │ Bucket  │ Contents          │    hash = val % 8
  ├─────────┼───────────────────┤
  │   [0]   │ [40]              │    40%8=0
  │   [1]   │ []                │
  │   [2]   │ [10]              │    10%8=2
  │   [3]   │ []                │
  │   [4]   │ [20]              │    20%8=4
  │   [5]   │ []                │
  │   [6]   │ [30]              │    30%8=6
  │   [7]   │ []                │
  └─────────┴───────────────────┘
  No collisions after resize!

  Then Add(20): dup, skip.  Add(50): hash(50)=50%8=2 → bucket[2]=[10,50]
  Final size = 5  (10, 20, 30, 40, 50)
```

---

## Example 8: Longest Consecutive Sequence

```go
package main

import "fmt"

// LeetCode 128: Given unsorted array, find length of longest consecutive sequence in O(n)
func longestConsecutive(nums []int) int {
    set := make(map[int]struct{})
    for _, n := range nums {
        set[n] = struct{}{}
    }

    best := 0
    for n := range set {
        // Only start counting from the beginning of a sequence
        if _, ok := set[n-1]; ok {
            continue // n is not the start of a sequence
        }

        length := 1
        curr := n
        for {
            if _, ok := set[curr+1]; ok {
                curr++
                length++
            } else {
                break
            }
        }
        if length > best {
            best = length
        }
    }
    return best
}

func main() {
    tests := [][]int{
        {100, 4, 200, 1, 3, 2},        // → 4 (1,2,3,4)
        {0, 3, 7, 2, 5, 8, 4, 6, 0, 1}, // → 9 (0-8)
        {9, 1, 4, 7, 3, -1, 0, 5, 8, -1, 6}, // → 7
    }

    for _, nums := range tests {
        fmt.Printf("nums=%v → longest=%d\n", nums, longestConsecutive(nums))
    }
}
```

**Textual Figure — Longest Consecutive Sequence (nums = [100,4,200,1,3,2]):**

```
  Step 1: Build set from array
  set = {100, 4, 200, 1, 3, 2}

  Step 2: Find sequence starts (n where n-1 is NOT in set)
  ┌─────┬─────────┬─────────────┬──────────────┐
  │  n  │  n-1    │  n-1 in set? │ Is start?    │
  ├─────┼─────────┼─────────────┼──────────────┤
  │ 100 │  99     │     No       │  Yes ✓       │
  │   4 │   3     │     Yes      │  No (skip)   │
  │ 200 │ 199     │     No       │  Yes ✓       │
  │   1 │   0     │     No       │  Yes ✓       │
  │   3 │   2     │     Yes      │  No (skip)   │
  │   2 │   1     │     Yes      │  No (skip)   │
  └─────┴─────────┴─────────────┴──────────────┘

  Step 3: Extend sequences from each start
    Start at 100: 100 → 101? No   → length = 1
    Start at 200: 200 → 201? No   → length = 1
    Start at 1:   1 → 2 → 3 → 4 → 5? No  → length = 4  ← best!

    1 ──── 2 ──── 3 ──── 4
    └────────────────────┘
         length = 4

  Result: 4
```

---

## Example 9: Set Intersection of Multiple Sets

```go
package main

import "fmt"

type Set map[int]struct{}

func NewSet(vals ...int) Set {
    s := make(Set)
    for _, v := range vals {
        s[v] = struct{}{}
    }
    return s
}

func IntersectAll(sets ...Set) Set {
    if len(sets) == 0 {
        return NewSet()
    }

    // Start with smallest set for efficiency
    smallest := 0
    for i, s := range sets {
        if len(s) < len(sets[smallest]) {
            smallest = i
        }
    }

    result := NewSet()
    for k := range sets[smallest] {
        inAll := true
        for i, s := range sets {
            if i == smallest {
                continue
            }
            if _, ok := s[k]; !ok {
                inAll = false
                break
            }
        }
        if inAll {
            result[k] = struct{}{}
        }
    }
    return result
}

func (s Set) Slice() []int {
    r := make([]int, 0, len(s))
    for k := range s {
        r = append(r, k)
    }
    return r
}

func main() {
    a := NewSet(1, 2, 3, 4, 5, 6)
    b := NewSet(2, 4, 6, 8, 10)
    c := NewSet(4, 5, 6, 7, 8)
    d := NewSet(1, 4, 6, 9)

    result := IntersectAll(a, b, c, d)
    fmt.Println("Intersection of 4 sets:", result.Slice()) // {4, 6}
}
```

**Textual Figure — Intersection of Multiple Sets:**

```
  A = {1, 2, 3, 4, 5, 6}    B = {2, 4, 6, 8, 10}
  C = {4, 5, 6, 7, 8}       D = {1, 4, 6, 9}

  Step 1: Find smallest set (optimize iteration)
    len(A)=6, len(B)=5, len(C)=5, len(D)=4  → smallest = D

  Step 2: For each element in D, check membership in A, B, C
  ┌────────┬───────┬───────┬───────┬───────────┐
  │ D elem │ in A? │ in B? │ in C? │ In result? │
  ├────────┼───────┼───────┼───────┼───────────┤
  │   1    │  ✓    │  ✗    │  —   │     ✗      │  ← short-circuit at B
  │   4    │  ✓    │  ✓    │  ✓   │     ✓      │
  │   6    │  ✓    │  ✓    │  ✓   │     ✓      │
  │   9    │  ✗    │  —   │  —   │     ✗      │  ← short-circuit at A
  └────────┴───────┴───────┴───────┴───────────┘

       A           B
  {1,2,3,4,5,6}  {2,4,6,8,10}
       │  │         │  │
       4  6  ━━━━━  4  6     ← common
       │  │         │  │
  {4,5,6,7,8}    {1,4,6,9}
       C           D

  Result: {4, 6}
```

---

## Example 10: Happy Number (Cycle Detection with Set)

```go
package main

import "fmt"

func digitSquareSum(n int) int {
    sum := 0
    for n > 0 {
        d := n % 10
        sum += d * d
        n /= 10
    }
    return sum
}

func isHappy(n int) bool {
    seen := make(map[int]struct{})
    for n != 1 {
        if _, ok := seen[n]; ok {
            return false // cycle detected
        }
        seen[n] = struct{}{}
        n = digitSquareSum(n)
    }
    return true
}

func main() {
    fmt.Println("Happy numbers from 1 to 50:")
    for i := 1; i <= 50; i++ {
        if isHappy(i) {
            fmt.Print(i, " ")
        }
    }
    fmt.Println()

    // Trace n=19
    fmt.Println("\nTrace for 19:")
    n := 19
    for n != 1 {
        next := digitSquareSum(n)
        fmt.Printf("  %d → %d\n", n, next)
        n = next
    }
    fmt.Println("  1 → Happy!")
}
```

**Textual Figure — Happy Number Cycle Detection (n=19):**

```
  digitSquareSum: sum of squares of each digit

  Trace for n = 19 (happy number):

  19 ──→ 1²+9² = 1+81 = 82
  82 ──→ 8²+2² = 64+4  = 68
  68 ──→ 6²+8² = 36+64 = 100
  100 ─→ 1²+0²+0² = 1    ← reached 1, HAPPY!

  seen set at each step:
    {}           ── check 19 → not in seen, add 19
    {19}         ── check 82 → not in seen, add 82
    {19,82}      ── check 68 → not in seen, add 68
    {19,82,68}   ── check 100 → not in seen, add 100
    {19,82,68,100} ── next = 1, loop ends ✓

  Trace for n = 2 (NOT happy — enters cycle):

    2 → 4 → 16 → 37 → 58 → 89 → 145 → 42 → 20 → 4
                                                    │
    ┌────────────────────────── cycle! ──────────┘
    │   4 → 16 → 37 → 58 → 89 → 145 → 42 → 20 → 4 ...
    └──── seen[4] already exists → return false

  Happy numbers 1–50: 1 7 10 13 19 23 28 31 32 44 49
```

---

## Example 11: Generic Set with Go Generics

```go
package main

import "fmt"

type Set[T comparable] map[T]struct{}

func NewGenericSet[T comparable](vals ...T) Set[T] {
    s := make(Set[T])
    for _, v := range vals {
        s[v] = struct{}{}
    }
    return s
}

func (s Set[T]) Add(val T) { s[val] = struct{}{} }

func (s Set[T]) Remove(val T) { delete(s, val) }

func (s Set[T]) Has(val T) bool {
    _, ok := s[val]
    return ok
}

func (s Set[T]) Union(other Set[T]) Set[T] {
    result := NewGenericSet[T]()
    for k := range s {
        result[k] = struct{}{}
    }
    for k := range other {
        result[k] = struct{}{}
    }
    return result
}

func (s Set[T]) Intersect(other Set[T]) Set[T] {
    result := NewGenericSet[T]()
    for k := range s {
        if other.Has(k) {
            result[k] = struct{}{}
        }
    }
    return result
}

func (s Set[T]) Values() []T {
    r := make([]T, 0, len(s))
    for k := range s {
        r = append(r, k)
    }
    return r
}

func main() {
    // String set
    langs := NewGenericSet("Go", "Rust", "Python")
    others := NewGenericSet("Go", "Java", "C++")
    fmt.Println("String union:", langs.Union(others).Values())
    fmt.Println("String intersect:", langs.Intersect(others).Values())

    // Float set
    floats := NewGenericSet(1.1, 2.2, 3.3)
    floats.Add(4.4)
    floats.Remove(2.2)
    fmt.Println("Float set:", floats.Values())
}
```

**Textual Figure — Generic Set with Type Parameters:**

```
  Go Generics: Set[T comparable] = map[T]struct{}

  String set operations:
  ┌─────────────────────────────────────────────────────────┐
  │  langs  = Set[string]{ "Go", "Rust", "Python" }    │
  │  others = Set[string]{ "Go", "Java", "C++" }        │
  ├─────────────────────────────────────────────────────────┤
  │  Union:     { "Go", "Rust", "Python", "Java", "C++" } │
  │  Intersect: { "Go" }                                  │
  └─────────────────────────────────────────────────────────┘

  Float set operations:
  ┌──────────────────────────────────────────────┐
  │  float set = Set[float64]{ 1.1, 2.2, 3.3 }    │
  │  Add(4.4)  →  { 1.1, 2.2, 3.3, 4.4 }          │
  │  Remove(2.2) → { 1.1, 3.3, 4.4 }               │
  └──────────────────────────────────────────────┘

  Type parameter [T comparable] allows:
    ┌──────────┬────────────────┬───────────────────────┐
    │   T      │   Concrete Type │   Backed by             │
    ├──────────┼────────────────┼───────────────────────┤
    │ string   │ Set[string]     │ map[string]struct{}     │
    │ int      │ Set[int]        │ map[int]struct{}        │
    │ float64  │ Set[float64]    │ map[float64]struct{}    │
    └──────────┴────────────────┴───────────────────────┘
```

---

## Hash Set Complexity

| Operation | Average | Worst |
|-----------|---------|-------|
| Add       | O(1)    | O(n)  |
| Contains  | O(1)    | O(n)  |
| Remove    | O(1)    | O(n)  |
| Union     | O(m+n)  | O(m+n)|
| Intersect | O(min)  | O(min)|

## Key Takeaways

1. **Go idiom**: `map[T]struct{}` — zero-size value, most memory-efficient
2. **O(1) membership testing** — hash sets are perfect for existence checks
3. **Set operations** (union, intersection, difference) are foundational
4. **Longest consecutive sequence** — classic interview problem using sets
5. **No duplicates** — adding same element again is a no-op
6. **Go 1.18+ generics** allow type-safe reusable sets
7. **Iterate over smaller set** for intersection (optimization)

> **Next up:** Frequency Maps →
