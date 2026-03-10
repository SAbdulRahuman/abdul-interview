# Phase 2: Arrays — Sliding Window

## What is the Sliding Window Technique?

A **sliding window** maintains a contiguous subarray/substring while expanding and shrinking to find optimal results. It reduces brute force O(n²) or O(n³) to **O(n)**.

---

## Two Types

| Type | Description | When to Use |
|------|-------------|-------------|
| Fixed-size | Window always has size k | "Find max sum of k elements" |
| Variable-size | Window grows/shrinks | "Find smallest window with condition" |

---

## Example 1: Maximum Sum of Subarray of Size K (Fixed Window)

```go
package main

import "fmt"

func maxSumK(nums []int, k int) int {
    // Compute first window
    windowSum := 0
    for i := 0; i < k; i++ {
        windowSum += nums[i]
    }
    maxSum := windowSum

    // Slide: add right, remove left
    for i := k; i < len(nums); i++ {
        windowSum += nums[i] - nums[i-k]
        if windowSum > maxSum {
            maxSum = windowSum
        }
    }
    return maxSum
}

func main() {
    fmt.Println(maxSumK([]int{2, 1, 5, 1, 3, 2}, 3))         // 9 → [5,1,3]
    fmt.Println(maxSumK([]int{2, 3, 4, 1, 5}, 2))             // 7 → [3,4]
    fmt.Println(maxSumK([]int{100, 200, 300, 400}, 2))         // 700 → [300,400]
}
```

**Textual Figure — Fixed-Size Sliding Window (Max Sum K):**

```
  nums = [2, 1, 5, 1, 3, 2]   k = 3

  Window slides →, adding right, removing left:

  [2, 1, 5] 1, 3, 2   sum=8
  └──k=3──┘

   2 [1, 5, 1] 3, 2   sum=8-2+1=7
      └──k=3──┘

   2, 1 [5, 1, 3] 2   sum=7-1+3=9  ← max!
         └──k=3──┘

   2, 1, 5 [1, 3, 2]  sum=9-5+2=6
            └──k=3──┘

  Result: max = 9  (window [5,1,3])

  Slide operation: O(1) per step!
    windowSum = windowSum + nums[i] - nums[i-k]
```

---

## Example 2: Minimum Size Subarray Sum (Variable Window)

```go
package main

import (
    "fmt"
    "math"
)

func minSubArrayLen(target int, nums []int) int {
    left := 0
    sum := 0
    minLen := math.MaxInt64

    for right := 0; right < len(nums); right++ {
        sum += nums[right]

        for sum >= target {
            windowLen := right - left + 1
            if windowLen < minLen {
                minLen = windowLen
            }
            sum -= nums[left]
            left++
        }
    }

    if minLen == math.MaxInt64 {
        return 0
    }
    return minLen
}

func main() {
    fmt.Println(minSubArrayLen(7, []int{2, 3, 1, 2, 4, 3})) // 2 → [4,3]
    fmt.Println(minSubArrayLen(4, []int{1, 4, 4}))           // 1 → [4]
    fmt.Println(minSubArrayLen(11, []int{1, 1, 1, 1}))       // 0 → impossible
}
```

**Textual Figure — Variable-Size Window (Min Size Subarray Sum):**

```
  nums = [2, 3, 1, 2, 4, 3]   target = 7

  Expand right until sum ≥ target, then shrink left:

  [2]                    sum=2 < 7 → expand
  [2, 3]                 sum=5 < 7 → expand
  [2, 3, 1]              sum=6 < 7 → expand
  [2, 3, 1, 2]           sum=8 ≥ 7 → len=4, shrink
     [3, 1, 2]           sum=6 < 7 → expand
     [3, 1, 2, 4]        sum=10 ≥ 7 → len=4, shrink
        [1, 2, 4]        sum=7 ≥ 7 → len=3, shrink
           [2, 4]        sum=6 < 7 → expand
           [2, 4, 3]     sum=9 ≥ 7 → len=3, shrink
              [4, 3]     sum=7 ≥ 7 → len=2, shrink  ← min!
                 [3]     sum=3 < 7 → done

  Result: min length = 2  (subarray [4,3])  ✓
```

---

## Example 3: Longest Substring Without Repeating Characters

```go
package main

import "fmt"

func lengthOfLongestSubstring(s string) int {
    charIndex := make(map[byte]int) // last seen position
    maxLen := 0
    left := 0

    for right := 0; right < len(s); right++ {
        ch := s[right]
        if idx, found := charIndex[ch]; found && idx >= left {
            left = idx + 1 // shrink past the duplicate
        }
        charIndex[ch] = right
        if right-left+1 > maxLen {
            maxLen = right - left + 1
        }
    }
    return maxLen
}

func main() {
    fmt.Println(lengthOfLongestSubstring("abcabcbb"))  // 3 → "abc"
    fmt.Println(lengthOfLongestSubstring("bbbbb"))     // 1 → "b"
    fmt.Println(lengthOfLongestSubstring("pwwkew"))    // 3 → "wke"
    fmt.Println(lengthOfLongestSubstring(""))          // 0
    fmt.Println(lengthOfLongestSubstring("dvdf"))      // 3 → "vdf"
}
```

**Textual Figure — Longest Substring Without Repeating Characters:**

```
  s = "abcabcbb"

  L=0, R expands:
  R=0: 'a' → {a:0}          window="a"     len=1
  R=1: 'b' → {a:0,b:1}      window="ab"    len=2
  R=2: 'c' → {a:0,b:1,c:2}  window="abc"   len=3
  R=3: 'a' → dup! a was at 0, L=0+1=1
       {a:3,b:1,c:2}        window="bca"   len=3
  R=4: 'b' → dup! b was at 1, L=1+1=2
       {a:3,b:4,c:2}        window="cab"   len=3
  R=5: 'c' → dup! c was at 2, L=2+1=3
       {a:3,b:4,c:5}        window="abc"   len=3
  R=6: 'b' → dup! b was at 4, L=4+1=5
       {a:3,b:6,c:5}        window="cb"    len=2
  R=7: 'b' → dup! b was at 6, L=6+1=7
       {a:3,b:7,c:5}        window="b"     len=1

  Max length = 3 ("abc")  ✓

  Key: on duplicate, jump L past the previous occurrence!
```

---

## Example 4: Maximum of All Subarrays of Size K (Deque Approach)

```go
package main

import "fmt"

func maxSlidingWindow(nums []int, k int) []int {
    var deque []int // stores indices, front = max
    var result []int

    for i := 0; i < len(nums); i++ {
        // Remove indices outside window
        for len(deque) > 0 && deque[0] <= i-k {
            deque = deque[1:]
        }
        // Remove smaller elements from back
        for len(deque) > 0 && nums[deque[len(deque)-1]] <= nums[i] {
            deque = deque[:len(deque)-1]
        }
        deque = append(deque, i)

        // Window is full → record result
        if i >= k-1 {
            result = append(result, nums[deque[0]])
        }
    }
    return result
}

func main() {
    fmt.Println(maxSlidingWindow([]int{1, 3, -1, -3, 5, 3, 6, 7}, 3))
    // [3 3 5 5 6 7]
    fmt.Println(maxSlidingWindow([]int{1}, 1))
    // [1]
    fmt.Println(maxSlidingWindow([]int{9, 11}, 2))
    // [11]
}
```

**Textual Figure — Sliding Window Maximum (Deque):**

```
  nums = [1, 3, -1, -3, 5, 3, 6, 7]   k = 3

  Deque stores indices, front = current window max

  i=0: deque=[0]              nums: 1
  i=1: 3>1, pop 0. deque=[1]  nums: 3
  i=2: deque=[1,2]            window full → max=nums[1]=3
  i=3: deque=[1,2,3]          pop expired? 1>3-3=0✓
                              window max=nums[1]=3
  i=4: 5>-3,-1,3 pop all. deque=[4]  max=nums[4]=5
  i=5: deque=[4,5]            max=nums[4]=5
  i=6: 6>3,5 pop. deque=[6]  max=nums[6]=6
  i=7: 7>6 pop. deque=[7]    max=nums[7]=7

  Result: [3, 3, 5, 5, 6, 7]

  Deque invariant:
  ┌─────────────────────────────────────┐
  │ Elements in deque are decreasing   │
  │ Front = index of window maximum    │
  │ Remove front if outside window     │
  │ Remove back if ≤ new element        │
  └─────────────────────────────────────┘
```

---

## Example 5: Longest Repeating Character Replacement

```go
package main

import "fmt"

func characterReplacement(s string, k int) int {
    count := [26]int{}
    maxFreq := 0
    maxLen := 0
    left := 0

    for right := 0; right < len(s); right++ {
        count[s[right]-'A']++
        if count[s[right]-'A'] > maxFreq {
            maxFreq = count[s[right]-'A']
        }

        // Window size - max frequency = chars to replace
        windowSize := right - left + 1
        if windowSize-maxFreq > k {
            count[s[left]-'A']--
            left++
        }

        if right-left+1 > maxLen {
            maxLen = right - left + 1
        }
    }
    return maxLen
}

func main() {
    fmt.Println(characterReplacement("ABAB", 2))    // 4
    fmt.Println(characterReplacement("AABABBA", 1)) // 4 → "AABA" or "ABBB"
    fmt.Println(characterReplacement("AAAA", 0))    // 4
}
```

**Textual Figure — Longest Repeating Character Replacement:**

```
  s = "AABABBA"   k = 1 (can replace 1 char)

  Window: chars_to_replace = windowSize - maxFreq
  If chars_to_replace > k → shrink left

  R=0: "A"       maxFreq=1  replace=1-1=0 ≤ 1 ✓ len=1
  R=1: "AA"      maxFreq=2  replace=2-2=0 ≤ 1 ✓ len=2
  R=2: "AAB"     maxFreq=2  replace=3-2=1 ≤ 1 ✓ len=3
  R=3: "AABA"    maxFreq=3  replace=4-3=1 ≤ 1 ✓ len=4
  R=4: "AABAB"   maxFreq=3  replace=5-3=2 > 1 ✗ shrink!
       "ABAB"    L=1       replace=4-2=2 > 1 ✗ shrink!
       "BAB"     L=2       replace=3-1=2 > 1 ✗ shrink!
       "AB"      L=3       replace=2-1=1 ≤ 1 ✓ len=2
  R=5: "ABB"     maxFreq=2  replace=3-2=1 ≤ 1 ✓ len=3
  R=6: "ABBA"    maxFreq=2  replace=4-2=2 > 1 ✗ shrink!
       "BBA"     L=4       replace=3-2=1 ≤ 1 ✓ len=3

  Max = 4 ("AABA" → replace B with A → "AAAA")
```

---

## Example 6: Count Subarrays with Product Less Than K

```go
package main

import "fmt"

func numSubarrayProductLessThanK(nums []int, k int) int {
    if k <= 1 {
        return 0
    }

    count := 0
    product := 1
    left := 0

    for right := 0; right < len(nums); right++ {
        product *= nums[right]

        for product >= k {
            product /= nums[left]
            left++
        }
        // All subarrays ending at right: [left,right], [left+1,right], ...
        count += right - left + 1
    }
    return count
}

func main() {
    fmt.Println(numSubarrayProductLessThanK([]int{10, 5, 2, 6}, 100)) // 8
    fmt.Println(numSubarrayProductLessThanK([]int{1, 2, 3}, 0))       // 0
    fmt.Println(numSubarrayProductLessThanK([]int{1, 1, 1}, 2))       // 6
}
```

**Textual Figure — Count Subarrays With Product < K:**

```
  nums = [10, 5, 2, 6]   k = 100

  Variable window: expand right, shrink left if product ≥ k

  R=0: product=10 < 100  → count += 1 (subarrays: [10])
  R=1: product=50 < 100  → count += 2 (subarrays: [5], [10,5])
  R=2: product=100 ≥ 100 → shrink! product/=10 → 10
       product=10 < 100  → count += 2 (subarrays: [2], [5,2])
  R=3: product=60 < 100  → count += 3 (subarrays: [6], [2,6], [5,2,6])

  Total: 1 + 2 + 2 + 3 = 8  ✓

  Why count += right - left + 1?
  ─────────────────────────
  Window [L..R] is valid. New subarrays ending at R:
  [R], [R-1,R], [R-2,R], ..., [L,R]
  That's exactly R - L + 1 subarrays!
```

---

## Example 7: Find All Anagrams in a String

```go
package main

import "fmt"

func findAnagrams(s string, p string) []int {
    if len(s) < len(p) {
        return nil
    }

    var pCount, sCount [26]int
    for _, c := range p {
        pCount[c-'a']++
    }

    var result []int
    for i := 0; i < len(s); i++ {
        sCount[s[i]-'a']++

        // Remove left element when window exceeds p length
        if i >= len(p) {
            sCount[s[i-len(p)]-'a']--
        }

        if sCount == pCount {
            result = append(result, i-len(p)+1)
        }
    }
    return result
}

func main() {
    fmt.Println(findAnagrams("cbaebabacd", "abc")) // [0 6]
    fmt.Println(findAnagrams("abab", "ab"))         // [0 1 2]
    fmt.Println(findAnagrams("hello", "xyz"))       // []
}
```

**Textual Figure — Find All Anagrams:**

```
  s = "cbaebabacd"   p = "abc"   pCount = {a:1, b:1, c:1}

  Fixed window of size len(p) = 3:

  i=0: window="c"      sCount={c:1}
  i=1: window="cb"     sCount={c:1,b:1}
  i=2: window="cba"    sCount={c:1,b:1,a:1} == pCount ✓ idx=0
  i=3: window="bae"    sCount={b:1,a:1,e:1}  remove c
  i=4: window="aeb"    sCount={a:1,e:1,b:1}  ✗
  i=5: window="eba"    sCount={e:1,b:1,a:1}  ✗
  i=6: window="bab"    sCount={b:2,a:1}       ✗
  i=7: window="aba"    sCount={a:2,b:1}       ✗
  i=8: window="bac"    sCount={b:1,a:1,c:1} == pCount ✓ idx=6
  i=9: window="acd"    sCount={a:1,c:1,d:1}  ✗

  Result: [0, 6]

  Trick: compare two [26]int arrays for O(1) matching!
```

---

## Example 8: Minimum Window Substring

```go
package main

import (
    "fmt"
    "math"
)

func minWindow(s string, t string) string {
    need := make(map[byte]int)
    for i := 0; i < len(t); i++ {
        need[t[i]]++
    }

    have := make(map[byte]int)
    formed := 0
    required := len(need)

    minLen := math.MaxInt64
    start := 0
    left := 0

    for right := 0; right < len(s); right++ {
        c := s[right]
        have[c]++
        if need[c] > 0 && have[c] == need[c] {
            formed++
        }

        for formed == required {
            windowLen := right - left + 1
            if windowLen < minLen {
                minLen = windowLen
                start = left
            }
            lc := s[left]
            have[lc]--
            if need[lc] > 0 && have[lc] < need[lc] {
                formed--
            }
            left++
        }
    }

    if minLen == math.MaxInt64 {
        return ""
    }
    return s[start : start+minLen]
}

func main() {
    fmt.Println(minWindow("ADOBECODEBANC", "ABC")) // "BANC"
    fmt.Println(minWindow("a", "a"))                 // "a"
    fmt.Println(minWindow("a", "aa"))                // ""
}
```

**Textual Figure — Minimum Window Substring:**

```
  s = "ADOBECODEBANC"   t = "ABC"
  need: {A:1, B:1, C:1}   required = 3

  Expand right until all chars found, then shrink left:

  R=0 'A': have={A:1} formed=1            "A"
  R=1 'D': formed=1                       "AD"
  R=2 'O': formed=1                       "ADO"
  R=3 'B': have={A:1,B:1} formed=2        "ADOB"
  R=4 'E': formed=2                       "ADOBE"
  R=5 'C': have={A:1,B:1,C:1} formed=3   "ADOBEC" len=6 ← valid!
    shrink: remove A → formed=2, L=1
  R=6..9: expanding...                    "DOBECODEB"
  R=10 'A': formed=3 again                "DOBECODEBA" len=10
    shrink: D,O,B,E,C,O,D → still valid
    shrink: remove E → "BANC" formed=3, len=4
    shrink: remove B → formed=2

  Best: "BANC" (start=9, len=4)  ✓

  Pattern: expand to satisfy, shrink to minimize
```

---

## Example 9: Maximum Consecutive Ones III (Flip at Most K Zeros)

```go
package main

import "fmt"

func longestOnes(nums []int, k int) int {
    left := 0
    zeros := 0
    maxLen := 0

    for right := 0; right < len(nums); right++ {
        if nums[right] == 0 {
            zeros++
        }

        for zeros > k {
            if nums[left] == 0 {
                zeros--
            }
            left++
        }

        if right-left+1 > maxLen {
            maxLen = right - left + 1
        }
    }
    return maxLen
}

func main() {
    fmt.Println(longestOnes([]int{1, 1, 1, 0, 0, 0, 1, 1, 1, 1, 0}, 2)) // 6
    fmt.Println(longestOnes([]int{0, 0, 1, 1, 0, 0, 1, 1, 1, 0, 1, 1, 0, 0, 0, 1, 1, 1, 1}, 3)) // 10
}
```

**Textual Figure — Max Consecutive Ones III (Flip K Zeros):**

```
  nums = [1, 1, 1, 0, 0, 0, 1, 1, 1, 1, 0]   k = 2

  Window tracks count of zeros. If zeros > k → shrink.

  [1, 1, 1]              zeros=0  len=3
  [1, 1, 1, 0]           zeros=1  len=4
  [1, 1, 1, 0, 0]        zeros=2  len=5
  [1, 1, 1, 0, 0, 0]     zeros=3 > k=2 → shrink!
     [1, 1, 0, 0, 0]     zeros=3 > k=2 → shrink!
        [1, 0, 0, 0]     zeros=3 > k=2 → shrink!
           [0, 0, 0]     zeros=3 > k=2 → shrink!
              [0, 0]     zeros=2  len=2
              [0, 0, 1]  zeros=2  len=3
              [0, 0, 1, 1]        len=4
              [0, 0, 1, 1, 1]     len=5
              [0, 0, 1, 1, 1, 1]  len=6  ← max!
              [0, 0, 1, 1, 1, 1, 0] zeros=3 → shrink
                 [0, 1, 1, 1, 1, 0] zeros=2  len=6  ← tie!

  Result: max = 6  (flip 2 zeros in [0,0,1,1,1,1])  ✓
```

---

## Example 10: Sliding Window Template

```go
package main

import "fmt"

// Generic sliding window template
func slidingWindowTemplate(nums []int, condition func(windowData map[int]int) bool) int {
    windowData := make(map[int]int)
    left := 0
    result := 0

    for right := 0; right < len(nums); right++ {
        // Expand: add nums[right] to window
        windowData[nums[right]]++

        // Shrink: while condition violated
        for !condition(windowData) {
            windowData[nums[left]]--
            if windowData[nums[left]] == 0 {
                delete(windowData, nums[left])
            }
            left++
        }

        // Update result (depends on problem)
        windowLen := right - left + 1
        if windowLen > result {
            result = windowLen
        }
    }
    return result
}

func main() {
    // Example: longest subarray with at most 2 distinct values
    nums := []int{1, 2, 1, 2, 3, 3, 2, 1}
    result := slidingWindowTemplate(nums, func(data map[int]int) bool {
        return len(data) <= 2
    })
    fmt.Println("Longest with ≤2 distinct:", result) // 4

    // Example: longest subarray where all elements are same
    nums2 := []int{1, 1, 1, 2, 2, 3, 3, 3, 3}
    result2 := slidingWindowTemplate(nums2, func(data map[int]int) bool {
        return len(data) <= 1
    })
    fmt.Println("Longest with 1 distinct:", result2) // 4
}
```

**Textual Figure — Sliding Window Template:**

```
  Template structure:
  ┌───────────────────────────────────────┐
  │  for right in range(n):               │
  │    1. EXPAND: add nums[right]          │
  │    2. SHRINK: while condition violated │
  │       remove nums[left], left++        │
  │    3. UPDATE: result = max/min/count   │
  └───────────────────────────────────────┘

  Example: longest subarray with ≤ 2 distinct values
  nums = [1, 2, 1, 2, 3, 3, 2, 1]

  R=0: {1:1}               len=1
  R=1: {1:1, 2:1}          len=2  (2 distinct ≤ 2 ✓)
  R=2: {1:2, 2:1}          len=3  (2 distinct ≤ 2 ✓)
  R=3: {1:2, 2:2}          len=4  (2 distinct ≤ 2 ✓) ← max!
  R=4: {1:2, 2:2, 3:1}     3 distinct > 2 ✗ shrink!
       L=1: {1:1, 2:2, 3:1}  still 3 ✗
       L=2: {2:2, 3:1}      2 ≤ 2 ✓  len=3
  ...continues...

  Result: 4
```

---

## Key Takeaways

1. **Fixed window**: compute first window, then slide (add right, remove left)
2. **Variable window**: expand right, shrink left when condition breaks
3. **Window validity**: use counters, maps, or variables to track state
4. **Count subarrays**: when window [left, right] is valid, `right-left+1` new valid subarrays
5. **Common trick**: track `maxFreq` to avoid recomputing (like character replacement)
6. **Time**: O(n) — each element enters and leaves the window at most once

> **Next up:** Kadane's Algorithm →
