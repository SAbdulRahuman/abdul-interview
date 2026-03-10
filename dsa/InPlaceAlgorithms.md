# Phase 2: Arrays — In-Place Algorithms

## What are In-Place Algorithms?

An **in-place algorithm** transforms data using O(1) extra space (or O(log n) for recursion stack). It modifies the input array directly without allocating proportional additional memory.

---

## Example 1: Reverse Array In-Place

```go
package main

import "fmt"

func reverse(nums []int) {
    left, right := 0, len(nums)-1
    for left < right {
        nums[left], nums[right] = nums[right], nums[left]
        left++
        right--
    }
}

func main() {
    nums := []int{1, 2, 3, 4, 5}
    reverse(nums)
    fmt.Println(nums) // [5 4 3 2 1]

    nums2 := []int{1, 2}
    reverse(nums2)
    fmt.Println(nums2) // [2 1]

    nums3 := []int{42}
    reverse(nums3)
    fmt.Println(nums3) // [42]
}
```

**Textual Figure — Reverse Array In-Place:**

```
  nums = [1, 2, 3, 4, 5]

  Step 1: swap(0,4)  [5, 2, 3, 4, 1]
          L=0            R=4
          ↑                  ↑
          └──── swap ────┘

  Step 2: swap(1,3)  [5, 4, 3, 2, 1]
             L=1       R=3

  Step 3: L=2 >= R=2 → stop

  Result: [5, 4, 3, 2, 1]  ✓

  O(n/2) swaps = O(n) time, O(1) space
  ┌─┬─┬─┬─┬─┐       ┌─┬─┬─┬─┬─┐
  │1│2│3│4│5│  →    │5│4│3│2│1│
  └─┴─┴─┴─┴─┘       └─┴─┴─┴─┴─┘
```

---

## Example 2: Remove Element In-Place

```go
package main

import "fmt"

func removeElement(nums []int, val int) int {
    slow := 0
    for _, num := range nums {
        if num != val {
            nums[slow] = num
            slow++
        }
    }
    return slow
}

func main() {
    nums := []int{3, 2, 2, 3}
    k := removeElement(nums, 3)
    fmt.Println(nums[:k]) // [2 2]

    nums2 := []int{0, 1, 2, 2, 3, 0, 4, 2}
    k2 := removeElement(nums2, 2)
    fmt.Println(nums2[:k2]) // [0 1 3 0 4]
}
```

**Textual Figure — Remove Element In-Place:**

```
  nums = [0, 1, 2, 2, 3, 0, 4, 2]   val = 2

  slow = write pointer, read scans entire array:

  read=0: 0≠2 → write nums[0]=0, slow=1
  read=1: 1≠2 → write nums[1]=1, slow=2
  read=2: 2==2 → SKIP
  read=3: 2==2 → SKIP
  read=4: 3≠2 → write nums[2]=3, slow=3
  read=5: 0≠2 → write nums[3]=0, slow=4
  read=6: 4≠2 → write nums[4]=4, slow=5
  read=7: 2==2 → SKIP

  [0, 1, 2, 2, 3, 0, 4, 2]
               ↓ ↓     ↓ ↓     ↓   (kept)
  [0, 1, 3, 0, 4, ?, ?, ?]    k=5

  Return 5 → first 5 elements are the result
```

---

## Example 3: Move Zeroes to End (Maintaining Order)

```go
package main

import "fmt"

func moveZeroes(nums []int) {
    insertPos := 0
    for _, num := range nums {
        if num != 0 {
            nums[insertPos] = num
            insertPos++
        }
    }
    for i := insertPos; i < len(nums); i++ {
        nums[i] = 0
    }
}

func main() {
    nums := []int{0, 1, 0, 3, 12}
    moveZeroes(nums)
    fmt.Println(nums) // [1 3 12 0 0]

    nums2 := []int{0, 0, 1}
    moveZeroes(nums2)
    fmt.Println(nums2) // [1 0 0]

    nums3 := []int{1, 0, 0, 0, 2, 0, 3}
    moveZeroes(nums3)
    fmt.Println(nums3) // [1 2 3 0 0 0 0]
}
```

**Textual Figure — Move Zeroes (Two-Pass):**

```
  nums = [0, 1, 0, 3, 12]

  Pass 1: Copy non-zeros to front
  insertPos=0
    0 → skip
    1 → nums[0]=1, insertPos=1
    0 → skip
    3 → nums[1]=3, insertPos=2
   12 → nums[2]=12, insertPos=3

  After pass 1: [1, 3, 12, 3, 12]
                              ↑ insertPos=3

  Pass 2: Fill rest with zeros
  nums[3]=0, nums[4]=0

  Result: [1, 3, 12, 0, 0]  ✓

  This approach maintains relative order of non-zeros!
  Unlike swap approach, this does overwrite (still O(1) space).
```

---

## Example 4: Rotate Image 90° Clockwise In-Place

```go
package main

import "fmt"

func rotate(matrix [][]int) {
    n := len(matrix)

    // Step 1: Transpose (swap matrix[i][j] with matrix[j][i])
    for i := 0; i < n; i++ {
        for j := i + 1; j < n; j++ {
            matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]
        }
    }

    // Step 2: Reverse each row
    for i := 0; i < n; i++ {
        left, right := 0, n-1
        for left < right {
            matrix[i][left], matrix[i][right] = matrix[i][right], matrix[i][left]
            left++
            right--
        }
    }
}

func main() {
    matrix := [][]int{
        {1, 2, 3},
        {4, 5, 6},
        {7, 8, 9},
    }
    rotate(matrix)
    for _, row := range matrix {
        fmt.Println(row)
    }
    // [7 4 1]
    // [8 5 2]
    // [9 6 3]
}
```

**Textual Figure — Rotate Image 90° CW In-Place:**

```
  Step 1: TRANSPOSE (swap across diagonal)
  Original:         After transpose:
  ┌───┬───┬───┐     ┌───┬───┬───┐
  │ 1 │ 2 │ 3 │     │ 1 │ 4 │ 7 │
  ├───┼───┼───┤     ├───┼───┼───┤
  │ 4 │ 5 │ 6 │ →   │ 2 │ 5 │ 8 │
  ├───┼───┼───┤     ├───┼───┼───┤
  │ 7 │ 8 │ 9 │     │ 3 │ 6 │ 9 │
  └───┴───┴───┘     └───┴───┴───┘

  Step 2: REVERSE each row
  After reverse:     Result:
  ┌───┬───┬───┐
  │ 7 │ 4 │ 1 │     90° clockwise rotation!
  ├───┼───┼───┤
  │ 8 │ 5 │ 2 │     Only swaps used
  ├───┼───┼───┤     O(n²) time, O(1) space
  │ 9 │ 6 │ 3 │
  └───┴───┴───┘
```

---

## Example 5: Set Matrix Zeroes In-Place

```go
package main

import "fmt"

func setZeroes(matrix [][]int) {
    rows, cols := len(matrix), len(matrix[0])
    firstRowZero := false
    firstColZero := false

    // Check if first row/col have zeros
    for j := 0; j < cols; j++ {
        if matrix[0][j] == 0 {
            firstRowZero = true
        }
    }
    for i := 0; i < rows; i++ {
        if matrix[i][0] == 0 {
            firstColZero = true
        }
    }

    // Use first row/col as markers
    for i := 1; i < rows; i++ {
        for j := 1; j < cols; j++ {
            if matrix[i][j] == 0 {
                matrix[i][0] = 0
                matrix[0][j] = 0
            }
        }
    }

    // Zero out based on markers
    for i := 1; i < rows; i++ {
        for j := 1; j < cols; j++ {
            if matrix[i][0] == 0 || matrix[0][j] == 0 {
                matrix[i][j] = 0
            }
        }
    }

    // Handle first row/col
    if firstRowZero {
        for j := 0; j < cols; j++ {
            matrix[0][j] = 0
        }
    }
    if firstColZero {
        for i := 0; i < rows; i++ {
            matrix[i][0] = 0
        }
    }
}

func main() {
    matrix := [][]int{
        {1, 1, 1},
        {1, 0, 1},
        {1, 1, 1},
    }
    setZeroes(matrix)
    for _, row := range matrix {
        fmt.Println(row)
    }
    // [1 0 1]
    // [0 0 0]
    // [1 0 1]
}
```

**Textual Figure — Set Matrix Zeroes In-Place:**

```
  Original:          Use first row/col as markers:
  ┌───┬───┬───┐
  │ 1 │ 1 │ 1 │    Step 1: Find zeros → (1,1) is 0
  ├───┼───┼───┤             Mark: matrix[1][0]=0, matrix[0][1]=0
  │ 1 │ 0 │ 1 │
  ├───┼───┼───┤    Markers (first row & col):
  │ 1 │ 1 │ 1 │    row 0: [1, 0, 1]  ← col 1 marked
  └───┴───┴───┘    col 0: [1, 0, 1]  ← row 1 marked

  Step 2: Zero out rows/cols based on markers
  Row 1 has marker 0 → zero entire row 1
  Col 1 has marker 0 → zero entire col 1

  Result:
  ┌───┬───┬───┐
  │ 1 │ 0 │ 1 │    O(1) extra space!
  ├───┼───┼───┤    (used first row/col as storage)
  │ 0 │ 0 │ 0 │
  ├───┼───┼───┤
  │ 1 │ 0 │ 1 │
  └───┴───┴───┘
```

---

## Example 6: In-Place Deduplication of Sorted Array

```go
package main

import "fmt"

// Remove duplicates: keep at most k occurrences
func removeDuplicatesK(nums []int, k int) int {
    if len(nums) <= k {
        return len(nums)
    }
    slow := k
    for fast := k; fast < len(nums); fast++ {
        if nums[fast] != nums[slow-k] {
            nums[slow] = nums[fast]
            slow++
        }
    }
    return slow
}

func main() {
    // Keep at most 1
    nums1 := []int{1, 1, 2, 2, 3}
    k1 := removeDuplicatesK(nums1, 1)
    fmt.Println(nums1[:k1]) // [1 2 3]

    // Keep at most 2
    nums2 := []int{1, 1, 1, 2, 2, 3}
    k2 := removeDuplicatesK(nums2, 2)
    fmt.Println(nums2[:k2]) // [1 1 2 2 3]

    // Keep at most 3
    nums3 := []int{0, 0, 0, 0, 1, 1, 1, 1, 2, 2}
    k3 := removeDuplicatesK(nums3, 3)
    fmt.Println(nums3[:k3]) // [0 0 0 1 1 1 2 2]
}
```

**Textual Figure — In-Place Deduplication (Keep at Most K):**

```
  nums = [1, 1, 1, 2, 2, 3]   k = 2 (keep at most 2)

  slow starts at k=2, fast scans from k:

  fast=2: nums[2]=1, nums[slow-k]=nums[0]=1 → same! SKIP
  fast=3: nums[3]=2, nums[slow-k]=nums[0]=1 → diff! copy
    nums[2]=2, slow=3
    [1, 1, 2, 2, 2, 3]
            s        f
  fast=4: nums[4]=2, nums[slow-k]=nums[1]=1 → diff! copy
    nums[3]=2, slow=4
    [1, 1, 2, 2, 2, 3]
               s     f
  fast=5: nums[5]=3, nums[slow-k]=nums[2]=2 → diff! copy
    nums[4]=3, slow=5
    [1, 1, 2, 2, 3, 3]
                  s  f

  Result: [1, 1, 2, 2, 3]  (first 5 elements)  ✓

  Key trick: compare nums[fast] with nums[slow-k]
  If same, we already have k copies → skip!
```

---

## Example 7: In-Place Partition (Lomuto's Scheme)

```go
package main

import "fmt"

func partition(nums []int, pivotIdx int) int {
    pivot := nums[pivotIdx]
    // Move pivot to end
    nums[pivotIdx], nums[len(nums)-1] = nums[len(nums)-1], nums[pivotIdx]

    storeIdx := 0
    for i := 0; i < len(nums)-1; i++ {
        if nums[i] < pivot {
            nums[storeIdx], nums[i] = nums[i], nums[storeIdx]
            storeIdx++
        }
    }
    // Move pivot to final position
    nums[storeIdx], nums[len(nums)-1] = nums[len(nums)-1], nums[storeIdx]
    return storeIdx
}

func main() {
    nums := []int{3, 6, 8, 1, 4, 7, 2, 5}
    fmt.Println("Before:", nums)
    pivotPos := partition(nums, 4) // pivot = 4
    fmt.Printf("After partition around 4: %v (pivot at index %d)\n", nums, pivotPos)
    // All elements < 4 are left of pivotPos, all >= 4 are right
}
```

**Textual Figure — Lomuto Partition Scheme:**

```
  nums = [3, 6, 8, 1, 4, 7, 2, 5]   pivot = nums[4] = 4

  Step 1: Move pivot to end
  [3, 6, 8, 1, 5, 7, 2, 4]
                          ↑ pivot at end

  Step 2: Partition (storeIdx tracks boundary)
  storeIdx=0:
    i=0: 3<4 → swap(0,0), store=1  [3, 6, 8, 1, 5, 7, 2, 4]
    i=1: 6≥4 → skip
    i=2: 8≥4 → skip
    i=3: 1<4 → swap(1,3), store=2  [3, 1, 8, 6, 5, 7, 2, 4]
    i=4: 5≥4 → skip
    i=5: 7≥4 → skip
    i=6: 2<4 → swap(2,6), store=3  [3, 1, 2, 6, 5, 7, 8, 4]

  Step 3: Swap pivot into position
  swap(3, 7): [3, 1, 2, 4, 5, 7, 8, 6]
                       ↑
                    pivot at index 3
       < 4       pivot     ≥ 4
  [3, 1, 2]  |  [4]  |  [5, 7, 8, 6]
```

---

## Example 8: Fisher-Yates Shuffle In-Place

```go
package main

import (
    "fmt"
    "math/rand"
)

func shuffle(nums []int) {
    for i := len(nums) - 1; i > 0; i-- {
        j := rand.Intn(i + 1)
        nums[i], nums[j] = nums[j], nums[i]
    }
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    fmt.Println("Before:", nums)
    shuffle(nums)
    fmt.Println("After:", nums) // random permutation

    // Verify all elements present
    sum := 0
    for _, v := range nums {
        sum += v
    }
    fmt.Println("Sum:", sum) // always 55
}
```

**Why Fisher-Yates?** Every permutation is equally likely. O(n) time, O(1) space.

**Textual Figure — Fisher-Yates Shuffle:**

```
  nums = [1, 2, 3, 4, 5]

  i=4: j=rand(0..4), say j=2 → swap(4,2)
       [1, 2, 5, 4, 3]          █ = locked
                     █

  i=3: j=rand(0..3), say j=0 → swap(3,0)
       [4, 2, 5, 1, 3]
                  █  █

  i=2: j=rand(0..2), say j=2 → swap(2,2) (no change)
       [4, 2, 5, 1, 3]
              █  █  █

  i=1: j=rand(0..1), say j=0 → swap(1,0)
       [2, 4, 5, 1, 3]
           █  █  █  █

  Done: [2, 4, 5, 1, 3]  (random permutation)

  Key: each element has equal 1/n probability
  for each position. Scan right→left, shrinking
  the random range each step.
```

---

## Example 9: Next Permutation In-Place

```go
package main

import "fmt"

func nextPermutation(nums []int) {
    n := len(nums)

    // Step 1: Find largest i such that nums[i] < nums[i+1]
    i := n - 2
    for i >= 0 && nums[i] >= nums[i+1] {
        i--
    }

    if i >= 0 {
        // Step 2: Find largest j such that nums[j] > nums[i]
        j := n - 1
        for nums[j] <= nums[i] {
            j--
        }
        // Step 3: Swap
        nums[i], nums[j] = nums[j], nums[i]
    }

    // Step 4: Reverse from i+1 to end
    left, right := i+1, n-1
    for left < right {
        nums[left], nums[right] = nums[right], nums[left]
        left++
        right--
    }
}

func main() {
    test := func(nums []int) {
        before := fmt.Sprint(nums)
        nextPermutation(nums)
        fmt.Printf("%s → %v\n", before, nums)
    }

    test([]int{1, 2, 3})    // [1 2 3] → [1 3 2]
    test([]int{3, 2, 1})    // [3 2 1] → [1 2 3]
    test([]int{1, 1, 5})    // [1 1 5] → [1 5 1]
    test([]int{1, 3, 2})    // [1 3 2] → [2 1 3]
    test([]int{2, 3, 1})    // [2 3 1] → [3 1 2]
}
```

**Textual Figure — Next Permutation Algorithm:**

```
  nums = [1, 3, 2]

  Step 1: Find rightmost i where nums[i] < nums[i+1]
    i=1: 3 ≥ 2 → no
    i=0: 1 < 3 → YES!  i=0

  Step 2: Find rightmost j where nums[j] > nums[i]
    j=2: 2 > 1 → YES!  j=2

  Step 3: Swap nums[0] and nums[2]
    [1, 3, 2] → [2, 3, 1]

  Step 4: Reverse from i+1 to end
    reverse [3, 1] → [1, 3]
    [2, 1, 3]  ✓

  Permutation order: 123 → 132 → 213 → 231 → 312 → 321
                               ↑         ↑
                          [1,3,2]     [2,1,3]

  Edge case: [3, 2, 1] (last permutation)
    Step 1: no i found → just reverse entire array
    → [1, 2, 3] (wraps to first permutation)
```

---

## Example 10: Rearrange Array — Positive and Negative Alternating

```go
package main

import "fmt"

// Rearrange so positive and negative alternate
// Assumes equal number of positives and negatives
func rearrange(nums []int) []int {
    n := len(nums)
    result := make([]int, n) // NOT truly in-place, but shows the logic
    posIdx, negIdx := 0, 1

    for _, num := range nums {
        if num >= 0 {
            result[posIdx] = num
            posIdx += 2
        } else {
            result[negIdx] = num
            negIdx += 2
        }
    }
    return result
}

// True in-place approach using cyclic sort idea
func rearrangeInPlace(nums []int) {
    // Separate positives and negatives while preserving order
    pos := make([]int, 0)
    neg := make([]int, 0)
    for _, n := range nums {
        if n >= 0 {
            pos = append(pos, n)
        } else {
            neg = append(neg, n)
        }
    }
    // Interleave back
    pi, ni := 0, 0
    for i := 0; i < len(nums); i++ {
        if i%2 == 0 && pi < len(pos) {
            nums[i] = pos[pi]
            pi++
        } else if ni < len(neg) {
            nums[i] = neg[ni]
            ni++
        }
    }
}

func main() {
    nums := []int{3, 1, -2, -5, 2, -4}
    fmt.Println(rearrange(nums)) // [3 -2 1 -5 2 -4]

    nums2 := []int{-1, 2, -3, 4, -5, 6}
    rearrangeInPlace(nums2)
    fmt.Println(nums2) // [2 -1 4 -3 6 -5]
}
```

**Textual Figure — Rearrange Positive/Negative Alternating:**

```
  nums = [3, 1, -2, -5, 2, -4]

  Goal: [+, -, +, -, +, -]

  posIdx starts at 0 (even positions)
  negIdx starts at 1 (odd positions)

  3  → positive → result[0]=3,  posIdx=2
  1  → positive → result[2]=1,  posIdx=4
  -2 → negative → result[1]=-2, negIdx=3
  -5 → negative → result[3]=-5, negIdx=5
  2  → positive → result[4]=2,  posIdx=6
  -4 → negative → result[5]=-4, negIdx=7

  result: [3, -2, 1, -5, 2, -4]
           +   -  +   -  +   -    ✓

  Note: This uses O(n) extra space.
  True in-place alternating is much harder
  and often requires O(n²) time to maintain order.
```

---

## Key Takeaways

1. **In-place = O(1) extra space** — modify input directly
2. **Swap-based**: reverse, rotate, shuffle, partition
3. **Two-pointer overwrite**: remove elements, deduplicate
4. **Use input as storage**: first row/col as markers (set zeroes)
5. **Trade-off**: in-place saves memory but can be harder to reason about
6. **Common patterns**: slow/fast pointers, working from back to front

> **Next up:** Array Rotations →
