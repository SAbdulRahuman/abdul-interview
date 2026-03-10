# Phase 2: Arrays — Indexing

## What is Indexing?

Indexing is how we access elements by their **position** in an array. Arrays store elements contiguously in memory, so `arr[i]` = base address + i × element size → **O(1)** access.

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| Zero-based indexing | Go arrays/slices start at index 0 |
| Bounds checking | Go panics on out-of-range access |
| Negative indexing | Not native in Go; must compute manually |
| Multi-dimensional | `matrix[row][col]` — row-major by default |
| Stride | Gap between consecutive elements in memory |

---

## Example 1: Basic Indexing — O(1) Access

```go
package main

import "fmt"

func main() {
    arr := [5]int{10, 20, 30, 40, 50}

    // Direct access — O(1)
    fmt.Println("First:", arr[0])   // 10
    fmt.Println("Third:", arr[2])   // 30
    fmt.Println("Last:", arr[4])    // 50

    // Modify via index — O(1)
    arr[2] = 999
    fmt.Println("After modify:", arr) // [10 20 999 40 50]
}
```

**Textual Figure — Basic O(1) Array Indexing:**

```
  arr = [10, 20, 30, 40, 50]

  Index:    0     1     2     3     4
          ┌─────┬─────┬─────┬─────┬─────┐
          │ 10  │ 20  │ 30  │ 40  │ 50  │
          └─────┴─────┴─────┴─────┴─────┘
                        ↑
  arr[0] = 10   ← O(1)  arr[2] = 30
  arr[4] = 50   ← O(1)

  Read / Write are both O(1) — just address arithmetic
```

---

## Example 2: Simulating Negative Indexing

```go
package main

import "fmt"

// negIndex returns s[i] where negative i counts from end
func negIndex(s []int, i int) int {
    if i < 0 {
        i = len(s) + i
    }
    return s[i]
}

func main() {
    s := []int{10, 20, 30, 40, 50}

    fmt.Println(negIndex(s, -1))  // 50  (last element)
    fmt.Println(negIndex(s, -2))  // 40  (second to last)
    fmt.Println(negIndex(s, -5))  // 10  (first element)
    fmt.Println(negIndex(s, 0))   // 10
    fmt.Println(negIndex(s, 2))   // 30
}
```

**Textual Figure — Negative Indexing (Python-style):**

```
  s = [10, 20, 30, 40, 50]    len = 5

  Positive:  0    1    2    3    4
           ┌────┬────┬────┬────┬────┐
           │ 10 │ 20 │ 30 │ 40 │ 50 │
           └────┴────┴────┴────┴────┘
  Negative: -5   -4   -3   -2   -1

  Formula: if i < 0, use len(s) + i
    negIndex(s, -1) → 5 + (-1) = 4 → s[4] = 50
    negIndex(s, -3) → 5 + (-3) = 2 → s[2] = 30
    negIndex(s, -5) → 5 + (-5) = 0 → s[0] = 10

  Go doesn't support this natively — must wrap it yourself!
```

---

## Example 3: 2D Indexing — Row and Column

```go
package main

import "fmt"

func main() {
    // 3×4 matrix
    matrix := [3][4]int{
        {1, 2, 3, 4},
        {5, 6, 7, 8},
        {9, 10, 11, 12},
    }

    // Access: matrix[row][col]
    fmt.Println("Row 0, Col 2:", matrix[0][2])  // 3
    fmt.Println("Row 2, Col 3:", matrix[2][3])  // 12

    // Convert 2D → 1D index
    rows, cols := 3, 4
    for r := 0; r < rows; r++ {
        for c := 0; c < cols; c++ {
            flatIndex := r*cols + c
            fmt.Printf("matrix[%d][%d] = flat[%d] = %d\n", r, c, flatIndex, matrix[r][c])
        }
    }
}
```

**Textual Figure — 2D to 1D Index Mapping:**

```
  3×4 matrix:
        col:  0    1    2    3
  row 0:   [  1,   2,   3,   4 ]    flat: 0  1  2  3
  row 1:   [  5,   6,   7,   8 ]    flat: 4  5  6  7
  row 2:   [  9,  10,  11,  12 ]    flat: 8  9  10 11

  Formula: flatIndex = row × cols + col

  matrix[0][2] = 3  →  flat[0×4+2] = flat[2] = 3   ✓
  matrix[2][3] = 12 →  flat[2×4+3] = flat[11] = 12 ✓

  Memory layout (row-major):
  ┌─┬─┬─┬─┬─┬─┬─┬─┬─┬──┬──┬──┐
  │1│2│3│4│5│6│7│8│9│10│11│12│
  └─┴─┴─┴─┴─┴─┴─┴─┴─┴──┴──┴──┘
  │-- row 0 --│-- row 1 --│--- row 2 ---│
```

---

```go
package main

import "fmt"

func main() {
    cols := 4
    flat := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12}

    // 1D to 2D
    for i, v := range flat {
        row := i / cols
        col := i % cols
        fmt.Printf("flat[%d]=%d → matrix[%d][%d]\n", i, v, row, col)
    }

    // 2D to 1D
    row, col := 2, 3
    flatIdx := row*cols + col
    fmt.Printf("\nmatrix[%d][%d] → flat[%d] = %d\n", row, col, flatIdx, flat[flatIdx])
}
```

**Why?** Flattening 2D arrays into 1D is common in competitive programming and cache-friendly code.

**Textual Figure — 1D ↔ 2D Conversion:**

```
  flat = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]   cols = 4

  1D → 2D:  row = i / cols    col = i % cols
  ───────────────────────────────────
  flat[0]  → 0/4=0, 0%4=0 → (0,0)   val=1
  flat[5]  → 5/4=1, 5%4=1 → (1,1)   val=6
  flat[11] → 11/4=2, 11%4=3 → (2,3) val=12

  2D → 1D:  flatIdx = row × cols + col
  ───────────────────────────────────
  (2,3) → 2×4+3 = 11 → flat[11] = 12  ✓

  Visual:
  flat: | 1  2  3  4 | 5  6  7  8 | 9  10 11 12 |
        |-- row 0 ---|- row 1 ----|- row 2 -----|
             i=0..3     i=4..7      i=8..11
```

---

## Example 5: Diagonal Indexing

```go
package main

import "fmt"

func main() {
    n := 5
    matrix := make([][]int, n)
    for i := range matrix {
        matrix[i] = make([]int, n)
        for j := range matrix[i] {
            matrix[i][j] = i*n + j + 1
        }
    }

    // Main diagonal: row == col
    fmt.Print("Main diagonal: ")
    for i := 0; i < n; i++ {
        fmt.Print(matrix[i][i], " ")
    }
    fmt.Println()

    // Anti-diagonal: row + col == n-1
    fmt.Print("Anti-diagonal: ")
    for i := 0; i < n; i++ {
        fmt.Print(matrix[i][n-1-i], " ")
    }
    fmt.Println()

    // All diagonals (top-left to bottom-right)
    fmt.Println("\nAll diagonals:")
    for d := -(n - 1); d < n; d++ {
        for i := 0; i < n; i++ {
            j := i - d
            if j >= 0 && j < n {
                fmt.Printf("%3d", matrix[i][j])
            }
        }
        fmt.Println()
    }
}
```

**Textual Figure — Diagonal Indexing in a Matrix:**

```
  5×5 matrix:
        0    1    2    3    4
  0 [  1,   2,   3,   4,   5 ]
  1 [  6,   7,   8,   9,  10 ]
  2 [ 11,  12,  13,  14,  15 ]
  3 [ 16,  17,  18,  19,  20 ]
  4 [ 21,  22,  23,  24,  25 ]

  Main diagonal (i == j):       1, 7, 13, 19, 25
       ╲
        1
          7
           13
             19
               25

  Anti-diagonal (i + j == n-1):  5, 9, 13, 17, 21
                                  /
                               5
                             9
                           13
                         17
                       21

  Diagonal property:
  ┌────────────────────────────────────┐
  │ Same diagonal → i - j = constant │
  │ Same anti-diag → i + j = constant│
  └────────────────────────────────────┘
```

---

```go
package main

import "fmt"

func spiralOrder(matrix [][]int) []int {
    if len(matrix) == 0 {
        return nil
    }
    rows, cols := len(matrix), len(matrix[0])
    result := make([]int, 0, rows*cols)
    top, bottom, left, right := 0, rows-1, 0, cols-1

    for top <= bottom && left <= right {
        // Traverse right
        for j := left; j <= right; j++ {
            result = append(result, matrix[top][j])
        }
        top++
        // Traverse down
        for i := top; i <= bottom; i++ {
            result = append(result, matrix[i][right])
        }
        right--
        // Traverse left
        if top <= bottom {
            for j := right; j >= left; j-- {
                result = append(result, matrix[bottom][j])
            }
            bottom--
        }
        // Traverse up
        if left <= right {
            for i := bottom; i >= top; i-- {
                result = append(result, matrix[i][left])
            }
            left++
        }
    }
    return result
}

func main() {
    matrix := [][]int{
        {1, 2, 3, 4},
        {5, 6, 7, 8},
        {9, 10, 11, 12},
    }
    fmt.Println(spiralOrder(matrix))
    // [1 2 3 4 8 12 11 10 9 5 6 7]
}
```

**Textual Figure — Spiral Order Traversal:**

```
  3×4 matrix:
  ┌────┬────┬────┬────┐
  │  1 │  2 │  3 │  4 │  → right (top row)
  ├────┼────┼────┼────┤
  │  5 │  6 │  7 │  8 │           ↓ down (right col)
  ├────┼────┼────┼────┤
  │  9 │ 10 │ 11 │ 12 │  ← left (bottom row)
  └────┴────┴────┴────┘
    ↑ up (left col)

  Traversal path:
    1 → 2 → 3 → 4
                  ↓
    5    6 ← 7    8
    ↑              ↓
    9 ← 10 ← 11 ← 12

  Result: [1, 2, 3, 4, 8, 12, 11, 10, 9, 5, 6, 7]

  Boundary pointers:
    top=0  bottom=2  left=0  right=3
    After each pass, shrink inward:
    right pass → top++
    down pass  → right--
    left pass  → bottom--
    up pass    → left++
```

---

```go
package main

import "fmt"

func main() {
    data := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11}

    // Access every 3rd element (stride = 3)
    stride := 3
    fmt.Print("Stride 3: ")
    for i := 0; i < len(data); i += stride {
        fmt.Print(data[i], " ") // 0 3 6 9
    }
    fmt.Println()

    // Access even-indexed elements (stride = 2)
    fmt.Print("Stride 2: ")
    for i := 0; i < len(data); i += 2 {
        fmt.Print(data[i], " ") // 0 2 4 6 8 10
    }
    fmt.Println()

    // Reverse stride
    fmt.Print("Reverse stride 2: ")
    for i := len(data) - 1; i >= 0; i -= 2 {
        fmt.Print(data[i], " ") // 11 9 7 5 3 1
    }
    fmt.Println()
}
```

**Textual Figure — Stride-based Access Patterns:**

```
  data = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11]
  Index:  0  1  2  3  4  5  6  7  8  9  10  11

  Stride = 3 (every 3rd element):
          ↓        ↓        ↓        ↓
         [0, _, _, 3, _, _, 6, _, _, 9, __,  __]
  Result: 0  3  6  9

  Stride = 2 (even-indexed):
          ↓     ↓     ↓     ↓     ↓      ↓
         [0, _, 2, _, 4, _, 6, _, 8, __,  10, __]
  Result: 0  2  4  6  8  10

  Reverse stride = 2 (from end):
             ↓     ↓     ↓     ↓     ↓     ↓
         [_, 1, _, 3, _, 5, _, 7, _, 9, __,  11]
  Result: 11  9  7  5  3  1

  Use cases: downsampling, channel interleaving, SIMD-style access
```

---

```go
package main

import "fmt"

func main() {
    buf := [5]int{10, 20, 30, 40, 50}
    n := len(buf)

    // Walk forward 8 positions starting from index 2
    start := 2
    fmt.Print("Circular walk: ")
    for step := 0; step < 8; step++ {
        idx := (start + step) % n
        fmt.Printf("buf[%d]=%d  ", idx, buf[idx])
    }
    fmt.Println()
    // buf[2]=30  buf[3]=40  buf[4]=50  buf[0]=10  buf[1]=20  buf[2]=30  ...

    // Walk backward
    fmt.Print("Backward: ")
    for step := 0; step < 8; step++ {
        idx := ((start - step) % n + n) % n // handles negative mod
        fmt.Printf("buf[%d]=%d  ", idx, buf[idx])
    }
    fmt.Println()
}
```

**Textual Figure — Circular Indexing (Ring Buffer):**

```
  buf = [10, 20, 30, 40, 50]     n = 5

  Circular layout:
          ┌────┐
     ┌────┤ 30 ├────┐       Index mapping:
     │ 20 │ [2] │ 40 │       idx = (start + step) % n
     │ [1]└────┘ [3] │
     └──┐          ┌─┘       start=2:
        │ 10   50  │         step 0: (2+0)%5 = 2 → 30
        │ [0]  [4] │         step 1: (2+1)%5 = 3 → 40
        └──────────┘         step 2: (2+2)%5 = 4 → 50
                             step 3: (2+3)%5 = 0 → 10  ← wraps!
  Linear view:               step 4: (2+4)%5 = 1 → 20
  [10, 20, 30, 40, 50]
          ↑ start=2
          30 → 40 → 50 → 10 → 20 → 30 → 40 → 50
                    wrap ↗

  Backward: idx = ((start - step) % n + n) % n
    step 0: 2 → 30
    step 1: 1 → 20
    step 2: 0 → 10
    step 3: 4 → 50  ← wraps backward!
```

---

```go
package main

import "fmt"

func main() {
    n := 4

    // 90° clockwise rotation index mapping
    // matrix[i][j] → matrix[j][n-1-i]
    fmt.Println("90° CW rotation mapping:")
    for i := 0; i < n; i++ {
        for j := 0; j < n; j++ {
            newRow, newCol := j, n-1-i
            fmt.Printf("(%d,%d)→(%d,%d)  ", i, j, newRow, newCol)
        }
        fmt.Println()
    }

    // Transpose mapping: matrix[i][j] ↔ matrix[j][i]
    matrix := [][]int{
        {1, 2, 3},
        {4, 5, 6},
        {7, 8, 9},
    }
    fmt.Println("\nTranspose:")
    for i := 0; i < 3; i++ {
        for j := i + 1; j < 3; j++ {
            matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]
        }
    }
    for _, row := range matrix {
        fmt.Println(row) // [[1 4 7], [2 5 8], [3 6 9]]
    }
}
```

**Textual Figure — Coordinate Transforms:**

```
  90° Clockwise Rotation: (i,j) → (j, n-1-i)

  Original:        After 90° CW:
  ┌───┬───┬───┐    ┌───┬───┬───┐
  │ 1 │ 2 │ 3 │    │ 7 │ 4 │ 1 │
  ├───┼───┼───┤    ├───┼───┼───┤
  │ 4 │ 5 │ 6 │ →  │ 8 │ 5 │ 2 │
  ├───┼───┼───┤    ├───┼───┼───┤
  │ 7 │ 8 │ 9 │    │ 9 │ 6 │ 3 │
  └───┴───┴───┘    └───┴───┴───┘

  Example: (0,0)→(0,2)  (0,2)→(2,2)  (2,2)→(2,0)  (2,0)→(0,0)

  Transpose: (i,j) ↔ (j,i)  — swap across main diagonal

  Original:        Transposed:
  ┌───┬───┬───┐    ┌───┬───┬───┐
  │ 1 │ 2 │ 3 │    │ 1 │ 4 │ 7 │
  ├───┼───┼───┤    ├───┼───┼───┤
  │ 4 │ 5 │ 6 │ →  │ 2 │ 5 │ 8 │
  ├───┼───┼───┤    ├───┼───┼───┤
  │ 7 │ 8 │ 9 │    │ 3 │ 6 │ 9 │
  └───┴───┴───┘    └───┴───┴───┘

  Tip: 90° CW = Transpose + Reverse each row
```

---

```go
package main

import "fmt"

// safeGet returns value at index or default if out of bounds
func safeGet(s []int, i int, defaultVal int) int {
    if i < 0 || i >= len(s) {
        return defaultVal
    }
    return s[i]
}

// neighbors4 returns valid 4-directional neighbors in a grid
func neighbors4(grid [][]int, r, c int) []int {
    rows, cols := len(grid), len(grid[0])
    dirs := [][2]int{{-1, 0}, {1, 0}, {0, -1}, {0, 1}}
    var result []int
    for _, d := range dirs {
        nr, nc := r+d[0], c+d[1]
        if nr >= 0 && nr < rows && nc >= 0 && nc < cols {
            result = append(result, grid[nr][nc])
        }
    }
    return result
}

func main() {
    s := []int{10, 20, 30}
    fmt.Println(safeGet(s, 1, -1))    // 20
    fmt.Println(safeGet(s, 5, -1))    // -1 (out of bounds)
    fmt.Println(safeGet(s, -1, -1))   // -1 (negative)

    grid := [][]int{
        {1, 2, 3},
        {4, 5, 6},
        {7, 8, 9},
    }
    fmt.Println("Neighbors of (1,1):", neighbors4(grid, 1, 1)) // [2 8 4 6]
    fmt.Println("Neighbors of (0,0):", neighbors4(grid, 0, 0)) // [4 2] — corner
}
```

**Textual Figure — Bounds Checking & Neighbor Access:**

```
  safeGet — safe 1D access:
  s = [10, 20, 30]    len = 3
        0    1    2
  ─ ─ ─┌────┬────┬────┐─ ─ ─
 X ← │ 10 │ 20 │ 30 │ → X      X = out of bounds
  ─ ─ ─└────┴────┴────┘─ ─ ─
  idx -1 → -1 (default)    idx 5 → -1 (default)
  idx  1 → 20 (valid)

  neighbors4 — 4-directional grid neighbors:
  ┌───┬───┬───┐
  │ 1 │ 2 │ 3 │    For (1,1)=5:
  ├───┼───┼───┤      UP    (0,1)=2  ✓
  │ 4 │[5]│ 6 │      DOWN  (2,1)=8  ✓
  ├───┼───┼───┤      LEFT  (1,0)=4  ✓
  │ 7 │ 8 │ 9 │      RIGHT (1,2)=6  ✓
  └───┴───┴───┘    Result: [2, 8, 4, 6]

  For (0,0)=1 (corner):
  ┌───┬───┬───┐      UP    (-1,0) ✗
  │[1]│ 2 │ 3 │      DOWN  (1,0)=4 ✓
  ├───┼───┼───┤      LEFT  (0,-1) ✗
  │ 4 │ 5 │ 6 │      RIGHT (0,1)=2 ✓
  ├───┼───┼───┤    Result: [4, 2]
  │ 7 │ 8 │ 9 │
  └───┴───┴───┘
  dirs = [(-1,0), (1,0), (0,-1), (0,1)]
```

---

## Key Takeaways

1. **Array indexing is O(1)** — base + offset multiplication
2. **Zero-based in Go** — first element is `arr[0]`
3. **2D→1D**: `flatIndex = row*cols + col`; **1D→2D**: `row = i/cols, col = i%cols`
4. **Circular indexing**: `(i + offset) % n`
5. **Diagonal**: main diagonal `i == j`, anti-diagonal `i + j == n-1`
6. **Always bounds-check** when indices are computed dynamically

> **Next up:** Prefix Sum →
