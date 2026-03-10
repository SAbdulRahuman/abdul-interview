# Phase 6: Stack — Monotonic Stack

## Overview

A **monotonic stack** maintains elements in strictly increasing or decreasing order. When a new element violates the monotonicity, we pop elements until the invariant is restored.

**Use cases:**
- Next Greater Element / Next Smaller Element
- Previous Greater / Previous Smaller Element
- Largest Rectangle in Histogram
- Trapping Rain Water
- Stock span problems

| Type | Invariant | Use |
|------|-----------|-----|
| Monotonic Increasing | bottom → top: small → large | Find next/prev smaller |
| Monotonic Decreasing | bottom → top: large → small | Find next/prev greater |

---

## Example 1: Next Greater Element

```go
package main

import "fmt"

// For each element, find the first larger element to its right
func nextGreaterElement(nums []int) []int {
    n := len(nums)
    result := make([]int, n)
    for i := range result {
        result[i] = -1 // default: no greater element
    }

    stack := []int{} // monotonic decreasing stack (stores indices)

    for i := 0; i < n; i++ {
        for len(stack) > 0 && nums[i] > nums[stack[len(stack)-1]] {
            idx := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[idx] = nums[i]
        }
        stack = append(stack, i)
    }

    return result
}

func main() {
    tests := [][]int{
        {4, 5, 2, 25},
        {13, 7, 6, 12},
        {1, 2, 3, 4},
        {4, 3, 2, 1},
    }

    for _, nums := range tests {
        fmt.Printf("%v → %v\n", nums, nextGreaterElement(nums))
    }
    // [4,5,2,25] → [5,25,25,-1]
    // [13,7,6,12] → [-1,12,12,-1]
}
```

**Textual Figure: Next Greater Element (Monotonic Decreasing Stack)**

```
nums = [4, 5, 2, 25]       Stack stores indices (decreasing by value)

i=0 (4):  stack=[]         → push 0         stack=[0]
                            (vals:  [4])
i=1 (5):  5>4 → pop 0      result[0]=5      stack=[]
          push 1            stack=[1]        (vals: [5])
i=2 (2):  2<5 → no pop     push 2           stack=[1,2]
                            (vals:  [5,2])
i=3 (25): 25>2 → pop 2     result[2]=25     stack=[1]
          25>5 → pop 1     result[1]=25     stack=[]
          push 3            stack=[3]        (vals: [25])

Remaining in stack: index 3 → result[3]=-1

  Index:  0   1   2   3
  Input:  4   5   2   25
  Result: 5   25  25  -1
          ↑   ↑   ↑   ↑
          5   25  25  none

  Monotonic decreasing stack invariant:
  ┌───────────────────────┐
  │  bottom → large  top │  Elements decrease from bottom to top
  │  When new > top: pop │  Popped element's answer = new element
  └───────────────────────┘
```

---

## Example 2: Next Smaller Element

```go
package main

import "fmt"

func nextSmallerElement(nums []int) []int {
    n := len(nums)
    result := make([]int, n)
    for i := range result {
        result[i] = -1
    }

    stack := []int{} // monotonic increasing stack

    for i := 0; i < n; i++ {
        for len(stack) > 0 && nums[i] < nums[stack[len(stack)-1]] {
            idx := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[idx] = nums[i]
        }
        stack = append(stack, i)
    }

    return result
}

func main() {
    nums := []int{4, 8, 5, 2, 25}
    fmt.Printf("%v → %v\n", nums, nextSmallerElement(nums))
    // [4,8,5,2,25] → [2,5,2,-1,-1]

    nums2 := []int{1, 3, 2, 4}
    fmt.Printf("%v → %v\n", nums2, nextSmallerElement(nums2))
    // [1,3,2,4] → [-1,2,-1,-1]
}
```

**Textual Figure: Next Smaller Element (Monotonic Increasing Stack)**

```
nums = [4, 8, 5, 2, 25]    Stack stores indices (increasing by value)

i=0 (4):  stack=[]         → push 0         stack=[0]
i=1 (8):  8>4 → no pop     push 1           stack=[0,1]
                            (vals:  [4,8])
i=2 (5):  5<8 → pop 1      result[1]=5      stack=[0]
          5>4 → no pop     push 2           stack=[0,2]
                            (vals:  [4,5])
i=3 (2):  2<5 → pop 2      result[2]=2      stack=[0]
          2<4 → pop 0      result[0]=2      stack=[]
          push 3            stack=[3]        (vals: [2])
i=4 (25): 25>2 → no pop    push 4           stack=[3,4]
                            (vals:  [2,25])

Remaining: indices 3,4 → result[3]=-1, result[4]=-1

  Index:  0   1   2   3    4
  Input:  4   8   5   2    25
  Result: 2   5   2   -1   -1

  Monotonic increasing stack:
  bottom → small .... large ← top
  When new < top: pop (answer for popped = new element)
```

---

## Example 3: Previous Greater Element

```go
package main

import "fmt"

func prevGreaterElement(nums []int) []int {
    n := len(nums)
    result := make([]int, n)
    for i := range result {
        result[i] = -1
    }

    stack := []int{} // monotonic decreasing stack

    for i := 0; i < n; i++ {
        // Pop elements smaller or equal
        for len(stack) > 0 && nums[stack[len(stack)-1]] <= nums[i] {
            stack = stack[:len(stack)-1]
        }
        if len(stack) > 0 {
            result[i] = nums[stack[len(stack)-1]]
        }
        stack = append(stack, i)
    }

    return result
}

func main() {
    nums := []int{10, 4, 2, 20, 40, 12, 30}
    fmt.Printf("%v\n→ %v\n", nums, prevGreaterElement(nums))
    // [-1, 10, 4, -1, -1, 40, 40]
}
```

**Textual Figure: Previous Greater Element (Monotonic Decreasing Stack)**

```
nums = [10, 4, 2, 20, 40, 12, 30]

i=0 (10): stack=[]          result[0]=-1    push 0   stack=[0]
i=1 (4):  4≤10 → no pop     result[1]=10    push 1   stack=[0,1]
                             (vals: [10,4])
i=2 (2):  2≤4 → no pop      result[2]=4     push 2   stack=[0,1,2]
                             (vals: [10,4,2])
i=3 (20): pop 2(2≤20)       pop 1(4≤20)     stack=[0]
          20≤10 → no pop    result[3]=-1    push 3   stack=[0,3]
                             (vals: [10,20])
i=4 (40): pop 3(20≤40)      pop 0(10≤40)    stack=[]
          result[4]=-1       push 4           stack=[4]
i=5 (12): 12≤40             result[5]=40    push 5   stack=[4,5]
i=6 (30): pop 5(12≤30)      result[6]=40    push 6   stack=[4,6]

  Index:  0    1   2   3    4    5    6
  Input:  10   4   2   20   40   12   30
  Result: -1   10  4   -1   -1   40   40
                                 ↑    ↑
                          stack top is prev greater
```

---

## Example 4: Daily Temperatures (LeetCode 739)

```go
package main

import "fmt"

// For each day, how many days until a warmer temperature?
func dailyTemperatures(temperatures []int) []int {
    n := len(temperatures)
    result := make([]int, n)
    stack := []int{} // decreasing stack of indices

    for i := 0; i < n; i++ {
        for len(stack) > 0 && temperatures[i] > temperatures[stack[len(stack)-1]] {
            prev := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[prev] = i - prev
        }
        stack = append(stack, i)
    }

    return result
}

func main() {
    temps := []int{73, 74, 75, 71, 69, 72, 76, 73}
    fmt.Println(dailyTemperatures(temps))
    // [1, 1, 4, 2, 1, 1, 0, 0]

    temps2 := []int{30, 40, 50, 60}
    fmt.Println(dailyTemperatures(temps2))
    // [1, 1, 1, 0]

    temps3 := []int{30, 60, 90}
    fmt.Println(dailyTemperatures(temps3))
    // [1, 1, 0]
}
```

**Textual Figure: Daily Temperatures (Monotonic Decreasing Stack)**

```
temps = [73, 74, 75, 71, 69, 72, 76, 73]
index:   0   1   2   3   4   5   6   7

Stack trace (decreasing stack of indices):

i=0 (73): push 0                     stack=[0]
i=1 (74): 74>73 pop 0 → r[0]=1-0=1   stack=[]     push 1
i=2 (75): 75>74 pop 1 → r[1]=2-1=1   stack=[]     push 2
i=3 (71): 71<75                       stack=[2]    push 3  → [2,3]
i=4 (69): 69<71                       stack=[2,3]  push 4  → [2,3,4]
i=5 (72): 72>69 pop 4 → r[4]=5-4=1
          72>71 pop 3 → r[3]=5-3=2   stack=[2]    push 5
i=6 (76): 76>72 pop 5 → r[5]=6-5=1
          76>75 pop 2 → r[2]=6-2=4   stack=[]     push 6
i=7 (73): 73<76                       stack=[6]    push 7

Remaining: r[6]=0, r[7]=0

  Index:   0  1  2  3  4  5  6  7
  Temp:   73 74 75 71 69 72 76 73
  Result:  1  1  4  2  1  1  0  0
           │  │  │  │  │  │
           └→┴→┤  └→┤  │     Days until warmer
                 └───┴──┘
```

---

## Example 5: Largest Rectangle in Histogram (LeetCode 84)

```go
package main

import "fmt"

func largestRectangleArea(heights []int) int {
    stack := []int{}
    maxArea := 0
    n := len(heights)

    for i := 0; i <= n; i++ {
        h := 0
        if i < n {
            h = heights[i]
        }

        for len(stack) > 0 && h < heights[stack[len(stack)-1]] {
            top := stack[len(stack)-1]
            stack = stack[:len(stack)-1]

            width := i
            if len(stack) > 0 {
                width = i - stack[len(stack)-1] - 1
            }

            area := heights[top] * width
            if area > maxArea {
                maxArea = area
            }
        }
        stack = append(stack, i)
    }

    return maxArea
}

func main() {
    tests := [][]int{
        {2, 1, 5, 6, 2, 3},
        {2, 4},
        {1, 1, 1, 1},
        {6, 2, 5, 4, 5, 1, 6},
    }

    for _, h := range tests {
        fmt.Printf("heights=%v → maxArea=%d\n", h, largestRectangleArea(h))
    }
    // [2,1,5,6,2,3] → 10
    // [2,4] → 4
    // [1,1,1,1] → 4
}
```

**Textual Figure: Largest Rectangle in Histogram**

```
heights = [2, 1, 5, 6, 2, 3]

       6
     ┌───┐
   5 │   │
  ┌──┤   │                       3
  │  │   │              ┌───┐
  │  │   │  2       2   │   │
  │  │   ├───┐  ┌───├───┤
  │  │   │   │  │   │   │
  ├──┤   │   │  │   │   │    1
  │  ├───┤   ├──┤   ├───┤  ┌───┐
  │  │   │   │  │   │   │  │   │
  └──┴───┴───┴──┴───┴───┘  └───┘
  [0]  [1]  [2] [3][4]  [5]

Stack trace (increasing stack):
  Push 0(h=2), pop at i=1 → area=2*1=2
  Push 1(h=1), push 2(h=5), push 3(h=6)
  At i=4(h=2): pop 3 → area=6*1=6
               pop 2 → area=5*2=10  ← MAX!
  Push 4(h=2), push 5(h=3)
  At end:      pop 5 → area=3*1=3
               pop 4 → area=2*4=8
               pop 1 → area=1*6=6

  Largest rectangle: 5×2 = 10 (indices 2-3)
  ██████████
  ██████████  ← width=2, height=5
```

---

## Example 6: Trapping Rain Water (Monotonic Stack)

```go
package main

import "fmt"

func trap(height []int) int {
    stack := []int{}
    water := 0

    for i := 0; i < len(height); i++ {
        for len(stack) > 0 && height[i] > height[stack[len(stack)-1]] {
            bottom := stack[len(stack)-1]
            stack = stack[:len(stack)-1]

            if len(stack) == 0 {
                break
            }

            left := stack[len(stack)-1]
            width := i - left - 1
            h := min(height[left], height[i]) - height[bottom]
            water += width * h
        }
        stack = append(stack, i)
    }

    return water
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    tests := [][]int{
        {0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1},
        {4, 2, 0, 3, 2, 5},
    }

    for _, h := range tests {
        fmt.Printf("height=%v → water=%d\n", h, trap(h))
    }
    // [0,1,0,2,1,0,1,3,2,1,2,1] → 6
    // [4,2,0,3,2,5] → 9
}
```

**Textual Figure: Trapping Rain Water (Monotonic Stack)**

```
height = [0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1]

              3
           ┌───┐
     2     │   │        2
  ┌───┐  │   │     ┌───┐
  │░░░│  │   │  2  │░░░│
  │░░░┤  │   ├───├───┤
1 │░░░│  │ 1 │░░░│░░░│ 1
┌├───├──┤ ├───├───├───├───┐
││ ░ │░░░│ │ ░ │░░░│░░░│   │
└┴───┴───┴─┴───┴───┴───┴───┘
0  1  0  2  1  0  1  3  2  1  2  1
░ = trapped water

Stack approach: decreasing stack of indices.
When height[i] > height[stack.top]:
  - Pop bottom bar
  - left = new stack top
  - width = i - left - 1
  - h = min(height[left], height[i]) - height[bottom]
  - water += width * h

Total trapped water = 6 units
```

---

## Example 7: Stock Span Problem

```go
package main

import "fmt"

// Stock span: number of consecutive days before (including today)
// where price was ≤ today's price
func stockSpan(prices []int) []int {
    n := len(prices)
    spans := make([]int, n)
    stack := []int{} // decreasing stack of indices

    for i := 0; i < n; i++ {
        for len(stack) > 0 && prices[stack[len(stack)-1]] <= prices[i] {
            stack = stack[:len(stack)-1]
        }

        if len(stack) == 0 {
            spans[i] = i + 1
        } else {
            spans[i] = i - stack[len(stack)-1]
        }
        stack = append(stack, i)
    }

    return spans
}

func main() {
    prices := []int{100, 80, 60, 70, 60, 75, 85}
    fmt.Println("Prices:", prices)
    fmt.Println("Spans: ", stockSpan(prices))
    // [1, 1, 1, 2, 1, 4, 6]
}
```

**Textual Figure: Stock Span Problem**

```
prices = [100, 80, 60, 70, 60, 75, 85]

Stack trace (decreasing stack of indices by price):

i=0 (100): stack=[]       span=0+1=1  push 0   stack=[0]
i=1 (80):  80≤100         span=1-0=1  push 1   stack=[0,1]
i=2 (60):  60≤80          span=2-1=1  push 2   stack=[0,1,2]
i=3 (70):  pop 2(60≤70)   span=3-1=2  push 3   stack=[0,1,3]
i=4 (60):  60≤70          span=4-3=1  push 4   stack=[0,1,3,4]
i=5 (75):  pop 4(60≤75)
           pop 3(70≤75)   span=5-1=4  push 5   stack=[0,1,5]
i=6 (85):  pop 5(75≤85)
           pop 1(80≤85)   span=6-0=6  push 6   stack=[0,6]

  Day:    0    1    2    3    4    5    6
  Price: 100  80   60   70   60   75   85
  Span:   1    1    1    2    1    4    6
               │    │    ┌┘    │    ┌───┘
               │    │    │     │    │
  Span = consecutive days (including today) where price ≤ today
```

---

## Example 8: Remove K Digits (LeetCode 402)

```go
package main

import (
    "fmt"
    "strings"
)

func removeKdigits(num string, k int) string {
    stack := []byte{}

    for i := 0; i < len(num); i++ {
        // Remove digits from stack that are greater than current
        for k > 0 && len(stack) > 0 && stack[len(stack)-1] > num[i] {
            stack = stack[:len(stack)-1]
            k--
        }
        stack = append(stack, num[i])
    }

    // Remove remaining k digits from end
    stack = stack[:len(stack)-k]

    // Remove leading zeros
    result := strings.TrimLeft(string(stack), "0")
    if result == "" {
        return "0"
    }
    return result
}

func main() {
    tests := []struct {
        num string
        k   int
    }{
        {"1432219", 3},
        {"10200", 1},
        {"10", 2},
        {"9", 1},
        {"112", 1},
    }

    for _, t := range tests {
        fmt.Printf("removeKdigits(%q, %d) = %q\n", t.num, t.k, removeKdigits(t.num, t.k))
    }
    // "1432219",3 → "1219"
    // "10200",1 → "200"
    // "10",2 → "0"
}
```

**Textual Figure: Remove K Digits (Monotonic Increasing Stack)**

```
num = "1432219", k = 3

Process each digit, maintain increasing stack:

  ch='1': stack=[1]                            k=3
  ch='4': 4>1 → push      stack=[1,4]          k=3
  ch='3': 3<4 → pop 4     stack=[1,3]          k=2
  ch='2': 2<3 → pop 3     stack=[1,2]          k=1
  ch='2': 2≤2 → push      stack=[1,2,2]        k=1
  ch='1': 1<2 → pop 2     stack=[1,2,1]        k=0
          (k=0, but we already removed)         k=0
  ch='9': 9>1 → push      stack=[1,2,1,9]      k=0

  Wait—let me retrace correctly:
  ch='1': stack=[1]                            k=3
  ch='4': push             stack=[1,4]          k=3
  ch='3': pop 4(4>3)       stack=[1,3]          k=2
  ch='2': pop 3(3>2)       stack=[1,2]          k=1
  ch='2': push             stack=[1,2,2]        k=1
  ch='1': pop 2(2>1)       stack=[1,2,1]        k=0
  ch='9': push             stack=[1,2,1,9]      k=0

  Result: "1219"

  Stack visualization:
  ┌─┐ ┌─┬─┐ ┌─┬─┐ ┌─┬─┐ ┌─┬─┬─┐ ┌─┬─┬─┐ ┌─┬─┬─┬─┐
  │1│ │4│1│ │3│1│ │2│1│ │2│2│1│ │1│2│1│ │9│1│2│1│
  └─┘ └─┴─┘ └─┴─┘ └─┴─┘ └─┴─┴─┘ └─┴─┴─┘ └─┴─┴─┴─┘
  '1'  '4'   '3'   '2'   '2'    '1'     '9'
  Greedy: remove larger digits early to minimize the number.
```

---

## Example 9: Asteroid Collision (LeetCode 735)

```go
package main

import "fmt"

func asteroidCollision(asteroids []int) []int {
    stack := []int{}

    for _, a := range asteroids {
        alive := true

        for alive && a < 0 && len(stack) > 0 && stack[len(stack)-1] > 0 {
            top := stack[len(stack)-1]

            if top < -a {
                // Top asteroid explodes
                stack = stack[:len(stack)-1]
            } else if top == -a {
                // Both explode
                stack = stack[:len(stack)-1]
                alive = false
            } else {
                // Current asteroid explodes
                alive = false
            }
        }

        if alive {
            stack = append(stack, a)
        }
    }

    return stack
}

func main() {
    tests := [][]int{
        {5, 10, -5},
        {8, -8},
        {10, 2, -5},
        {-2, -1, 1, 2},
        {1, -2, -2, -2},
    }

    for _, t := range tests {
        fmt.Printf("%v → %v\n", t, asteroidCollision(t))
    }
    // [5,10,-5] → [5,10]
    // [8,-8] → []
    // [10,2,-5] → [10]
    // [-2,-1,1,2] → [-2,-1,1,2]
}
```

**Textual Figure: Asteroid Collision**

```
asteroids = [5, 10, -5]

Process each asteroid (positive=right, negative=left):

  a=5:   stack=[5]        (moving right, no collision)
  a=10:  stack=[5,10]     (moving right, no collision)
  a=-5:  │-5│ vs │10│      10 > 5  → -5 explodes!
         stack=[5,10]     (10 survives)

  Result: [5, 10]

  Collision rules:
  ┌───────────────────────────────────────┐
  │  (+) and (-) moving toward each other:  │
  │    │+│ > │-│  →  (-) explodes          │
  │    │+│ < │-│  →  (+) explodes          │
  │    │+│ = │-│  →  both explode          │
  │  Same direction → no collision          │
  └───────────────────────────────────────┘

  [8, -8]:  8 = 8 → both explode → []
  [10, 2, -5]:  -5 vs 2 → 2 explodes; -5 vs 10 → -5 explodes → [10]
```

---

## Example 10: Next Greater Element II (Circular Array, LeetCode 503)

```go
package main

import "fmt"

func nextGreaterElements(nums []int) []int {
    n := len(nums)
    result := make([]int, n)
    for i := range result {
        result[i] = -1
    }

    stack := []int{} // monotonic decreasing

    // Traverse twice to handle circular
    for i := 0; i < 2*n; i++ {
        idx := i % n
        for len(stack) > 0 && nums[idx] > nums[stack[len(stack)-1]] {
            top := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[top] = nums[idx]
        }
        if i < n {
            stack = append(stack, i)
        }
    }

    return result
}

func main() {
    tests := [][]int{
        {1, 2, 1},
        {1, 2, 3, 4, 3},
        {5, 4, 3, 2, 1},
    }

    for _, nums := range tests {
        fmt.Printf("%v → %v\n", nums, nextGreaterElements(nums))
    }
    // [1,2,1] → [2,-1,2]
    // [1,2,3,4,3] → [2,3,4,-1,4]
    // [5,4,3,2,1] → [-1,5,5,5,5]
}
```

**Textual Figure: Next Greater Element II (Circular Array)**

```
nums = [1, 2, 1]  (circular: wraps around)

Traverse 2*n = 6 elements using i % n:

i=0 (1): push 0             stack=[0]
i=1 (2): 2>1 pop 0→r[0]=2   stack=[]   push 1  stack=[1]
i=2 (1): 1<2                 push 2     stack=[1,2]
i=3 (idx=0, val=1): 1<1 no pop                   (only push in first pass)
i=4 (idx=1, val=2): 2>1 pop 2→r[2]=2   stack=[1] (no push, second pass)
i=5 (idx=2, val=1): 1<2 no pop

Remaining: index 1 → r[1]=-1 (no greater in circular scan)

  Circular array visualization:
       ┌───┐
    ┌──┤ 1 ├──┐
    │  └───┘  │
  ┌─┴─┐      ┌─┴─┐
  │ 1 │      │ 2 │
  └─┬─┘      └─┬─┘
    │  ┌───┐  │       Index: 0  1  2
    └──┤   ├──┘       Input: 1  2  1
       └───┘           Result: 2 -1  2
  Traverse indices 0..2n-1 with idx = i%n
  to handle wrap-around.
```

---

## Example 11: Sum of Subarray Minimums (LeetCode 907)

```go
package main

import "fmt"

func sumSubarrayMins(arr []int) int {
    const MOD = 1_000_000_007
    n := len(arr)

    // left[i]  = number of subarrays ending at i where arr[i] is min
    // right[i] = number of subarrays starting at i where arr[i] is min
    left := make([]int, n)
    right := make([]int, n)

    // Previous less element
    stack := []int{}
    for i := 0; i < n; i++ {
        for len(stack) > 0 && arr[stack[len(stack)-1]] >= arr[i] {
            stack = stack[:len(stack)-1]
        }
        if len(stack) == 0 {
            left[i] = i + 1
        } else {
            left[i] = i - stack[len(stack)-1]
        }
        stack = append(stack, i)
    }

    // Next less element
    stack = stack[:0]
    for i := n - 1; i >= 0; i-- {
        for len(stack) > 0 && arr[stack[len(stack)-1]] > arr[i] {
            stack = stack[:len(stack)-1]
        }
        if len(stack) == 0 {
            right[i] = n - i
        } else {
            right[i] = stack[len(stack)-1] - i
        }
        stack = append(stack, i)
    }

    result := 0
    for i := 0; i < n; i++ {
        result = (result + arr[i]*left[i]*right[i]) % MOD
    }

    return result
}

func main() {
    tests := [][]int{
        {3, 1, 2, 4},
        {11, 81, 94, 43, 3},
    }

    for _, arr := range tests {
        fmt.Printf("arr=%v → sum=%d\n", arr, sumSubarrayMins(arr))
    }
    // [3,1,2,4] → 17
    // [11,81,94,43,3] → 444
}
```

**Textual Figure: Sum of Subarray Minimums**

```
arr = [3, 1, 2, 4]

For each element, count subarrays where it's the minimum:

  left[i] = # of subarrays ending at i where arr[i] is min
  right[i] = # of subarrays starting at i where arr[i] is min

  Prev less (left):           Next less (right):
  i=0: no prev less, left=1   i=3: no next less, right=1
  i=1: no prev less, left=2   i=2: next less at 1, right=1
  i=2: prev less at 1, left=1 i=1: no next less, right=3
  i=3: prev less at 1, left=2 i=0: next less at 1, right=1  (strict)

  Index:  0    1    2    3
  Value:  3    1    2    4
  left:   1    2    1    2
  right:  1    3    1    1

  Contribution of each element:
  arr[i] * left[i] * right[i]
  3*1*1 = 3
  1*2*3 = 6
  2*1*1 = 2
  4*2*1 = 8   (wait, should be 4*1*1 with correct boundaries)

  Actually:
  left=[1,2,1,1], right=[1,3,2,1]
  3*1*1 + 1*2*3 + 2*1*2 + 4*1*1 = 3+6+4+4 = 17  ✓

  All subarrays and their mins:
  [3]=3  [1]=1  [2]=2  [4]=4
  [3,1]=1  [1,2]=1  [2,4]=2
  [3,1,2]=1  [1,2,4]=1
  [3,1,2,4]=1
  Sum = 3+1+2+4+1+1+2+1+1+1 = 17
```

---

## Monotonic Stack Pattern Summary

| Problem | Stack Type | Direction |
|---------|-----------|-----------|
| Next Greater | Decreasing | Left → Right |
| Next Smaller | Increasing | Left → Right |
| Prev Greater | Decreasing | Left → Right |
| Prev Smaller | Increasing | Left → Right |
| Histogram | Increasing | Left → Right |
| Rain Water | Decreasing | Left → Right |
| Stock Span | Decreasing | Left → Right |

## Key Takeaways

1. **Monotonic decreasing stack** → finds next/prev **greater** element
2. **Monotonic increasing stack** → finds next/prev **smaller** element
3. Stack stores **indices** (not values) — enables distance/width calculation
4. Every element is pushed and popped **at most once** → O(n) total
5. For **circular arrays**, traverse `2*n` elements using modulo
6. **Histogram/rain water**: think about what each bar "contributes" when popped

> **Next up:** Expression Evaluation →
