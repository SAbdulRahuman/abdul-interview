# Phase 16: Sorting Algorithms — In-Place Sorting

## Overview

An **in-place** sorting algorithm sorts using O(1) extra memory (or O(log n) for the call stack). It rearranges elements within the original array without allocating a proportional auxiliary array.

| In-Place (O(1) extra) | Not In-Place |
|------------------------|--------------|
| Bubble, Selection, Insertion | Merge Sort (standard) |
| Quick Sort (O(log n) stack) | Counting Sort |
| Heap Sort | Radix Sort |
| Shell Sort | Bucket Sort |

---

## Example 1: In-Place Swap-Based Sorting (Bubble Sort)

```go
package main

import "fmt"

func bubbleSort(arr []int) {
	n := len(arr)
	for i := 0; i < n-1; i++ {
		swapped := false
		for j := 0; j < n-1-i; j++ {
			if arr[j] > arr[j+1] {
				arr[j], arr[j+1] = arr[j+1], arr[j]
				swapped = true
			}
		}
		if !swapped { break }
	}
}

func main() {
	arr := []int{64, 34, 25, 12, 22, 11, 90}
	bubbleSort(arr) // O(1) extra space
	fmt.Println(arr) // [11 12 22 25 34 64 90]
}
```

---

## Example 2: In-Place Partition (Quick Sort Core)

```go
package main

import "fmt"

func partition(arr []int, lo, hi int) int {
	pivot := arr[hi]
	i := lo
	for j := lo; j < hi; j++ {
		if arr[j] <= pivot {
			arr[i], arr[j] = arr[j], arr[i]
			i++
		}
	}
	arr[i], arr[hi] = arr[hi], arr[i]
	return i
}

func quickSort(arr []int, lo, hi int) {
	if lo < hi {
		p := partition(arr, lo, hi)
		quickSort(arr, lo, p-1)
		quickSort(arr, p+1, hi)
	}
}

func main() {
	arr := []int{10, 80, 30, 90, 40, 50, 70}
	quickSort(arr, 0, len(arr)-1) // O(log n) stack space
	fmt.Println(arr) // [10 30 40 50 70 80 90]
}
```

---

## Example 3: In-Place Reversal

```go
package main

import "fmt"

func reverseInPlace(arr []int) {
	left, right := 0, len(arr)-1
	for left < right {
		arr[left], arr[right] = arr[right], arr[left]
		left++
		right--
	}
}

func main() {
	arr := []int{1, 2, 3, 4, 5}
	reverseInPlace(arr)
	fmt.Println(arr) // [5 4 3 2 1]
}
```

---

## Example 4: In-Place Dutch National Flag (3-Way Partition)

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
		}
	}
}

func main() {
	nums := []int{2, 0, 2, 1, 1, 0}
	sortColors(nums) // O(1) space, single pass
	fmt.Println(nums) // [0 0 1 1 2 2]
}
```

---

## Example 5: In-Place Merge (O(1) Space Merge Sort)

```go
package main

import "fmt"

// In-place merge using rotation — O(n²) but O(1) space
func inPlaceMerge(arr []int, lo, mid, hi int) {
	i, j := lo, mid+1
	for i <= mid && j <= hi {
		if arr[i] <= arr[j] {
			i++
		} else {
			val := arr[j]
			// Shift elements right
			for k := j; k > i; k-- {
				arr[k] = arr[k-1]
			}
			arr[i] = val
			i++
			mid++
			j++
		}
	}
}

func inPlaceMergeSort(arr []int, lo, hi int) {
	if lo < hi {
		mid := lo + (hi-lo)/2
		inPlaceMergeSort(arr, lo, mid)
		inPlaceMergeSort(arr, mid+1, hi)
		inPlaceMerge(arr, lo, mid, hi)
	}
}

func main() {
	arr := []int{12, 11, 13, 5, 6, 7}
	inPlaceMergeSort(arr, 0, len(arr)-1)
	fmt.Println(arr) // [5 6 7 11 12 13]
}
```

---

## Example 6: In-Place Array Rotation (for In-Place Algorithms)

```go
package main

import "fmt"

func reverse(arr []int, lo, hi int) {
	for lo < hi {
		arr[lo], arr[hi] = arr[hi], arr[lo]
		lo++
		hi--
	}
}

func rotateLeft(arr []int, k int) {
	n := len(arr)
	k %= n
	// Three reversals — O(1) space
	reverse(arr, 0, k-1)
	reverse(arr, k, n-1)
	reverse(arr, 0, n-1)
}

func rotateRight(arr []int, k int) {
	n := len(arr)
	k %= n
	reverse(arr, 0, n-1)
	reverse(arr, 0, k-1)
	reverse(arr, k, n-1)
}

func main() {
	arr := []int{1, 2, 3, 4, 5, 6, 7}
	rotateLeft(arr, 2)
	fmt.Println("Left by 2:", arr) // [3 4 5 6 7 1 2]

	arr2 := []int{1, 2, 3, 4, 5, 6, 7}
	rotateRight(arr2, 3)
	fmt.Println("Right by 3:", arr2) // [5 6 7 1 2 3 4]
}
```

---

## Example 7: In-Place Move Zeroes to End

```go
package main

import "fmt"

func moveZeroes(nums []int) {
	writeIdx := 0
	for _, v := range nums {
		if v != 0 {
			nums[writeIdx] = v
			writeIdx++
		}
	}
	for writeIdx < len(nums) {
		nums[writeIdx] = 0
		writeIdx++
	}
}

func main() {
	nums := []int{0, 1, 0, 3, 12}
	moveZeroes(nums)
	fmt.Println(nums) // [1 3 12 0 0]
}
```

---

## Example 8: In-Place Remove Duplicates from Sorted Array

```go
package main

import "fmt"

func removeDuplicates(nums []int) int {
	if len(nums) == 0 { return 0 }
	writeIdx := 1
	for i := 1; i < len(nums); i++ {
		if nums[i] != nums[i-1] {
			nums[writeIdx] = nums[i]
			writeIdx++
		}
	}
	return writeIdx
}

func main() {
	nums := []int{0, 0, 1, 1, 1, 2, 2, 3, 3, 4}
	n := removeDuplicates(nums)
	fmt.Println(nums[:n]) // [0 1 2 3 4]
}
```

---

## Example 9: Shell Sort (In-Place, Better Than Insertion)

```go
package main

import "fmt"

func shellSort(arr []int) {
	n := len(arr)
	// Start with large gap, reduce
	for gap := n / 2; gap > 0; gap /= 2 {
		for i := gap; i < n; i++ {
			temp := arr[i]
			j := i
			for j >= gap && arr[j-gap] > temp {
				arr[j] = arr[j-gap]
				j -= gap
			}
			arr[j] = temp
		}
	}
}

func main() {
	arr := []int{12, 34, 54, 2, 3}
	shellSort(arr) // O(1) extra space
	fmt.Println(arr) // [2 3 12 34 54]
}
```

---

## Example 10: In-Place Sorting Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== In-Place Sorting Summary ===")
	fmt.Println()

	fmt.Println("Definition:")
	fmt.Println("  O(1) extra memory (excluding input & call stack)")
	fmt.Println("  Some count O(log n) stack as 'in-place' (quicksort)")
	fmt.Println()

	fmt.Println("In-place algorithms:")
	fmt.Println("  ┌──────────────────┬───────────┬────────────┬────────┐")
	fmt.Println("  │ Algorithm        │ Time      │ Space      │ Stable │")
	fmt.Println("  ├──────────────────┼───────────┼────────────┼────────┤")
	fmt.Println("  │ Bubble Sort      │ O(n²)     │ O(1)       │ Yes    │")
	fmt.Println("  │ Selection Sort   │ O(n²)     │ O(1)       │ No     │")
	fmt.Println("  │ Insertion Sort   │ O(n²)     │ O(1)       │ Yes    │")
	fmt.Println("  │ Shell Sort       │ O(n^1.5)  │ O(1)       │ No     │")
	fmt.Println("  │ Heap Sort        │ O(n lg n) │ O(1)       │ No     │")
	fmt.Println("  │ Quick Sort       │ O(n lg n) │ O(lg n)*   │ No     │")
	fmt.Println("  └──────────────────┴───────────┴────────────┴────────┘")
	fmt.Println("  * Quick sort uses O(log n) stack space")
	fmt.Println()

	fmt.Println("Key techniques for in-place operations:")
	fmt.Println("  • Swap: exchange two elements")
	fmt.Println("  • Partition: rearrange around pivot")
	fmt.Println("  • Two pointers: read/write pointers")
	fmt.Println("  • Reversal: three-reversal rotation")
	fmt.Println("  • Overwrite: use negative/sentinel values")
}
```

---

## Key Takeaways

1. In-place = O(1) extra space (input array modified directly)
2. All O(n²) simple sorts are in-place
3. Heap sort is the only O(n log n) in-place sort that's always O(n log n)
4. In-place merge sort exists but trades time O(n²) for space O(1)
5. Key pattern: two-pointer / read-write pointer approach

> **Next up:** Divide and Conquer in Sorting →
