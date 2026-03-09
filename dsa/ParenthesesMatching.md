# Phase 6: Stack — Parentheses Matching

## Overview

**Parentheses matching** is the most classic stack problem: push opening brackets, pop on closing brackets, and check they match.

**Key insight:** A stack naturally tracks nesting depth and pairing order.

---

## Example 1: Valid Parentheses (LeetCode 20)

```go
package main

import "fmt"

func isValid(s string) bool {
    stack := []byte{}
    match := map[byte]byte{
        ')': '(',
        ']': '[',
        '}': '{',
    }

    for i := 0; i < len(s); i++ {
        ch := s[i]
        if ch == '(' || ch == '[' || ch == '{' {
            stack = append(stack, ch)
        } else {
            if len(stack) == 0 || stack[len(stack)-1] != match[ch] {
                return false
            }
            stack = stack[:len(stack)-1]
        }
    }

    return len(stack) == 0
}

func main() {
    tests := []string{
        "()",
        "()[]{}",
        "(]",
        "([)]",
        "{[]}",
        "",
        "(((",
        ")",
    }

    for _, s := range tests {
        fmt.Printf("%-10q → %v\n", s, isValid(s))
    }
}
```

---

## Example 2: Minimum Add to Make Valid (LeetCode 921)

```go
package main

import "fmt"

func minAddToMakeValid(s string) int {
    open := 0  // unmatched '('
    close := 0 // unmatched ')'

    for _, ch := range s {
        if ch == '(' {
            open++
        } else {
            if open > 0 {
                open--
            } else {
                close++
            }
        }
    }

    return open + close
}

func main() {
    tests := []string{
        "())",
        "(((",
        "()",
        "()))((",
    }

    for _, s := range tests {
        fmt.Printf("%-10q → %d additions\n", s, minAddToMakeValid(s))
    }
    // "())"    → 1
    // "((("    → 3
    // "()"     → 0
    // "()))((" → 4
}
```

---

## Example 3: Longest Valid Parentheses (LeetCode 32)

```go
package main

import "fmt"

func longestValidParentheses(s string) int {
    stack := []int{-1} // base index
    maxLen := 0

    for i := 0; i < len(s); i++ {
        if s[i] == '(' {
            stack = append(stack, i)
        } else {
            stack = stack[:len(stack)-1]
            if len(stack) == 0 {
                stack = append(stack, i) // new base
            } else {
                length := i - stack[len(stack)-1]
                if length > maxLen {
                    maxLen = length
                }
            }
        }
    }

    return maxLen
}

func main() {
    tests := []string{
        "(()",
        ")()())",
        "",
        "()(())",
        "()()()",
        "(()()",
    }

    for _, s := range tests {
        fmt.Printf("%-10q → %d\n", s, longestValidParentheses(s))
    }
    // "(()": 2, ")()())": 4, "()()()": 6
}
```

---

## Example 4: Remove Invalid Parentheses (Minimum Removal)

```go
package main

import "fmt"

// Remove minimum parentheses to make string valid
func minRemoveToMakeValid(s string) string {
    bytes := []byte(s)
    stack := []int{} // indices of unmatched '('

    // Mark unmatched ')'
    for i, ch := range bytes {
        if ch == '(' {
            stack = append(stack, i)
        } else if ch == ')' {
            if len(stack) > 0 {
                stack = stack[:len(stack)-1]
            } else {
                bytes[i] = '#' // mark for removal
            }
        }
    }

    // Mark unmatched '('
    for _, idx := range stack {
        bytes[idx] = '#'
    }

    // Build result without '#'
    result := []byte{}
    for _, ch := range bytes {
        if ch != '#' {
            result = append(result, ch)
        }
    }

    return string(result)
}

func main() {
    tests := []string{
        "lee(t(c)o)de)",
        "a)b(c)d",
        "))((",
        "(a(b(c)d)",
    }

    for _, s := range tests {
        fmt.Printf("%-20q → %q\n", s, minRemoveToMakeValid(s))
    }
}
```

---

## Example 5: Generate Parentheses (LeetCode 22)

```go
package main

import "fmt"

func generateParenthesis(n int) []string {
    result := []string{}

    var backtrack func(current string, open, close int)
    backtrack = func(current string, open, close int) {
        if len(current) == 2*n {
            result = append(result, current)
            return
        }

        if open < n {
            backtrack(current+"(", open+1, close)
        }
        if close < open {
            backtrack(current+")", open, close+1)
        }
    }

    backtrack("", 0, 0)
    return result
}

func main() {
    for n := 1; n <= 4; n++ {
        combos := generateParenthesis(n)
        fmt.Printf("n=%d (%d combos): %v\n", n, len(combos), combos)
    }
    // n=1: ["()"]
    // n=2: ["(())", "()()"]
    // n=3: ["((()))", "(()())", "(())()", "()(())", "()()()"]
}
```

---

## Example 6: Score of Parentheses (LeetCode 856)

```go
package main

import "fmt"

// () = 1, (A) = 2*A, AB = A + B
func scoreOfParentheses(s string) int {
    stack := []int{0} // running scores at each depth

    for _, ch := range s {
        if ch == '(' {
            stack = append(stack, 0)
        } else {
            inner := stack[len(stack)-1]
            stack = stack[:len(stack)-1]

            var score int
            if inner == 0 {
                score = 1 // () = 1
            } else {
                score = 2 * inner // (A) = 2*A
            }
            stack[len(stack)-1] += score
        }
    }

    return stack[0]
}

func main() {
    tests := []string{
        "()",
        "(())",
        "()()",
        "(()(()))",
    }

    for _, s := range tests {
        fmt.Printf("%-12q → %d\n", s, scoreOfParentheses(s))
    }
    // "()"       → 1
    // "(())"     → 2
    // "()()"     → 2
    // "(()(()))" → 6
}
```

---

## Example 7: Remove Outermost Parentheses (LeetCode 1021)

```go
package main

import "fmt"

func removeOuterParentheses(s string) string {
    result := []byte{}
    depth := 0

    for i := 0; i < len(s); i++ {
        if s[i] == '(' {
            if depth > 0 {
                result = append(result, s[i])
            }
            depth++
        } else {
            depth--
            if depth > 0 {
                result = append(result, s[i])
            }
        }
    }

    return string(result)
}

func main() {
    tests := []string{
        "(()())(())",
        "(()())(())(()(()))",
        "()()",
    }

    for _, s := range tests {
        fmt.Printf("%-25q → %q\n", s, removeOuterParentheses(s))
    }
    // "(()())(())"           → "()()()"
    // "(()())(())(()(()))"   → "()()()()(())"
    // "()()"                 → ""
}
```

---

## Example 8: Check if Parentheses String Can Be Valid (LeetCode 2116)

```go
package main

import "fmt"

// s has '(' and ')'. locked[i]='1' means s[i] is fixed, '0' means changeable.
func canBeValid(s string, locked string) bool {
    n := len(s)
    if n%2 != 0 {
        return false
    }

    // Left to right: ensure ')' never exceeds possible '('
    open := 0
    for i := 0; i < n; i++ {
        if locked[i] == '0' || s[i] == '(' {
            open++
        } else {
            open--
        }
        if open < 0 {
            return false
        }
    }

    // Right to left: ensure '(' never exceeds possible ')'
    close := 0
    for i := n - 1; i >= 0; i-- {
        if locked[i] == '0' || s[i] == ')' {
            close++
        } else {
            close--
        }
        if close < 0 {
            return false
        }
    }

    return true
}

func main() {
    tests := []struct {
        s, locked string
    }{
        {"))()))", "010100"},
        {"()()", "0000"},
        {")", "0"},
    }

    for _, t := range tests {
        fmt.Printf("s=%q locked=%q → %v\n", t.s, t.locked, canBeValid(t.s, t.locked))
    }
    // true, true, false
}
```

---

## Example 9: Maximum Nesting Depth of Parentheses (LeetCode 1614)

```go
package main

import "fmt"

func maxDepth(s string) int {
    depth := 0
    maxD := 0

    for _, ch := range s {
        if ch == '(' {
            depth++
            if depth > maxD {
                maxD = depth
            }
        } else if ch == ')' {
            depth--
        }
    }

    return maxD
}

func main() {
    tests := []string{
        "(1+(2*3)+((8)/4))+1",
        "(1)+((2))+(((3)))",
        "1+(2*3)/(2-1)",
        "1",
    }

    for _, s := range tests {
        fmt.Printf("%-30q → depth=%d\n", s, maxDepth(s))
    }
    // 3, 3, 1, 0
}
```

---

## Example 10: Reverse Substrings Between Each Pair of Parentheses (LeetCode 1190)

```go
package main

import "fmt"

func reverseParentheses(s string) string {
    stack := [][]byte{}
    current := []byte{}

    for i := 0; i < len(s); i++ {
        if s[i] == '(' {
            stack = append(stack, current)
            current = []byte{}
        } else if s[i] == ')' {
            // Reverse current
            for l, r := 0, len(current)-1; l < r; l, r = l+1, r-1 {
                current[l], current[r] = current[r], current[l]
            }
            // Append to previous level
            prev := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            current = append(prev, current...)
        } else {
            current = append(current, s[i])
        }
    }

    return string(current)
}

func main() {
    tests := []string{
        "(abcd)",
        "(u(love)i)",
        "(ed(et(oc))el)",
        "a(bcdefghijkl(mno)p)q",
    }

    for _, s := range tests {
        fmt.Printf("%-30q → %q\n", s, reverseParentheses(s))
    }
    // "(abcd)"          → "dcba"
    // "(u(love)i)"      → "iloveu"
    // "(ed(et(oc))el)"  → "leetcode"
}
```

---

## Example 11: Validate Wildcard Parentheses (LeetCode 678)

```go
package main

import "fmt"

// '*' can be '(', ')' or empty
func checkValidString(s string) bool {
    // Track range of possible open count
    minOpen := 0 // min possible '(' count
    maxOpen := 0 // max possible '(' count

    for _, ch := range s {
        switch ch {
        case '(':
            minOpen++
            maxOpen++
        case ')':
            minOpen--
            maxOpen--
        case '*':
            minOpen-- // treat as ')'
            maxOpen++ // treat as '('
        }

        if maxOpen < 0 {
            return false // too many ')'
        }
        if minOpen < 0 {
            minOpen = 0 // can't have negative open count
        }
    }

    return minOpen == 0
}

func main() {
    tests := []string{
        "()",
        "(*)",
        "(*))",
        "(*)))",
        "(((*)",
        "(((******))",
    }

    for _, s := range tests {
        fmt.Printf("%-20q → %v\n", s, checkValidString(s))
    }
}
```

---

## Parentheses Problem Patterns

| Problem | Technique | Key Insight |
|---------|-----------|-------------|
| Valid parentheses | Stack matching | Push open, pop on close |
| Longest valid | Stack with base index | Track start of valid sequence |
| Min add to valid | Counter | Count unmatched open and close |
| Generate all | Backtracking | open < n and close < open |
| Score | Stack of scores | Depth-based scoring |
| Max depth | Counter | Track max of running depth |
| Wildcard (*) | Range tracking | minOpen ≤ actual ≤ maxOpen |

## Key Takeaways

1. **Basic matching**: push opening, pop and compare on closing — O(n) time, O(n) space
2. **Counter approach**: for single bracket type, just track `open` count — O(1) space
3. **Longest valid**: use stack with a "base" index to compute lengths
4. **Wildcard**: maintain a range `[minOpen, maxOpen]` — greedy O(n) solution
5. **Nested problems** (decode, reverse): stack of partial results at each depth level
6. **Generation**: backtracking with constraint `close < open ≤ n`

> **Next up:** Stack-Based Traversal →
