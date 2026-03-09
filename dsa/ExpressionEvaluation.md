# Phase 6: Stack — Expression Evaluation

## Overview

Stacks are the standard tool for **evaluating mathematical expressions**:
- **Infix** (`3 + 4 * 2`) — human-readable, needs precedence rules
- **Postfix / RPN** (`3 4 2 * +`) — no parentheses needed, easy to evaluate
- **Prefix / Polish** (`+ 3 * 4 2`) — operator before operands

| Notation | Evaluation | Parentheses |
|----------|-----------|-------------|
| Infix | Complex (precedence) | Required |
| Postfix (RPN) | Simple stack scan | Not needed |
| Prefix | Reverse scan with stack | Not needed |

---

## Example 1: Evaluate Postfix (RPN) Expression (LeetCode 150)

```go
package main

import (
    "fmt"
    "strconv"
)

func evalRPN(tokens []string) int {
    stack := []int{}

    for _, token := range tokens {
        switch token {
        case "+", "-", "*", "/":
            b := stack[len(stack)-1]
            a := stack[len(stack)-2]
            stack = stack[:len(stack)-2]

            var result int
            switch token {
            case "+":
                result = a + b
            case "-":
                result = a - b
            case "*":
                result = a * b
            case "/":
                result = a / b // truncates toward zero
            }
            stack = append(stack, result)
        default:
            num, _ := strconv.Atoi(token)
            stack = append(stack, num)
        }
    }

    return stack[0]
}

func main() {
    tests := [][]string{
        {"2", "1", "+", "3", "*"},         // (2+1)*3 = 9
        {"4", "13", "5", "/", "+"},        // 4+(13/5) = 6
        {"10", "6", "9", "3", "+", "-11", "*", "/", "*", "17", "+", "5", "+"},
    }

    for _, t := range tests {
        fmt.Printf("%v → %d\n", t, evalRPN(t))
    }
}
```

---

## Example 2: Evaluate Prefix Expression

```go
package main

import (
    "fmt"
    "strconv"
)

func evalPrefix(tokens []string) int {
    stack := []int{}

    // Scan right to left
    for i := len(tokens) - 1; i >= 0; i-- {
        token := tokens[i]

        if token == "+" || token == "-" || token == "*" || token == "/" {
            a := stack[len(stack)-1]
            b := stack[len(stack)-2]
            stack = stack[:len(stack)-2]

            var result int
            switch token {
            case "+":
                result = a + b
            case "-":
                result = a - b
            case "*":
                result = a * b
            case "/":
                result = a / b
            }
            stack = append(stack, result)
        } else {
            num, _ := strconv.Atoi(token)
            stack = append(stack, num)
        }
    }

    return stack[0]
}

func main() {
    // + 3 * 4 2 → 3 + (4*2) = 11
    fmt.Println(evalPrefix([]string{"+", "3", "*", "4", "2"}))

    // * + 2 3 - 7 4 → (2+3) * (7-4) = 15
    fmt.Println(evalPrefix([]string{"*", "+", "2", "3", "-", "7", "4"}))
}
```

---

## Example 3: Infix to Postfix (Shunting-Yard Algorithm)

```go
package main

import (
    "fmt"
    "strings"
    "unicode"
)

func precedence(op byte) int {
    switch op {
    case '+', '-':
        return 1
    case '*', '/':
        return 2
    case '^':
        return 3
    }
    return 0
}

func isRightAssoc(op byte) bool {
    return op == '^'
}

func infixToPostfix(expr string) string {
    var output []string
    var opStack []byte

    i := 0
    for i < len(expr) {
        ch := expr[i]

        if ch == ' ' {
            i++
            continue
        }

        if unicode.IsDigit(rune(ch)) {
            // Read full number
            j := i
            for j < len(expr) && unicode.IsDigit(rune(expr[j])) {
                j++
            }
            output = append(output, expr[i:j])
            i = j
            continue
        }

        if ch == '(' {
            opStack = append(opStack, ch)
        } else if ch == ')' {
            for len(opStack) > 0 && opStack[len(opStack)-1] != '(' {
                output = append(output, string(opStack[len(opStack)-1]))
                opStack = opStack[:len(opStack)-1]
            }
            opStack = opStack[:len(opStack)-1] // remove '('
        } else {
            // Operator
            for len(opStack) > 0 && opStack[len(opStack)-1] != '(' {
                topOp := opStack[len(opStack)-1]
                if precedence(topOp) > precedence(ch) ||
                    (precedence(topOp) == precedence(ch) && !isRightAssoc(ch)) {
                    output = append(output, string(topOp))
                    opStack = opStack[:len(opStack)-1]
                } else {
                    break
                }
            }
            opStack = append(opStack, ch)
        }
        i++
    }

    for len(opStack) > 0 {
        output = append(output, string(opStack[len(opStack)-1]))
        opStack = opStack[:len(opStack)-1]
    }

    return strings.Join(output, " ")
}

func main() {
    tests := []string{
        "3 + 4 * 2",
        "(3 + 4) * 2",
        "3 + 4 * 2 / (1 - 5)",
        "2 ^ 3 ^ 2",
    }

    for _, expr := range tests {
        fmt.Printf("Infix: %-25s → Postfix: %s\n", expr, infixToPostfix(expr))
    }
    // 3 + 4 * 2           → 3 4 2 * +
    // (3 + 4) * 2         → 3 4 + 2 *
    // 3 + 4 * 2 / (1 - 5) → 3 4 2 * 1 5 - / +
    // 2 ^ 3 ^ 2           → 2 3 2 ^ ^  (right-associative)
}
```

---

## Example 4: Infix to Prefix

```go
package main

import (
    "fmt"
    "strings"
    "unicode"
)

func precedence(op byte) int {
    switch op {
    case '+', '-':
        return 1
    case '*', '/':
        return 2
    }
    return 0
}

func reverse(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    // Swap parentheses
    for i, r := range runes {
        if r == '(' {
            runes[i] = ')'
        } else if r == ')' {
            runes[i] = '('
        }
    }
    return string(runes)
}

func infixToPostfix(expr string) []string {
    var output []string
    var opStack []byte

    i := 0
    for i < len(expr) {
        ch := expr[i]
        if ch == ' ' {
            i++
            continue
        }
        if unicode.IsDigit(rune(ch)) {
            j := i
            for j < len(expr) && unicode.IsDigit(rune(expr[j])) {
                j++
            }
            output = append(output, expr[i:j])
            i = j
            continue
        }
        if ch == '(' {
            opStack = append(opStack, ch)
        } else if ch == ')' {
            for len(opStack) > 0 && opStack[len(opStack)-1] != '(' {
                output = append(output, string(opStack[len(opStack)-1]))
                opStack = opStack[:len(opStack)-1]
            }
            opStack = opStack[:len(opStack)-1]
        } else {
            for len(opStack) > 0 && opStack[len(opStack)-1] != '(' &&
                precedence(opStack[len(opStack)-1]) >= precedence(ch) {
                output = append(output, string(opStack[len(opStack)-1]))
                opStack = opStack[:len(opStack)-1]
            }
            opStack = append(opStack, ch)
        }
        i++
    }
    for len(opStack) > 0 {
        output = append(output, string(opStack[len(opStack)-1]))
        opStack = opStack[:len(opStack)-1]
    }
    return output
}

func infixToPrefix(expr string) string {
    // Step 1: Reverse, swap parens
    rev := reverse(expr)

    // Step 2: Convert to postfix
    postfix := infixToPostfix(rev)

    // Step 3: Reverse result
    for i, j := 0, len(postfix)-1; i < j; i, j = i+1, j-1 {
        postfix[i], postfix[j] = postfix[j], postfix[i]
    }

    return strings.Join(postfix, " ")
}

func main() {
    expr := "(3 + 4) * 2"
    fmt.Printf("Infix: %s → Prefix: %s\n", expr, infixToPrefix(expr))
    // + 3 4 → wait, (3+4)*2 → * + 3 4 2
}
```

---

## Example 5: Direct Infix Evaluation (with Precedence)

```go
package main

import (
    "fmt"
    "unicode"
)

func calculate(s string) int {
    numStack := []int{}
    opStack := []byte{}

    precedence := func(op byte) int {
        if op == '+' || op == '-' {
            return 1
        }
        if op == '*' || op == '/' {
            return 2
        }
        return 0
    }

    applyOp := func() {
        b := numStack[len(numStack)-1]
        a := numStack[len(numStack)-2]
        numStack = numStack[:len(numStack)-2]
        op := opStack[len(opStack)-1]
        opStack = opStack[:len(opStack)-1]

        var result int
        switch op {
        case '+':
            result = a + b
        case '-':
            result = a - b
        case '*':
            result = a * b
        case '/':
            result = a / b
        }
        numStack = append(numStack, result)
    }

    i := 0
    for i < len(s) {
        ch := s[i]
        if ch == ' ' {
            i++
            continue
        }

        if unicode.IsDigit(rune(ch)) {
            num := 0
            for i < len(s) && unicode.IsDigit(rune(s[i])) {
                num = num*10 + int(s[i]-'0')
                i++
            }
            numStack = append(numStack, num)
            continue
        }

        if ch == '(' {
            opStack = append(opStack, ch)
        } else if ch == ')' {
            for opStack[len(opStack)-1] != '(' {
                applyOp()
            }
            opStack = opStack[:len(opStack)-1] // remove '('
        } else {
            for len(opStack) > 0 && opStack[len(opStack)-1] != '(' &&
                precedence(opStack[len(opStack)-1]) >= precedence(ch) {
                applyOp()
            }
            opStack = append(opStack, ch)
        }
        i++
    }

    for len(opStack) > 0 {
        applyOp()
    }

    return numStack[0]
}

func main() {
    tests := []string{
        "3 + 4 * 2",
        "(3 + 4) * 2",
        "10 + 2 * 6",
        "100 * 2 + 12",
        "100 * (2 + 12)",
        "100 * (2 + 12) / 14",
    }

    for _, expr := range tests {
        fmt.Printf("%-25s = %d\n", expr, calculate(expr))
    }
}
```

---

## Example 6: Basic Calculator (LeetCode 224)

```go
package main

import (
    "fmt"
    "unicode"
)

// Handles +, -, (, ) and unary minus
func basicCalculator(s string) int {
    stack := []int{}
    result := 0
    num := 0
    sign := 1

    for i := 0; i < len(s); i++ {
        ch := s[i]

        if unicode.IsDigit(rune(ch)) {
            num = num*10 + int(ch-'0')
        } else if ch == '+' {
            result += sign * num
            num = 0
            sign = 1
        } else if ch == '-' {
            result += sign * num
            num = 0
            sign = -1
        } else if ch == '(' {
            // Save current result and sign
            stack = append(stack, result)
            stack = append(stack, sign)
            result = 0
            sign = 1
        } else if ch == ')' {
            result += sign * num
            num = 0
            // Pop sign and previous result
            prevSign := stack[len(stack)-1]
            prevResult := stack[len(stack)-2]
            stack = stack[:len(stack)-2]
            result = prevResult + prevSign*result
        }
    }

    return result + sign*num
}

func main() {
    tests := []string{
        "1 + 1",
        " 2-1 + 2 ",
        "(1+(4+5+2)-3)+(6+8)",
        "-(3+4)+5",
    }

    for _, s := range tests {
        fmt.Printf("%-30s = %d\n", s, basicCalculator(s))
    }
    // 2, 3, 23, -2
}
```

---

## Example 7: Basic Calculator II (LeetCode 227) — With * /

```go
package main

import (
    "fmt"
    "unicode"
)

func calculateII(s string) int {
    stack := []int{}
    num := 0
    op := byte('+')

    for i := 0; i < len(s); i++ {
        ch := s[i]

        if unicode.IsDigit(rune(ch)) {
            num = num*10 + int(ch-'0')
        }

        if (!unicode.IsDigit(rune(ch)) && ch != ' ') || i == len(s)-1 {
            switch op {
            case '+':
                stack = append(stack, num)
            case '-':
                stack = append(stack, -num)
            case '*':
                prev := stack[len(stack)-1]
                stack[len(stack)-1] = prev * num
            case '/':
                prev := stack[len(stack)-1]
                stack[len(stack)-1] = prev / num
            }
            op = ch
            num = 0
        }
    }

    result := 0
    for _, v := range stack {
        result += v
    }
    return result
}

func main() {
    tests := []string{
        "3+2*2",
        " 3/2 ",
        " 3+5 / 2 ",
        "14-3/2",
    }

    for _, s := range tests {
        fmt.Printf("%-15s = %d\n", s, calculateII(s))
    }
    // 7, 1, 5, 13
}
```

---

## Example 8: Evaluate Expression Tree

```go
package main

import (
    "fmt"
    "strconv"
)

type TreeNode struct {
    Val   string
    Left  *TreeNode
    Right *TreeNode
}

func evaluate(root *TreeNode) int {
    if root == nil {
        return 0
    }

    // Leaf node = operand
    if root.Left == nil && root.Right == nil {
        num, _ := strconv.Atoi(root.Val)
        return num
    }

    left := evaluate(root.Left)
    right := evaluate(root.Right)

    switch root.Val {
    case "+":
        return left + right
    case "-":
        return left - right
    case "*":
        return left * right
    case "/":
        return left / right
    }

    return 0
}

func main() {
    // Expression: (3 + 4) * 2
    //        *
    //       / \
    //      +   2
    //     / \
    //    3   4
    tree := &TreeNode{
        Val: "*",
        Left: &TreeNode{
            Val:   "+",
            Left:  &TreeNode{Val: "3"},
            Right: &TreeNode{Val: "4"},
        },
        Right: &TreeNode{Val: "2"},
    }

    fmt.Println("(3 + 4) * 2 =", evaluate(tree)) // 14

    // Expression: 10 - (5 + 3)
    tree2 := &TreeNode{
        Val:  "-",
        Left: &TreeNode{Val: "10"},
        Right: &TreeNode{
            Val:   "+",
            Left:  &TreeNode{Val: "5"},
            Right: &TreeNode{Val: "3"},
        },
    }
    fmt.Println("10 - (5 + 3) =", evaluate(tree2)) // 2
}
```

---

## Example 9: Decode String (LeetCode 394)

```go
package main

import (
    "fmt"
    "strings"
    "unicode"
)

func decodeString(s string) string {
    countStack := []int{}
    strStack := []string{}
    current := ""
    k := 0

    for _, ch := range s {
        if unicode.IsDigit(ch) {
            k = k*10 + int(ch-'0')
        } else if ch == '[' {
            countStack = append(countStack, k)
            strStack = append(strStack, current)
            current = ""
            k = 0
        } else if ch == ']' {
            count := countStack[len(countStack)-1]
            countStack = countStack[:len(countStack)-1]
            prev := strStack[len(strStack)-1]
            strStack = strStack[:len(strStack)-1]
            current = prev + strings.Repeat(current, count)
        } else {
            current += string(ch)
        }
    }

    return current
}

func main() {
    tests := []string{
        "3[a]2[bc]",
        "3[a2[c]]",
        "2[abc]3[cd]ef",
        "abc3[cd]xyz",
    }

    for _, s := range tests {
        fmt.Printf("%-20s → %s\n", s, decodeString(s))
    }
    // "3[a]2[bc]"     → "aaabcbc"
    // "3[a2[c]]"      → "accaccacc"
    // "2[abc]3[cd]ef"  → "abcabccdcdcdef"
}
```

---

## Example 10: Postfix to Expression Tree

```go
package main

import "fmt"

type TreeNode struct {
    Val   string
    Left  *TreeNode
    Right *TreeNode
}

func isOperator(s string) bool {
    return s == "+" || s == "-" || s == "*" || s == "/"
}

func buildTree(postfix []string) *TreeNode {
    stack := []*TreeNode{}

    for _, token := range postfix {
        node := &TreeNode{Val: token}
        if isOperator(token) {
            node.Right = stack[len(stack)-1]
            node.Left = stack[len(stack)-2]
            stack = stack[:len(stack)-2]
        }
        stack = append(stack, node)
    }

    return stack[0]
}

func inorder(root *TreeNode) {
    if root == nil {
        return
    }
    if isOperator(root.Val) {
        fmt.Print("(")
    }
    inorder(root.Left)
    fmt.Print(root.Val)
    inorder(root.Right)
    if isOperator(root.Val) {
        fmt.Print(")")
    }
}

func main() {
    // Postfix: 3 4 + 2 * → (3 + 4) * 2
    postfix := []string{"3", "4", "+", "2", "*"}
    tree := buildTree(postfix)

    fmt.Print("Infix: ")
    inorder(tree)
    fmt.Println() // ((3+4)*2)

    // Postfix: 5 1 2 + 4 * + 3 -
    postfix2 := []string{"5", "1", "2", "+", "4", "*", "+", "3", "-"}
    tree2 := buildTree(postfix2)
    fmt.Print("Infix: ")
    inorder(tree2)
    fmt.Println() // ((5+((1+2)*4))-3)
}
```

---

## Expression Evaluation Cheat Sheet

| Task | Method | Time |
|------|--------|------|
| Evaluate Postfix | Single stack, left to right | O(n) |
| Evaluate Prefix | Single stack, right to left | O(n) |
| Infix → Postfix | Shunting-Yard (two stacks) | O(n) |
| Evaluate Infix | Two stacks (nums + ops) | O(n) |
| Build Expression Tree | Stack of tree nodes | O(n) |

## Key Takeaways

1. **Postfix evaluation** is simplest — scan left-to-right, push numbers, apply operators
2. **Shunting-Yard** converts infix → postfix respecting precedence and associativity
3. **Two-stack approach** evaluates infix directly: number stack + operator stack
4. **Basic Calculator I**: handles `+`, `-`, `(`, `)` — use sign tracking
5. **Basic Calculator II**: handles `+`, `-`, `*`, `/` — use delayed evaluation with previous operator
6. **Decode String**: classic nested structure → stack of (count, string) pairs

> **Next up:** Parentheses Matching →
