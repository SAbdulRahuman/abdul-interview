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

---

## Key Takeaways

1. **Fixed window**: compute first window, then slide (add right, remove left)
2. **Variable window**: expand right, shrink left when condition breaks
3. **Window validity**: use counters, maps, or variables to track state
4. **Count subarrays**: when window [left, right] is valid, `right-left+1` new valid subarrays
5. **Common trick**: track `maxFreq` to avoid recomputing (like character replacement)
6. **Time**: O(n) — each element enters and leaves the window at most once

> **Next up:** Kadane's Algorithm →
