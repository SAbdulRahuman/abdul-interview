# Phase 3: Strings — Character Encoding

## What is Character Encoding?

Character encoding maps characters to numbers. Go uses **UTF-8** natively — strings are byte sequences where ASCII chars use 1 byte and other Unicode chars use 2-4 bytes.

---

## Example 1: ASCII Basics

```go
package main

import "fmt"

func main() {
    // ASCII: 0-127, each char = 1 byte
    for _, ch := range "AZaz09!@" {
        fmt.Printf("'%c' = %d (0x%02X)\n", ch, ch, ch)
    }
    // 'A' = 65, 'Z' = 90, 'a' = 97, 'z' = 122, '0' = 48, '9' = 57

    // Character arithmetic
    fmt.Println('Z' - 'A')       // 25 (number of letters - 1)
    fmt.Println('a' - 'A')       // 32 (case difference)
    fmt.Println('5' - '0')       // 5 (digit value)
    fmt.Println('A' + 3)         // 68 = 'D'
}
```

**Textual Figure — ASCII Table (Key Ranges):**

```
  ASCII Character Map (0-127):

  ┌──────────┬─────────┬────────────────────┐
  │ Range    │ Decimal │ Characters           │
  ├──────────┼─────────┼────────────────────┤
  │ '0'-'9'  │  48-57  │ Digits               │
  │ 'A'-'Z'  │  65-90  │ Uppercase letters    │
  │ 'a'-'z'  │  97-122 │ Lowercase letters    │
  └──────────┴─────────┴────────────────────┘

  Character Arithmetic:
  'Z' - 'A' = 90 - 65 = 25     (letter distance)
  'a' - 'A' = 97 - 65 = 32     (case offset)
  '5' - '0' = 53 - 48 = 5      (digit value)
  'A' + 3   = 65 + 3  = 68 'D' (shift in alphabet)

  Common interview tricks:
  • char - 'a'  →  index 0-25 (for [26]int arrays)
  • char - '0'  →  digit value 0-9
  • char ^ 32   →  toggle case (ASCII trick)
```

---

## Example 2: UTF-8 Multi-Byte Characters

```go
package main

import "fmt"

func main() {
    s := "Hello, 世界! 🌍"

    fmt.Println("Byte length:", len(s))            // 20
    fmt.Println("Rune count:", len([]rune(s)))      // 12

    // Iterate by bytes
    fmt.Print("Bytes: ")
    for i := 0; i < len(s); i++ {
        fmt.Printf("%02X ", s[i])
    }
    fmt.Println()

    // Iterate by runes (correct way)
    fmt.Println("Runes:")
    for i, r := range s {
        fmt.Printf("  byte %2d: U+%04X '%c' (%d bytes)\n",
            i, r, r, len(string(r)))
    }
}
```

**Textual Figure — UTF-8 Multi-Byte Characters:**

```
  s = "Hello, 世界! 🌍"

  Byte layout:
  Position: 0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16 17 18 19
  Hex:      48 65 6C 6C 6F 2C 20 E4 B8 96 E7 95 8C 21 20 F0 9F 8C 8D
            H  e  l  l  o  ,     世(×3 bytes) 界(×3 bytes) !     🌍 (×4 bytes)
            └─── ASCII (1B each) ──┘  └─ 3B ─┘ └─ 3B ─┘     └── 4B ──┘

  len(s) = 20 bytes,  len([]rune(s)) = 12 runes

  Rune iteration (via range):
  byte  0: U+0048 'H' (1 byte)
  byte  1: U+0065 'e' (1 byte)
  ...                            range skips to next rune start!
  byte  7: U+4E16 '世' (3 bytes)
  byte 10: U+754C '界' (3 bytes)
  byte 15: U+1F30D '🌍' (4 bytes)

  Key: range s gives (byte_index, rune), NOT (rune_index, rune)
```

---

## Example 3: Rune vs Byte — Critical Difference

```go
package main

import (
    "fmt"
    "unicode/utf8"
)

func main() {
    examples := []string{"hello", "café", "日本語", "🎉🌟", "naïve"}

    for _, s := range examples {
        bytes := len(s)
        runes := utf8.RuneCountInString(s)
        fmt.Printf("%-10s bytes=%2d runes=%2d\n", s, bytes, runes)
    }
    // hello      bytes= 5 runes= 5
    // café       bytes= 5 runes= 4
    // 日本語     bytes= 9 runes= 3
    // 🎉🌟      bytes= 8 runes= 2
    // naïve      bytes= 6 runes= 5

    // Safe character access
    s := "café"
    fmt.Printf("s[3] = byte 0x%02X (WRONG: partial rune)\n", s[3])
    runes := []rune(s)
    fmt.Printf("rune[3] = '%c' (CORRECT)\n", runes[3]) // é
}
```

**Textual Figure — Rune vs Byte Access:**

```
  String      bytes  runes  Why different?
  ──────────────────────────────────────────────
  "hello"       5      5    All ASCII (1 byte each)
  "café"        5      4    é = 2 bytes
  "日本語"       9      3    Each CJK = 3 bytes
  "🎉🌟"        8      2    Each emoji = 4 bytes
  "naïve"       6      5    ï = 2 bytes

  Accessing "café":
  Index:  0    1    2    3    4
  Bytes: [63] [61] [66] [C3] [A9]
          c    a    f    └─é─┘

  s[3] = 0xC3  ✘  (half of é, invalid rune!)

  Runes: [99] [97] [102] [233]
          c    a     f    é

  rune[3] = 233 = 'é'  ✓

  Rule: NEVER index a string by byte for non-ASCII!
```

---

## Example 4: Converting Between Encodings

```go
package main

import "fmt"

func main() {
    // string → []byte (UTF-8 bytes)
    s := "hello"
    b := []byte(s)
    fmt.Println("Bytes:", b) // [104 101 108 108 111]

    // []byte → string
    s2 := string(b)
    fmt.Println("String:", s2) // hello

    // string → []rune (Unicode code points)
    r := []rune("café")
    fmt.Println("Runes:", r) // [99 97 102 233]

    // []rune → string
    s3 := string(r)
    fmt.Println("String:", s3) // café

    // int/rune → string (single character)
    fmt.Println(string(rune(65)))   // A
    fmt.Println(string(rune(9731))) // ☃ (snowman)
    fmt.Println(string(rune(128640))) // 🚀
}
```

**Textual Figure — Encoding Conversions:**

```
  Conversion paths:

  string ──── []byte(s) ────→ []byte     (copies UTF-8 bytes)
    │                              │
    │    string(b) ─────────────┘         (copies back)
    │
    ├──── []rune(s) ────→ []rune    (decodes to code points)
    │                              │
    │    string(r) ─────────────┘         (encodes back to UTF-8)
    │
    └──── string(rune(65)) ──→ "A"      (single code point)

  Example:
  "café" ──[]byte──→ [99, 97, 102, 195, 169]   (5 bytes)
  "café" ──[]rune──→ [99, 97, 102, 233]        (4 runes)

  ⚠ Each conversion allocates new memory!
```

---

## Example 5: Case Conversion

```go
package main

import (
    "fmt"
    "strings"
    "unicode"
)

func main() {
    s := "Hello, World! café 日本"

    fmt.Println(strings.ToUpper(s))  // HELLO, WORLD! CAFÉ 日本
    fmt.Println(strings.ToLower(s))  // hello, world! café 日本
    fmt.Println(strings.Title(s))    // Hello, World! Café 日本

    // Manual case conversion (ASCII only)
    toUpperASCII := func(c byte) byte {
        if c >= 'a' && c <= 'z' {
            return c - 32 // 'a' - 'A' = 32
        }
        return c
    }
    b := []byte("hello")
    for i := range b {
        b[i] = toUpperASCII(b[i])
    }
    fmt.Println(string(b)) // HELLO

    // Unicode-aware character classification
    for _, r := range "Hello123!é" {
        fmt.Printf("'%c': letter=%v digit=%v upper=%v lower=%v\n",
            r, unicode.IsLetter(r), unicode.IsDigit(r),
            unicode.IsUpper(r), unicode.IsLower(r))
    }
}
```

**Textual Figure — Case Conversion:**

```
  ASCII case conversion trick:

  'A' = 0100 0001 (65)
  'a' = 0110 0001 (97)       difference = 32 = bit 5

  To lowercase: c | 0x20   (set bit 5)
  To uppercase: c & ^0x20  (clear bit 5)
  Toggle case:  c ^ 0x20   (flip bit 5)

  Manual: c - 32 = uppercase,  c + 32 = lowercase

  Unicode classification:
  ┌───────┬────────┬───────┬───────┬───────┐
  │ Char  │ Letter │ Digit │ Upper │ Lower │
  ├───────┼────────┼───────┼───────┼───────┤
  │ 'H'   │  ✓     │  ✘    │  ✓    │  ✘    │
  │ 'e'   │  ✓     │  ✘    │  ✘    │  ✓    │
  │ '1'   │  ✘     │  ✓    │  ✘    │  ✘    │
  │ '!'   │  ✘     │  ✘    │  ✘    │  ✘    │
  │ 'é'   │  ✓     │  ✘    │  ✘    │  ✓    │
  └───────┴────────┴───────┴───────┴───────┘
```

---

```go
package main

import "fmt"

func charFrequency(s string) map[rune]int {
    freq := make(map[rune]int)
    for _, r := range s { // range iterates runes, not bytes
        freq[r]++
    }
    return freq
}

// ASCII-only version (array-based, faster)
func asciiFrequency(s string) [128]int {
    var freq [128]int
    for i := 0; i < len(s); i++ {
        if s[i] < 128 {
            freq[s[i]]++
        }
    }
    return freq
}

func main() {
    freq := charFrequency("hello 世界 世界 hello")
    for r, count := range freq {
        fmt.Printf("'%c': %d\n", r, count)
    }

    // ASCII frequency for letter counting
    f := asciiFrequency("abracadabra")
    for ch := byte('a'); ch <= byte('z'); ch++ {
        if f[ch] > 0 {
            fmt.Printf("%c: %d\n", ch, f[ch])
        }
    }
    // a:5 b:2 c:1 d:1 r:2
}
```

**Textual Figure — Character Frequency Counting:**

```
  Two approaches:

  1. map[rune]int (Unicode-safe):
     "hello 世界" → iterate runes via range
     map: {'h':1, 'e':1, 'l':2, 'o':1, ' ':1, '世':1, '界':1}

  2. [128]int array (ASCII-only, faster):
     "abracadabra"

     Index:   97  98  99 100 114    (a=97, b=98, ...)
     Array:  [ 5,  2,  1,  1, ..., 2, ...]   freq['a'-0]=5

     s[i]    freq[s[i]]++
     'a'  →  freq[97]++   → 5
     'b'  →  freq[98]++   → 2
     'r'  →  freq[114]++  → 2
     'c'  →  freq[99]++   → 1
     'd'  →  freq[100]++  → 1

  Speed comparison:
  ┌──────────────┬──────────┬──────────────┐
  │ Method       │ Lookup   │ Unicode?     │
  ├──────────────┼──────────┼──────────────┤
  │ [128]int     │ O(1)     │ ASCII only   │
  │ map[rune]int │ O(1) avg │ Yes          │
  └──────────────┴──────────┴──────────────┘
```

---

## Example 7: Is Anagram — Using Character Counts

```go
package main

import "fmt"

func isAnagram(s, t string) bool {
    if len(s) != len(t) {
        return false
    }
    var count [256]int
    for i := 0; i < len(s); i++ {
        count[s[i]]++
        count[t[i]]--
    }
    for _, c := range count {
        if c != 0 {
            return false
        }
    }
    return true
}

// Unicode-safe version
func isAnagramUnicode(s, t string) bool {
    sr, tr := []rune(s), []rune(t)
    if len(sr) != len(tr) {
        return false
    }
    freq := make(map[rune]int)
    for _, r := range sr { freq[r]++ }
    for _, r := range tr { freq[r]-- }
    for _, v := range freq {
        if v != 0 { return false }
    }
    return true
}

func main() {
    fmt.Println(isAnagram("anagram", "nagaram"))   // true
    fmt.Println(isAnagram("rat", "car"))            // false
    fmt.Println(isAnagramUnicode("café", "éfac"))   // true
}
```

**Textual Figure — Anagram Detection:**

```
  isAnagram("anagram", "nagaram"):

  Use a single [256]int count array:
  For s: count[ch]++     For t: count[ch]--

  s = "anagram"     t = "nagaram"
  ┌───┬──────────┬──────────┐
  │ i │ s[i]++ →  │ t[i]-- →  │
  ├───┼──────────┼──────────┤
  │ 0 │ a: +1    │ n: -1    │
  │ 1 │ n: +1    │ a: -1    │
  │ 2 │ a: +1    │ g: -1    │
  │ 3 │ g: +1    │ a: -1    │
  │ 4 │ r: +1    │ r: -1    │
  │ 5 │ a: +1    │ a: -1    │
  │ 6 │ m: +1    │ m: -1    │
  └───┴──────────┴──────────┘

  Final counts: all zeros → true (anagram!) ✓

  One-pass trick: increment for s, decrement for t simultaneously.
```

---

## Example 8: UTF-8 Encoding Details

```go
package main

import (
    "fmt"
    "unicode/utf8"
)

func main() {
    // UTF-8 encoding rules:
    // U+0000..U+007F:   0xxxxxxx                  (1 byte, ASCII)
    // U+0080..U+07FF:   110xxxxx 10xxxxxx          (2 bytes)
    // U+0800..U+FFFF:   1110xxxx 10xxxxxx 10xxxxxx (3 bytes)
    // U+10000..U+10FFFF: 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx (4 bytes)

    chars := []rune{'A', 'é', '世', '🌍'}
    for _, ch := range chars {
        buf := make([]byte, 4)
        n := utf8.EncodeRune(buf, ch)
        fmt.Printf("'%c' U+%04X → %d bytes: % X\n", ch, ch, n, buf[:n])
    }
    // 'A' U+0041 → 1 bytes: 41
    // 'é' U+00E9 → 2 bytes: C3 A9
    // '世' U+4E16 → 3 bytes: E4 B8 96
    // '🌍' U+1F30D → 4 bytes: F0 9F 8C 8D

    // Decode first rune
    s := "日本語"
    r, size := utf8.DecodeRuneInString(s)
    fmt.Printf("First rune: '%c', size: %d bytes\n", r, size) // 日, 3

    // Validate UTF-8
    fmt.Println(utf8.ValidString("hello"))   // true
    fmt.Println(utf8.Valid([]byte{0xff, 0xfe})) // false — invalid UTF-8
}
```

**Textual Figure — UTF-8 Encoding Rules:**

```
  UTF-8 Variable-Length Encoding:

  ┌───────────────────┬───────┬─────────────────────────────────┐
  │ Code Point Range    │ Bytes │ Bit Pattern                      │
  ├───────────────────┼───────┼─────────────────────────────────┤
  │ U+0000..U+007F     │   1   │ 0xxxxxxx                         │
  │ U+0080..U+07FF     │   2   │ 110xxxxx 10xxxxxx                 │
  │ U+0800..U+FFFF     │   3   │ 1110xxxx 10xxxxxx 10xxxxxx        │
  │ U+10000..U+10FFFF  │   4   │ 11110xxx 10xxxxxx 10xxxxxx 10xx.. │
  └───────────────────┴───────┴─────────────────────────────────┘

  Examples:
  'A'  U+0041 → 01000001                   = 0x41 (1 byte)
  'é'  U+00E9 → 11000011 10101001          = 0xC3 0xA9 (2 bytes)
  '世' U+4E16 → 11100100 10111000 10010110 = 0xE4 0xB8 0x96 (3 bytes)
  '🌍' U+1F30D→ 11110000 10011111 10001100 10001101 (4 bytes)

  Leading bits tell the decoder how many bytes to read.
```

---

## Example 9: String as []byte for Pattern Problems

```go
package main

import "fmt"

func main() {
    // For interview problems with ASCII, using []byte or [26]int is common

    // Count lowercase letter frequencies
    s := "leetcode"
    var freq [26]int
    for i := 0; i < len(s); i++ {
        freq[s[i]-'a']++
    }
    fmt.Println("Frequencies:")
    for i, f := range freq {
        if f > 0 {
            fmt.Printf("  %c: %d\n", 'a'+i, f)
        }
    }

    // Check if two strings are permutations
    isPermutation := func(a, b string) bool {
        if len(a) != len(b) {
            return false
        }
        var count [26]int
        for i := 0; i < len(a); i++ {
            count[a[i]-'a']++
            count[b[i]-'a']--
        }
        for _, c := range count {
            if c != 0 {
                return false
            }
        }
        return true
    }
    fmt.Println(isPermutation("abc", "cba")) // true
    fmt.Println(isPermutation("abc", "abd")) // false
}
```

**Textual Figure — [26]int Array for Letter Problems:**

```
  s = "leetcode"

  freq[ch - 'a']++ for each byte:

  Index:  0  1  2  3  4  5  ... 11  ... 14  ... 19  ...
  Letter: a  b  c  d  e  f      l       o       t
  Count: [0, 0, 1, 1, 3, 0, ..., 1, ..., 1, ..., 1, ...]
                  c  d  e        l       o       t

  Permutation check with [26]int:
  a = "abc",  b = "cba"

  Process simultaneously:
  i=0: count['a']++, count['c']--  → [+1, 0, -1, ...]
  i=1: count['b']++, count['b']--  → [+1, 0, -1, ...]
  i=2: count['c']++, count['a']--  → [ 0, 0,  0, ...]

  All zeros → permutation!  ✓

  This technique: O(n) time, O(1) space (fixed 26 slots)
```

---

## Example 10: Handling Special Characters and Escapes

```go
package main

import (
    "fmt"
    "strconv"
    "strings"
    "unicode"
)

func main() {
    // Common escape sequences
    s := "Hello\tWorld\nNew \"Line\"\t\\Backslash"
    fmt.Println(s)

    // Raw strings — no escape processing
    raw := `Hello\tWorld\nNo escapes here "quotes ok"`
    fmt.Println(raw)

    // Strip non-alphanumeric characters
    clean := func(s string) string {
        var sb strings.Builder
        for _, r := range s {
            if unicode.IsLetter(r) || unicode.IsDigit(r) {
                sb.WriteRune(r)
            }
        }
        return sb.String()
    }
    fmt.Println(clean("A man, a plan, a canal: Panama"))
    // AmanaplanacanelPanama

    // Quote/Unquote
    quoted := strconv.Quote("hello\nworld")
    fmt.Println(quoted)            // "hello\nworld"
    unquoted, _ := strconv.Unquote(quoted)
    fmt.Println(unquoted)          // hello
                                   // world
}
```

**Textual Figure — Escape Sequences and Raw Strings:**

```
  Regular string (escapes processed):
  "Hello\tWorld\n"  →  Hello→(tab)World↓(newline)

  Common escape sequences:
  ┌──────┬────────────┬─────────┬────────┐
  │ Esc  │ Meaning    │ Decimal │ Hex    │
  ├──────┼────────────┼─────────┼────────┤
  │ \n   │ Newline    │ 10      │ 0x0A   │
  │ \t   │ Tab        │ 9       │ 0x09   │
  │ \\   │ Backslash  │ 92      │ 0x5C   │
  │ \"   │ Quote      │ 34      │ 0x22   │
  └──────┴────────────┴─────────┴────────┘

  Raw string (backtick, no escapes):
  `Hello\tWorld\n`  →  Hello\tWorld\n  (literal backslashes!)

  Clean non-alphanumeric:
  "A man, a plan..." → check each rune:
  'A'✓ ' '✘ 'm'✓ 'a'✓ 'n'✓ ','✘ ' '✘ ...
  Result: "AmanaplanacanelPanama"
```

---

## Key Takeaways

1. **Go strings are UTF-8** — ASCII chars are 1 byte, others 1-4 bytes
2. **`len(s)` = byte count**, `len([]rune(s))` = character count
3. **`range s` iterates runes**, `s[i]` accesses bytes
4. **Use `[26]int` or `[128]int`** for ASCII frequency counting
5. **Use `map[rune]int`** for Unicode frequency counting
6. **`unicode/utf8`** package for encoding/decoding details

> **Next up:** Pattern Matching →
