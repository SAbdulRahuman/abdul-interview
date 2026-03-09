# Phase 15: Dynamic Programming — State Machine DP

## Overview

**State Machine DP** models problems where you transition between discrete states. Each state has defined transitions with costs. The classic example: stock buy/sell with various constraints.

| Concept | Description |
|---------|-------------|
| **States** | Finite set of conditions (holding, not holding, cooldown) |
| **Transitions** | Rules for moving between states |
| **DP table** | dp[i][state] = optimal value at step i in given state |

---

## Example 1: Best Time to Buy and Sell Stock (One Transaction)

```go
package main

import "fmt"

func maxProfit(prices []int) int {
	// State machine: NOT_HOLDING → HOLDING → SOLD
	minPrice := prices[0]
	maxProf := 0

	for _, p := range prices {
		if p < minPrice { minPrice = p }
		if p-minPrice > maxProf { maxProf = p - minPrice }
	}
	return maxProf
}

func main() {
	fmt.Println(maxProfit([]int{7, 1, 5, 3, 6, 4})) // 5
	fmt.Println(maxProfit([]int{7, 6, 4, 3, 1}))     // 0
}
```

---

## Example 2: Buy and Sell Stock II (Unlimited Transactions)

```go
package main

import "fmt"

func maxProfit(prices []int) int {
	// Two states: holding, notHolding
	hold := -prices[0] // bought on day 0
	notHold := 0

	for i := 1; i < len(prices); i++ {
		newHold := hold
		newNotHold := notHold

		// Transition: buy (notHold → hold)
		if notHold-prices[i] > newHold { newHold = notHold - prices[i] }
		// Transition: sell (hold → notHold)
		if hold+prices[i] > newNotHold { newNotHold = hold + prices[i] }

		hold, notHold = newHold, newNotHold
	}
	return notHold
}

func main() {
	fmt.Println(maxProfit([]int{7, 1, 5, 3, 6, 4})) // 7
	fmt.Println(maxProfit([]int{1, 2, 3, 4, 5}))     // 4
}
```

---

## Example 3: Buy and Sell Stock with Cooldown (LeetCode 309)

```go
package main

import "fmt"

func maxProfit(prices []int) int {
	// Three states: hold, sold (cooldown next), rest
	hold := -prices[0]
	sold := 0
	rest := 0

	for i := 1; i < len(prices); i++ {
		newHold := hold
		newSold := hold + prices[i]
		newRest := max(rest, sold)

		// buy: rest → hold
		if rest-prices[i] > newHold { newHold = rest - prices[i] }

		hold, sold, rest = newHold, newSold, newRest
	}
	return max(sold, rest)
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	fmt.Println(maxProfit([]int{1, 2, 3, 0, 2})) // 3
	fmt.Println(maxProfit([]int{1}))               // 0
}
```

---

## Example 4: Buy and Sell Stock with Transaction Fee (LeetCode 714)

```go
package main

import "fmt"

func maxProfit(prices []int, fee int) int {
	hold := -prices[0]
	cash := 0

	for i := 1; i < len(prices); i++ {
		// sell: hold → cash (pay fee)
		newCash := cash
		if hold+prices[i]-fee > newCash { newCash = hold + prices[i] - fee }
		// buy: cash → hold
		newHold := hold
		if cash-prices[i] > newHold { newHold = cash - prices[i] }

		hold, cash = newHold, newCash
	}
	return cash
}

func main() {
	fmt.Println(maxProfit([]int{1, 3, 2, 8, 4, 9}, 2)) // 8
	fmt.Println(maxProfit([]int{1, 3, 7, 5, 10, 3}, 3)) // 6
}
```

---

## Example 5: Buy and Sell Stock III (At Most 2 Transactions)

```go
package main

import "fmt"

func maxProfit(prices []int) int {
	// 4 states per day: buy1, sell1, buy2, sell2
	buy1 := -prices[0]
	sell1 := 0
	buy2 := -prices[0]
	sell2 := 0

	for i := 1; i < len(prices); i++ {
		// First buy
		if -prices[i] > buy1 { buy1 = -prices[i] }
		// First sell
		if buy1+prices[i] > sell1 { sell1 = buy1 + prices[i] }
		// Second buy (using profit from first sell)
		if sell1-prices[i] > buy2 { buy2 = sell1 - prices[i] }
		// Second sell
		if buy2+prices[i] > sell2 { sell2 = buy2 + prices[i] }
	}
	return sell2
}

func main() {
	fmt.Println(maxProfit([]int{3, 3, 5, 0, 0, 3, 1, 4})) // 6
	fmt.Println(maxProfit([]int{1, 2, 3, 4, 5}))           // 4
}
```

---

## Example 6: Buy and Sell Stock IV (At Most K Transactions - LeetCode 188)

```go
package main

import "fmt"

func maxProfit(k int, prices []int) int {
	n := len(prices)
	if n == 0 { return 0 }

	// If k >= n/2, unlimited transactions
	if k >= n/2 {
		profit := 0
		for i := 1; i < n; i++ {
			if prices[i] > prices[i-1] { profit += prices[i] - prices[i-1] }
		}
		return profit
	}

	// dp[j][0] = max profit after j-th buy, dp[j][1] = after j-th sell
	buy := make([]int, k+1)
	sell := make([]int, k+1)
	for j := range buy { buy[j] = -1 << 31 }

	for _, p := range prices {
		for j := 1; j <= k; j++ {
			if sell[j-1]-p > buy[j] { buy[j] = sell[j-1] - p }
			if buy[j]+p > sell[j] { sell[j] = buy[j] + p }
		}
	}
	return sell[k]
}

func main() {
	fmt.Println(maxProfit(2, []int{2, 4, 1}))       // 2
	fmt.Println(maxProfit(2, []int{3, 2, 6, 5, 0, 3})) // 7
}
```

---

## Example 7: Paint House (LeetCode 256)

```go
package main

import "fmt"

// Paint n houses with 3 colors, no two adjacent same color
func minCost(costs [][]int) int {
	n := len(costs)
	// States: color 0, 1, 2
	dp := [3]int{costs[0][0], costs[0][1], costs[0][2]}

	for i := 1; i < n; i++ {
		newDP := [3]int{
			costs[i][0] + min(dp[1], dp[2]),
			costs[i][1] + min(dp[0], dp[2]),
			costs[i][2] + min(dp[0], dp[1]),
		}
		dp = newDP
	}
	return min3(dp[0], dp[1], dp[2])
}

func min(a, b int) int { if a < b { return a }; return b }
func min3(a, b, c int) int { return min(a, min(b, c)) }

func main() {
	costs := [][]int{{17, 2, 17}, {16, 16, 5}, {14, 3, 19}}
	fmt.Println("Min paint cost:", minCost(costs)) // 10
}
```

---

## Example 8: Paint House II (K Colors, LeetCode 265)

```go
package main

import (
	"fmt"
	"math"
)

// n houses, k colors, no adjacent same — O(nk) solution
func minCostII(costs [][]int) int {
	n := len(costs)
	k := len(costs[0])

	dp := make([]int, k)
	copy(dp, costs[0])

	for i := 1; i < n; i++ {
		// Find min and second-min of dp
		min1, min2 := math.MaxInt32, math.MaxInt32
		minIdx := -1
		for j := 0; j < k; j++ {
			if dp[j] < min1 {
				min2 = min1
				min1 = dp[j]
				minIdx = j
			} else if dp[j] < min2 {
				min2 = dp[j]
			}
		}

		newDP := make([]int, k)
		for j := 0; j < k; j++ {
			if j == minIdx {
				newDP[j] = costs[i][j] + min2
			} else {
				newDP[j] = costs[i][j] + min1
			}
		}
		dp = newDP
	}

	ans := dp[0]
	for _, v := range dp { if v < ans { ans = v } }
	return ans
}

func main() {
	costs := [][]int{{1, 5, 3}, {2, 9, 4}}
	fmt.Println("Min cost (k colors):", minCostII(costs)) // 5
}
```

---

## Example 9: Decode Ways Variant (State Machine)

```go
package main

import "fmt"

// Count decodings of digit string — state machine approach
func numDecodings(s string) int {
	if len(s) == 0 || s[0] == '0' { return 0 }

	// States: prev1 = dp[i-1], prev2 = dp[i-2]
	prev2, prev1 := 1, 1

	for i := 1; i < len(s); i++ {
		curr := 0

		// Single digit decode
		if s[i] != '0' {
			curr += prev1
		}

		// Two digit decode
		twoDigit := (s[i-1]-'0')*10 + (s[i] - '0')
		if twoDigit >= 10 && twoDigit <= 26 {
			curr += prev2
		}

		prev2, prev1 = prev1, curr
	}
	return prev1
}

func main() {
	fmt.Println(numDecodings("12"))   // 2
	fmt.Println(numDecodings("226"))  // 3
	fmt.Println(numDecodings("06"))   // 0
	fmt.Println(numDecodings("11106")) // 2
}
```

---

## Example 10: State Machine DP Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== State Machine DP Patterns ===")
	fmt.Println()

	problems := []struct{ name, states, transitions string }{
		{"Stock I (1 txn)", "minPrice, maxProfit",
			"Track min so far, max profit so far"},
		{"Stock II (unlimited)", "hold, cash",
			"buy: cash→hold, sell: hold→cash"},
		{"Stock cooldown", "hold, sold, rest",
			"sell: hold→sold, cooldown: sold→rest, buy: rest→hold"},
		{"Stock with fee", "hold, cash",
			"sell: hold→cash (pay fee), buy: cash→hold"},
		{"Stock III (2 txn)", "buy1, sell1, buy2, sell2",
			"Sequential: buy1→sell1→buy2→sell2"},
		{"Stock IV (k txn)", "buy[j], sell[j] for j=1..k",
			"Generalized k-transaction version"},
		{"Paint House", "dp[color] for each color",
			"Transition: pick min from other colors"},
		{"Paint House II", "dp[color], track min1/min2",
			"O(nk) with first/second minimum trick"},
	}

	for i, p := range problems {
		fmt.Printf("%d. %s\n", i+1, p.name)
		fmt.Printf("   States: %s\n", p.states)
		fmt.Printf("   Transitions: %s\n\n", p.transitions)
	}

	fmt.Println("General approach:")
	fmt.Println("1. Identify all possible states")
	fmt.Println("2. Define valid transitions between states")
	fmt.Println("3. Write recurrence for each state")
	fmt.Println("4. Often O(1) space using variables for each state")
}
```

---

## Key Takeaways

1. State Machine DP: define states + transitions, iterate through input updating all states
2. Stock problems are the classic example — states = {hold, not_hold, cooldown}
3. Often optimizable to O(1) space since only previous step's states matter
4. Paint house extends to k colors with O(nk) using min/second-min trick
5. Draw the state diagram first — transitions become the recurrence relation

> **Next up:** Digit DP →
