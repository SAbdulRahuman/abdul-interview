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

---

## Example 4: 1D ↔ 2D Index Conversion

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

---

## Example 6: Spiral Order Indexing

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

---

## Example 7: Stride-based Access

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

---

## Example 8: Circular Indexing (Ring Buffer)

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

---

## Example 9: Index Mapping — Coordinate Transforms

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

---

## Example 10: Bounds Checking and Safe Access

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

---

## Key Takeaways

1. **Array indexing is O(1)** — base + offset multiplication
2. **Zero-based in Go** — first element is `arr[0]`
3. **2D→1D**: `flatIndex = row*cols + col`; **1D→2D**: `row = i/cols, col = i%cols`
4. **Circular indexing**: `(i + offset) % n`
5. **Diagonal**: main diagonal `i == j`, anti-diagonal `i + j == n-1`
6. **Always bounds-check** when indices are computed dynamically

> **Next up:** Prefix Sum →
