# Phase 15: Dynamic Programming — DP on Strings

## Overview

String DP problems typically involve two strings (or one string against itself) with transitions based on character matching. Common patterns: LCS, edit distance, palindromic subsequences, interleaving.

| Pattern | State | Transition |
|---------|-------|------------|
| **LCS** | dp[i][j] = LCS of s[0..i-1], t[0..j-1] | match → dp[i-1][j-1]+1, else max(dp[i-1][j], dp[i][j-1]) |
| **Edit Distance** | dp[i][j] = ops to convert s[0..i-1] → t[0..j-1] | match → dp[i-1][j-1], else 1+min(ins, del, rep) |
| **Palindrome** | dp[i][j] on same string, i=start, j=end | s[i]==s[j] → dp[i+1][j-1]+2, else max(dp[i+1][j], dp[i][j-1]) |

---

## Example 1: Longest Common Subsequence (LeetCode 1143)

```go
package main

import "fmt"

func longestCommonSubsequence(text1, text2 string) int {
	m, n := len(text1), len(text2)
	dp := make([][]int, m+1)
	for i := range dp { dp[i] = make([]int, n+1) }

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if text1[i-1] == text2[j-1] {
				dp[i][j] = dp[i-1][j-1] + 1
			} else {
				dp[i][j] = max(dp[i-1][j], dp[i][j-1])
			}
		}
	}
	return dp[m][n]
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	fmt.Println(longestCommonSubsequence("abcde", "ace"))   // 3
	fmt.Println(longestCommonSubsequence("abc", "def"))     // 0
	fmt.Println(longestCommonSubsequence("abcba", "abcbcba")) // 5
}
```

---

## Example 2: Print LCS (Reconstruct)

```go
package main

import "fmt"

func printLCS(s, t string) string {
	m, n := len(s), len(t)
	dp := make([][]int, m+1)
	for i := range dp { dp[i] = make([]int, n+1) }

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if s[i-1] == t[j-1] {
				dp[i][j] = dp[i-1][j-1] + 1
			} else {
				dp[i][j] = max(dp[i-1][j], dp[i][j-1])
			}
		}
	}

	// Backtrack
	result := []byte{}
	i, j := m, n
	for i > 0 && j > 0 {
		if s[i-1] == t[j-1] {
			result = append([]byte{s[i-1]}, result...)
			i--; j--
		} else if dp[i-1][j] > dp[i][j-1] {
			i--
		} else {
			j--
		}
	}
	return string(result)
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	fmt.Println(printLCS("AGGTAB", "GXTXAYB")) // GTAB
	fmt.Println(printLCS("abcde", "ace"))       // ace
}
```

---

## Example 3: Edit Distance (LeetCode 72)

```go
package main

import "fmt"

func minDistance(word1, word2 string) int {
	m, n := len(word1), len(word2)
	dp := make([][]int, m+1)
	for i := range dp {
		dp[i] = make([]int, n+1)
		dp[i][0] = i
	}
	for j := 0; j <= n; j++ { dp[0][j] = j }

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if word1[i-1] == word2[j-1] {
				dp[i][j] = dp[i-1][j-1]
			} else {
				dp[i][j] = 1 + min3(
					dp[i-1][j],   // delete
					dp[i][j-1],   // insert
					dp[i-1][j-1], // replace
				)
			}
		}
	}
	return dp[m][n]
}

func min3(a, b, c int) int {
	if a < b { if a < c { return a }; return c }
	if b < c { return b }; return c
}

func main() {
	fmt.Println(minDistance("horse", "ros"))      // 3
	fmt.Println(minDistance("intention", "execution")) // 5
	fmt.Println(minDistance("", "abc"))           // 3
}
```

---

## Example 4: Longest Palindromic Subsequence (LeetCode 516)

```go
package main

import "fmt"

func longestPalindromeSubseq(s string) int {
	n := len(s)
	dp := make([][]int, n)
	for i := range dp {
		dp[i] = make([]int, n)
		dp[i][i] = 1
	}

	for length := 2; length <= n; length++ {
		for i := 0; i <= n-length; i++ {
			j := i + length - 1
			if s[i] == s[j] {
				dp[i][j] = dp[i+1][j-1] + 2
			} else {
				dp[i][j] = max(dp[i+1][j], dp[i][j-1])
			}
		}
	}
	return dp[0][n-1]
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	fmt.Println(longestPalindromeSubseq("bbbab"))   // 4 (bbbb)
	fmt.Println(longestPalindromeSubseq("cbbd"))    // 2 (bb)
	fmt.Println(longestPalindromeSubseq("racecar")) // 7
}
```

---

## Example 5: Palindrome Partitioning — Min Cuts (LeetCode 132)

```go
package main

import "fmt"

func minCut(s string) int {
	n := len(s)

	// isPalin[i][j] = true if s[i..j] is palindrome
	isPalin := make([][]bool, n)
	for i := range isPalin { isPalin[i] = make([]bool, n) }
	for i := n - 1; i >= 0; i-- {
		for j := i; j < n; j++ {
			if s[i] == s[j] && (j-i <= 2 || isPalin[i+1][j-1]) {
				isPalin[i][j] = true
			}
		}
	}

	// dp[i] = min cuts for s[0..i]
	dp := make([]int, n)
	for i := range dp { dp[i] = i } // worst case: i cuts

	for i := 1; i < n; i++ {
		if isPalin[0][i] {
			dp[i] = 0
			continue
		}
		for j := 1; j <= i; j++ {
			if isPalin[j][i] && dp[j-1]+1 < dp[i] {
				dp[i] = dp[j-1] + 1
			}
		}
	}
	return dp[n-1]
}

func main() {
	fmt.Println(minCut("aab"))    // 1
	fmt.Println(minCut("a"))     // 0
	fmt.Println(minCut("aaabba")) // 1
}
```

---

## Example 6: Distinct Subsequences (LeetCode 115)

```go
package main

import "fmt"

// How many distinct subsequences of s equal t?
func numDistinct(s, t string) int {
	m, n := len(s), len(t)
	dp := make([][]int, m+1)
	for i := range dp {
		dp[i] = make([]int, n+1)
		dp[i][0] = 1 // empty t is always a subsequence
	}

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			dp[i][j] = dp[i-1][j] // skip s[i-1]
			if s[i-1] == t[j-1] {
				dp[i][j] += dp[i-1][j-1]
			}
		}
	}
	return dp[m][n]
}

func main() {
	fmt.Println(numDistinct("rabbbit", "rabbit")) // 3
	fmt.Println(numDistinct("babgbag", "bag"))    // 5
}
```

---

## Example 7: Shortest Common Supersequence (LeetCode 1092)

```go
package main

import "fmt"

func shortestCommonSupersequence(str1, str2 string) string {
	m, n := len(str1), len(str2)

	// LCS table
	dp := make([][]int, m+1)
	for i := range dp { dp[i] = make([]int, n+1) }
	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if str1[i-1] == str2[j-1] {
				dp[i][j] = dp[i-1][j-1] + 1
			} else {
				dp[i][j] = max(dp[i-1][j], dp[i][j-1])
			}
		}
	}

	// Build SCS from LCS table
	result := []byte{}
	i, j := m, n
	for i > 0 && j > 0 {
		if str1[i-1] == str2[j-1] {
			result = append([]byte{str1[i-1]}, result...)
			i--; j--
		} else if dp[i-1][j] > dp[i][j-1] {
			result = append([]byte{str1[i-1]}, result...)
			i--
		} else {
			result = append([]byte{str2[j-1]}, result...)
			j--
		}
	}
	for i > 0 { result = append([]byte{str1[i-1]}, result...); i-- }
	for j > 0 { result = append([]byte{str2[j-1]}, result...); j-- }

	return string(result)
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	fmt.Println(shortestCommonSupersequence("abac", "cab"))    // cabac
	fmt.Println(shortestCommonSupersequence("dynamic", "programming")) 
}
```

---

## Example 8: Interleaving String (LeetCode 97)

```go
package main

import "fmt"

// Is s3 formed by interleaving s1 and s2?
func isInterleave(s1, s2, s3 string) bool {
	m, n := len(s1), len(s2)
	if m+n != len(s3) { return false }

	dp := make([][]bool, m+1)
	for i := range dp { dp[i] = make([]bool, n+1) }
	dp[0][0] = true

	for i := 1; i <= m; i++ { dp[i][0] = dp[i-1][0] && s1[i-1] == s3[i-1] }
	for j := 1; j <= n; j++ { dp[0][j] = dp[0][j-1] && s2[j-1] == s3[j-1] }

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			dp[i][j] = (dp[i-1][j] && s1[i-1] == s3[i+j-1]) ||
				(dp[i][j-1] && s2[j-1] == s3[i+j-1])
		}
	}
	return dp[m][n]
}

func main() {
	fmt.Println(isInterleave("aabcc", "dbbca", "aadbbcbcac")) // true
	fmt.Println(isInterleave("aabcc", "dbbca", "aadbbbaccc")) // false
}
```

---

## Example 9: Longest Common Substring

```go
package main

import "fmt"

func longestCommonSubstring(s, t string) (int, string) {
	m, n := len(s), len(t)
	dp := make([][]int, m+1)
	for i := range dp { dp[i] = make([]int, n+1) }

	maxLen, endIdx := 0, 0
	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if s[i-1] == t[j-1] {
				dp[i][j] = dp[i-1][j-1] + 1
				if dp[i][j] > maxLen {
					maxLen = dp[i][j]
					endIdx = i
				}
			}
			// else dp[i][j] = 0 (default)
		}
	}
	return maxLen, s[endIdx-maxLen : endIdx]
}

func main() {
	length, sub := longestCommonSubstring("abcdef", "zbcdf")
	fmt.Printf("Length: %d, Substring: %q\n", length, sub) // 3, "bcd"

	length, sub = longestCommonSubstring("GeeksforGeeks", "GeeksQuiz")
	fmt.Printf("Length: %d, Substring: %q\n", length, sub) // 5, "Geeks"
}
```

---

## Example 10: Wildcard Matching (LeetCode 44)

```go
package main

import "fmt"

// '?' matches any single character, '*' matches any sequence
func isMatch(s, p string) bool {
	m, n := len(s), len(p)
	dp := make([][]bool, m+1)
	for i := range dp { dp[i] = make([]bool, n+1) }
	dp[0][0] = true

	// Pattern starting with * can match empty string
	for j := 1; j <= n; j++ {
		if p[j-1] == '*' { dp[0][j] = dp[0][j-1] }
	}

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if p[j-1] == '*' {
				dp[i][j] = dp[i][j-1] || dp[i-1][j]
			} else if p[j-1] == '?' || s[i-1] == p[j-1] {
				dp[i][j] = dp[i-1][j-1]
			}
		}
	}
	return dp[m][n]
}

func main() {
	fmt.Println(isMatch("adceb", "*a*b"))  // true
	fmt.Println(isMatch("acdcb", "a*c?b")) // false
	fmt.Println(isMatch("", "*"))           // true
}
```

---

## Example 11: Regular Expression Matching (LeetCode 10)

```go
package main

import "fmt"

// '.' matches any single char, '*' means zero or more of preceding
func isMatch(s, p string) bool {
	m, n := len(s), len(p)
	dp := make([][]bool, m+1)
	for i := range dp { dp[i] = make([]bool, n+1) }
	dp[0][0] = true

	// Patterns like a*, a*b*, a*b*c* can match empty string
	for j := 2; j <= n; j++ {
		if p[j-1] == '*' { dp[0][j] = dp[0][j-2] }
	}

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if p[j-1] == '*' {
				dp[i][j] = dp[i][j-2] // zero occurrences
				if p[j-2] == '.' || p[j-2] == s[i-1] {
					dp[i][j] = dp[i][j] || dp[i-1][j] // one+ occurrences
				}
			} else if p[j-1] == '.' || p[j-1] == s[i-1] {
				dp[i][j] = dp[i-1][j-1]
			}
		}
	}
	return dp[m][n]
}

func main() {
	fmt.Println(isMatch("aa", "a"))     // false
	fmt.Println(isMatch("aa", "a*"))    // true
	fmt.Println(isMatch("ab", ".*"))    // true
	fmt.Println(isMatch("aab", "c*a*b")) // true
}
```

---

## Key Takeaways

1. Most string DP uses 2D table: dp[i][j] for prefixes s[0..i-1] and t[0..j-1]
2. LCS pattern: match → diagonal + 1, else max(top, left)
3. Edit distance: match → diagonal, else 1 + min(three neighbors)
4. Palindrome DP works on substrings of same string: dp[i][j], length expansion
5. Reconstruction: backtrack through the DP table following the optimal choices
6. Space can often be optimized to O(min(m,n)) using rolling array

> **Next up:** DP on Trees →
