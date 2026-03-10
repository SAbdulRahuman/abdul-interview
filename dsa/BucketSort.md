# Phase 16: Sorting Algorithms — Bucket Sort

## Overview

**Bucket Sort** distributes elements into buckets, sorts each bucket individually, then concatenates results. It achieves O(n) average time when input is uniformly distributed.

| Aspect | Detail |
|--------|--------|
| **Time** | O(n + k) average, O(n²) worst |
| **Space** | O(n + k) |
| **Stable** | Yes (if bucket sort is stable) |
| **Best for** | Uniformly distributed data |

---

## Example 1: Basic Bucket Sort (Floats in [0, 1))

```go
package main

import (
	"fmt"
	"sort"
)

func bucketSort(arr []float64) {
	n := len(arr)
	if n <= 1 { return }

	// Create n buckets
	buckets := make([][]float64, n)
	for i := range buckets {
		buckets[i] = []float64{}
	}

	// Distribute into buckets
	for _, v := range arr {
		idx := int(v * float64(n))
		if idx >= n { idx = n - 1 }
		buckets[idx] = append(buckets[idx], v)
	}

	// Sort each bucket
	for i := range buckets {
		sort.Float64s(buckets[i])
	}

	// Concatenate
	idx := 0
	for _, bucket := range buckets {
		for _, v := range bucket {
			arr[idx] = v
			idx++
		}
	}
}

func main() {
	arr := []float64{0.78, 0.17, 0.39, 0.26, 0.72, 0.94, 0.21, 0.12, 0.23, 0.68}
	bucketSort(arr)
	fmt.Println(arr)
}
```

**Textual Figure:**

```
Bucket Sort (Floats): arr = [0.78, 0.17, 0.39, 0.26, 0.72, 0.94, 0.21, 0.12, 0.23, 0.68]
n = 10 buckets, idx = int(v × 10)

  Distribute into buckets:
  ┌──────────┬───────────────────────┐
  │ Bucket 0 │                       │
  │ Bucket 1 │ 0.17, 0.12            │
  │ Bucket 2 │ 0.26, 0.21, 0.23      │
  │ Bucket 3 │ 0.39                  │
  │ Bucket 4 │                       │
  │ Bucket 5 │                       │
  │ Bucket 6 │ 0.68                  │
  │ Bucket 7 │ 0.78, 0.72            │
  │ Bucket 8 │                       │
  │ Bucket 9 │ 0.94                  │
  └──────────┴───────────────────────┘

  Sort each bucket → Concatenate:
  [0.12, 0.17, 0.21, 0.23, 0.26, 0.39, 0.68, 0.72, 0.78, 0.94]
```

---

## Example 2: Bucket Sort for Integers

```go
package main

import (
	"fmt"
	"sort"
)

func bucketSortInt(arr []int) {
	if len(arr) <= 1 { return }

	minVal, maxVal := arr[0], arr[0]
	for _, v := range arr {
		if v < minVal { minVal = v }
		if v > maxVal { maxVal = v }
	}

	bucketCount := len(arr)
	rang := maxVal - minVal + 1
	buckets := make([][]int, bucketCount)
	for i := range buckets {
		buckets[i] = []int{}
	}

	for _, v := range arr {
		idx := (v - minVal) * (bucketCount - 1) / rang
		buckets[idx] = append(buckets[idx], v)
	}

	for i := range buckets {
		sort.Ints(buckets[i])
	}

	idx := 0
	for _, bucket := range buckets {
		for _, v := range bucket {
			arr[idx] = v
			idx++
		}
	}
}

func main() {
	arr := []int{42, 32, 33, 52, 37, 47, 51}
	bucketSortInt(arr)
	fmt.Println(arr) // [32 33 37 42 47 51 52]
}
```

**Textual Figure:**

```
Bucket Sort (Integers): arr = [42, 32, 33, 52, 37, 47, 51]
min=32, max=52, range=21, 7 buckets
idx = (v - 32) × 6 / 21

  ┌──────────┬────────────────┐
  │ Bucket 0 │ 32, 33         │  (32-34)
  │ Bucket 1 │ 37             │  (35-37)
  │ Bucket 2 │                │
  │ Bucket 3 │ 42             │  (41-43)
  │ Bucket 4 │ 47             │  (44-46)
  │ Bucket 5 │ 51, 52         │  (50-52)
  │ Bucket 6 │                │
  └──────────┴────────────────┘

  Sort buckets → Concatenate:
  ┌────┬────┬────┬────┬────┬────┬────┐
  │ 32 │ 33 │ 37 │ 42 │ 47 │ 51 │ 52 │
  └────┴────┴────┴────┴────┴────┴────┘
```

---

## Example 3: Bucket Sort with Custom Bucket Count

```go
package main

import (
	"fmt"
	"sort"
)

func bucketSortN(arr []int, numBuckets int) {
	if len(arr) <= 1 { return }

	minVal, maxVal := arr[0], arr[0]
	for _, v := range arr {
		if v < minVal { minVal = v }
		if v > maxVal { maxVal = v }
	}

	rang := float64(maxVal-minVal+1) / float64(numBuckets)
	buckets := make([][]int, numBuckets)
	for i := range buckets {
		buckets[i] = []int{}
	}

	for _, v := range arr {
		idx := int(float64(v-minVal) / rang)
		if idx >= numBuckets { idx = numBuckets - 1 }
		buckets[idx] = append(buckets[idx], v)
	}

	fmt.Println("Bucket distribution:")
	for i, b := range buckets {
		fmt.Printf("  Bucket %d: %v\n", i, b)
	}

	for i := range buckets {
		sort.Ints(buckets[i])
	}

	idx := 0
	for _, bucket := range buckets {
		for _, v := range bucket {
			arr[idx] = v
			idx++
		}
	}
}

func main() {
	arr := []int{29, 25, 3, 49, 9, 37, 21, 43}
	bucketSortN(arr, 4)
	fmt.Println("Sorted:", arr)
}
```

**Textual Figure:**

```
Custom Bucket Count: arr = [29,25,3,49,9,37,21,43], numBuckets=4
min=3, max=49, range=47/4 ≈ 11.75 per bucket

  ┌──────────┬────────────────┬─────────────┐
  │ Bucket 0 │ 3, 9           │ [3..14]     │
  │ Bucket 1 │ 25, 21         │ [15..26]    │
  │ Bucket 2 │ 29, 37         │ [27..38]    │
  │ Bucket 3 │ 49, 43         │ [39..49]    │
  └──────────┴────────────────┴─────────────┘

  Sort each → Concatenate:
  ┌───┬───┬────┬────┬────┬────┬────┬────┐
  │ 3 │ 9 │ 21 │ 25 │ 29 │ 37 │ 43 │ 49 │
  └───┴───┴────┴────┴────┴────┴────┴────┘

  Fewer buckets → bigger buckets → more per-bucket work
  More buckets  → smaller buckets → more overhead
```

---

## Example 4: Top Frequent Elements Using Bucket Sort

```go
package main

import "fmt"

func topKFrequent(nums []int, k int) []int {
	// Count frequency
	freq := map[int]int{}
	for _, v := range nums {
		freq[v]++
	}

	// Bucket by frequency — bucket[i] = elements with frequency i
	n := len(nums)
	buckets := make([][]int, n+1)
	for num, count := range freq {
		buckets[count] = append(buckets[count], num)
	}

	// Collect from highest frequency
	result := []int{}
	for i := n; i >= 0 && len(result) < k; i-- {
		result = append(result, buckets[i]...)
	}
	return result[:k]
}

func main() {
	fmt.Println(topKFrequent([]int{1, 1, 1, 2, 2, 3}, 2)) // [1 2]
	fmt.Println(topKFrequent([]int{1}, 1))                  // [1]
}
```

**Textual Figure:**

```
Top K Frequent: nums = [1,1,1,2,2,3], k=2

  Step 1 — Count frequencies:
  ┌─────┬───────┐
  │ Num │ Freq  │
  ├─────┼───────┤
  │  1  │  3    │
  │  2  │  2    │
  │  3  │  1    │
  └─────┴───────┘

  Step 2 — Bucket by frequency (idx = freq):
  ┌──────────┬──────────┐
  │ Bucket 1 │ [3]      │  ← freq=1
  │ Bucket 2 │ [2]      │  ← freq=2
  │ Bucket 3 │ [1]      │  ← freq=3
  │ Bucket 4 │ []       │
  │ Bucket 5 │ []       │
  │ Bucket 6 │ []       │
  └──────────┴──────────┘

  Step 3 — Collect from highest bucket, k=2:
  Bucket 3: [1] → result = [1]
  Bucket 2: [2] → result = [1, 2]  ← done (len=k)
```

---

## Example 5: Sort Points by Distance (Bucket Sort)

```go
package main

import (
	"fmt"
	"math"
	"sort"
)

type Point struct {
	X, Y float64
}

func (p Point) dist() float64 {
	return math.Sqrt(p.X*p.X + p.Y*p.Y)
}

func sortPointsByDist(points []Point, numBuckets int) {
	if len(points) <= 1 { return }

	maxDist := 0.0
	for _, p := range points {
		if d := p.dist(); d > maxDist { maxDist = d }
	}

	buckets := make([][]Point, numBuckets)
	for i := range buckets {
		buckets[i] = []Point{}
	}

	for _, p := range points {
		idx := int(p.dist() / maxDist * float64(numBuckets-1))
		if idx >= numBuckets { idx = numBuckets - 1 }
		buckets[idx] = append(buckets[idx], p)
	}

	for i := range buckets {
		sort.Slice(buckets[i], func(a, b int) bool {
			return buckets[i][a].dist() < buckets[i][b].dist()
		})
	}

	idx := 0
	for _, bucket := range buckets {
		for _, p := range bucket {
			points[idx] = p
			idx++
		}
	}
}

func main() {
	points := []Point{{3, 4}, {1, 1}, {5, 0}, {0, 2}, {2, 3}}
	sortPointsByDist(points, 5)
	for _, p := range points {
		fmt.Printf("(%.0f,%.0f) dist=%.2f\n", p.X, p.Y, p.dist())
	}
}
```

**Textual Figure:**

```
Sort Points by Distance: [(3,4),(1,1),(5,0),(0,2),(2,3)]

  Compute distances from origin:
  ┌───────┬──────────────────────┐
  │ Point │ dist = √(x²+y²)     │
  ├───────┼──────────────────────┤
  │ (3,4) │ 5.00                 │
  │ (1,1) │ 1.41                 │
  │ (5,0) │ 5.00                 │
  │ (0,2) │ 2.00                 │
  │ (2,3) │ 3.61                 │
  └───────┴──────────────────────┘
  maxDist = 5.00, 5 buckets

  ┌──────────┬─────────────────┐
  │ Bucket 0 │                 │ [0..1)
  │ Bucket 1 │ (1,1)           │ [1..2)
  │ Bucket 2 │ (0,2)           │ [2..3)
  │ Bucket 3 │ (2,3)           │ [3..4)
  │ Bucket 4 │ (3,4),(5,0)     │ [4..5]
  └──────────┴─────────────────┘

  Sorted: (1,1) → (0,2) → (2,3) → (3,4) → (5,0)
```

---

## Example 6: Sort Strings by Length (Bucket Sort)

```go
package main

import "fmt"

func sortByLength(words []string) []string {
	if len(words) == 0 { return words }

	maxLen := 0
	for _, w := range words {
		if len(w) > maxLen { maxLen = len(w) }
	}

	// Buckets indexed by length
	buckets := make([][]string, maxLen+1)
	for _, w := range words {
		buckets[len(w)] = append(buckets[len(w)], w)
	}

	result := make([]string, 0, len(words))
	for _, bucket := range buckets {
		result = append(result, bucket...)
	}
	return result
}

func main() {
	words := []string{"go", "python", "c", "java", "rust", "js"}
	sorted := sortByLength(words)
	fmt.Println(sorted) // [c go js java rust python]
}
```

**Textual Figure:**

```
Sort Strings by Length: ["go","python","c","java","rust","js"]

  Buckets indexed by string length:
  ┌──────────┬────────────────┐
  │ Len 1    │ "c"            │
  │ Len 2    │ "go", "js"     │
  │ Len 3    │                │
  │ Len 4    │ "java", "rust" │
  │ Len 5    │                │
  │ Len 6    │ "python"       │
  └──────────┴────────────────┘

  Concatenate: ["c", "go", "js", "java", "rust", "python"]

  Note: Within same length, original order is preserved (stable).
```

---

## Example 7: Sort Ages (Small Range → Bucket/Counting Hybrid)

```go
package main

import "fmt"

type Person struct {
	Name string
	Age  int
}

func sortByAge(people []Person) {
	// Ages typically 0-120
	buckets := make([][]Person, 121)
	for _, p := range people {
		buckets[p.Age] = append(buckets[p.Age], p)
	}

	idx := 0
	for _, bucket := range buckets {
		for _, p := range bucket {
			people[idx] = p
			idx++
		}
	}
}

func main() {
	people := []Person{
		{"Alice", 30}, {"Bob", 25}, {"Charlie", 30},
		{"Dave", 20}, {"Eve", 25},
	}
	sortByAge(people)
	for _, p := range people {
		fmt.Printf("%s(%d) ", p.Name, p.Age)
	}
	// Dave(20) Bob(25) Eve(25) Alice(30) Charlie(30) — stable!
	fmt.Println()
}
```

**Textual Figure:**

```
Sort Ages: [(Alice,30),(Bob,25),(Charlie,30),(Dave,20),(Eve,25)]

  Buckets indexed by age (0..120):
  ┌───────────┬────────────────────────┐
  │ Age   0  │                        │
  │  ...     │                        │
  │ Age  20  │ Dave                   │
  │  ...     │                        │
  │ Age  25  │ Bob, Eve               │  ← stable: Bob before Eve
  │  ...     │                        │
  │ Age  30  │ Alice, Charlie         │  ← stable: Alice before Charlie
  │  ...     │                        │
  └───────────┴────────────────────────┘

  Result: Dave(20) Bob(25) Eve(25) Alice(30) Charlie(30)
  Counting/bucket hybrid — O(n + k) where k=121
```

---

## Example 8: Maximum Gap Using Bucket Sort (LC 164)

```go
package main

import (
	"fmt"
	"math"
)

func maximumGap(nums []int) int {
	n := len(nums)
	if n < 2 { return 0 }

	minVal, maxVal := nums[0], nums[0]
	for _, v := range nums {
		if v < minVal { minVal = v }
		if v > maxVal { maxVal = v }
	}
	if minVal == maxVal { return 0 }

	// Bucket size = ceiling of (max-min)/(n-1)
	// By pigeonhole, max gap must be between buckets
	bucketSize := int(math.Ceil(float64(maxVal-minVal) / float64(n-1)))
	bucketCount := (maxVal-minVal)/bucketSize + 1

	bucketMin := make([]int, bucketCount)
	bucketMax := make([]int, bucketCount)
	used := make([]bool, bucketCount)

	for i := range bucketMin {
		bucketMin[i] = math.MaxInt64
		bucketMax[i] = math.MinInt64
	}

	for _, v := range nums {
		idx := (v - minVal) / bucketSize
		used[idx] = true
		if v < bucketMin[idx] { bucketMin[idx] = v }
		if v > bucketMax[idx] { bucketMax[idx] = v }
	}

	// Max gap is between consecutive buckets
	maxGap := 0
	prevMax := minVal
	for i := 0; i < bucketCount; i++ {
		if !used[i] { continue }
		gap := bucketMin[i] - prevMax
		if gap > maxGap { maxGap = gap }
		prevMax = bucketMax[i]
	}
	return maxGap
}

func main() {
	fmt.Println(maximumGap([]int{3, 6, 9, 1}))    // 3
	fmt.Println(maximumGap([]int{10}))              // 0
	fmt.Println(maximumGap([]int{1, 10000000}))     // 9999999
}
```

**Textual Figure:**

```
Maximum Gap: nums = [3, 6, 9, 1]
min=1, max=9, n=4, bucketSize = ceil(8/3) = 3
bucketCount = (9-1)/3 + 1 = 3

  Distribute (idx = (v-1)/3):
  ┌──────────┬───────┬───────┬─────────┐
  │ Bucket   │ Range │ Min   │ Max     │
  ├──────────┼───────┼───────┼─────────┤
  │ B0       │ 1-3   │ 1     │ 3       │
  │ B1       │ 4-6   │ 6     │ 6       │
  │ B2       │ 7-9   │ 9     │ 9       │
  └──────────┴───────┴───────┴─────────┘

  Scan gaps between consecutive buckets:
  B0.max=3 → B1.min=6: gap = 3 ← max
  B1.max=6 → B2.min=9: gap = 3 ← max

  Pigeonhole: max gap ≥ ceil((max-min)/(n-1)) = 3
  Answer: 3
```

---

## Example 9: Bucket Sort with Insertion Sort for Small Buckets

```go
package main

import "fmt"

func insertionSort(arr []int) {
	for i := 1; i < len(arr); i++ {
		key := arr[i]
		j := i - 1
		for j >= 0 && arr[j] > key {
			arr[j+1] = arr[j]
			j--
		}
		arr[j+1] = key
	}
}

func bucketSortWithInsertion(arr []int) {
	if len(arr) <= 1 { return }

	minVal, maxVal := arr[0], arr[0]
	for _, v := range arr {
		if v < minVal { minVal = v }
		if v > maxVal { maxVal = v }
	}
	if minVal == maxVal { return }

	numBuckets := len(arr)
	rang := float64(maxVal - minVal + 1)
	buckets := make([][]int, numBuckets)
	for i := range buckets {
		buckets[i] = []int{}
	}

	for _, v := range arr {
		idx := int(float64(v-minVal) / rang * float64(numBuckets))
		if idx >= numBuckets { idx = numBuckets - 1 }
		buckets[idx] = append(buckets[idx], v)
	}

	// Use insertion sort for each bucket (efficient for small arrays)
	for i := range buckets {
		insertionSort(buckets[i])
	}

	idx := 0
	for _, bucket := range buckets {
		for _, v := range bucket {
			arr[idx] = v
			idx++
		}
	}
}

func main() {
	arr := []int{29, 25, 3, 49, 9, 37, 21, 43, 15, 8}
	bucketSortWithInsertion(arr)
	fmt.Println(arr) // [3 8 9 15 21 25 29 37 43 49]
}
```

**Textual Figure:**

```
Bucket Sort + Insertion Sort:
arr = [29,25,3,49,9,37,21,43,15,8], 10 buckets
min=3, max=49, range=47

  Distribute into buckets:
  ┌──────────┬─────────────┐
  │ Bucket 0 │ 3             │
  │ Bucket 1 │ 9, 8          │
  │ Bucket 2 │ 15            │
  │ Bucket 3 │ 21            │
  │ Bucket 4 │ 25            │
  │ Bucket 5 │ 29            │
  │ Bucket 7 │ 37            │
  │ Bucket 8 │ 43            │
  │ Bucket 9 │ 49            │
  └──────────┴─────────────┘

  Insertion sort on B1: [9,8] → [8,9]
  (Other buckets have ≤1 element, no sorting needed)

  Concatenate: [3, 8, 9, 15, 21, 25, 29, 37, 43, 49]

  Insertion sort = O(k²) for tiny k ≈ O(1) per bucket
```

---

## Example 10: When to Use Bucket Sort

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Bucket Sort — Decision Guide ===")
	fmt.Println()

	fmt.Println("USE when:")
	fmt.Println("  • Input is uniformly distributed")
	fmt.Println("  • Floating point numbers in known range")
	fmt.Println("  • Can choose good bucket boundaries")
	fmt.Println("  • n elements, k buckets where k ≈ n → O(n)")
	fmt.Println()

	fmt.Println("AVOID when:")
	fmt.Println("  • Input is heavily skewed (all in one bucket → O(n²))")
	fmt.Println("  • Unknown distribution")
	fmt.Println("  • Memory constrained (needs O(n + k) space)")
	fmt.Println()

	fmt.Println("Comparison with other linear sorts:")
	fmt.Println("  ┌────────────────┬─────────────┬───────────────┐")
	fmt.Println("  │ Algorithm      │ Best For    │ Time          │")
	fmt.Println("  ├────────────────┼─────────────┼───────────────┤")
	fmt.Println("  │ Counting Sort  │ Small range │ O(n + k)      │")
	fmt.Println("  │ Radix Sort     │ Integers    │ O(d × (n+k))  │")
	fmt.Println("  │ Bucket Sort    │ Uniform dist│ O(n) average  │")
	fmt.Println("  └────────────────┴─────────────┴───────────────┘")
}
```

**Textual Figure:**

```
Bucket Sort Decision Guide:

  ┌─────────────────────────┐
  │ Is data uniformly      │
  │ distributed?           │
  └────────────┬────────────┘
          Yes│No
     ┌─────┴──────┬─────────────┐
     │ Bucket Sort │ Try other   │
     │ O(n) avg    │ algorithms  │
     └────────────┴─────────────┘

  Complexity Comparison:
  ┌────────────────┬─────────────┬───────────────┐
  │ Algorithm      │ Best For    │ Time          │
  ├────────────────┼─────────────┼───────────────┤
  │ Counting Sort  │ Small range │ O(n + k)      │
  │ Radix Sort     │ Integers    │ O(d × (n+k))  │
  │ Bucket Sort    │ Uniform     │ O(n) avg      │
  │ Merge Sort     │ General     │ O(n log n)    │
  │ Quick Sort     │ General     │ O(n log n)avg │
  └────────────────┴─────────────┴───────────────┘
```

---

## Key Takeaways

1. O(n) average when elements are uniformly distributed across buckets
2. Performance degrades to O(n²) if all elements land in one bucket
3. Excellent for floating-point data in a known range
4. Pigeonhole principle application for maximum gap problem
5. Choose bucket count ≈ n for best performance

> **Next up:** Stability of Sorting Algorithms →
