# Phase 2: Arrays — Dutch National Flag Algorithm

## What is the Dutch National Flag Problem?

Given an array with three distinct values (like 0, 1, 2), sort them in a single pass using **O(n) time and O(1) space**. Proposed by Edsger Dijkstra.

The algorithm uses three pointers:
- `lo`: boundary of region 0 (left)
- `mid`: current element being examined
- `hi`: boundary of region 2 (right)

```
[0 0 0 | 1 1 1 | ? ? ? | 2 2 2]
       lo       mid     hi
```

---

## Example 1: Sort Colors (LeetCode 75)

```go
package main

import "fmt"

func sortColors(nums []int) {
    lo, mid, hi := 0, 0, len(nums)-1

    for mid <= hi {
        switch nums[mid] {
        case 0:
            nums[lo], nums[mid] = nums[mid], nums[lo]
            lo++
            mid++
        case 1:
            mid++
        case 2:
            nums[mid], nums[hi] = nums[hi], nums[mid]
            hi--
            // Don't advance mid! The swapped element needs checking
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

    nums3 := []int{0}
    sortColors(nums3)
    fmt.Println(nums3) // [0]
}
```

---

## Example 2: Tracing the Algorithm Step by Step

```go
package main

import "fmt"

func sortColorsVerbose(nums []int) {
    lo, mid, hi := 0, 0, len(nums)-1
    step := 0

    for mid <= hi {
        step++
        fmt.Printf("Step %d: lo=%d mid=%d hi=%d arr=%v → ", step, lo, mid, hi, nums)
        switch nums[mid] {
        case 0:
            nums[lo], nums[mid] = nums[mid], nums[lo]
            fmt.Printf("swap(%d,%d), advance lo,mid\n", lo, mid)
            lo++
            mid++
        case 1:
            fmt.Println("mid is 1, advance mid")
            mid++
        case 2:
            nums[mid], nums[hi] = nums[hi], nums[mid]
            fmt.Printf("swap(%d,%d), shrink hi\n", mid, hi)
            hi--
        }
    }
    fmt.Printf("Final: %v\n\n", nums)
}

func main() {
    sortColorsVerbose([]int{2, 0, 2, 1, 1, 0})
    sortColorsVerbose([]int{1, 2, 0})
}
```

---

## Example 3: Generic Three-Way Partition

```go
package main

import "fmt"

func threeWayPartition(arr []int, pivot int) {
    lo, mid, hi := 0, 0, len(arr)-1

    for mid <= hi {
        if arr[mid] < pivot {
            arr[lo], arr[mid] = arr[mid], arr[lo]
            lo++
            mid++
        } else if arr[mid] > pivot {
            arr[mid], arr[hi] = arr[hi], arr[mid]
            hi--
        } else {
            mid++
        }
    }
}

func main() {
    arr := []int{4, 9, 4, 4, 1, 9, 4, 4, 9, 4, 4, 1, 4}
    threeWayPartition(arr, 4)
    fmt.Println(arr) // [1 1 4 4 4 4 4 4 4 9 9 9] — all < 4, then 4s, then > 4

    arr2 := []int{3, 5, 3, 7, 3, 1, 3, 8, 3, 2}
    threeWayPartition(arr2, 3)
    fmt.Println(arr2) // [2 1 3 3 3 3 8 7 5 ...] — partitioned around 3
}
```

---

## Example 4: Sort Array of 'R', 'W', 'B' Characters

```go
package main

import "fmt"

func sortRWB(s []byte) {
    lo, mid, hi := 0, 0, len(s)-1

    for mid <= hi {
        switch s[mid] {
        case 'R':
            s[lo], s[mid] = s[mid], s[lo]
            lo++
            mid++
        case 'W':
            mid++
        case 'B':
            s[mid], s[hi] = s[hi], s[mid]
            hi--
        }
    }
}

func main() {
    s := []byte("BWRRWBR")
    sortRWB(s)
    fmt.Println(string(s)) // RRRWWBB

    s2 := []byte("BBBWWWRRR")
    sortRWB(s2)
    fmt.Println(string(s2)) // RRRWWWBBB
}
```

---

## Example 5: Partition Negatives, Zeros, Positives

```go
package main

import "fmt"

func partitionNZP(arr []int) {
    lo, mid, hi := 0, 0, len(arr)-1

    for mid <= hi {
        if arr[mid] < 0 {
            arr[lo], arr[mid] = arr[mid], arr[lo]
            lo++
            mid++
        } else if arr[mid] > 0 {
            arr[mid], arr[hi] = arr[hi], arr[mid]
            hi--
        } else {
            mid++
        }
    }
}

func main() {
    arr := []int{0, -1, 2, -3, 0, 4, -5, 0, 6}
    partitionNZP(arr)
    fmt.Println(arr) // [-1 -3 -5 0 0 0 6 4 2] — negatives, zeros, positives

    arr2 := []int{3, 0, -2, 0, 5, -1}
    partitionNZP(arr2)
    fmt.Println(arr2) // [-2 -1 0 0 5 3]
}
```

---

## Example 6: Four-Way Partition Using DNF Twice

```go
package main

import "fmt"

func fourWayPartition(arr []int, low, high int) {
    // First pass: partition into [< low | >= low]
    lo, mid, hi := 0, 0, len(arr)-1
    for mid <= hi {
        if arr[mid] < low {
            arr[lo], arr[mid] = arr[mid], arr[lo]
            lo++
            mid++
        } else {
            mid++
        }
    }

    // Second pass on the right portion: partition [low..high | > high]
    mid = lo
    hi = len(arr) - 1
    for mid <= hi {
        if arr[mid] > high {
            arr[mid], arr[hi] = arr[hi], arr[mid]
            hi--
        } else {
            mid++
        }
    }
}

func main() {
    arr := []int{1, 14, 5, 20, 4, 2, 54, 20, 87, 98, 3, 1, 32}
    fourWayPartition(arr, 10, 20)
    fmt.Println(arr)
    // [< 10] | [10-20] | [> 20]
}
```

---

## Example 7: DNF with Counting Verification

```go
package main

import "fmt"

func sortAndVerify(nums []int) bool {
    // Count expected
    counts := [3]int{}
    for _, v := range nums {
        counts[v]++
    }

    // DNF sort
    lo, mid, hi := 0, 0, len(nums)-1
    for mid <= hi {
        switch nums[mid] {
        case 0:
            nums[lo], nums[mid] = nums[mid], nums[lo]
            lo++
            mid++
        case 1:
            mid++
        case 2:
            nums[mid], nums[hi] = nums[hi], nums[mid]
            hi--
        }
    }

    // Verify
    afterCounts := [3]int{}
    for _, v := range nums {
        afterCounts[v]++
    }

    // Check sorted
    for i := 1; i < len(nums); i++ {
        if nums[i] < nums[i-1] {
            return false
        }
    }
    return counts == afterCounts
}

func main() {
    tests := [][]int{
        {2, 0, 2, 1, 1, 0},
        {0, 0, 0},
        {2, 2, 2},
        {1, 0, 2, 0, 1, 2, 0, 1},
        {0, 1, 2},
    }
    for _, test := range tests {
        original := make([]int, len(test))
        copy(original, test)
        ok := sortAndVerify(test)
        fmt.Printf("%v → %v (valid: %v)\n", original, test, ok)
    }
}
```

---

## Example 8: Sort by Priority Levels

```go
package main

import "fmt"

type Task struct {
    Name     string
    Priority int // 0=high, 1=medium, 2=low
}

func sortByPriority(tasks []Task) {
    lo, mid, hi := 0, 0, len(tasks)-1

    for mid <= hi {
        switch tasks[mid].Priority {
        case 0: // high → front
            tasks[lo], tasks[mid] = tasks[mid], tasks[lo]
            lo++
            mid++
        case 1: // medium → middle
            mid++
        case 2: // low → back
            tasks[mid], tasks[hi] = tasks[hi], tasks[mid]
            hi--
        }
    }
}

func main() {
    tasks := []Task{
        {"Email", 2}, {"Deploy", 0}, {"Meeting", 1},
        {"Hotfix", 0}, {"Review", 1}, {"Docs", 2}, {"Test", 0},
    }
    sortByPriority(tasks)
    for _, t := range tasks {
        priority := []string{"HIGH", "MED", "LOW"}[t.Priority]
        fmt.Printf("[%s] %s\n", priority, t.Name)
    }
    // [HIGH] Deploy, [HIGH] Test, [HIGH] Hotfix, [MED] Meeting, [MED] Review, [LOW] Docs, [LOW] Email
}
```

---

## Example 9: DNF for QuickSort with Many Duplicates

```go
package main

import (
    "fmt"
    "math/rand"
)

func quickSort3Way(arr []int, lo, hi int) {
    if lo >= hi {
        return
    }

    // Random pivot
    pivotIdx := lo + rand.Intn(hi-lo+1)
    pivot := arr[pivotIdx]

    // Three-way partition
    lt, gt := lo, hi
    i := lo
    for i <= gt {
        if arr[i] < pivot {
            arr[lt], arr[i] = arr[i], arr[lt]
            lt++
            i++
        } else if arr[i] > pivot {
            arr[i], arr[gt] = arr[gt], arr[i]
            gt--
        } else {
            i++
        }
    }
    // arr[lo..lt-1] < pivot, arr[lt..gt] == pivot, arr[gt+1..hi] > pivot
    quickSort3Way(arr, lo, lt-1)
    quickSort3Way(arr, gt+1, hi)
}

func main() {
    arr := []int{4, 9, 4, 4, 1, 9, 4, 4, 9, 4, 4, 1, 4}
    quickSort3Way(arr, 0, len(arr)-1)
    fmt.Println(arr) // [1 1 4 4 4 4 4 4 4 9 9 9]

    arr2 := []int{2, 2, 2, 1, 1, 3, 3}
    quickSort3Way(arr2, 0, len(arr2)-1)
    fmt.Println(arr2) // [1 1 2 2 2 3 3]
}
```

**Why?** Standard QuickSort degrades to O(n²) with many duplicates. Three-way partitioning handles duplicates in O(n).

---

## Example 10: DNF Variant — Segregate 0s and 1s

```go
package main

import "fmt"

func segregate01(arr []int) {
    lo, hi := 0, len(arr)-1
    for lo < hi {
        if arr[lo] == 0 {
            lo++
        } else if arr[hi] == 1 {
            hi--
        } else {
            arr[lo], arr[hi] = arr[hi], arr[lo]
            lo++
            hi--
        }
    }
}

func main() {
    arr := []int{0, 1, 0, 1, 1, 0, 1, 0, 0, 1}
    segregate01(arr)
    fmt.Println(arr) // [0 0 0 0 0 1 1 1 1 1]

    arr2 := []int{1, 1, 0, 0}
    segregate01(arr2)
    fmt.Println(arr2) // [0 0 1 1]

    arr3 := []int{1, 0}
    segregate01(arr3)
    fmt.Println(arr3) // [0 1]

    arr4 := []int{0, 0, 0}
    segregate01(arr4)
    fmt.Println(arr4) // [0 0 0]
}
```

---

## Algorithm Invariants

Throughout the DNF algorithm, these invariants hold:
```
arr[0..lo-1]     = all 0s (or < pivot)
arr[lo..mid-1]   = all 1s (or == pivot)
arr[mid..hi]     = unknown (not yet processed)
arr[hi+1..n-1]   = all 2s (or > pivot)
```

---

## Key Takeaways

1. **Single pass** — O(n) time, O(1) space
2. **Three pointers**: lo, mid, hi — mid scans, lo/hi mark boundaries
3. **Don't advance mid when swapping with hi** — need to check the new element
4. **Do advance mid when swapping with lo** — lo region is already processed
5. **Extends to**: three-way QuickSort, multi-category partitioning
6. **Common mistake**: forgetting the `hi--` without `mid++` on case 2

> **Phase 2 complete! Next up:** Phase 3 — Strings →
