# Phase 2: Arrays — Dynamic Arrays

## What is a Dynamic Array?

A dynamic array can **grow and shrink** at runtime. In Go, **slices** are dynamic arrays backed by a fixed-size underlying array. When capacity is exceeded, a new larger array is allocated and elements are copied.

---

## Key Properties

| Operation | Time |
|-----------|------|
| Access by index | O(1) |
| Append | O(1) amortized |
| Insert at index | O(n) |
| Delete at index | O(n) |
| Search | O(n) |

---

## Example 1: Slice Internals — len, cap, pointer

```go
package main

import "fmt"

func main() {
    // A slice has 3 fields: pointer, length, capacity
    s := make([]int, 3, 5)
    s[0], s[1], s[2] = 10, 20, 30

    fmt.Printf("len=%d, cap=%d, data=%v\n", len(s), cap(s), s)
    // len=3, cap=5, data=[10 20 30]

    // Append within capacity — no reallocation
    s = append(s, 40)
    fmt.Printf("len=%d, cap=%d, data=%v\n", len(s), cap(s), s)
    // len=4, cap=5

    s = append(s, 50)
    fmt.Printf("len=%d, cap=%d, data=%v\n", len(s), cap(s), s)
    // len=5, cap=5

    // Append beyond capacity — reallocation!
    s = append(s, 60)
    fmt.Printf("len=%d, cap=%d, data=%v\n", len(s), cap(s), s)
    // len=6, cap=10 (doubled!)
}
```

**Textual Figure — Slice Internals (ptr, len, cap):**

```
  s := make([]int, 3, 5)   →   len=3, cap=5

  Slice header:         Backing array:
  ┌──────────┐           ┌────┬────┬────┬────┬────┐
  │ ptr ────┼───────▶ │ 10 │ 20 │ 30 │    │    │
  │ len = 3  │           └────┴────┴────┴────┴────┘
  │ cap = 5  │            │── used ──│─ available ─│
  └──────────┘

  append(s, 40)   →  len=4, cap=5 (room available, no realloc)
  append(s, 50)   →  len=5, cap=5 (exactly full)
  append(s, 60)   →  len=6, cap=10 (DOUBLED! new array allocated)

  Before reallocation:    After reallocation:
  ┌───┬───┬───┬───┬───┐    ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │10 │20 │30 │40 │50 │    │10 │20 │30 │40 │50 │60 │   │   │   │   │
  └───┴───┴───┴───┴───┘    └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  cap=5  FULL              cap=10  new array!
```

---

## Example 2: Growth Strategy — How Capacity Doubles

```go
package main

import "fmt"

func main() {
    var s []int
    prevCap := 0

    for i := 0; i < 50; i++ {
        s = append(s, i)
        if cap(s) != prevCap {
            fmt.Printf("len=%2d → cap changed from %3d to %3d\n", len(s), prevCap, cap(s))
            prevCap = cap(s)
        }
    }
    // Growth pattern: 0→1→2→4→8→16→32→64
    // Ensures amortized O(1) append
}
```

**Textual Figure — Capacity Growth Pattern:**

```
  Appending 50 elements, tracking capacity changes:

  len:  1  2  3  4  5  6  7  8  9 ... 17 ... 33 ... 50
  cap:  1  2  4  4  8  8  8  8  16    32     64     64
               ↑        ↑           ↑      ↑
             double   double       double  double

  Cap  ▲
  64   │                                   ████████
  32   │                        ████████
  16   │              ████████
   8   │        █████
   4   │     ███
   2   │   ██
   1   │  █
       └────────────────────────────────────▶ len

  Doubling ensures:
  • Number of reallocations = O(log n)
  • Total elements copied = n + n/2 + n/4 + ... ≈ 2n
  • Amortized cost per append = O(1)
```

---

## Example 3: Building a Custom Dynamic Array

```go
package main

import "fmt"

type DynamicArray struct {
    data []int
    size int
}

func NewDynamicArray() *DynamicArray {
    return &DynamicArray{data: make([]int, 1), size: 0}
}

func (d *DynamicArray) Get(index int) int { return d.data[index] }
func (d *DynamicArray) Set(index, val int) { d.data[index] = val }
func (d *DynamicArray) Len() int { return d.size }
func (d *DynamicArray) Cap() int { return len(d.data) }

func (d *DynamicArray) Append(val int) {
    if d.size == len(d.data) {
        d.resize(2 * len(d.data)) // double capacity
    }
    d.data[d.size] = val
    d.size++
}

func (d *DynamicArray) resize(newCap int) {
    newData := make([]int, newCap)
    copy(newData, d.data[:d.size])
    d.data = newData
    fmt.Printf("  Resized to capacity %d\n", newCap)
}

func (d *DynamicArray) String() string {
    return fmt.Sprintf("%v", d.data[:d.size])
}

func main() {
    arr := NewDynamicArray()
    for i := 1; i <= 10; i++ {
        arr.Append(i * 10)
    }
    fmt.Println("Array:", arr)
    fmt.Printf("Size: %d, Capacity: %d\n", arr.Len(), arr.Cap())
}
```

**Textual Figure — Custom Dynamic Array Resize Steps:**

```
  Append: 10, 20, 30, 40, 50, 60, 70, 80, 90, 100

  Append(10): cap=1  ┌───┐
                     │ 10 │  size=1
                     └───┘

  Append(20): RESIZE cap=2
                     ┌───┬───┐
                     │ 10 │ 20 │  size=2
                     └───┴───┘

  Append(30): RESIZE cap=4
                     ┌───┬───┬───┬───┐
                     │ 10 │ 20 │ 30 │   │  size=3, room for 1 more
                     └───┴───┴───┴───┘

  Append(40): fits!  size=4, cap=4

  Append(50): RESIZE cap=8
                     ┌───┬───┬───┬───┬───┬───┬───┬───┐
                     │ 10 │ 20 │ 30 │ 40 │ 50 │   │   │   │  size=5
                     └───┴───┴───┴───┴───┴───┴───┴───┘

  Resize happens at: size=1→2, 2→4, 4→8, 8→16
  Pattern: capacity doubles each time
```

---

## Example 4: Insert at Index — O(n)

```go
package main

import "fmt"

func insertAt(s []int, index, value int) []int {
    s = append(s, 0)             // make room: O(1) amortized
    copy(s[index+1:], s[index:]) // shift right: O(n)
    s[index] = value
    return s
}

func main() {
    s := []int{10, 20, 30, 40, 50}
    fmt.Println("Before:", s)

    s = insertAt(s, 0, 5)   // insert at beginning: worst case O(n)
    fmt.Println("Insert 5 at 0:", s) // [5 10 20 30 40 50]

    s = insertAt(s, 3, 25)
    fmt.Println("Insert 25 at 3:", s) // [5 10 20 25 30 40 50]

    s = insertAt(s, len(s), 60) // insert at end: O(1)
    fmt.Println("Insert 60 at end:", s) // [5 10 20 25 30 40 50 60]
}
```

**Textual Figure — Insert at Index (Shifting):**

```
  Insert 25 at index 3:

  Before: [5, 10, 20, 30, 40, 50]
                       ↑ insert here

  Step 1: append(s, 0)  →  make room at end
          [5, 10, 20, 30, 40, 50, 0]

  Step 2: copy(s[4:], s[3:])  →  shift right from index 3
          [5, 10, 20, 30, 30, 40, 50]
                       ↑───▶↑───▶↑───▶↑

  Step 3: s[3] = 25
          [5, 10, 20, 25, 30, 40, 50]
                       ↑ inserted!

  Insert positions and their cost:
  ┌───────────────┬────────────────────┐
  │  Position      │  Elements shifted │
  ├───────────────┼────────────────────┤
  │  Beginning (0) │  n (all) — worst  │
  │  Middle (n/2)  │  n/2             │
  │  End (n)       │  0 — O(1)         │
  └───────────────┴────────────────────┘
```

---

## Example 5: Delete at Index — O(n)

```go
package main

import "fmt"

// Delete maintaining order — O(n)
func deleteAt(s []int, index int) []int {
    return append(s[:index], s[index+1:]...) // shift left
}

// Delete without maintaining order — O(1)
func deleteSwap(s []int, index int) []int {
    s[index] = s[len(s)-1] // swap with last
    return s[:len(s)-1]    // shrink
}

func main() {
    s := []int{10, 20, 30, 40, 50}
    fmt.Println("Original:", s)

    s1 := make([]int, len(s))
    copy(s1, s)
    s1 = deleteAt(s1, 2)
    fmt.Println("Delete index 2 (ordered):", s1) // [10 20 40 50]

    s2 := make([]int, len(s))
    copy(s2, s)
    s2 = deleteSwap(s2, 2)
    fmt.Println("Delete index 2 (swap):", s2) // [10 20 50 40] — order changed!
}
```

**Textual Figure — Ordered Delete vs Swap Delete:**

```
  Delete index 2 from [10, 20, 30, 40, 50]:

  Ordered delete: O(n) — shift left, preserves order
  ────────────────────────────────────────
  [10, 20, 30, 40, 50]     delete s[2]=30
          ←──┬───┘
  [10, 20, 40, 50]         O(n) shifts
  Order preserved ✓

  Swap delete: O(1) — swap with last, order broken
  ────────────────────────────────────────
  [10, 20, 30, 40, 50]     swap s[2] with s[4]
              ↑─────────↑
  [10, 20, 50, 40]         O(1), but order changed!

  Choose based on your needs:
  ┌────────────────┬────────┬─────────────┐
  │ Method         │ Time   │ Preserves    │
  │                │        │ Order?       │
  ├────────────────┼────────┼─────────────┤
  │ Ordered delete │ O(n)   │ Yes ✓        │
  │ Swap delete    │ O(1)   │ No  ✗        │
  └────────────────┴────────┴─────────────┘
```

---

## Example 6: Slice of Slices — 2D Dynamic Array

```go
package main

import "fmt"

func main() {
    rows, cols := 3, 4

    // Create 2D dynamic array
    matrix := make([][]int, rows)
    for i := range matrix {
        matrix[i] = make([]int, cols)
    }

    // Fill
    count := 1
    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            matrix[i][j] = count
            count++
        }
    }

    // Each row can have different length (jagged array)
    jagged := [][]int{
        {1, 2, 3},
        {4, 5},
        {6, 7, 8, 9},
    }
    for _, row := range jagged {
        fmt.Println(row)
    }

    // Dynamic: append a new row
    jagged = append(jagged, []int{10, 11})
    fmt.Println("After append:", jagged)
}
```

**Textual Figure — 2D Dynamic Array (Jagged):**

```
  Regular 3×4 matrix:              Jagged array:
  ┌───┬───┬───┬───┐               Row 0: ┌─┬─┬─┐
  │ 1 │ 2 │ 3 │ 4 │                      │1│2│3│
  ├───┼───┼───┼───┤                      └─┴─┴─┘
  │ 5 │ 6 │ 7 │ 8 │               Row 1: ┌─┬─┐
  ├───┼───┼───┼───┤                      │4│5│
  │ 9 │10 │11 │12 │                      └─┴─┘
  └───┴───┴───┴───┘               Row 2: ┌─┬─┬─┬─┐
  All rows = 4 cols                       │6│7│8│9│
                                          └─┴─┴─┴─┘
                                   Rows can have different lengths!

  Memory: each row is a separate slice with its own pointer
  ┌─────────┐     ┌─┬─┬─┐
  │ ptr ───▶┼────▶│1│2│3│   row 0
  ├─────────┤     └─┴─┴─┘
  │ ptr ───▶┼──▶┌─┬─┐
  ├─────────┤   │4│5│         row 1
  │ ptr ───▶┼──┘└─┴─┘
  └─────────┘  └▶┌─┬─┬─┬─┐
                  │6│7│8│9│  row 2
                  └─┴─┴─┴─┘
```

---

## Example 7: Slice Tricks — Common Operations

```go
package main

import "fmt"

func main() {
    // 1. Copy a slice
    original := []int{1, 2, 3, 4, 5}
    copied := make([]int, len(original))
    copy(copied, original)

    // 2. Reverse a slice
    for i, j := 0, len(copied)-1; i < j; i, j = i+1, j-1 {
        copied[i], copied[j] = copied[j], copied[i]
    }
    fmt.Println("Reversed:", copied) // [5 4 3 2 1]

    // 3. Filter (keep even numbers)
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8}
    evens := nums[:0] // reuse underlying array
    for _, n := range nums {
        if n%2 == 0 {
            evens = append(evens, n)
        }
    }
    fmt.Println("Evens:", evens) // [2 4 6 8]

    // 4. Deduplicate sorted slice
    sorted := []int{1, 1, 2, 2, 3, 3, 3, 4, 5, 5}
    deduped := sorted[:1]
    for i := 1; i < len(sorted); i++ {
        if sorted[i] != sorted[i-1] {
            deduped = append(deduped, sorted[i])
        }
    }
    fmt.Println("Deduped:", deduped) // [1 2 3 4 5]

    // 5. Pop from end
    stack := []int{10, 20, 30}
    top := stack[len(stack)-1]
    stack = stack[:len(stack)-1]
    fmt.Println("Popped:", top, "Stack:", stack) // 30 [10 20]
}
```

**Textual Figure — Common Slice Tricks:**

```
  1. Copy: make new slice + copy()
     ┌─┬─┬─┬─┬─┐       ┌─┬─┬─┬─┬─┐
     │1│2│3│4│5│  ─▶  │1│2│3│4│5│  independent copy
     └─┴─┴─┴─┴─┘       └─┴─┴─┴─┴─┘

  2. Reverse in-place:
     ┌─┬─┬─┬─┬─┐      swap(0,4)  swap(1,3)
     │1│2│3│4│5│  ─▶  ┌─┬─┬─┬─┬─┐
     └─┴─┴─┴─┴─┘       │5│4│3│2│1│
                        └─┴─┴─┴─┴─┘

  3. Filter (in-place, reuse backing):
     [1,2,3,4,5,6,7,8]  keep evens
     evens := nums[:0]  ← same array, len=0
     → [2,4,6,8]        overwritten in-place

  4. Dedup sorted:
     [1,1,2,2,3,3,3,4,5,5]
      ↑ keep
        ↑ skip (same as prev)
          ↑ keep (new value) ...
     → [1,2,3,4,5]

  5. Pop from end: O(1)
     [10,20,30]  →  top=30, stack=[10,20]
```

---

## Example 8: Pre-allocating Capacity for Performance

```go
package main

import (
    "fmt"
    "time"
)

func appendNoPrealloc(n int) []int {
    var s []int // starts with cap=0
    for i := 0; i < n; i++ {
        s = append(s, i) // many reallocations
    }
    return s
}

func appendWithPrealloc(n int) []int {
    s := make([]int, 0, n) // pre-allocate capacity
    for i := 0; i < n; i++ {
        s = append(s, i) // zero reallocations
    }
    return s
}

func main() {
    n := 10_000_000

    start := time.Now()
    appendNoPrealloc(n)
    fmt.Printf("No pre-alloc:   %v\n", time.Since(start))

    start = time.Now()
    appendWithPrealloc(n)
    fmt.Printf("With pre-alloc: %v\n", time.Since(start))

    // Pre-allocation avoids O(log n) reallocations and copies
    fmt.Println("\nTip: Always pre-allocate when you know the size!")
}
```

**Textual Figure — Pre-allocation vs No Pre-allocation:**

```
  Without pre-alloc (var s []int):
  ───────────────────────────────────
  Append 1:  alloc cap=1   copy 0
  Append 2:  alloc cap=2   copy 1
  Append 3:  alloc cap=4   copy 2
  Append 5:  alloc cap=8   copy 4
  Append 9:  alloc cap=16  copy 8
  ...        ...           ...
  Total: ~log₂(n) allocations, ~2n copies

  With pre-alloc (make([]int, 0, n)):
  ───────────────────────────────────
  Alloc cap=n once!
  Append 1..n: zero reallocations, zero copies

  For n=10,000,000:
  No pre-alloc:    ~24 reallocations, ~20M copies
  With pre-alloc:   1 allocation,      0 copies

  Speed difference: 2-5× faster with pre-allocation!
```

---

## Example 9: Slice Memory Leaks and Gotchas

```go
package main

import "fmt"

func main() {
    // Gotcha 1: Slicing doesn't copy — shares underlying array
    original := []int{1, 2, 3, 4, 5}
    sub := original[1:3]
    sub[0] = 999
    fmt.Println("Original modified!", original) // [1 999 3 4 5]

    // Gotcha 2: Large backing array kept in memory
    bigSlice := make([]int, 1000000)
    for i := range bigSlice { bigSlice[i] = i }

    // This small slice keeps the entire 1M array alive!
    smallSlice := bigSlice[:3]
    _ = smallSlice

    // Fix: copy to a new slice
    safeCopy := make([]int, 3)
    copy(safeCopy, bigSlice[:3])
    bigSlice = nil // now the big array can be garbage collected
    fmt.Println("Safe copy:", safeCopy)

    // Gotcha 3: Append may or may not create a new array
    a := make([]int, 3, 5)
    a[0], a[1], a[2] = 1, 2, 3
    b := append(a, 4) // uses existing capacity — shares array!
    b[0] = 999
    fmt.Println("a:", a) // [999 2 3] — modified through b!
}
```

**Textual Figure — Slice Gotchas Visualized:**

```
  Gotcha 1: Slicing shares underlying array
  ──────────────────────────────────────
  original := [1, 2, 3, 4, 5]
  sub := original[1:3]

  original: ─▶ ┌─┬─┬─┬─┬─┐
               │1│2│3│4│5│  ← shared!
  sub:      ─▶    ↑─↑
                  │ │
               sub[0] sub[1]

  sub[0] = 999 → original becomes [1, 999, 3, 4, 5] ⚠

  Gotcha 2: Small slice keeps big array alive
  ──────────────────────────────────────
  bigSlice (1M elements) ─▶ ┌───────────────────┐
                            │  1,000,000 ints      │
  smallSlice = big[:3] ─▶  │▲                      │
                            └┴───────────────────┘
  Only 3 elements needed, but entire 8MB stays allocated!

  Fix: copy(safeCopy, bigSlice[:3])  → independent!
```

---

## Example 10: Dynamic Array Use Cases

```go
package main

import "fmt"

// Stack using dynamic array
type Stack struct{ data []int }
func (s *Stack) Push(v int) { s.data = append(s.data, v) }
func (s *Stack) Pop() int   { v := s.data[len(s.data)-1]; s.data = s.data[:len(s.data)-1]; return v }
func (s *Stack) Len() int   { return len(s.data) }

// Queue using dynamic array (not ideal — just for demo)
type Queue struct{ data []int }
func (q *Queue) Enqueue(v int) { q.data = append(q.data, v) }
func (q *Queue) Dequeue() int  { v := q.data[0]; q.data = q.data[1:]; return v }

// Collect results dynamically
func findAllEvens(nums []int) []int {
    result := make([]int, 0, len(nums)/2) // estimate capacity
    for _, n := range nums {
        if n%2 == 0 {
            result = append(result, n)
        }
    }
    return result
}

func main() {
    // Stack
    s := &Stack{}
    s.Push(1); s.Push(2); s.Push(3)
    fmt.Println(s.Pop(), s.Pop(), s.Pop()) // 3 2 1

    // Queue
    q := &Queue{}
    q.Enqueue(10); q.Enqueue(20); q.Enqueue(30)
    fmt.Println(q.Dequeue(), q.Dequeue()) // 10 20

    // Dynamic collection
    fmt.Println(findAllEvens([]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}))
    // [2 4 6 8 10]
}
```

**Textual Figure — Dynamic Array for Stack, Queue, Collection:**

```
  Stack (LIFO) — append/pop from end: O(1)
  ───────────────────────────────────
  Push 1,2,3:  [1, 2, 3]
                        ↑ top
  Pop → 3:     [1, 2]
                     ↑ top

  Queue (FIFO) — append at end, dequeue from front: O(n)
  ───────────────────────────────────
  Enqueue 10,20,30:     [10, 20, 30]
                         ↑ front       ↑ back
  Dequeue → 10:         [20, 30]
  (Note: s = s[1:] doesn't free memory — use ring buffer for real queues)

  Dynamic collection — filter with pre-allocated capacity:
  ───────────────────────────────────
  Input:  [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
  Filter: keep evens
  Output: [2, 4, 6, 8, 10]
  Capacity hint: make([]int, 0, n/2) → minimal allocs
```

---

## Key Takeaways

1. **Go slices = dynamic arrays** — header (ptr, len, cap) + backing array
2. **Append is O(1) amortized** — capacity doubles when exceeded
3. **Pre-allocate with `make([]T, 0, n)`** when size is known
4. **Slicing shares memory** — be careful with modifications
5. **Insert/delete at arbitrary index is O(n)** — requires shifting
6. **Swap-delete is O(1)** if order doesn't matter

> **Next up:** Indexing →
