# Matrix Binary Search

Binary search can be extended to 2D matrices when rows and/or columns are sorted. The key insight is treating a sorted matrix as a virtual 1D array, or using staircase search to exploit both row and column sorting. These techniques reduce search time from O(m×n) to O(log(m×n)) or O(m+n).

---

## Example 1: Search a 2D Matrix (LeetCode 74)

Each row is sorted, and the first element of each row is greater than the last element of the previous row. Treat as a flattened sorted array.

```go
package main

import "fmt"

func searchMatrix(matrix [][]int, target int) bool {
	m, n := len(matrix), len(matrix[0])
	lo, hi := 0, m*n-1
	for lo <= hi {
		mid := lo + (hi-lo)/2
		val := matrix[mid/n][mid%n]
		if val == target {
			return true
		} else if val < target {
			lo = mid + 1
		} else {
			hi = mid - 1
		}
	}
	return false
}

func main() {
	matrix := [][]int{
		{1, 3, 5, 7},
		{10, 11, 16, 20},
		{23, 30, 34, 60},
	}
	fmt.Println(searchMatrix(matrix, 3))  // true
	fmt.Println(searchMatrix(matrix, 13)) // false
}
```

**Textual Figure: searchMatrix(matrix, target=3)**

```
    Matrix (3×4), treat as flattened 1D array of length 12:
    ┌────┬────┬────┬────┐
    │  1 │  3 │  5 │  7 │  row 0   (indices 0–3)
    ├────┼────┼────┼────┤
    │ 10 │ 11 │ 16 │ 20 │  row 1   (indices 4–7)
    ├────┼────┼────┼────┤
    │ 23 │ 30 │ 34 │ 60 │  row 2   (indices 8–11)
    └────┴────┴────┴────┘

    1D view: [1, 3, 5, 7, 10, 11, 16, 20, 23, 30, 34, 60]
    index:    0  1  2  3   4   5   6   7   8   9  10  11

    Searching for target = 3:

    Iteration 1:  lo=0, hi=11, mid=5
        mid/n = 5/4 = row 1, mid%n = 5%4 = col 1
        matrix[1][1] = 11 > 3  →  hi = mid-1 = 4
        ┌────┬────┬────┬────┐
        │  1 │  3 │  5 │  7 │
        ├────┼────┼────┼────┤
        │ 10 │█11█│ 16 │ 20 │  ← mid here, 11>3, go left
        ├────┼────┼────┼────┤
        │ 23 │ 30 │ 34 │ 60 │
        └────┴────┴────┴────┘

    Iteration 2:  lo=0, hi=4, mid=2
        mid/n = 2/4 = row 0, mid%n = 2%4 = col 2
        matrix[0][2] = 5 > 3  →  hi = mid-1 = 1
        ┌────┬────┬────┬────┐
        │  1 │  3 │█ 5█│  7 │  ← mid here, 5>3, go left
        ├────┼────┼────┼────┤
        │ 10 │ 11 │ 16 │ 20 │
        ├────┼────┼────┼────┤
        │ 23 │ 30 │ 34 │ 60 │
        └────┴────┴────┴────┘

    Iteration 3:  lo=0, hi=1, mid=0
        matrix[0][0] = 1 < 3  →  lo = mid+1 = 1

    Iteration 4:  lo=1, hi=1, mid=1
        matrix[0][1] = 3 == 3  →  FOUND!
        ┌────┬────┬────┬────┐
        │  1 │► 3◄│  5 │  7 │  ← FOUND at [0][1]
        ├────┼────┼────┼────┤
        │ 10 │ 11 │ 16 │ 20 │
        ├────┼────┼────┼────┤
        │ 23 │ 30 │ 34 │ 60 │
        └────┴────┴────┴────┘

    Result: true (found at row 0, col 1)
```

---

## Example 2: Search a 2D Matrix II (LeetCode 240)

Each row is sorted left to right. Each column is sorted top to bottom. Start from top-right corner (staircase search).

```go
package main

import "fmt"

func searchMatrixII(matrix [][]int, target int) bool {
	m, n := len(matrix), len(matrix[0])
	row, col := 0, n-1 // start top-right
	for row < m && col >= 0 {
		if matrix[row][col] == target {
			return true
		} else if matrix[row][col] < target {
			row++ // move down
		} else {
			col-- // move left
		}
	}
	return false
}

func main() {
	matrix := [][]int{
		{1, 4, 7, 11, 15},
		{2, 5, 8, 12, 19},
		{3, 6, 9, 16, 22},
		{10, 13, 14, 17, 24},
		{18, 21, 23, 26, 30},
	}
	fmt.Println(searchMatrixII(matrix, 5))  // true
	fmt.Println(searchMatrixII(matrix, 20)) // false
}
```

**Textual Figure: searchMatrixII(matrix, target=5)**

```
    Matrix (5×5), row-sorted & column-sorted:
    ┌────┬────┬────┬────┬────┐
    │  1 │  4 │  7 │ 11 │ 15 │  ← Start here (top-right)
    ├────┼────┼────┼────┼────┤
    │  2 │  5 │  8 │ 12 │ 19 │
    ├────┼────┼────┼────┼────┤
    │  3 │  6 │  9 │ 16 │ 22 │
    ├────┼────┼────┼────┼────┤
    │ 10 │ 13 │ 14 │ 17 │ 24 │
    ├────┼────┼────┼────┼────┤
    │ 18 │ 21 │ 23 │ 26 │ 30 │
    └────┴────┴────┴────┴────┘

    Staircase search for target = 5:
    Start: row=0, col=4

    Step 1: matrix[0][4]=15 > 5 → move LEFT (col--)
    ┌────┬────┬────┬────┬────┐
    │  1 │  4 │  7 │ 11 │[15]│ ←── 15>5, go left
    └────┴────┴────┴────┴────┘

    Step 2: matrix[0][3]=11 > 5 → move LEFT
    Step 3: matrix[0][2]=7  > 5 → move LEFT
    Step 4: matrix[0][1]=4  < 5 → move DOWN (row++)

    ┌────┬────┬────┬────┬────┐
    │  1 │[ 4]│  7 │ 11 │ 15 │  4<5, go down
    ├────┼────┼────┼────┼────┤
    │  2 │  5 │    │    │    │
    └────┴────┴    ┴    ┴    ┘
           ↓
    Step 5: matrix[1][1]=5 == 5 → FOUND!

    Search path on matrix:
    ┌────┬────┬────┬────┬────┐
    │  1 │  4 │  7 │ 11 │ 15 │
    ├────┼────┼    ←───←───←┤  ←── path
    │  2 │► 5◄│  8 │ 12 │ 19 │
    ├────┼↑───┼────┼────┼────┤
    │  3 │  6 │  9 │ 16 │ 22 │
    ├────┼────┼────┼────┼────┤
    │ 10 │ 13 │ 14 │ 17 │ 24 │
    ├────┼────┼────┼────┼────┤
    │ 18 │ 21 │ 23 │ 26 │ 30 │
    └────┴────┴────┴────┴────┘

    Result: true (found at row 1, col 1)
```

---

## Example 3: Count Negative Numbers in a Sorted Matrix (LeetCode 1351)

Matrix is sorted in non-increasing order in both rows and columns. Count all negatives using staircase.

```go
package main

import "fmt"

func countNegatives(grid [][]int) int {
	m, n := len(grid), len(grid[0])
	count := 0
	row, col := 0, n-1 // start top-right
	for row < m && col >= 0 {
		if grid[row][col] < 0 {
			count += m - row // all below are also negative
			col--
		} else {
			row++
		}
	}
	return count
}

func main() {
	grid := [][]int{
		{4, 3, 2, -1},
		{3, 2, 1, -1},
		{1, 1, -1, -2},
		{-1, -1, -2, -3},
	}
	fmt.Println(countNegatives(grid)) // 8
}
```

**Textual Figure: countNegatives(grid) = 8**

```
    Grid (4×4), sorted non-increasing in rows & columns:
    ┌────┬────┬────┬────┐
    │  4 │  3 │  2 │ -1 │  row 0
    ├────┼────┼────┼────┤
    │  3 │  2 │  1 │ -1 │  row 1
    ├────┼────┼────┼────┤
    │  1 │  1 │ -1 │ -2 │  row 2
    ├────┼────┼────┼────┤
    │ -1 │ -1 │ -2 │ -3 │  row 3
    └────┴────┴────┴────┘

    Staircase from top-right (row=0, col=3):

    Step 1: grid[0][3]=-1 < 0 → negative!
       count += (4 - 0) = 4   (all rows below col 3 are negative)
       col-- → col=2
    ┌────┬────┬────┬████┐
    │  4 │  3 │  2 │█-1█│  ← counted 4 below (including this)
    ├────┼────┼────┼████┤
    │  3 │  2 │  1 │█-1█│
    ├────┼────┼────┼████┤
    │  1 │  1 │ -1 │█-2█│
    ├────┼────┼────┼████┤
    │ -1 │ -1 │ -2 │█-3█│
    └────┴────┴────┴████┘  count=4

    Step 2: grid[0][2]=2 ≥ 0 → row++ → row=1
    Step 3: grid[1][2]=1 ≥ 0 → row++ → row=2

    Step 4: grid[2][2]=-1 < 0 → negative!
       count += (4 - 2) = 2   → count=6
       col-- → col=1

    Step 5: grid[2][1]=1 ≥ 0 → row++ → row=3

    Step 6: grid[3][1]=-1 < 0 → negative!
       count += (4 - 3) = 1   → count=7
       col-- → col=0

    Step 7: grid[3][0]=-1 < 0 → negative!
       count += (4 - 3) = 1   → count=8
       col-- → col=-1  STOP

    Final grid with all negatives marked:
    ┌────┬────┬────┬────┐
    │  4 │  3 │  2 │►-1◄│
    ├────┼────┼────┼────┤
    │  3 │  2 │  1 │►-1◄│
    ├────┼────┼────┼────┤
    │  1 │  1 │►-1◄│►-2◄│
    ├────┼────┼────┼────┤
    │►-1◄│►-1◄│►-2◄│►-3◄│
    └────┴────┴────┴────┘

    Result: 8 negative numbers
```

---

## Example 4: Row With Maximum Ones (Binary Search per Row)

Given a binary matrix where each row is sorted (0s then 1s), find the row with the most 1s.

```go
package main

import "fmt"

func rowWithMaxOnes(mat [][]int) int {
	maxOnes, resultRow := 0, -1
	for i, row := range mat {
		// binary search for first 1 in this row
		lo, hi := 0, len(row)
		for lo < hi {
			mid := lo + (hi-lo)/2
			if row[mid] < 1 {
				lo = mid + 1
			} else {
				hi = mid
			}
		}
		ones := len(row) - lo
		if ones > maxOnes {
			maxOnes = ones
			resultRow = i
		}
	}
	return resultRow
}

func main() {
	mat := [][]int{
		{0, 0, 0, 1},
		{0, 1, 1, 1},
		{0, 0, 1, 1},
		{0, 0, 0, 0},
	}
	fmt.Println(rowWithMaxOnes(mat)) // 1
}
```

**Textual Figure: rowWithMaxOnes(mat) → row 1**

```
    Binary matrix (4×4), each row sorted [0..0, 1..1]:
    ┌───┬───┬───┬───┐
    │ 0 │ 0 │ 0 │ 1 │  row 0:  1 one
    ├───┼───┼───┼───┤
    │ 0 │ 1 │ 1 │ 1 │  row 1:  3 ones  ← MAX
    ├───┼───┼───┼───┤
    │ 0 │ 0 │ 1 │ 1 │  row 2:  2 ones
    ├───┼───┼───┼───┤
    │ 0 │ 0 │ 0 │ 0 │  row 3:  0 ones
    └───┴───┴───┴───┘

    Binary search per row for first '1':

    Row 0: [0, 0, 0, 1]
        lo=0  hi=4
        mid=2: row[2]=0 < 1 → lo=3
        mid=3: row[3]=1 ≥ 1 → hi=3
        lo==hi==3 → first 1 at index 3
        ones = 4 - 3 = 1
        ┌───┬───┬───┬───┐
        │ 0 │ 0 │ 0 │►1◄│  first 1 found at idx 3
        └───┴───┴───┴───┘
                     ↑lo

    Row 1: [0, 1, 1, 1]
        lo=0  hi=4
        mid=2: row[2]=1 ≥ 1 → hi=2
        mid=1: row[1]=1 ≥ 1 → hi=1
        mid=0: row[0]=0 < 1 → lo=1
        lo==hi==1 → first 1 at index 1
        ones = 4 - 1 = 3  ← NEW MAX
        ┌───┬───┬───┬───┐
        │ 0 │►1◄│ 1 │ 1 │  first 1 at idx 1, three 1s
        └───┴───┴───┴───┘
             ↑lo

    Row 2: ones=2 (< 3, not max)
    Row 3: ones=0 (< 3, not max)

    Result: row 1 (3 ones)
```

---

## Example 5: Kth Smallest Element in Sorted Matrix (LeetCode 378)

Matrix sorted row-wise and column-wise. Use binary search on value range, counting elements ≤ mid.

```go
package main

import "fmt"

func kthSmallest(matrix [][]int, k int) int {
	n := len(matrix)
	lo, hi := matrix[0][0], matrix[n-1][n-1]
	for lo < hi {
		mid := lo + (hi-lo)/2
		count := countLessEqual(matrix, mid, n)
		if count < k {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

func countLessEqual(matrix [][]int, target, n int) int {
	count := 0
	row, col := n-1, 0 // start bottom-left
	for row >= 0 && col < n {
		if matrix[row][col] <= target {
			count += row + 1
			col++
		} else {
			row--
		}
	}
	return count
}

func main() {
	matrix := [][]int{
		{1, 5, 9},
		{10, 11, 13},
		{12, 13, 15},
	}
	fmt.Println(kthSmallest(matrix, 8)) // 13
}
```

**Textual Figure: kthSmallest(matrix, k=8) → 13**

```
    Matrix (3×3), sorted row-wise and column-wise:
    ┌────┬────┬────┐
    │  1 │  5 │  9 │
    ├────┼────┼────┤
    │ 10 │ 11 │ 13 │
    ├────┼────┼────┤
    │ 12 │ 13 │ 15 │
    └────┴────┴────┘
    Value range: lo=1, hi=15, k=8

    Binary search on VALUE (not index):

    Iter 1: lo=1, hi=15, mid=8
        countLessEqual(8):
        Start bottom-left: row=2, col=0
        ┌────┬────┬────┐
        │  1 │  5 │  9 │
        ├────┼────┼────┤
        │ 10 │ 11 │ 13 │
        ├────┼────┼────┤
      → │ 12 │ 13 │ 15 │  12>8 → row--
        └────┴────┴────┘
      → │ 10 │    │    │  10>8 → row--
      → │  1 │  5 │    │  1≤8 → count+=1, col++
                            5≤8 → count+=1, col++
                                 col=2, 9>8 → row-- (row<0, stop)
        count = 2  (< k=8)  →  lo = 9

    Iter 2: lo=9, hi=15, mid=12
        countLessEqual(12):
        row=2,col=0: 12≤12 → count+=3, col++
        row=2,col=1: 13>12 → row--
        row=1,col=1: 11≤12 → count+=2, col++
        row=1,col=2: 13>12 → row--
        row=0,col=2: 9≤12  → count+=1, col++
        count = 6  (< k=8)  →  lo = 13

    Iter 3: lo=13, hi=15, mid=14
        countLessEqual(14):
        row=2,col=0: 12≤14 → count+=3, col++
        row=2,col=1: 13≤14 → count+=3, col++
        row=2,col=2: 15>14 → row--
        row=1,col=2: 13≤14 → count+=2, col++
        count = 8  (≥ k=8)  →  hi = 14

    Iter 4: lo=13, hi=14, mid=13
        countLessEqual(13):
        row=2,col=0: 12≤13 → count+=3, col++
        row=2,col=1: 13≤13 → count+=3, col++
        row=2,col=2: 15>13 → row--
        row=1,col=2: 13≤13 → count+=2, col++
        count = 8  (≥ k=8)  →  hi = 13

    lo == hi == 13  →  STOP

    Result: 13
    ┌────┬────┬────┐
    │  1 │  5 │  9 │  All ≤8: [1,5]         (2 elements)
    ├────┼────┼────┤
    │ 10 │ 11 │►13◄│  All ≤13: [1,5,9,10,11,12,13,13]
    ├────┼────┼────┤           8th smallest = 13
    │ 12 │►13◄│ 15 │
    └────┴────┴────┘
```

---

## Key Takeaways

| Problem | Approach | Time Complexity |
|---|---|---|
| Search 2D Matrix (fully sorted) | Flatten to 1D binary search | O(log(m×n)) |
| Search 2D Matrix II (row+col sorted) | Staircase from top-right | O(m+n) |
| Count Negatives | Staircase from top-right | O(m+n) |
| Row with Max Ones | Binary search each row | O(m log n) |
| Kth Smallest | Binary search on value + count | O(n log(max−min)) |

---
