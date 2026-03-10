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

**Textual Figure: Valid Parentheses Matching**

```
Input: "{[]}"   → true

  Char   Stack          Action
  ────   ─────          ──────
  '{'    │ { │          push '{'
  '['    │ [ │ { │      push '['
  ']'    │ { │          pop '[' ↔ ']' match ✓
  '}'    (empty)         pop '{' ↔ '}' match ✓
  END    stack empty     → valid!

Input: "([)]"   → false

  Char   Stack          Action
  '('    │ ( │          push '('
  '['    │ [ │ ( │      push '['
  ')'    │ [ │ ( │      top='[' ≠ match[')']='('  → INVALID!

  Match map:
  ┌─────────────────────┐
  │  ')' → '('           │
  │  ']' → '['           │
  │  '}' → '{'           │
  └─────────────────────┘
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

**Textual Figure: Minimum Additions to Make Valid**

```
Input: "()))(("    → 4 additions needed

  Char   open   close   Action
  ────   ────   ─────   ──────
  '('    1      0       open++ (unmatched '(')
  ')'    0      0       open-- (matched!)
  ')'    0      1       open=0, close++ (unmatched ')')
  ')'    0      2       close++
  '('    1      2       open++
  '('    2      2       open++

  Result: open + close = 2 + 2 = 4

  Need to add:  2 × ')' to close the '('s
                2 × '(' to open for the ')'s

  ┌─────────────────────────────────┐
  │ No stack needed!                │
  │ Just two counters: open, close  │
  │ O(1) space, O(n) time           │
  └─────────────────────────────────┘
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

**Textual Figure: Longest Valid Parentheses**

```
Input: ")()())"   → longest = 4

Stack stores indices (initialized with -1 as base):

  i  char  Stack          Action               maxLen
  ─  ────  ─────          ──────               ──────
  -  init  [-1]           base index
  0  ')'   [0]            pop -1, stack empty    0
                          push 0 as new base
  1  '('   [0, 1]         push index             0
  2  ')'   [0]            pop 1, len=2-0=2       2
  3  '('   [0, 3]         push index             2
  4  ')'   [0]            pop 3, len=4-0=4       4
  5  ')'   [5]            pop 0, stack empty     4
                          push 5 as new base

  Longest valid substring: positions 1-4 = "()()"

  ┌───┬───┬───┬───┬───┬───┐
  │ ) │ ( │ ) │ ( │ ) │ ) │
  └───┴───┴───┴───┴───┴───┘
   [0]  [1]  [2]  [3]  [4]  [5]
    x   █████████████████   x    ← valid span = 4
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

**Textual Figure: Remove Invalid Parentheses**

```
Input: "lee(t(c)o)de)"

Pass 1: Mark unmatched ')' with '#'
  l e e ( t ( c ) o ) d e )
                              ↑ unmatched ')' → '#'

Pass 2: Mark unmatched '(' with '#'
  Stack of '(' indices: push index 3 and 5
  Index 5 '(' matched by index 7 ')'
  Index 3 '(' matched by index 9 ')'
  Stack empty → no unmatched '('

  Before: l  e  e  (  t  (  c  )  o  )  d  e  )
  Marks:                                         #
  After:  l  e  e  (  t  (  c  )  o  )  d  e

  Result: "lee(t(c)o)de"

  Algorithm:
  ┌───────────────────────────────────────┐
  │ 1. Forward pass: mark unmatched ')'   │
  │ 2. Stack tracks unmatched '(' indices  │
  │ 3. Remove all '#' marked characters    │
  └───────────────────────────────────────┘
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

**Textual Figure: Generate Parentheses (Backtracking Tree)**

```
n=3:  Generate all valid combinations of 3 pairs.

  Backtracking decision tree (open < n, close < open):

                          ""
                          │
                         "("
                       ┌──┴──┐
                    "(("      "()"
                  ┌──┴──┐      │
              "((("    "(()"   "()("
                │     ┌─┴─┐    ┌─┴─┐
            "((()""(()(" "(())" "()((" "()()"
               │    │     │     │     │
           "((())""(()()""(())(""()(()" "()()("
              │    │     │     │      │
              ✓    ✓   "(())()" ✓    "()()()"
                         │              │
                         ✓              ✓

  Result: ["((()))", "(()())", "(())()", "()(())", "()()()"]

  Rules: • Add '(' if open < n
         • Add ')' if close < open
         • Complete when length = 2n
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
    // "(()"     → 2
    // "()("     → 2
    // "(()(()))" → 6
}
```

**Textual Figure: Score of Parentheses**

```
Input: "(()(()))"   → score = 6

Stack tracks running scores at each depth level:

  Char  Stack             Action
  ────  ─────             ──────
  init  [0]               base score
  '('   [0, 0]            push 0 (enter depth 1)
  '('   [0, 0, 0]         push 0 (enter depth 2)
  ')'   [0, 1]            inner=0 → score=1; add to prev
  '('   [0, 1, 0]         push 0 (enter depth 2)
  '('   [0, 1, 0, 0]      push 0 (enter depth 3)
  ')'   [0, 1, 1]         inner=0 → score=1; add to prev
  ')'   [0, 3]            inner=1 → score=2*1=2; 1+2=3
  ')'   [6]               inner=3 → score=2*3=6; 0+6=6

  Result: stack[0] = 6

  Scoring rules:
  ┌───────────────────────┐
  │  ()     = 1             │
  │  (A)   = 2 * A          │
  │  A B   = A + B           │
  └───────────────────────┘
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

**Textual Figure: Remove Outermost Parentheses**

```
Input: "(()())(())"  →  "()()()"

  Char  depth   Include?   Output
  ────  ─────  ────────  ──────
  '('   0→1     NO  (outer)  ""
  '('   1→2     YES          "("
  ')'   2→1     YES          "()"
  '('   1→2     YES          "()("
  ')'   2→1     YES          "()()"
  ')'   1→0     NO  (outer)  "()()"
  '('   0→1     NO  (outer)  "()()"
  '('   1→2     YES          "()()("
  ')'   2→1     YES          "()()()"
  ')'   1→0     NO  (outer)  "()()()"

  Primitive groups:
  ┌──────────┐  ┌──────┐
  │ (()())   │  │ (()) │
  └──────────┘  └──────┘
   ↑        ↑    ↑    ↑
  outer   outer outer outer  ← removed
   Result: ()()  +  () = "()()()"
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

**Textual Figure: Check if Parentheses Can Be Valid**

```
s = "))()))", locked = "010100"

  Index:   0    1    2    3    4    5
  s:       )    )    (    )    )    )
  locked:  0    1    0    1    0    0
           ↑         ↑         ↑    ↑
         flex       flex     flex  flex  (changeable)

Left-to-right scan (can we keep ')' ≤ possible '('):
  i=0: locked=0 or '(' → open++     open=1
  i=1: locked=1 and ')' → open--    open=0
  i=2: locked=0         → open++    open=1
  i=3: locked=1 and ')' → open--    open=0
  i=4: locked=0         → open++    open=1
  i=5: locked=0         → open++    open=2
  open ≥ 0 throughout ✓

Right-to-left scan (can we keep '(' ≤ possible ')'):
  Similar check from right → close ≥ 0 throughout ✓

  Result: true (possible valid assignment exists)

  Two-pass greedy: O(n) time, O(1) space
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

**Textual Figure: Maximum Nesting Depth**

```
Input: "(1+(2*3)+((8)/4))+1"

  Char     depth   maxDepth
  ────     ─────   ────────
  '('      1       1
   1
   +
  '('      2       2
   2 * 3
  ')'      1
   +
  '('      2       2
  '('      3       3  ← MAX
   8
  ')'      2
   / 4
  ')'      1
  ')'      0
   + 1

  Nesting visualization:
  Depth 0: ......................+1
  Depth 1: (──────────────────)
  Depth 2:   (─────)  (──────)
  Depth 3:            ((──)   )

  Maximum nesting depth = 3
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

**Textual Figure: Reverse Substrings Between Parentheses**

```
Input: "(u(love)i)"

Stack of partial strings at each depth:

  Char    stack              current
  ────    ─────              ───────
  '('     [""]               ""         save & reset
  'u'     [""]               "u"
  '('     ["", "u"]          ""         save & reset
  'l'     ["", "u"]          "l"
  'o'     ["", "u"]          "lo"
  'v'     ["", "u"]          "lov"
  'e'     ["", "u"]          "love"
  ')'     [""]               "u"+rev("love")  reverse & pop
                             = "u" + "evol"
                             = "uevol"
  'i'     [""]               "uevoli"
  ')'     []                 ""+rev("uevoli")  reverse & pop
                             = "iloveu"

  Stack at deepest point:
  ┌─────┐
  │ "u" │←top   current = "love"
  ├─────┤
  │ ""  │
  └─────┘

  Result: "iloveu"
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

**Textual Figure: Validate Wildcard Parentheses**

```
Input: "(*))"

'*' can be '(' or ')' or empty.
Track range [minOpen, maxOpen] of possible open-paren counts:

  Char  minOpen  maxOpen   Reasoning
  ────  ───────  ───────   ─────────
  '('   1        1         must be '('
  '*'   0        2         treat as ')'→min--
                           treat as '('→max++
  ')'   0        1         min-- → -1→clamp to 0
                           max--
  ')'   0        0         min-- → -1→clamp to 0
                           max-- = 0 (≥0, ok)

  Final: minOpen = 0  → valid! ✓

  Visualization of possible open counts:

  After '(':   [1, 1]     min=max=1
  After '*':   [0, 2]     ─────────
  After ')':   [0, 1]       ──────
  After ')':   [0, 0]         ─     ← range includes 0 → valid

  If maxOpen < 0 at any point → impossible → false
  If minOpen = 0 at end → valid
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
