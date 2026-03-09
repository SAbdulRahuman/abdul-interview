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
