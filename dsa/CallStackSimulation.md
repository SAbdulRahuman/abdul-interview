# Phase 6: Stack — Call Stack Simulation

## Overview

**Call stack simulation** means converting recursive algorithms to iterative ones using an explicit stack. This avoids stack overflow, gives control over execution order, and is essential for understanding how recursion works internally.

**Pattern:** Replace each recursive call with pushing a "frame" (parameters/state) onto a stack. Loop until the stack is empty.

---

## Example 1: Simulate Recursive Factorial

```go
package main

import "fmt"

// Recursive
func factorialRec(n int) int {
    if n <= 1 {
        return 1
    }
    return n * factorialRec(n-1)
}

// Iterative with explicit stack (simulating call stack)
func factorialStack(n int) int {
    type Frame struct {
        N      int
        Return int
        Phase  int // 0=descend, 1=multiply
    }

    stack := []Frame{{N: n, Phase: 0}}
    result := 1

    for len(stack) > 0 {
        frame := &stack[len(stack)-1]

        switch frame.Phase {
        case 0: // "calling" phase
            if frame.N <= 1 {
                result = 1
                stack = stack[:len(stack)-1] // return
            } else {
                frame.Phase = 1 // mark to process on return
                // "recursive call"
                stack = append(stack, Frame{N: frame.N - 1, Phase: 0})
            }
        case 1: // "return" phase — multiply
            result = frame.N * result
            stack = stack[:len(stack)-1]
        }
    }

    return result
}

func main() {
    for _, n := range []int{0, 1, 5, 10} {
        fmt.Printf("factorial(%d) = %d (rec=%d)\n", n, factorialStack(n), factorialRec(n))
    }
}
```

---

## Example 2: Simulate Recursive Fibonacci

```go
package main

import "fmt"

// Direct recursive (exponential)
func fibRec(n int) int {
    if n <= 1 {
        return n
    }
    return fibRec(n-1) + fibRec(n-2)
}

// Simulated call stack
func fibStack(n int) int {
    type Frame struct {
        N     int
        Phase int // 0=start, 1=after first call, 2=after second call
        Left  int // result of fib(n-1)
    }

    stack := []Frame{{N: n, Phase: 0}}
    result := 0

    for len(stack) > 0 {
        frame := &stack[len(stack)-1]

        switch frame.Phase {
        case 0: // initial
            if frame.N <= 1 {
                result = frame.N
                stack = stack[:len(stack)-1]
            } else {
                frame.Phase = 1
                stack = append(stack, Frame{N: frame.N - 1, Phase: 0})
            }
        case 1: // after fib(n-1) returned
            frame.Left = result
            frame.Phase = 2
            stack = append(stack, Frame{N: frame.N - 2, Phase: 0})
        case 2: // after fib(n-2) returned
            result = frame.Left + result
            stack = stack[:len(stack)-1]
        }
    }

    return result
}

func main() {
    for i := 0; i <= 10; i++ {
        fmt.Printf("fib(%d) = %d\n", i, fibStack(i))
    }
}
```

---

## Example 3: Tower of Hanoi (Iterative)

```go
package main

import "fmt"

// Recursive
func hanoiRec(n int, from, to, aux string) {
    if n == 0 {
        return
    }
    hanoiRec(n-1, from, aux, to)
    fmt.Printf("Move disk %d: %s → %s\n", n, from, to)
    hanoiRec(n-1, aux, to, from)
}

// Iterative using explicit stack
func hanoiStack(n int, from, to, aux string) {
    type Frame struct {
        N     int
        From  string
        To    string
        Aux   string
        Phase int
    }

    stack := []Frame{{N: n, From: from, To: to, Aux: aux, Phase: 0}}

    for len(stack) > 0 {
        frame := &stack[len(stack)-1]

        if frame.N == 0 {
            stack = stack[:len(stack)-1]
            continue
        }

        switch frame.Phase {
        case 0:
            frame.Phase = 1
            // hanoi(n-1, from, aux, to)
            stack = append(stack, Frame{N: frame.N - 1, From: frame.From, To: frame.Aux, Aux: frame.To, Phase: 0})
        case 1:
            fmt.Printf("Move disk %d: %s → %s\n", frame.N, frame.From, frame.To)
            frame.Phase = 2
            // hanoi(n-1, aux, to, from)
            stack = append(stack, Frame{N: frame.N - 1, From: frame.Aux, To: frame.To, Aux: frame.From, Phase: 0})
        case 2:
            stack = stack[:len(stack)-1]
        }
    }
}

func main() {
    fmt.Println("=== Recursive ===")
    hanoiRec(3, "A", "C", "B")
    fmt.Println("\n=== Iterative ===")
    hanoiStack(3, "A", "C", "B")
}
```

---

## Example 4: Simulate Recursive Binary Search

```go
package main

import "fmt"

func binarySearchRec(arr []int, target, lo, hi int) int {
    if lo > hi {
        return -1
    }
    mid := lo + (hi-lo)/2
    if arr[mid] == target {
        return mid
    }
    if target < arr[mid] {
        return binarySearchRec(arr, target, lo, mid-1)
    }
    return binarySearchRec(arr, target, mid+1, hi)
}

// Simulating with explicit stack (tail recursion → easy)
func binarySearchStack(arr []int, target int) int {
    type Frame struct {
        Lo, Hi int
    }

    stack := []Frame{{0, len(arr) - 1}}

    for len(stack) > 0 {
        frame := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        if frame.Lo > frame.Hi {
            continue
        }

        mid := frame.Lo + (frame.Hi-frame.Lo)/2
        if arr[mid] == target {
            return mid
        }
        if target < arr[mid] {
            stack = append(stack, Frame{frame.Lo, mid - 1})
        } else {
            stack = append(stack, Frame{mid + 1, frame.Hi})
        }
    }

    return -1
}

func main() {
    arr := []int{1, 3, 5, 7, 9, 11, 13, 15}
    for _, target := range []int{7, 1, 15, 6} {
        idx := binarySearchStack(arr, target)
        fmt.Printf("Search %d → index %d\n", target, idx)
    }
}
```

---

## Example 5: Simulate Tree DFS with State Machine

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// State machine approach: each frame has a phase indicating what to do next
func dfsStateMachine(root *TreeNode) (pre, in, post []int) {
    if root == nil {
        return
    }

    type Frame struct {
        Node  *TreeNode
        Phase int // 0=pre, 1=left done, 2=right done
    }

    stack := []Frame{{root, 0}}

    for len(stack) > 0 {
        frame := &stack[len(stack)-1]

        switch frame.Phase {
        case 0: // Preorder position
            pre = append(pre, frame.Node.Val)
            frame.Phase = 1
            if frame.Node.Left != nil {
                stack = append(stack, Frame{frame.Node.Left, 0})
            }
        case 1: // Inorder position (left subtree done)
            in = append(in, frame.Node.Val)
            frame.Phase = 2
            if frame.Node.Right != nil {
                stack = append(stack, Frame{frame.Node.Right, 0})
            }
        case 2: // Postorder position (both subtrees done)
            post = append(post, frame.Node.Val)
            stack = stack[:len(stack)-1]
        }
    }

    return
}

func main() {
    //     1
    //    / \
    //   2   3
    //  / \
    // 4   5
    root := &TreeNode{
        Val: 1,
        Left: &TreeNode{
            Val:   2,
            Left:  &TreeNode{Val: 4},
            Right: &TreeNode{Val: 5},
        },
        Right: &TreeNode{Val: 3},
    }

    pre, in, post := dfsStateMachine(root)
    fmt.Println("Preorder: ", pre)  // [1 2 4 5 3]
    fmt.Println("Inorder:  ", in)   // [4 2 5 1 3]
    fmt.Println("Postorder:", post) // [4 5 2 3 1]
}
```

---

## Example 6: Simulate Merge Sort

```go
package main

import "fmt"

func mergeSortIterativeStack(arr []int) []int {
    type Frame struct {
        Lo, Hi int
        Phase  int
        Left   []int
        Right  []int
    }

    result := make([]int, len(arr))
    copy(result, arr)

    // Simpler: bottom-up merge sort (natural iterative version)
    n := len(result)
    for width := 1; width < n; width *= 2 {
        for i := 0; i < n; i += 2 * width {
            lo := i
            mid := min(i+width, n)
            hi := min(i+2*width, n)
            merge(result, lo, mid, hi)
        }
    }

    return result
}

func merge(arr []int, lo, mid, hi int) {
    temp := make([]int, hi-lo)
    i, j, k := lo, mid, 0

    for i < mid && j < hi {
        if arr[i] <= arr[j] {
            temp[k] = arr[i]
            i++
        } else {
            temp[k] = arr[j]
            j++
        }
        k++
    }
    for i < mid {
        temp[k] = arr[i]
        i++
        k++
    }
    for j < hi {
        temp[k] = arr[j]
        j++
        k++
    }
    copy(arr[lo:hi], temp)
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    arr := []int{38, 27, 43, 3, 9, 82, 10}
    fmt.Println("Before:", arr)
    fmt.Println("After: ", mergeSortIterativeStack(arr))
}
```

---

## Example 7: Simulate Quick Sort

```go
package main

import "fmt"

func quickSortIterative(arr []int) {
    type Range struct {
        Lo, Hi int
    }

    stack := []Range{{0, len(arr) - 1}}

    for len(stack) > 0 {
        r := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        if r.Lo >= r.Hi {
            continue
        }

        pivot := partition(arr, r.Lo, r.Hi)

        // Push larger partition first (limits stack depth to O(log n))
        if pivot-r.Lo > r.Hi-pivot {
            stack = append(stack, Range{r.Lo, pivot - 1})
            stack = append(stack, Range{pivot + 1, r.Hi})
        } else {
            stack = append(stack, Range{pivot + 1, r.Hi})
            stack = append(stack, Range{r.Lo, pivot - 1})
        }
    }
}

func partition(arr []int, lo, hi int) int {
    pivot := arr[hi]
    i := lo
    for j := lo; j < hi; j++ {
        if arr[j] < pivot {
            arr[i], arr[j] = arr[j], arr[i]
            i++
        }
    }
    arr[i], arr[hi] = arr[hi], arr[i]
    return i
}

func main() {
    arr := []int{10, 7, 8, 9, 1, 5}
    fmt.Println("Before:", arr)
    quickSortIterative(arr)
    fmt.Println("After: ", arr)

    arr2 := []int{3, 1, 4, 1, 5, 9, 2, 6}
    quickSortIterative(arr2)
    fmt.Println("Sorted:", arr2)
}
```

---

## Example 8: Simulate Recursive Permutations

```go
package main

import "fmt"

func permutationsIterative(nums []int) [][]int {
    type Frame struct {
        Perm  []int
        Avail []bool
    }

    n := len(nums)
    result := [][]int{}
    avail := make([]bool, n)
    for i := range avail {
        avail[i] = true
    }

    stack := []Frame{{Perm: []int{}, Avail: avail}}

    for len(stack) > 0 {
        frame := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        if len(frame.Perm) == n {
            p := make([]int, n)
            copy(p, frame.Perm)
            result = append(result, p)
            continue
        }

        for i := 0; i < n; i++ {
            if frame.Avail[i] {
                newAvail := make([]bool, n)
                copy(newAvail, frame.Avail)
                newAvail[i] = false

                newPerm := make([]int, len(frame.Perm)+1)
                copy(newPerm, frame.Perm)
                newPerm[len(frame.Perm)] = nums[i]

                stack = append(stack, Frame{Perm: newPerm, Avail: newAvail})
            }
        }
    }

    return result
}

func main() {
    nums := []int{1, 2, 3}
    perms := permutationsIterative(nums)
    fmt.Printf("Permutations of %v (%d total):\n", nums, len(perms))
    for _, p := range perms {
        fmt.Println(p)
    }
}
```

---

## Example 9: Simulate Recursive Flood Fill

```go
package main

import "fmt"

func floodFill(image [][]int, sr, sc, color int) [][]int {
    original := image[sr][sc]
    if original == color {
        return image
    }

    rows, cols := len(image), len(image[0])
    type Point struct{ R, C int }

    stack := []Point{{sr, sc}}
    dirs := []Point{{-1, 0}, {1, 0}, {0, -1}, {0, 1}}

    for len(stack) > 0 {
        p := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        if p.R < 0 || p.R >= rows || p.C < 0 || p.C >= cols {
            continue
        }
        if image[p.R][p.C] != original {
            continue
        }

        image[p.R][p.C] = color

        for _, d := range dirs {
            stack = append(stack, Point{p.R + d.R, p.C + d.C})
        }
    }

    return image
}

func main() {
    image := [][]int{
        {1, 1, 1},
        {1, 1, 0},
        {1, 0, 1},
    }

    fmt.Println("Before:")
    for _, row := range image {
        fmt.Println(row)
    }

    floodFill(image, 1, 1, 2)

    fmt.Println("\nAfter flood fill (1,1) with color 2:")
    for _, row := range image {
        fmt.Println(row)
    }
}
```

---

## Example 10: Visualize Call Stack Depth

```go
package main

import (
    "fmt"
    "strings"
)

type CallInfo struct {
    FuncName string
    Args     string
    Depth    int
}

// Trace fibonacci calls showing stack depth
func fibTrace(n int) int {
    type Frame struct {
        N     int
        Phase int
        Left  int
        Depth int
    }

    stack := []Frame{{N: n, Phase: 0, Depth: 0}}
    result := 0
    maxDepth := 0
    callCount := 0

    for len(stack) > 0 {
        frame := &stack[len(stack)-1]

        if frame.Depth > maxDepth {
            maxDepth = frame.Depth
        }

        switch frame.Phase {
        case 0:
            callCount++
            indent := strings.Repeat("  ", frame.Depth)
            fmt.Printf("%sfib(%d) called [stack depth: %d]\n", indent, frame.N, len(stack))

            if frame.N <= 1 {
                result = frame.N
                stack = stack[:len(stack)-1]
            } else {
                frame.Phase = 1
                stack = append(stack, Frame{N: frame.N - 1, Phase: 0, Depth: frame.Depth + 1})
            }
        case 1:
            frame.Left = result
            frame.Phase = 2
            stack = append(stack, Frame{N: frame.N - 2, Phase: 0, Depth: frame.Depth + 1})
        case 2:
            result = frame.Left + result
            indent := strings.Repeat("  ", frame.Depth)
            fmt.Printf("%sfib(%d) = %d\n", indent, frame.N, result)
            stack = stack[:len(stack)-1]
        }
    }

    fmt.Printf("\nTotal calls: %d, Max stack depth: %d\n", callCount, maxDepth+1)
    return result
}

func main() {
    n := 5
    result := fibTrace(n)
    fmt.Printf("\nfib(%d) = %d\n", n, result)
}
```

---

## Conversion Recipe: Recursion → Iteration

| Step | Action |
|------|--------|
| 1 | Define a `Frame` struct with function parameters + a `Phase` field |
| 2 | Initialize stack with one frame for the initial call |
| 3 | Loop while stack is non-empty |
| 4 | Each recursive call → push new frame, increment phase |
| 5 | Base case → set result, pop frame |
| 6 | After returning → resume at next phase, use returned value |

## Key Takeaways

1. **Every recursive function** can be converted to iterative using an explicit stack
2. **Frame struct** holds: parameters + local variables + phase counter
3. **Phase counter** tracks which "line" of the function we're at (before/after each recursive call)
4. **Tail recursion** (single call at end) is simplest to convert — just push new params
5. **Binary recursion** (like fib): needs 3 phases (start, after-left, after-right)
6. **Bottom-up** (merge sort widths) is often simpler than true call stack simulation
7. **Benefits**: no stack overflow, explicit depth control, can be paused/resumed

> **Phase 6 Complete!** Next up: Phase 7 — Queue →
