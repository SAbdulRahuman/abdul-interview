# Phase 2: Arrays — Static Arrays

## What is a Static Array?

A static array is a **fixed-size**, contiguous block of memory. Once created, its size **cannot change**. Elements are stored sequentially in memory, allowing **O(1) random access** by index.

In Go, arrays (not slices) are static: `var arr [5]int` — size is part of the type.

---

## Key Properties

| Property | Value |
|----------|-------|
| Access by index | O(1) |
| Search (unsorted) | O(n) |
| Insert/Delete | O(n) — must shift elements |
| Size | Fixed at creation |
| Memory | Contiguous |

---

## Example 1: Declaring and Accessing Static Arrays

```go
package main

import "fmt"

func main() {
    // Static array — size is part of the type
    var arr [5]int // [0 0 0 0 0] — zero-initialized
    arr[0] = 10
    arr[1] = 20
    arr[2] = 30
    arr[3] = 40
    arr[4] = 50

    // O(1) access by index — direct memory offset
    fmt.Println(arr[0])  // 10
    fmt.Println(arr[4])  // 50
    fmt.Println(arr)     // [10 20 30 40 50]

    // Array literal
    arr2 := [3]string{"Go", "is", "fast"}
    fmt.Println(arr2) // [Go is fast]

    // Size inferred
    arr3 := [...]int{1, 2, 3, 4}
    fmt.Println(arr3, len(arr3)) // [1 2 3 4] 4
}
```

---

## Example 2: O(1) Random Access — How It Works

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    arr := [5]int{100, 200, 300, 400, 500}

    // O(1) access: address = base + index * sizeof(element)
    base := uintptr(unsafe.Pointer(&arr[0]))
    elemSize := unsafe.Sizeof(arr[0])

    for i := 0; i < len(arr); i++ {
        addr := base + uintptr(i)*elemSize
        fmt.Printf("arr[%d] = %d  (address: %x, offset: %d bytes)\n",
            i, arr[i], addr, uintptr(i)*elemSize)
    }
    // Each element is exactly 8 bytes apart (64-bit int)
    // Address calculation is O(1) — no searching needed!
}
```

---

## Example 3: Array vs Slice in Go

```go
package main

import "fmt"

func main() {
    // ARRAY — static, fixed size, value type
    arr := [5]int{1, 2, 3, 4, 5}

    // Arrays are VALUE TYPES — passed by copy
    arr2 := arr      // creates a COPY
    arr2[0] = 999
    fmt.Println(arr)  // [1 2 3 4 5] — unchanged!
    fmt.Println(arr2) // [999 2 3 4 5] — only copy changed

    // SLICE — dynamic, reference type
    slice := []int{1, 2, 3, 4, 5}
    slice2 := slice    // shares underlying array
    slice2[0] = 999
    fmt.Println(slice) // [999 2 3 4 5] — CHANGED!

    // Array size is part of type
    // var x [5]int = [3]int{1,2,3}  // COMPILE ERROR: different types!

    fmt.Printf("Array type: %T, size: %d\n", arr, len(arr))   // [5]int
    fmt.Printf("Slice type: %T, size: %d\n", slice, len(slice)) // []int
}
```

---

## Example 4: Linear Search in Static Array

```go
package main

import "fmt"

// O(n) search — must check each element
func search(arr [10]int, target int) int {
    for i, v := range arr {
        if v == target {
            return i
        }
    }
    return -1
}

// O(n) — find minimum
func findMin(arr [10]int) int {
    min := arr[0]
    for _, v := range arr[1:] {
        if v < min {
            min = v
        }
    }
    return min
}

func main() {
    arr := [10]int{45, 23, 67, 12, 89, 34, 56, 78, 90, 11}
    fmt.Println("Search 34:", search(arr, 34))  // 5
    fmt.Println("Search 99:", search(arr, 99))  // -1
    fmt.Println("Minimum:", findMin(arr))        // 11
}
```

---

## Example 5: Simulating Insert and Delete (Shifting)

```go
package main

import "fmt"

// Insert at index — O(n) due to shifting
func insertAt(arr *[10]int, size *int, index, value int) {
    // Shift elements right
    for i := *size; i > index; i-- {
        arr[i] = arr[i-1]
    }
    arr[index] = value
    *size++
}

// Delete at index — O(n) due to shifting
func deleteAt(arr *[10]int, size *int, index int) {
    // Shift elements left
    for i := index; i < *size-1; i++ {
        arr[i] = arr[i+1]
    }
    arr[*size-1] = 0 // clear last
    *size--
}

func main() {
    var arr [10]int
    size := 0

    // Build array: [10, 20, 30, 40, 50]
    for _, v := range []int{10, 20, 30, 40, 50} {
        arr[size] = v
        size++
    }
    fmt.Println("Initial:", arr[:size]) // [10 20 30 40 50]

    // Insert 25 at index 2
    insertAt(&arr, &size, 2, 25)
    fmt.Println("After insert 25 at idx 2:", arr[:size]) // [10 20 25 30 40 50]

    // Delete at index 1
    deleteAt(&arr, &size, 1)
    fmt.Println("After delete idx 1:", arr[:size]) // [10 25 30 40 50]
}
```

---

## Example 6: Array as Fixed-Size Buffer

```go
package main

import "fmt"

// Ring buffer / circular buffer using static array
type RingBuffer struct {
    data  [8]int // fixed-size array
    head  int
    tail  int
    count int
}

func (r *RingBuffer) Push(val int) bool {
    if r.count == len(r.data) {
        return false // full
    }
    r.data[r.tail] = val
    r.tail = (r.tail + 1) % len(r.data)
    r.count++
    return true
}

func (r *RingBuffer) Pop() (int, bool) {
    if r.count == 0 {
        return 0, false // empty
    }
    val := r.data[r.head]
    r.head = (r.head + 1) % len(r.data)
    r.count--
    return val, true
}

func main() {
    rb := &RingBuffer{}
    for i := 1; i <= 8; i++ {
        rb.Push(i * 10) // fill buffer
    }
    fmt.Println("Full:", rb.Push(90)) // false — buffer full

    for rb.count > 0 {
        val, _ := rb.Pop()
        fmt.Print(val, " ")
    }
    fmt.Println() // 10 20 30 40 50 60 70 80
}
```

---

## Example 7: Multi-Dimensional Static Arrays

```go
package main

import "fmt"

func main() {
    // 2D static array — 3×4 matrix
    var matrix [3][4]int

    // Fill with values
    count := 1
    for i := 0; i < 3; i++ {
        for j := 0; j < 4; j++ {
            matrix[i][j] = count
            count++
        }
    }

    // Print matrix
    for _, row := range matrix {
        fmt.Println(row)
    }
    // [1 2 3 4]
    // [5 6 7 8]
    // [9 10 11 12]

    // O(1) access: matrix[1][2] = 7
    fmt.Println("matrix[1][2] =", matrix[1][2])

    // 3D array
    var cube [2][3][4]int
    cube[1][2][3] = 42
    fmt.Println("cube[1][2][3] =", cube[1][2][3])
}
```

---

## Example 8: Comparing Static Array Performance

```go
package main

import (
    "fmt"
    "time"
)

// Static array — allocated on stack (fast)
func sumArray() int {
    arr := [1000]int{}
    for i := range arr { arr[i] = i }
    sum := 0
    for _, v := range arr { sum += v }
    return sum
}

// Slice — allocated on heap (slightly slower due to indirection)
func sumSlice() int {
    s := make([]int, 1000)
    for i := range s { s[i] = i }
    sum := 0
    for _, v := range s { sum += v }
    return sum
}

func main() {
    iterations := 1000000

    start := time.Now()
    for i := 0; i < iterations; i++ { sumArray() }
    arrTime := time.Since(start)

    start = time.Now()
    for i := 0; i < iterations; i++ { sumSlice() }
    sliceTime := time.Since(start)

    fmt.Printf("Array:  %v\n", arrTime)
    fmt.Printf("Slice:  %v\n", sliceTime)
    fmt.Println("Arrays can be stack-allocated → no GC pressure")
}
```

---

## Example 9: Array-Based Stack (Fixed Capacity)

```go
package main

import (
    "errors"
    "fmt"
)

type FixedStack struct {
    data [100]int // static array
    top  int
}

func (s *FixedStack) Push(val int) error {
    if s.top >= len(s.data) {
        return errors.New("stack overflow")
    }
    s.data[s.top] = val
    s.top++
    return nil
}

func (s *FixedStack) Pop() (int, error) {
    if s.top == 0 {
        return 0, errors.New("stack underflow")
    }
    s.top--
    return s.data[s.top], nil
}

func (s *FixedStack) Peek() int { return s.data[s.top-1] }
func (s *FixedStack) Size() int { return s.top }

func main() {
    s := &FixedStack{}
    for i := 1; i <= 5; i++ { s.Push(i * 10) }

    fmt.Println("Size:", s.Size()) // 5
    fmt.Println("Peek:", s.Peek()) // 50

    for s.Size() > 0 {
        val, _ := s.Pop()
        fmt.Print(val, " ")
    }
    fmt.Println() // 50 40 30 20 10
}
```

---

## Example 10: Using Arrays for Counting (Fixed Alphabet)

```go
package main

import "fmt"

// Count character frequencies — fixed array of 26 for lowercase
func charFrequency(s string) [26]int {
    var freq [26]int // static: exactly 26 slots
    for _, ch := range s {
        if ch >= 'a' && ch <= 'z' {
            freq[ch-'a']++
        }
    }
    return freq
}

// Check if two strings are anagrams
func isAnagram(s, t string) bool {
    if len(s) != len(t) { return false }
    freq := charFrequency(s)
    for _, ch := range t {
        freq[ch-'a']--
    }
    for _, count := range freq {
        if count != 0 { return false }
    }
    return true
}

func main() {
    freq := charFrequency("hello world")
    for i, count := range freq {
        if count > 0 {
            fmt.Printf("%c: %d  ", 'a'+i, count)
        }
    }
    fmt.Println()

    fmt.Println(isAnagram("listen", "silent")) // true
    fmt.Println(isAnagram("hello", "world"))   // false
}
```

---

## Static Array vs Dynamic Array Summary

| Feature | Static Array | Dynamic Array (Slice) |
|---------|-------------|----------------------|
| Size | Fixed at compile time | Grows dynamically |
| Memory | Stack or heap | Always heap |
| Access | O(1) | O(1) |
| Append | N/A (fixed) | O(1) amortized |
| Type | `[5]int` | `[]int` |
| Copy behavior | Value copy | Reference (shared backing) |

---

## Key Takeaways

1. **Static arrays** have fixed size — size is part of the type in Go
2. **O(1) access** via index: `address = base + index × element_size`
3. **Insert/delete is O(n)** — requires shifting elements
4. **Value type** in Go — assigning copies the entire array
5. **Use when size is known**: counting arrays, buffers, small fixed collections
6. **Slices** are Go's dynamic alternative — backed by arrays internally

> **Next up:** Dynamic Arrays →
