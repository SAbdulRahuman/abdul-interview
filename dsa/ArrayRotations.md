# Phase 2: Arrays — Array Rotations

## What is Array Rotation?

Rotating an array shifts elements cyclically. A **left rotation by k** moves the first k elements to the end. A **right rotation by k** moves the last k elements to the front.

```
Left rotate [1,2,3,4,5] by 2 → [3,4,5,1,2]
Right rotate [1,2,3,4,5] by 2 → [4,5,1,2,3]
```

---

## Example 1: Right Rotate Using Reverse (O(n) time, O(1) space)

```go
package main

import "fmt"

func reverse(nums []int, l, r int) {
    for l < r {
        nums[l], nums[r] = nums[r], nums[l]
        l++
        r--
    }
}

func rotateRight(nums []int, k int) {
    n := len(nums)
    k = k % n
    if k == 0 {
        return
    }
    // [1,2,3,4,5] k=2
    reverse(nums, 0, n-1)   // [5,4,3,2,1]
    reverse(nums, 0, k-1)   // [4,5,3,2,1]
    reverse(nums, k, n-1)   // [4,5,1,2,3]
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7}
    rotateRight(nums, 3)
    fmt.Println(nums) // [5 6 7 1 2 3 4]

    nums2 := []int{-1, -100, 3, 99}
    rotateRight(nums2, 2)
    fmt.Println(nums2) // [3 99 -1 -100]
}
```

**Why triple-reverse?**
- Reverse all: puts elements in opposite order
- Reverse first k: fixes the rotated portion
- Reverse rest: fixes the remaining portion

**Textual Figure — Right Rotate Using Triple Reverse:**

```
  nums = [1, 2, 3, 4, 5, 6, 7]   k = 3

  Step 1: Reverse entire array
  [1, 2, 3, 4, 5, 6, 7]  →  [7, 6, 5, 4, 3, 2, 1]

  Step 2: Reverse first k=3 elements
  [7, 6, 5, 4, 3, 2, 1]  →  [5, 6, 7, 4, 3, 2, 1]
  └─reverse─┘

  Step 3: Reverse remaining elements
  [5, 6, 7, 4, 3, 2, 1]  →  [5, 6, 7, 1, 2, 3, 4]
            └──reverse──┘

  Result: [5, 6, 7, 1, 2, 3, 4]  ✓

  Visual:
  Before: |1 2 3 4|5 6 7|    k=3 from end
  After:  |5 6 7|1 2 3 4|    moved to front!
```

---

## Example 2: Left Rotate Using Reverse

```go
package main

import "fmt"

func reverse(nums []int, l, r int) {
    for l < r {
        nums[l], nums[r] = nums[r], nums[l]
        l++
        r--
    }
}

func rotateLeft(nums []int, k int) {
    n := len(nums)
    k = k % n
    if k == 0 {
        return
    }
    // [1,2,3,4,5] k=2
    reverse(nums, 0, k-1)   // [2,1,3,4,5]
    reverse(nums, k, n-1)   // [2,1,5,4,3]
    reverse(nums, 0, n-1)   // [3,4,5,1,2]
}

func main() {
    nums := []int{1, 2, 3, 4, 5}
    rotateLeft(nums, 2)
    fmt.Println(nums) // [3 4 5 1 2]

    nums2 := []int{10, 20, 30, 40, 50, 60}
    rotateLeft(nums2, 4)
    fmt.Println(nums2) // [50 60 10 20 30 40]
}
```

**Textual Figure — Left Rotate Using Triple Reverse:**

```
  nums = [1, 2, 3, 4, 5]   k = 2 (left rotate)

  Step 1: Reverse first k=2 elements
  [1, 2, 3, 4, 5]  →  [2, 1, 3, 4, 5]
  └─rev─┘

  Step 2: Reverse remaining elements
  [2, 1, 3, 4, 5]  →  [2, 1, 5, 4, 3]
         └──rev──┘

  Step 3: Reverse entire array
  [2, 1, 5, 4, 3]  →  [3, 4, 5, 1, 2]
  └────reverse────┘

  Result: [3, 4, 5, 1, 2]  ✓

  Comparison:
  Right rotate: rev ALL → rev first k → rev rest
  Left rotate:  rev first k → rev rest → rev ALL

  Left rotate by k = Right rotate by (n-k)
```

---

## Example 3: Rotate Using Extra Array

```go
package main

import "fmt"

func rotateWithCopy(nums []int, k int) []int {
    n := len(nums)
    k = k % n
    result := make([]int, n)
    for i := 0; i < n; i++ {
        result[(i+k)%n] = nums[i]
    }
    return result
}

func main() {
    nums := []int{1, 2, 3, 4, 5}
    fmt.Println("Right 2:", rotateWithCopy(nums, 2)) // [4 5 1 2 3]
    fmt.Println("Right 1:", rotateWithCopy(nums, 1)) // [5 1 2 3 4]
    fmt.Println("Right 5:", rotateWithCopy(nums, 5)) // [1 2 3 4 5] (full cycle)
}
```

**Textual Figure — Rotate Using Extra Array:**

```
  nums = [1, 2, 3, 4, 5]   k = 2 (right rotate)

  Formula: result[(i + k) % n] = nums[i]

  i=0: result[(0+2)%5] = result[2] = 1
  i=1: result[(1+2)%5] = result[3] = 2
  i=2: result[(2+2)%5] = result[4] = 3
  i=3: result[(3+2)%5] = result[0] = 4
  i=4: result[(4+2)%5] = result[1] = 5

  result: [4, 5, 1, 2, 3]  ✓

  nums:   ┌─┬─┬─┬─┬─┐
          │1│2│3│4│5│
          └─┴─┴─┴─┴─┘
           │││↓↓     each shifts right by k=2
  result: ┌─┬─┬─┬─┬─┐
          │4│5│1│2│3│
          └─┴─┴─┴─┴─┘

  Simple O(n) time but O(n) extra space.
```

---

## Example 4: Rotate Using Cyclic Replacements

```go
package main

import "fmt"

func rotateCyclic(nums []int, k int) {
    n := len(nums)
    k = k % n
    if k == 0 {
        return
    }

    count := 0 // total elements placed
    for start := 0; count < n; start++ {
        current := start
        prev := nums[start]
        for {
            next := (current + k) % n
            nums[next], prev = prev, nums[next]
            current = next
            count++
            if current == start {
                break
            }
        }
    }
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6}
    rotateCyclic(nums, 2)
    fmt.Println(nums) // [5 6 1 2 3 4]

    nums2 := []int{1, 2, 3, 4, 5, 6, 7, 8}
    rotateCyclic(nums2, 3)
    fmt.Println(nums2) // [6 7 8 1 2 3 4 5]
}
```

**Why?** Each element is moved exactly once. O(n) time, O(1) space.

**Textual Figure — Cyclic Replacement Rotation:**

```
  nums = [1, 2, 3, 4, 5, 6]   k = 2

  Start at index 0, follow the cycle:
  ┌───────────────────────┐
  │ idx 0 → idx 2 → idx 4 → idx 0 (cycle!)
  │  1→3   3→5   5→1
  └───────────────────────┘
  After cycle 1: [5, 2, 1, 4, 3, 6]
  count = 3 (moved 3 elements)

  Start at index 1, follow cycle:
  ┌───────────────────────┐
  │ idx 1 → idx 3 → idx 5 → idx 1 (cycle!)
  │  2→4   4→6   6→2
  └───────────────────────┘
  After cycle 2: [5, 6, 1, 2, 3, 4]
  count = 6 (all moved!) ✓

  Number of cycles = GCD(n, k) = GCD(6,2) = 2
```

---

## Example 5: Juggling Algorithm (GCD-based Rotation)

```go
package main

import "fmt"

func gcd(a, b int) int {
    for b != 0 {
        a, b = b, a%b
    }
    return a
}

func rotateLeftJuggling(nums []int, d int) {
    n := len(nums)
    d = d % n
    sets := gcd(n, d)

    for i := 0; i < sets; i++ {
        temp := nums[i]
        j := i
        for {
            k := (j + d) % n
            if k == i {
                break
            }
            nums[j] = nums[k]
            j = k
        }
        nums[j] = temp
    }
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12}
    rotateLeftJuggling(nums, 3)
    fmt.Println(nums) // [4 5 6 7 8 9 10 11 12 1 2 3]

    nums2 := []int{1, 2, 3, 4, 5, 6}
    rotateLeftJuggling(nums2, 2)
    fmt.Println(nums2) // [3 4 5 6 1 2]
}
```

**Textual Figure — Juggling Algorithm (GCD-Based):**

```
  nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]   d = 3
  n = 12,  GCD(12, 3) = 3  →  3 cycles

  Cycle 0 (start=0): 0→3→6→9→0
    save temp=nums[0]=1
    nums[0]=nums[3]=4
    nums[3]=nums[6]=7
    nums[6]=nums[9]=10
    nums[9]=temp=1

  Cycle 1 (start=1): 1→4→7→10→1
    save temp=nums[1]=2
    nums[1]=nums[4]=5
    nums[4]=nums[7]=8
    nums[7]=nums[10]=11
    nums[10]=temp=2

  Cycle 2 (start=2): 2→5→8→11→2
    (similar...)

  Result: [4, 5, 6, 7, 8, 9, 10, 11, 12, 1, 2, 3]  ✓

  Each element moves exactly once!
  Total cycles = GCD(n, d)
```

---

## Example 6: Search in Rotated Sorted Array

```go
package main

import "fmt"

func search(nums []int, target int) int {
    left, right := 0, len(nums)-1

    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            return mid
        }

        // Left half is sorted
        if nums[left] <= nums[mid] {
            if target >= nums[left] && target < nums[mid] {
                right = mid - 1
            } else {
                left = mid + 1
            }
        } else {
            // Right half is sorted
            if target > nums[mid] && target <= nums[right] {
                left = mid + 1
            } else {
                right = mid - 1
            }
        }
    }
    return -1
}

func main() {
    nums := []int{4, 5, 6, 7, 0, 1, 2}
    fmt.Println(search(nums, 0)) // 4
    fmt.Println(search(nums, 3)) // -1
    fmt.Println(search(nums, 5)) // 1

    nums2 := []int{3, 1}
    fmt.Println(search(nums2, 1)) // 1
}
```

**Textual Figure — Search in Rotated Sorted Array:**

```
  nums = [4, 5, 6, 7, 0, 1, 2]   target = 0

  Iteration 1: left=0, right=6, mid=3
    nums[mid]=7, not target
    nums[left]=4 ≤ nums[mid]=7 → left half [4,5,6,7] sorted
    target=0 NOT in [4..7] → go right
    left = 4

  Iteration 2: left=4, right=6, mid=5
    nums[mid]=1, not target
    nums[left]=0 ≤ nums[mid]=1 → left half [0,1] sorted
    target=0 in [0..1] → go left
    right = 4

  Iteration 3: left=4, right=4, mid=4
    nums[mid]=0 == target! → return 4  ✓

  Key Insight:
  ┌─────sorted─────┐ ┌──sorted──┐
  [4, 5, 6, 7,  |  0, 1, 2]
   left half    pivot  right half
  One half is ALWAYS sorted → check if target is in sorted half.
```

---

## Example 7: Find Minimum in Rotated Sorted Array

```go
package main

import "fmt"

func findMin(nums []int) int {
    left, right := 0, len(nums)-1

    for left < right {
        mid := left + (right-left)/2
        if nums[mid] > nums[right] {
            left = mid + 1 // min is in right half
        } else {
            right = mid // min could be mid
        }
    }
    return nums[left]
}

func main() {
    fmt.Println(findMin([]int{3, 4, 5, 1, 2}))       // 1
    fmt.Println(findMin([]int{4, 5, 6, 7, 0, 1, 2})) // 0
    fmt.Println(findMin([]int{11, 13, 15, 17}))       // 11 (not rotated)
    fmt.Println(findMin([]int{2, 1}))                  // 1
}
```

**Textual Figure — Find Minimum in Rotated Sorted Array:**

```
  nums = [4, 5, 6, 7, 0, 1, 2]

  Binary Search for Minimum:

  Iter 1: left=0, right=6, mid=3
    nums[mid]=7 > nums[right]=2 → min in RIGHT half
    left = 4

  Iter 2: left=4, right=6, mid=5
    nums[mid]=1 ≤ nums[right]=2 → min could be mid or LEFT
    right = 5

  Iter 3: left=4, right=5, mid=4
    nums[mid]=0 ≤ nums[right]=1 → right = 4

  left == right == 4 → return nums[4] = 0  ✓

  Visual Pattern:
       7
      / \
     6   \
    /     0         Sorted-rotated = two ascending segments.
   5       \        The min is the "drop-off" point.
  /         \       
  4          1
              \
               2
```

---

## Example 8: Find Rotation Count

```go
package main

import "fmt"

// In a rotated sorted array, the number of rotations = index of minimum element
func findRotationCount(nums []int) int {
    left, right := 0, len(nums)-1

    // Not rotated
    if nums[left] <= nums[right] {
        return 0
    }

    for left < right {
        mid := left + (right-left)/2
        if nums[mid] > nums[right] {
            left = mid + 1
        } else {
            right = mid
        }
    }
    return left
}

func main() {
    fmt.Println(findRotationCount([]int{15, 18, 2, 3, 6, 12})) // 2
    fmt.Println(findRotationCount([]int{7, 9, 11, 12, 5}))     // 4
    fmt.Println(findRotationCount([]int{7, 9, 11, 12, 15}))    // 0 (not rotated)
}
```

**Textual Figure — Find Rotation Count:**

```
  Rotation count = Index of minimum element

  Original sorted: [2, 3, 6, 12, 15, 18]
  After 2 rotations (right): [15, 18, 2, 3, 6, 12]
                               0   1  2  3  4   5
                                      ↑
                              min at index 2 = rotation count

  Example: nums = [7, 9, 11, 12, 5]

  Iter 1: left=0, right=4, mid=2
    nums[0]=7 > nums[4]=5 → it IS rotated
    nums[mid]=11 > nums[right]=5 → left = 3

  Iter 2: left=3, right=4, mid=3
    nums[mid]=12 > nums[right]=5 → left = 4

  left == right == 4 → rotation count = 4  ✓

  [7, 9, 11, 12, 5]  was rotated 4 times from [5, 7, 9, 11, 12]
```

---

## Example 9: Rotate a 2D Matrix Layer by Layer

```go
package main

import "fmt"

func rotateMatrixRing(matrix [][]int, ring, k int) {
    n := len(matrix)
    top, left := ring, ring
    bottom, right := n-1-ring, n-1-ring

    // Extract ring elements
    var elements []int
    for j := left; j <= right; j++ { elements = append(elements, matrix[top][j]) }
    for i := top + 1; i <= bottom; i++ { elements = append(elements, matrix[i][right]) }
    for j := right - 1; j >= left; j-- { elements = append(elements, matrix[bottom][j]) }
    for i := bottom - 1; i > top; i-- { elements = append(elements, matrix[i][left]) }

    // Rotate
    m := len(elements)
    k = k % m
    rotated := make([]int, m)
    for i := 0; i < m; i++ {
        rotated[(i+k)%m] = elements[i]
    }

    // Put back
    idx := 0
    for j := left; j <= right; j++ { matrix[top][j] = rotated[idx]; idx++ }
    for i := top + 1; i <= bottom; i++ { matrix[i][right] = rotated[idx]; idx++ }
    for j := right - 1; j >= left; j-- { matrix[bottom][j] = rotated[idx]; idx++ }
    for i := bottom - 1; i > top; i-- { matrix[i][left] = rotated[idx]; idx++ }
}

func main() {
    matrix := [][]int{
        {1, 2, 3, 4},
        {5, 6, 7, 8},
        {9, 10, 11, 12},
        {13, 14, 15, 16},
    }
    rotateMatrixRing(matrix, 0, 2) // rotate outer ring by 2
    for _, row := range matrix {
        fmt.Println(row)
    }
}
```

**Textual Figure — Rotate 2D Matrix Ring (Layer by Layer):**

```
  4×4 Matrix:
  ┌───┬───┬───┬───┐
  │ 1 │ 2 │ 3 │ 4 │  ← Outer ring (ring=0)
  ├───┼───┼───┼───┤
  │ 5 │ 6 │ 7 │ 8 │
  ├───┼───┼───┼───┤
  │ 9 │10 │11 │12 │
  ├───┼───┼───┼───┤
  │13 │14 │15 │16 │
  └───┴───┴───┴───┘

  Step 1: Extract outer ring (clockwise):
  ┌ → → → ┐
  ↑         ↓     Elements: [1,2,3,4,8,12,16,15,14,13,9,5]
  ↑         ↓
  └ ← ← ← ┘

  Step 2: Rotate by k=2 positions:
  [1,2,3,4,8,12,16,15,14,13,9,5]
  → [9,5,1,2,3,4,8,12,16,15,14,13]

  Step 3: Place back in ring positions.

  This rotates just one ring without affecting inner elements.
```

---

## Example 10: Block Swap Algorithm for Rotation

```go
package main

import "fmt"

func swap(arr []int, fi, si, d int) {
    for i := 0; i < d; i++ {
        arr[fi+i], arr[si+i] = arr[si+i], arr[fi+i]
    }
}

func blockSwapRotate(arr []int, d int) {
    n := len(arr)
    if d == 0 || d == n {
        return
    }
    d = d % n

    i, j := d, n-d
    for i != j {
        if i < j {
            // A is shorter: swap A with rightmost B
            swap(arr, d-i, d+j-i, i)
            j -= i
        } else {
            // B is shorter: swap B with leftmost A
            swap(arr, d-i, d, j)
            i -= j
        }
    }
    swap(arr, d-i, d, i)
}

func main() {
    arr := []int{1, 2, 3, 4, 5, 6, 7}
    blockSwapRotate(arr, 2)
    fmt.Println(arr) // [3 4 5 6 7 1 2]

    arr2 := []int{1, 2, 3, 4, 5, 6, 7, 8}
    blockSwapRotate(arr2, 3)
    fmt.Println(arr2) // [4 5 6 7 8 1 2 3]
}
```

**Textual Figure — Block Swap Algorithm:**

```
  arr = [1, 2, 3, 4, 5, 6, 7]   d = 2 (left rotate)

  Split into A=[1,2] and B=[3,4,5,6,7]
  A.len=2 (i=2), B.len=5 (j=5)

  Since i < j: swap A with rightmost part of B
  [1, 2, | 3, 4, 5, | 6, 7]
   A(2)      B             swap A with [6,7]
  [6, 7, | 3, 4, 5, | 1, 2]   j = j-i = 3
   done✓     new A          placed✓

  Now i=2, j=3: still i < j
  [6, 7, | 3, | 4, 5, | 1, 2]
  swap [6,7] with [4,5]
  [4, 5, | 3, | 6, 7, | 1, 2]   j = j-i = 1

  Now i=2, j=1: i > j
  swap B=[3] with leftmost of A=[4]
  [3, 5, | 4, | 6, 7, | 1, 2]   i = i-j = 1

  Now i=1, j=1: equal! Final swap.
  [3, 4, 5, 6, 7, 1, 2]  ✓

  Key: Recursively swap blocks until equal size, then one last swap.
```

---

## Comparison Table

| Method | Time | Space | Notes |
|--------|------|-------|-------|
| Triple reverse | O(n) | O(1) | Best general-purpose |
| Extra array | O(n) | O(n) | Simplest |
| Cyclic replace | O(n) | O(1) | Each element moved once |
| Juggling (GCD) | O(n) | O(1) | Good for large k |
| Block swap | O(n) | O(1) | Recursive swapping |

---

## Key Takeaways

1. **Triple reverse** is the cleanest O(n), O(1) rotation
2. **Always mod k by n** — rotating by n is a no-op
3. **Right rotate by k** = left rotate by (n-k)
4. **Rotated sorted arrays** enable O(log n) search
5. **Rotation count** = index of minimum element

> **Next up:** Partitioning →
