# Phase 2: Arrays — Two Pointer Technique

## What is the Two Pointer Technique?

The **two pointer technique** uses two indices that move through an array simultaneously to solve problems in O(n) time, avoiding O(n²) brute force.

---

## Patterns

| Pattern | Description | Example |
|---------|-------------|---------|
| Opposite ends | Left→ and ←Right | Two Sum (sorted), Container With Most Water |
| Same direction | Slow + Fast | Remove duplicates, Partition |
| Sliding window | Left + Right expand/shrink | Covered separately |

---

## Example 1: Two Sum on Sorted Array

```go
package main

import "fmt"

func twoSum(nums []int, target int) (int, int) {
    left, right := 0, len(nums)-1

    for left < right {
        sum := nums[left] + nums[right]
        if sum == target {
            return left, right
        } else if sum < target {
            left++ // need larger sum
        } else {
            right-- // need smaller sum
        }
    }
    return -1, -1
}

func main() {
    nums := []int{1, 3, 5, 7, 9, 11}
    i, j := twoSum(nums, 12)
    fmt.Printf("Indices: %d, %d → %d + %d = 12\n", i, j, nums[i], nums[j])
    // 1, 4 → 3 + 9 = 12

    i, j = twoSum(nums, 16)
    fmt.Printf("Indices: %d, %d → %d + %d = 16\n", i, j, nums[i], nums[j])
    // 2, 5 → 5 + 11 = 16
}
```

**Textual Figure — Two Sum (Sorted Array, Opposite-End Pointers):**

```
  nums = [1, 3, 5, 7, 9, 11]   target = 12

  Step 1: L=0, R=5 → 1+11=12 ✓ Found!
  ┌───┬───┬───┬───┬───┬────┐
  │ 1 │ 3 │ 5 │ 7 │ 9 │ 11 │
  └───┴───┴───┴───┴───┴────┘
    L→                   ←R

  But let's try target = 16:
  Step 1: L=0, R=5 →  1+11=12 < 16 → L++
  Step 2: L=1, R=5 →  3+11=14 < 16 → L++
  Step 3: L=2, R=5 →  5+11=16 = 16 ✓ Found!

  Logic:
  ┌──────────────────────────────┐
  │ sum < target → move L right  │
  │ sum > target → move R left   │
  │ sum == target → found!       │
  └──────────────────────────────┘
```

---

## Example 2: Container With Most Water

```go
package main

import "fmt"

func maxArea(height []int) int {
    left, right := 0, len(height)-1
    maxWater := 0

    for left < right {
        h := height[left]
        if height[right] < h {
            h = height[right]
        }
        width := right - left
        area := h * width
        if area > maxWater {
            maxWater = area
        }
        // Move the shorter side inward
        if height[left] < height[right] {
            left++
        } else {
            right--
        }
    }
    return maxWater
}

func main() {
    fmt.Println(maxArea([]int{1, 8, 6, 2, 5, 4, 8, 3, 7})) // 49
    fmt.Println(maxArea([]int{1, 1}))                         // 1
    fmt.Println(maxArea([]int{4, 3, 2, 1, 4}))               // 16
}
```

**Textual Figure — Container With Most Water:**

```
  height = [1, 8, 6, 2, 5, 4, 8, 3, 7]

     8 |    █        █
     7 |    █  █     █     █
     6 |    █  █     █     █
     5 |    █  █  █  █     █
     4 |    █  █  █  █  █  █
     3 |    █  █  █  █  █  █
     2 | █  █  █  █  █  █  █
     1 | █  █  █  █  █  █  █  █  █
       ─────────────────────────
         0  1  2  3  4  5  6  7  8
         L→                   ←R

  Best: L=1(h=8), R=8(h=7) → min(8,7) × (8-1) = 7×7 = 49

  Why move the shorter side?
  ─ Moving the taller side can only decrease height
  ─ Moving the shorter side might find a taller wall
  ─ Width always decreases by 1, so we need more height
```

---

## Example 3: Remove Duplicates from Sorted Array (In-Place)

```go
package main

import "fmt"

func removeDuplicates(nums []int) int {
    if len(nums) == 0 {
        return 0
    }
    slow := 0 // points to last unique element
    for fast := 1; fast < len(nums); fast++ {
        if nums[fast] != nums[slow] {
            slow++
            nums[slow] = nums[fast]
        }
    }
    return slow + 1
}

func main() {
    nums := []int{1, 1, 2, 2, 3, 3, 3, 4, 5, 5}
    k := removeDuplicates(nums)
    fmt.Println("Unique count:", k)
    fmt.Println("Result:", nums[:k]) // [1 2 3 4 5]

    nums2 := []int{0, 0, 1, 1, 1, 2, 2, 3, 3, 4}
    k2 := removeDuplicates(nums2)
    fmt.Println("Result:", nums2[:k2]) // [0 1 2 3 4]
}
```

**Textual Figure — Remove Duplicates (Slow/Fast Pointers):**

```
  nums = [1, 1, 2, 2, 3, 3, 3, 4, 5, 5]

  slow=0, fast scans forward:

  fast=1: nums[1]=1 == nums[0]=1 → skip
  fast=2: nums[2]=2 != nums[0]=1 → slow++, copy
    [1, 2, 2, 2, 3, 3, 3, 4, 5, 5]
     s     f
  fast=3: nums[3]=2 == nums[1]=2 → skip
  fast=4: nums[4]=3 != nums[1]=2 → slow++, copy
    [1, 2, 3, 2, 3, 3, 3, 4, 5, 5]
        s        f
  fast=5,6: skip (3==3)
  fast=7: nums[7]=4 != nums[2]=3 → slow++, copy
    [1, 2, 3, 4, 3, 3, 3, 4, 5, 5]
           s              f
  fast=8: nums[8]=5 != nums[3]=4 → slow++, copy
    [1, 2, 3, 4, 5, 3, 3, 4, 5, 5]
              s              f
  fast=9: skip (5==5)

  Result: first 5 elements = [1, 2, 3, 4, 5]

  Slow pointer = write position
  Fast pointer = read position (scanner)
```

---

## Example 4: Three Sum

```go
package main

import (
    "fmt"
    "sort"
)

func threeSum(nums []int) [][]int {
    sort.Ints(nums)
    var result [][]int

    for i := 0; i < len(nums)-2; i++ {
        if i > 0 && nums[i] == nums[i-1] {
            continue // skip duplicate first element
        }

        left, right := i+1, len(nums)-1
        target := -nums[i]

        for left < right {
            sum := nums[left] + nums[right]
            if sum == target {
                result = append(result, []int{nums[i], nums[left], nums[right]})
                // Skip duplicates
                for left < right && nums[left] == nums[left+1] {
                    left++
                }
                for left < right && nums[right] == nums[right-1] {
                    right--
                }
                left++
                right--
            } else if sum < target {
                left++
            } else {
                right--
            }
        }
    }
    return result
}

func main() {
    fmt.Println(threeSum([]int{-1, 0, 1, 2, -1, -4}))
    // [[-1 -1 2] [-1 0 1]]

    fmt.Println(threeSum([]int{0, 0, 0, 0}))
    // [[0 0 0]]
}
```

**Textual Figure — Three Sum:**

```
  nums = [-1, 0, 1, 2, -1, -4]
  sorted = [-4, -1, -1, 0, 1, 2]

  For each i, find pairs using two pointers:

  i=0: nums[0]=-4,  target=4
    L=1, R=5: -1+2=1 < 4 → L++
    L=2, R=5: -1+2=1 < 4 → L++
    L=3, R=5:  0+2=2 < 4 → L++
    L=4, R=5:  1+2=3 < 4 → L++  (L>=R, done)

  i=1: nums[1]=-1,  target=1
    L=2, R=5: -1+2=1 == 1 ✓ → [-1,-1,2]
    skip dups, L++, R--
    L=3, R=4:  0+1=1 == 1 ✓ → [-1,0,1]

  i=2: nums[2]=-1, same as nums[1] → SKIP (avoid duplicates)

  i=3: nums[3]=0, target=0
    L=4, R=5: 1+2=3 > 0 → R--  (L>=R, done)

  Result: [[-1,-1,2], [-1,0,1]]

  ┌─────┬────┬────┬───┬───┬───┐
  │ -4  │ -1 │ -1 │ 0 │ 1 │ 2 │
  └─────┴────┴────┴───┴───┴───┘
    i          L→          ←R
```

---

## Example 5: Move Zeroes to End

```go
package main

import "fmt"

func moveZeroes(nums []int) {
    slow := 0 // position to place next non-zero
    for fast := 0; fast < len(nums); fast++ {
        if nums[fast] != 0 {
            nums[slow], nums[fast] = nums[fast], nums[slow]
            slow++
        }
    }
}

func main() {
    nums := []int{0, 1, 0, 3, 12}
    moveZeroes(nums)
    fmt.Println(nums) // [1 3 12 0 0]

    nums2 := []int{0, 0, 1}
    moveZeroes(nums2)
    fmt.Println(nums2) // [1 0 0]

    nums3 := []int{4, 2, 4, 0, 0, 3, 0, 5, 1, 0}
    moveZeroes(nums3)
    fmt.Println(nums3) // [4 2 4 3 5 1 0 0 0 0]
}
```

**Textual Figure — Move Zeroes to End:**

```
  nums = [0, 1, 0, 3, 12]

  slow=0 (write position), fast scans:

  fast=0: nums[0]=0 → skip (it's zero)
  [0, 1, 0, 3, 12]
   s  f

  fast=1: nums[1]=1 ≠ 0 → swap(slow,fast), slow++
  [1, 0, 0, 3, 12]
      s  f

  fast=2: nums[2]=0 → skip
  [1, 0, 0, 3, 12]
      s     f

  fast=3: nums[3]=3 ≠ 0 → swap(slow,fast), slow++
  [1, 3, 0, 0, 12]
         s     f

  fast=4: nums[4]=12 ≠ 0 → swap(slow,fast), slow++
  [1, 3, 12, 0, 0]
             s     f

  Result: [1, 3, 12, 0, 0]  ✓

  All non-zeros moved left, zeros pushed right!
```

---

## Example 6: Palindrome Check

```go
package main

import (
    "fmt"
    "strings"
    "unicode"
)

func isPalindrome(s string) bool {
    s = strings.ToLower(s)
    runes := []rune(s)
    left, right := 0, len(runes)-1

    for left < right {
        // Skip non-alphanumeric
        for left < right && !unicode.IsLetter(runes[left]) && !unicode.IsDigit(runes[left]) {
            left++
        }
        for left < right && !unicode.IsLetter(runes[right]) && !unicode.IsDigit(runes[right]) {
            right--
        }
        if runes[left] != runes[right] {
            return false
        }
        left++
        right--
    }
    return true
}

func main() {
    fmt.Println(isPalindrome("A man, a plan, a canal: Panama")) // true
    fmt.Println(isPalindrome("race a car"))                      // false
    fmt.Println(isPalindrome("Was it a car or a cat I saw?"))    // true
    fmt.Println(isPalindrome(""))                                 // true
}
```

**Textual Figure — Palindrome Check (Two Pointers):**

```
  s = "A man, a plan, a canal: Panama"
  cleaned = "amanaplanacanalpanama"

  L=0           R=19
  ↓             ↓
  a m a n a p l a n a c a n a l p a n a m a
  ↑                                         ↑
  L                                         R
  a == a ✓

  Step 2: L=1, R=18:  m == m ✓
  Step 3: L=2, R=17:  a == a ✓
  Step 4: L=3, R=16:  n == n ✓
  ...continues matching...
  Step 10: L=9, R=10: a == a ✓
  L >= R → palindrome!

  Non-alphanumeric characters are skipped:
  "A man," → skip spaces/commas → compare letters only
```

---

## Example 7: Sort Colors (Dutch National Flag — Two Pointers Variant)

```go
package main

import "fmt"

func sortColors(nums []int) {
    low, mid, high := 0, 0, len(nums)-1

    for mid <= high {
        switch nums[mid] {
        case 0:
            nums[low], nums[mid] = nums[mid], nums[low]
            low++
            mid++
        case 1:
            mid++
        case 2:
            nums[mid], nums[high] = nums[high], nums[mid]
            high--
            // Don't advance mid — need to check swapped element
        }
    }
}

func main() {
    nums := []int{2, 0, 2, 1, 1, 0}
    sortColors(nums)
    fmt.Println(nums) // [0 0 1 1 2 2]

    nums2 := []int{2, 0, 1}
    sortColors(nums2)
    fmt.Println(nums2) // [0 1 2]
}
```

**Textual Figure — Sort Colors (Dutch National Flag):**

```
  nums = [2, 0, 2, 1, 1, 0]    low=0, mid=0, high=5

  Three regions:
  [0..low-1] = 0s    [low..mid-1] = 1s    [high+1..n-1] = 2s

  Step 1: nums[mid]=2 → swap(mid,high), high--
  [0, 0, 2, 1, 1, 2]   low=0 mid=0 high=4

  Step 2: nums[mid]=0 → swap(low,mid), low++, mid++
  [0, 0, 2, 1, 1, 2]   low=1 mid=1 high=4

  Step 3: nums[mid]=0 → swap(low,mid), low++, mid++
  [0, 0, 2, 1, 1, 2]   low=2 mid=2 high=4

  Step 4: nums[mid]=2 → swap(mid,high), high--
  [0, 0, 1, 1, 2, 2]   low=2 mid=2 high=3

  Step 5: nums[mid]=1 → mid++
  [0, 0, 1, 1, 2, 2]   low=2 mid=3 high=3

  Step 6: nums[mid]=1 → mid++
  [0, 0, 1, 1, 2, 2]   low=2 mid=4 high=3

  mid > high → DONE!  [0, 0, 1, 1, 2, 2] ✓

   0s     1s     2s
  └───┘ └────┘ └───┘
```

---

## Example 8: Trapping Rain Water

```go
package main

import "fmt"

func trap(height []int) int {
    left, right := 0, len(height)-1
    leftMax, rightMax := 0, 0
    water := 0

    for left < right {
        if height[left] < height[right] {
            if height[left] >= leftMax {
                leftMax = height[left]
            } else {
                water += leftMax - height[left]
            }
            left++
        } else {
            if height[right] >= rightMax {
                rightMax = height[right]
            } else {
                water += rightMax - height[right]
            }
            right--
        }
    }
    return water
}

func main() {
    fmt.Println(trap([]int{0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1})) // 6
    fmt.Println(trap([]int{4, 2, 0, 3, 2, 5}))                     // 9
    fmt.Println(trap([]int{1, 0, 1}))                               // 1
}
```

**Textual Figure — Trapping Rain Water:**

```
  height = [0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1]

     3 |                        █
     2 |          █╶╶╶╶╶╶╶╶█╶╶█
     1 |    █╶╶█  █╶█  █     █
     0 | ─  █  █  █  █  █  █  █  █  █
       ───────────────────────────
         0  1  2  3  4  5  6  7  8  9  10 11

  Two pointers: L=0, R=11
  Track leftMax and rightMax

  At each step, process the shorter side:
  - If h[L] < h[R]: water at L = leftMax - h[L]
  - Else:           water at R = rightMax - h[R]

  Water trapped:
  idx 2: leftMax=1, h=0 → 1 unit
  idx 4: leftMax=2, h=1 → 1 unit
  idx 5: leftMax=2, h=0 → 2 units
  idx 6: leftMax=2, h=1 → 1 unit
  idx 9: rightMax=2, h=1 → 1 unit
  Total = 6 units  ✓
```

---

## Example 9: Merge Two Sorted Arrays

```go
package main

import "fmt"

func merge(nums1 []int, m int, nums2 []int, n int) {
    // Fill from the back to avoid shifting
    p1 := m - 1
    p2 := n - 1
    p := m + n - 1

    for p1 >= 0 && p2 >= 0 {
        if nums1[p1] > nums2[p2] {
            nums1[p] = nums1[p1]
            p1--
        } else {
            nums1[p] = nums2[p2]
            p2--
        }
        p--
    }
    // Copy remaining from nums2
    for p2 >= 0 {
        nums1[p] = nums2[p2]
        p2--
        p--
    }
}

func main() {
    nums1 := []int{1, 2, 3, 0, 0, 0}
    merge(nums1, 3, []int{2, 5, 6}, 3)
    fmt.Println(nums1) // [1 2 2 3 5 6]

    nums2 := []int{4, 5, 6, 0, 0, 0}
    merge(nums2, 3, []int{1, 2, 3}, 3)
    fmt.Println(nums2) // [1 2 3 4 5 6]
}
```

**Textual Figure — Merge Two Sorted Arrays (Back-Fill):**

```
  nums1 = [1, 2, 3, 0, 0, 0]   m=3
  nums2 = [2, 5, 6]             n=3

  Fill from the back to avoid shifting:

  p1=2, p2=2, p=5:
    nums1[2]=3 vs nums2[2]=6 → 6 wins
    [1, 2, 3, 0, 0, 6]   p1=2 p2=1 p=4

  p1=2, p2=1, p=4:
    nums1[2]=3 vs nums2[1]=5 → 5 wins
    [1, 2, 3, 0, 5, 6]   p1=2 p2=0 p=3

  p1=2, p2=0, p=3:
    nums1[2]=3 vs nums2[0]=2 → 3 wins
    [1, 2, 3, 3, 5, 6]   p1=1 p2=0 p=2

  p1=1, p2=0, p=2:
    nums1[1]=2 vs nums2[0]=2 → 2 (nums2)
    [1, 2, 2, 3, 5, 6]   p1=1 p2=-1 p=1

  p2 < 0 → done!  Result: [1, 2, 2, 3, 5, 6]  ✓

  Key insight: filling from back means we never
  overwrite unprocessed elements!
```

---

## Example 10: Partition — Separate Odd and Even

```go
package main

import "fmt"

// Partition: all evens before odds (stable not required)
func partitionEvenOdd(nums []int) {
    left, right := 0, len(nums)-1
    for left < right {
        for left < right && nums[left]%2 == 0 {
            left++
        }
        for left < right && nums[right]%2 != 0 {
            right--
        }
        if left < right {
            nums[left], nums[right] = nums[right], nums[left]
            left++
            right--
        }
    }
}

func main() {
    nums := []int{3, 1, 2, 4, 5, 6, 7, 8}
    partitionEvenOdd(nums)
    fmt.Println(nums) // [8 6 2 4 5 1 7 3] — evens first, odds last

    nums2 := []int{1, 2, 3, 4, 5}
    partitionEvenOdd(nums2)
    fmt.Println(nums2) // [4 2 3 1 5]
}
```

**Textual Figure — Partition Odd/Even:**

```
  nums = [3, 1, 2, 4, 5, 6, 7, 8]

  L finds odds, R finds evens, then swap:

  L=0(3=odd), R=7(8=even) → swap
  [8, 1, 2, 4, 5, 6, 7, 3]   L=1 R=6

  L=1(1=odd), R=6(7=odd) → R--
  R=5(6=even) → swap(L,R)
  [8, 6, 2, 4, 5, 1, 7, 3]   L=2 R=4

  L=2(2=even) → L++
  L=3(4=even) → L++
  L=4(5=odd), R=4 → L >= R, stop

  Result: [8, 6, 2, 4 | 5, 1, 7, 3]
          └── evens ──┘  └── odds ──┘
```

---

## Example 11: Squares of a Sorted Array

```go
package main

import "fmt"

func sortedSquares(nums []int) []int {
    n := len(nums)
    result := make([]int, n)
    left, right := 0, n-1

    // Fill from largest to smallest
    for i := n - 1; i >= 0; i-- {
        lSq := nums[left] * nums[left]
        rSq := nums[right] * nums[right]
        if lSq > rSq {
            result[i] = lSq
            left++
        } else {
            result[i] = rSq
            right--
        }
    }
    return result
}

func main() {
    fmt.Println(sortedSquares([]int{-4, -1, 0, 3, 10})) // [0 1 9 16 100]
    fmt.Println(sortedSquares([]int{-7, -3, 2, 3, 11}))  // [4 9 9 49 121]
    fmt.Println(sortedSquares([]int{-5, -3, -2, -1}))    // [1 4 9 25]
}
```

**Textual Figure — Squares of Sorted Array:**

```
  nums = [-4, -1, 0, 3, 10]    Squares: [16, 1, 0, 9, 100]

  Note: squares of negatives can be larger than positives!
  Use two pointers: compare |left| vs |right|

  Fill result from the BACK (largest first):

  i=4: L=0(-4²=16) vs R=4(10²=100) → 100 wins
    result = [_, _, _, _, 100]   R--

  i=3: L=0(-4²=16) vs R=3(3²=9) → 16 wins
    result = [_, _, _, 16, 100]   L++

  i=2: L=1(-1²=1) vs R=3(3²=9) → 9 wins
    result = [_, _, 9, 16, 100]   R--

  i=1: L=1(-1²=1) vs R=2(0²=0) → 1 wins
    result = [_, 1, 9, 16, 100]   L++

  i=0: L=2(0²=0) vs R=2 → 0
    result = [0, 1, 9, 16, 100]   ✓
```

---

## Key Takeaways

1. **Opposite-end pointers**: sorted array problems → O(n)
2. **Same-direction pointers**: in-place removal/partitioning → O(n)
3. **Always ask**: is the array sorted? Can I sort it?
4. **Skip duplicates** carefully in problems like Three Sum
5. **Fill from back** when merging in-place
6. **Two pointers reduce O(n²) brute force to O(n)**

> **Next up:** Sliding Window →
