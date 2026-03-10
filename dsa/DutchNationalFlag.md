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

**Textual Figure — Sort Colors (DNF):**

```
  nums = [2, 0, 2, 1, 1, 0]   lo=0, mid=0, hi=5

  Invariant regions:
  [0s done | 1s done | unprocessed | 2s done]
   0..lo-1   lo..mid-1  mid..hi     hi+1..n-1

  Step 1: nums[mid]=2 → swap(mid,hi), hi--
          [0, 0, 2, 1, 1, 2]  lo=0 mid=0 hi=4
  Step 2: nums[mid]=0 → swap(lo,mid), lo++, mid++
          [0, 0, 2, 1, 1, 2]  lo=1 mid=1 hi=4
  Step 3: nums[mid]=0 → swap(lo,mid), lo++, mid++
          [0, 0, 2, 1, 1, 2]  lo=2 mid=2 hi=4
  Step 4: nums[mid]=2 → swap(mid,hi), hi--
          [0, 0, 1, 1, 2, 2]  lo=2 mid=2 hi=3
  Step 5: nums[mid]=1 → mid++
          [0, 0, 1, 1, 2, 2]  lo=2 mid=3 hi=3
  Step 6: nums[mid]=1 → mid++
          [0, 0, 1, 1, 2, 2]  lo=2 mid=4 hi=3

  mid > hi → DONE!  Result: [0, 0, 1, 1, 2, 2]  ✓
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

**Textual Figure — Step-by-Step Trace:**

```
  Input: [1, 2, 0]    lo=0, mid=0, hi=2

  Step 1: nums[mid=0]=1 (case 1)
          mid is 1, advance mid
          [1, 2, 0]  lo=0 mid=1 hi=2
          ┌─┬─┬─┐
          │1│2│0│    lo=0, mid=1, hi=2
          └─┴─┴─┘
           L M   H

  Step 2: nums[mid=1]=2 (case 2)
          swap(mid=1, hi=2), shrink hi
          [1, 0, 2]  lo=0 mid=1 hi=1
          ┌─┬─┬─┐
          │1│0│2│    lo=0, mid=1, hi=1
          └─┴─┴─┘
           L MH  ✓(placed)

  Step 3: nums[mid=1]=0 (case 0)
          swap(lo=0, mid=1), advance lo, mid
          [0, 1, 2]  lo=1 mid=2 hi=1
          ┌─┬─┬─┐
          │0│1│2│    mid=2 > hi=1 → DONE!
          └─┴─┴─┘
           ✓ ✓ ✓

  Final: [0, 1, 2]  ✓
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

**Textual Figure — Generic Three-Way Partition:**

```
  arr = [4, 9, 4, 4, 1, 9, 4, 4, 9, 4, 4, 1, 4]   pivot = 4

  Rules:
    arr[mid] < pivot  →  swap to lo, advance both
    arr[mid] == pivot →  just advance mid
    arr[mid] > pivot  →  swap to hi, shrink hi only

  Processing:
  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ 4 │ 9 │ 4 │ 4 │ 1 │ 9 │ 4 │ 4 │ 9 │ 4 │ 4 │ 1 │ 4 │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  lo,mid                                                    hi

  After processing:
  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ 1 │ 1 │ 4 │ 4 │ 4 │ 4 │ 4 │ 4 │ 4 │ 9 │ 9 │ 9 │   │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  │─< 4─│─────── == 4 ───────│─── > 4 ───│
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

**Textual Figure — Sort R/W/B Characters:**

```
  s = "BWRRWBR"    Mapping: R=0 (front), W=1 (mid), B=2 (back)

  lo=0, mid=0, hi=6

  mid=0: 'B' → swap(0,6), hi=5   s = "RWRRWBB"
  mid=0: 'R' → swap(0,0), lo=1, mid=1  s = "RWRRWBB"
  mid=1: 'W' → mid=2             s = "RWRRWBB"
  mid=2: 'R' → swap(1,2), lo=2, mid=3  s = "RRRWWBB"
  mid=3: 'W' → mid=4             s = "RRRWWBB"
  mid=4: 'W' → mid=5             s = "RRRWWBB"
  mid=5: 'B' → swap(5,5), hi=4   s = "RRRWWBB"
  mid=5 > hi=4 → DONE!

  Result: R R R W W B B
          └─red─┘ └wht┘ └blu┘

  Same DNF pattern applied to any 3-value domain!
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

**Textual Figure — Partition Negatives/Zeros/Positives:**

```
  arr = [0, -1, 2, -3, 0, 4, -5, 0, 6]

  Mapping: negative=0 (front), zero=1 (mid), positive=2 (back)

  Initial:  lo=0, mid=0, hi=8

  mid=0: 0 (zero) → mid++
  mid=1: -1 (neg) → swap(0,1), lo++, mid++
         [-1, 0, 2, -3, 0, 4, -5, 0, 6]
  mid=2: 2 (pos) → swap(2,8), hi--
         [-1, 0, 6, -3, 0, 4, -5, 0, 2]
  mid=2: 6 (pos) → swap(2,7), hi--
         [-1, 0, 0, -3, 0, 4, -5, 6, 2]
  mid=2: 0 → mid++
  mid=3: -3 (neg) → swap(1,3), lo++, mid++
         [-1, -3, 0, 0, 0, 4, -5, 6, 2]
  ...continues...

  Final: [-1, -3, -5, 0, 0, 0, 6, 4, 2]
          └─ neg ──┘  └zero┘  └─ pos ─┘
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

**Textual Figure — Four-Way Partition (Two DNF Passes):**

```
  arr = [1, 14, 5, 20, 4, 2, 54, 20, 87, 98, 3, 1, 32]
  low=10, high=20

  Pass 1: Separate [< low | ≥ low]
    Scan with lo/mid: move elements < 10 to front
    After: [1, 5, 4, 2, 3, 1 | 14, 20, 54, 20, 87, 98, 32]
            ─── < 10 ────    ─────── ≥ 10 ────────

  Pass 2: In the ≥ low portion, separate [≤ high | > high]
    Scan with mid/hi: move elements > 20 to back
    After: [1, 5, 4, 2, 3, 1 | 14, 20, 20 | 54, 87, 98, 32]
            ─── < 10 ────   ─ 10-20 ─   ─── > 20 ───

  Four regions:
  ┌─────────┬──────────┬───────────┬──────────┐
  │  < 10   │ 10 - 20  │  > 20       │          │
  └─────────┴──────────┴───────────┴──────────┘
  Two-pass DNF = four-way partition!
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

**Textual Figure — DNF with Counting Verification:**

```
  Input: [2, 0, 2, 1, 1, 0]

  Step 1: Count BEFORE sorting
    counts[0] = 2  (two 0s)
    counts[1] = 2  (two 1s)
    counts[2] = 2  (two 2s)

  Step 2: DNF Sort
    [2, 0, 2, 1, 1, 0] → [0, 0, 1, 1, 2, 2]

  Step 3: Count AFTER sorting
    afterCounts[0] = 2  ✓ matches
    afterCounts[1] = 2  ✓ matches
    afterCounts[2] = 2  ✓ matches

  Step 4: Verify sorted order
    0≤0 ≤ 1≤1 ≤ 2≤2  ✓ non-decreasing

  Verification checklist:
  ┌────────────────────────────┐
  │ ✓ Element counts match   │
  │ ✓ Array is sorted        │
  │ ✓ No elements lost/added │
  └────────────────────────────┘
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

**Textual Figure — Sort by Priority Levels:**

```
  Tasks:
  Email(2) Deploy(0) Meeting(1) Hotfix(0) Review(1) Docs(2) Test(0)

  DNF with Priority: 0=HIGH(front), 1=MED(mid), 2=LOW(back)

  Processing (lo=0, mid=0, hi=6):

  mid=0: Email(2)   → swap to hi   [Test(0)...Email(2)] hi=5
  mid=0: Test(0)    → swap to lo   lo=1, mid=1
  mid=1: Deploy(0)  → swap to lo   lo=2, mid=2
  mid=2: Meeting(1) → mid++        mid=3
  mid=3: Hotfix(0)  → swap to lo   lo=3, mid=4
  mid=4: Review(1)  → mid++        mid=5
  mid=5: Docs(2)    → swap to hi   hi=4
  mid=5 > hi=4 → DONE!

  Result:
  ┌───────────────────┬────────────────┬────────────┐
  │ Deploy Test Hotfix │ Meeting Review │ Docs Email │
  │    HIGH (0)        │  MEDIUM (1)    │  LOW (2)   │
  └───────────────────┴────────────────┴────────────┘
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

**Textual Figure — 3-Way QuickSort with DNF:**

```
  arr = [4, 9, 4, 4, 1, 9, 4, 4, 9, 4, 4, 1, 4]

  Level 0: pivot = 4 (random)
  DNF partition:
  [1, 1 | 4, 4, 4, 4, 4, 4, 4 | 9, 9, 9]
   <4      ======= =4 =======    >4
   lt=2                        gt=9

  Recurse LEFT [1, 1]: already sorted (base case)
  Recurse RIGHT [9, 9, 9]: already sorted (base case)
  SKIP equal region [4,4,4,...] ← KEY advantage!

  Standard QuickSort vs 3-Way:
  ┌──────────────┬───────────────┬─────────────────┐
  │              │ Standard       │ 3-Way            │
  ├──────────────┼───────────────┼─────────────────┤
  │ Partitions   │ [< p] [p] [>p] │ [<p] [==p] [>p]   │
  │ Duplicates   │ O(n²) worst    │ O(n) per group   │
  │ Recurse into │ Both sides     │ Skip equal region│
  └──────────────┴───────────────┴─────────────────┘
```

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

**Textual Figure — Segregate 0s and 1s:**

```
  arr = [0, 1, 0, 1, 1, 0, 1, 0, 0, 1]
  lo=0, hi=9

  Simplified DNF (only 2 values → 2 pointers enough):

  lo=0: arr[0]=0 → lo++           lo=1
  lo=1: arr[1]=1, arr[hi=9]=1 → hi--  hi=8
  lo=1: arr[1]=1, arr[hi=8]=0 → swap  lo=2, hi=7
        [0, 0, 0, 1, 1, 0, 1, 1, 0, 1]
  lo=2: arr[2]=0 → lo++           lo=3
  lo=3: arr[3]=1, arr[hi=7]=1 → hi--  hi=6
  lo=3: arr[3]=1, arr[hi=6]=1 → hi--  hi=5
  lo=3: arr[3]=1, arr[hi=5]=0 → swap  lo=4, hi=4
        [0, 0, 0, 0, 1, 0, 1, 1, 0, 1]
  lo=4: arr[4]=1, arr[hi=4]=1 → hi--  hi=3
  lo=4 > hi=3 → DONE!

  Result: [0, 0, 0, 0, 0, 1, 1, 1, 1, 1]  ✓
           └── all 0s ──┘  └── all 1s ──┘

  With only 2 values, DNF simplifies to 2 pointers!
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
