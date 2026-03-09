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
