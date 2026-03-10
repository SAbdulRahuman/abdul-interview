# Chapter 33 — Iota & Enum Patterns in Go

Go doesn't have a dedicated `enum` keyword. Instead, it uses `const` blocks with `iota` — an auto-incrementing constant generator. This chapter covers all iota mechanics and idiomatic patterns for building type-safe, string-representable enumerations.

```
┌──────────────────────────────────────────────────────────────┐
│                    Iota & Enum Patterns                       │
│                                                              │
│  const (                                                     │
│      Sunday    Weekday = iota   // 0                         │
│      Monday                     // 1  (iota auto-increments) │
│      Tuesday                    // 2                         │
│      Wednesday                  // 3                         │
│      Thursday                   // 4                         │
│      Friday                     // 5                         │
│      Saturday                   // 6                         │
│  )                                                           │
│                                                              │
│  Key Rules:                                                  │
│  • iota starts at 0, increments per const spec               │
│  • Resets to 0 in each new const block                       │
│  • Blank identifier _ can skip values                        │
│  • Expressions with iota are repeated implicitly             │
│  • Custom types + String() method = type-safe enums          │
└──────────────────────────────────────────────────────────────┘
```

## Iota Basics

**Tutorial: How Iota Works in Const Blocks**

`iota` is a predeclared identifier that represents the index of the current constant specification within a `const` block, starting from 0. Each time a new `const` declaration starts, `iota` resets to 0. Within a block, `iota` increments by 1 for each constant specification (line), not each constant name.

```
┌────────────────────────────────────────────────────────────┐
│  const (          iota value                               │
│      A = iota     // 0   ← iota starts at 0               │
│      B            // 1   ← implicit repeat of "= iota"    │
│      C            // 2                                     │
│  )                                                         │
│                                                            │
│  const (          iota resets                               │
│      X = iota     // 0   ← new block, reset to 0          │
│      Y            // 1                                     │
│  )                                                         │
│                                                            │
│  const Z = iota   // 0   ← single const, always 0         │
│                                                            │
│  const (                                                   │
│      _  = iota    // 0   ← skipped with blank identifier   │
│      P            // 1                                     │
│      Q            // 2                                     │
│  )                                                         │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Basic iota — starts at 0, auto-increments
const (
	Red   = iota // 0
	Green        // 1
	Blue         // 2
)

// Iota resets per const block
const (
	Small  = iota // 0
	Medium        // 1
	Large         // 2
)

// Skip zero with blank identifier
const (
	_         = iota // 0 skipped
	January          // 1
	February         // 2
	March            // 3
)

func main() {
	fmt.Println("Colors:", Red, Green, Blue)       // 0 1 2
	fmt.Println("Sizes:", Small, Medium, Large)     // 0 1 2
	fmt.Println("Months:", January, February, March) // 1 2 3
}
```

---

## Iota with Expressions

**Tutorial: Using Expressions That Repeat with Iota**

When a const spec uses an expression involving `iota`, subsequent specs repeat the same expression with `iota` incrementing. This enables patterns like powers of 2, offsets, and multiplied sequences without repeating the expression.

```
┌────────────────────────────────────────────────────────────┐
│  const (                                                   │
│      KB = 1 << (10 * (iota + 1))                           │
│      MB                          // repeats expression     │
│      GB                          // with iota = 2          │
│      TB                          // with iota = 3          │
│  )                                                         │
│                                                            │
│  iota=0: 1 << (10 * 1) = 1 << 10 = 1024         (KB)      │
│  iota=1: 1 << (10 * 2) = 1 << 20 = 1,048,576    (MB)      │
│  iota=2: 1 << (10 * 3) = 1 << 30 = 1,073,741,824 (GB)     │
│  iota=3: 1 << (10 * 4) = 1 << 40 = 1,099,511,627,776 (TB) │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Byte sizes — powers of 1024
const (
	KB = 1 << (10 * (iota + 1)) // 1 << 10 = 1024
	MB                          // 1 << 20 = 1,048,576
	GB                          // 1 << 30
	TB                          // 1 << 40
)

// Starting from a specific value
const (
	StatusPending    = iota + 100 // 100
	StatusProcessing              // 101
	StatusCompleted               // 102
	StatusFailed                  // 103
)

// Multiplied values
const (
	None     = 0
	Low      = (iota) * 10 // 10  (iota=1)
	Medium2               // 20  (iota=2)
	High                  // 30  (iota=3)
	Critical              // 40  (iota=4)
)

func main() {
	fmt.Printf("KB=%d  MB=%d  GB=%d  TB=%d\n", KB, MB, GB, TB)
	// KB=1024  MB=1048576  GB=1073741824  TB=1099511627776

	fmt.Println("Status:", StatusPending, StatusProcessing, StatusCompleted, StatusFailed)
	// 100 101 102 103

	fmt.Println("Priority:", None, Low, Medium2, High, Critical)
	// 0 10 20 30 40
}
```

---

## Bitmask / Bitflag Enums

**Tutorial: Using Iota for Permission Flags**

Bitflags use `1 << iota` to create powers of 2. Each constant occupies one bit, so they can be combined with `|` (OR) and tested with `&` (AND). This is the standard pattern for permission systems, feature flags, and option sets.

```
┌────────────────────────────────────────────────────────────┐
│  Bitmask Pattern: 1 << iota                                │
│                                                            │
│  const (                                                   │
│      Read    = 1 << iota  // 0001 = 1                      │
│      Write                // 0010 = 2                      │
│      Execute              // 0100 = 4                      │
│      Admin                // 1000 = 8                      │
│  )                                                         │
│                                                            │
│  Combine:  Read | Write        = 0011 = 3                  │
│  Test:     (perms & Write) != 0  → has Write?              │
│  All:      Read | Write | Execute | Admin = 1111 = 15      │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Bit:    3    2    1    0                             │  │
│  │          Admin Exec Write Read                        │  │
│  │                                                      │  │
│  │  Read|Write = 0011 → can read and write              │  │
│  │  Admin|Exec = 1100 → admin + execute only            │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"strings"
)

type Permission uint8

const (
	Read    Permission = 1 << iota // 1
	Write                          // 2
	Execute                        // 4
	Admin                          // 8
)

func (p Permission) String() string {
	var parts []string
	if p&Read != 0 {
		parts = append(parts, "Read")
	}
	if p&Write != 0 {
		parts = append(parts, "Write")
	}
	if p&Execute != 0 {
		parts = append(parts, "Execute")
	}
	if p&Admin != 0 {
		parts = append(parts, "Admin")
	}
	if len(parts) == 0 {
		return "None"
	}
	return strings.Join(parts, "|")
}

func (p Permission) Has(flag Permission) bool {
	return p&flag != 0
}

func main() {
	// Combine permissions with |
	userPerms := Read | Write
	adminPerms := Read | Write | Execute | Admin

	fmt.Println("User perms:", userPerms)   // Read|Write
	fmt.Println("Admin perms:", adminPerms) // Read|Write|Execute|Admin

	// Test individual permissions
	fmt.Println("User has Write?", userPerms.Has(Write))     // true
	fmt.Println("User has Admin?", userPerms.Has(Admin))     // false
	fmt.Println("Admin has Execute?", adminPerms.Has(Execute)) // true

	// Add/remove permission
	userPerms |= Execute          // add Execute
	fmt.Println("After +Execute:", userPerms) // Read|Write|Execute

	userPerms &^= Write           // remove Write (AND NOT)
	fmt.Println("After -Write:", userPerms)   // Read|Execute
}
```

---

## Type-Safe Enums with Custom Type

**Tutorial: Preventing Invalid Enum Values with Named Types**

Using a named type for your enum prevents accidentally mixing unrelated constants. The compiler catches type mismatches at compile time.

```
┌────────────────────────────────────────────────────────────┐
│  type Color int   ← named type, not raw int               │
│                                                            │
│  const (                                                   │
│      Red   Color = iota                                    │
│      Green                                                 │
│      Blue                                                  │
│  )                                                         │
│                                                            │
│  func Paint(c Color) { ... }                               │
│                                                            │
│  Paint(Red)    ✓  (Red is Color)                           │
│  Paint(0)      ✓  (untyped const 0 converts to Color)      │
│  Paint(Blue)   ✓                                           │
│                                                            │
│  var x int = 5                                             │
│  Paint(x)      ✗  COMPILE ERROR: int ≠ Color               │
│  Paint(Color(x)) ✓ (explicit conversion)                   │
│                                                            │
│  ⚠️ Limitation: Color(99) compiles — no range check at     │
│  compile time. Validate at runtime if needed.              │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type Direction int

const (
	North Direction = iota
	East
	South
	West
)

// String method — makes Direction printable
func (d Direction) String() string {
	switch d {
	case North:
		return "North"
	case East:
		return "East"
	case South:
		return "South"
	case West:
		return "West"
	default:
		return fmt.Sprintf("Direction(%d)", int(d))
	}
}

// IsValid checks if the direction is within the valid range
func (d Direction) IsValid() bool {
	return d >= North && d <= West
}

// Opposite returns the reverse direction
func (d Direction) Opposite() Direction {
	return (d + 2) % 4
}

// Type safety: function accepts only Direction, not raw int
func Move(d Direction, steps int) {
	fmt.Printf("Moving %s for %d steps\n", d, steps)
}

func main() {
	Move(North, 5)              // Moving North for 5 steps
	Move(East, 3)               // Moving East for 3 steps

	fmt.Println(North.Opposite()) // South
	fmt.Println(East.Opposite())  // West

	// var x int = 1
	// Move(x, 5)  ← COMPILE ERROR: cannot use x (type int) as Direction

	// Runtime validation for values from external input
	d := Direction(99)
	fmt.Println(d.IsValid()) // false
	fmt.Println(d)            // Direction(99)
}
```

---

## String Enums with go:generate stringer

**Tutorial: Auto-Generating String() Methods with stringer**

The `stringer` tool (from `golang.org/x/tools`) auto-generates `String()` methods for integer-based enums. Run `go generate` and it creates a `_string.go` file with an efficient string lookup.

```
┌────────────────────────────────────────────────────────────┐
│  //go:generate stringer -type=Weekday                      │
│  type Weekday int                                          │
│  const (                                                   │
│      Sunday Weekday = iota                                 │
│      Monday                                                │
│      ...                                                   │
│  )                                                         │
│                                                            │
│  $ go generate ./...                                       │
│  → creates weekday_string.go:                              │
│                                                            │
│  func (i Weekday) String() string {                        │
│      // efficient lookup table                             │
│      ...                                                   │
│  }                                                         │
│                                                            │
│  fmt.Println(Monday)  → "Monday"                           │
│  fmt.Println(Friday)  → "Friday"                           │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// In production, use: //go:generate stringer -type=Weekday
// Install: go install golang.org/x/tools/cmd/stringer@latest

type Weekday int

const (
	Sunday Weekday = iota
	Monday
	Tuesday
	Wednesday
	Thursday
	Friday
	Saturday
)

// Manual String() — stringer generates this automatically
func (w Weekday) String() string {
	names := [...]string{
		"Sunday", "Monday", "Tuesday", "Wednesday",
		"Thursday", "Friday", "Saturday",
	}
	if w < Sunday || w > Saturday {
		return fmt.Sprintf("Weekday(%d)", int(w))
	}
	return names[w]
}

func (w Weekday) IsWeekend() bool {
	return w == Sunday || w == Saturday
}

func (w Weekday) IsWeekday() bool {
	return !w.IsWeekend()
}

func main() {
	for d := Sunday; d <= Saturday; d++ {
		status := "workday"
		if d.IsWeekend() {
			status = "weekend"
		}
		fmt.Printf("%-10s → %s\n", d, status)
	}
}
```

---

## String-Based Enums

**Tutorial: Enums with String Underlying Type**

Sometimes you want enums to be strings (e.g., for JSON, database values). Use `type Status string` with const values. This avoids the int-to-string mapping entirely but loses `iota`.

```go
package main

import (
	"encoding/json"
	"fmt"
)

type Status string

const (
	StatusActive   Status = "active"
	StatusInactive Status = "inactive"
	StatusPending  Status = "pending"
	StatusBanned   Status = "banned"
)

func (s Status) IsValid() bool {
	switch s {
	case StatusActive, StatusInactive, StatusPending, StatusBanned:
		return true
	}
	return false
}

type User struct {
	Name   string `json:"name"`
	Status Status `json:"status"`
}

func main() {
	u := User{Name: "Alice", Status: StatusActive}

	// Marshals naturally to JSON without custom marshaler
	data, _ := json.Marshal(u)
	fmt.Println(string(data))
	// {"name":"Alice","status":"active"}

	// Unmarshals from JSON directly
	var u2 User
	json.Unmarshal([]byte(`{"name":"Bob","status":"pending"}`), &u2)
	fmt.Println(u2.Name, u2.Status, u2.Status.IsValid())
	// Bob pending true

	// Invalid status from untrusted input
	var u3 User
	json.Unmarshal([]byte(`{"name":"Eve","status":"hacker"}`), &u3)
	fmt.Println(u3.Name, u3.Status, u3.Status.IsValid())
	// Eve hacker false
}
```

---

## Sentinel Value Pattern — Enum Boundaries

**Tutorial: Using Sentinels for Validation and Iteration**

Define sentinel constants at the beginning and/or end of your enum to enable range validation and iteration. This pattern catches invalid values at runtime.

```
┌────────────────────────────────────────────────────────────┐
│  const (                                                   │
│      logLevelBegin LogLevel = iota   // 0 — sentinel       │
│      Debug                           // 1                  │
│      Info                            // 2                  │
│      Warn                            // 3                  │
│      Error                           // 4                  │
│      Fatal                           // 5                  │
│      logLevelEnd                     // 6 — sentinel       │
│  )                                                         │
│                                                            │
│  Valid range: logLevelBegin < level < logLevelEnd           │
│  IsValid: level > 0 && level < 6                           │
│  Count:   logLevelEnd - logLevelBegin - 1 = 5              │
│  Iterate: for l := Debug; l < logLevelEnd; l++ { ... }     │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type LogLevel int

const (
	logLevelBegin LogLevel = iota // sentinel (unexported)
	Debug
	Info
	Warn
	Error
	Fatal
	logLevelEnd // sentinel (unexported)
)

var logLevelNames = map[LogLevel]string{
	Debug: "DEBUG",
	Info:  "INFO",
	Warn:  "WARN",
	Error: "ERROR",
	Fatal: "FATAL",
}

func (l LogLevel) String() string {
	if name, ok := logLevelNames[l]; ok {
		return name
	}
	return fmt.Sprintf("LogLevel(%d)", int(l))
}

func (l LogLevel) IsValid() bool {
	return l > logLevelBegin && l < logLevelEnd
}

func AllLogLevels() []LogLevel {
	levels := make([]LogLevel, 0, logLevelEnd-logLevelBegin-1)
	for l := logLevelBegin + 1; l < logLevelEnd; l++ {
		levels = append(levels, l)
	}
	return levels
}

func main() {
	// Iterate all valid levels
	fmt.Println("All log levels:")
	for _, l := range AllLogLevels() {
		fmt.Printf("  %d: %s (valid=%t)\n", l, l, l.IsValid())
	}

	// Validate external input
	fromUser := LogLevel(99)
	fmt.Printf("\nLogLevel(99) valid? %t\n", fromUser.IsValid()) // false
}
```

---

## Iota with Multiple Constants Per Line

**Tutorial: Iota Increments Per Spec, Not Per Name**

A subtle rule: `iota` increments per constant *specification* (line), not per constant *name*. Multiple constants on the same line share the same `iota` value.

```go
package main

import "fmt"

// Multiple constants per line — same iota
const (
	A, B = iota, iota * 10 // iota=0: A=0, B=0
	C, D                   // iota=1: C=1, D=10
	E, F                   // iota=2: E=2, F=20
)

// Practical: paired constants
const (
	StatusCode200, StatusText200 = 200 + iota, "OK"         // iota=0
	StatusCode201, StatusText201 = 200 + iota, "Created"    // iota=1
	StatusCode202, StatusText202 = 200 + iota, "Accepted"   // iota=2
)

func main() {
	fmt.Println(A, B) // 0 0
	fmt.Println(C, D) // 1 10
	fmt.Println(E, F) // 2 20

	fmt.Println(StatusCode200, StatusText200) // 200 OK
	fmt.Println(StatusCode201, StatusText201) // 201 Created
	fmt.Println(StatusCode202, StatusText202) // 202 Accepted
}
```

---

## Enum with JSON Marshal/Unmarshal

**Tutorial: Custom JSON Encoding for Integer Enums**

Integer enums need custom `MarshalJSON`/`UnmarshalJSON` to appear as strings in JSON rather than numbers.

```go
package main

import (
	"encoding/json"
	"fmt"
)

type Priority int

const (
	PriorityLow Priority = iota
	PriorityMedium
	PriorityHigh
	PriorityCritical
)

var priorityNames = map[Priority]string{
	PriorityLow:      "low",
	PriorityMedium:   "medium",
	PriorityHigh:     "high",
	PriorityCritical: "critical",
}

var priorityValues = map[string]Priority{
	"low":      PriorityLow,
	"medium":   PriorityMedium,
	"high":     PriorityHigh,
	"critical": PriorityCritical,
}

func (p Priority) String() string {
	if name, ok := priorityNames[p]; ok {
		return name
	}
	return fmt.Sprintf("Priority(%d)", int(p))
}

func (p Priority) MarshalJSON() ([]byte, error) {
	return json.Marshal(p.String())
}

func (p *Priority) UnmarshalJSON(data []byte) error {
	var s string
	if err := json.Unmarshal(data, &s); err != nil {
		return err
	}
	val, ok := priorityValues[s]
	if !ok {
		return fmt.Errorf("invalid priority: %q", s)
	}
	*p = val
	return nil
}

type Task struct {
	Title    string   `json:"title"`
	Priority Priority `json:"priority"`
}

func main() {
	task := Task{Title: "Fix bug", Priority: PriorityHigh}

	// Marshal — outputs "high" not 2
	data, _ := json.MarshalIndent(task, "", "  ")
	fmt.Println(string(data))
	// {
	//   "title": "Fix bug",
	//   "priority": "high"
	// }

	// Unmarshal — parses "medium" back to PriorityMedium
	var task2 Task
	json.Unmarshal([]byte(`{"title":"Deploy","priority":"medium"}`), &task2)
	fmt.Printf("Parsed: %s (priority=%d)\n", task2.Title, task2.Priority)
	// Parsed: Deploy (priority=1)

	// Invalid value
	var task3 Task
	err := json.Unmarshal([]byte(`{"title":"X","priority":"urgent"}`), &task3)
	fmt.Println("Error:", err)
	// Error: invalid priority: "urgent"
}
```

---

## Interview Questions

1. **What is `iota` in Go?**
   - `iota` is a predeclared identifier used in `const` blocks. It starts at 0 and increments by 1 for each constant specification. It resets to 0 at each new `const` block. Used to create auto-incrementing enums.

2. **How do you skip a value with iota?**
   - Use the blank identifier: `_ = iota` to skip. The next constant gets `iota+1`. Example: `_, January, February = iota, iota, iota` or simply `_ = iota` on its own line followed by `January` which gets 1.

3. **How do you create bitflag enums?**
   - Use `1 << iota`. Each constant occupies one bit. Combine with `|`, test with `&`. Example: `Read = 1 << iota` (1), `Write` (2), `Execute` (4). Combine: `perms := Read | Write`.

4. **How does iota behave with multiple constants on one line?**
   - `iota` increments per constant *spec* (line), not per name. `A, B = iota, iota*10` gives A=0, B=0. Next line `C, D` gives C=1, D=10 (same expression repeated).

5. **How do you make an enum JSON-friendly?**
   - For string-based enums (`type Status string`), JSON works automatically. For integer enums, implement `MarshalJSON` (return string) and `UnmarshalJSON` (parse string, map to int). Alternative: use the `stringer` tool.

6. **How do you validate enum values at runtime?**
   - Use sentinel constants: `enumBegin = iota` at start, `enumEnd` at end. `IsValid()` checks `value > enumBegin && value < enumEnd`. This catches out-of-range values from external input or type conversions.

7. **What is `go generate stringer`?**
   - `stringer` auto-generates `String()` methods for integer enums. Add `//go:generate stringer -type=MyEnum` above your type. Run `go generate ./...` to create the `_string.go` file. Part of `golang.org/x/tools`.

---

## Interview Problems & Solutions

### Problem 1 — HTTP Status Code Enum with Methods

**Problem:** Design an HTTP status code enum with methods: `IsSuccess()`, `IsClientError()`, `IsServerError()`, `IsRedirect()`. Include a `Category()` method that returns a string. Support JSON marshaling as the standard text representation (e.g., "200 OK").

```go
package main

import (
	"encoding/json"
	"fmt"
)

type HTTPStatus int

const (
	StatusOK                  HTTPStatus = 200
	StatusCreated             HTTPStatus = 201
	StatusMovedPermanently    HTTPStatus = 301
	StatusFound               HTTPStatus = 302
	StatusBadRequest          HTTPStatus = 400
	StatusUnauthorized        HTTPStatus = 401
	StatusForbidden           HTTPStatus = 403
	StatusNotFound            HTTPStatus = 404
	StatusInternalServerError HTTPStatus = 500
	StatusBadGateway          HTTPStatus = 502
	StatusServiceUnavailable  HTTPStatus = 503
)

var statusTexts = map[HTTPStatus]string{
	StatusOK:                  "OK",
	StatusCreated:             "Created",
	StatusMovedPermanently:    "Moved Permanently",
	StatusFound:               "Found",
	StatusBadRequest:          "Bad Request",
	StatusUnauthorized:        "Unauthorized",
	StatusForbidden:           "Forbidden",
	StatusNotFound:            "Not Found",
	StatusInternalServerError: "Internal Server Error",
	StatusBadGateway:          "Bad Gateway",
	StatusServiceUnavailable:  "Service Unavailable",
}

func (s HTTPStatus) String() string {
	if text, ok := statusTexts[s]; ok {
		return fmt.Sprintf("%d %s", int(s), text)
	}
	return fmt.Sprintf("%d", int(s))
}

func (s HTTPStatus) IsSuccess() bool     { return s >= 200 && s < 300 }
func (s HTTPStatus) IsRedirect() bool    { return s >= 300 && s < 400 }
func (s HTTPStatus) IsClientError() bool { return s >= 400 && s < 500 }
func (s HTTPStatus) IsServerError() bool { return s >= 500 && s < 600 }

func (s HTTPStatus) Category() string {
	switch {
	case s.IsSuccess():
		return "Success"
	case s.IsRedirect():
		return "Redirect"
	case s.IsClientError():
		return "Client Error"
	case s.IsServerError():
		return "Server Error"
	default:
		return "Unknown"
	}
}

func (s HTTPStatus) MarshalJSON() ([]byte, error) {
	return json.Marshal(s.String())
}

type APIResponse struct {
	Status HTTPStatus `json:"status"`
	Body   string     `json:"body"`
}

func main() {
	codes := []HTTPStatus{
		StatusOK, StatusCreated, StatusMovedPermanently,
		StatusNotFound, StatusInternalServerError,
	}

	for _, code := range codes {
		fmt.Printf("%-30s → %s\n", code, code.Category())
	}

	// JSON marshaling
	resp := APIResponse{Status: StatusNotFound, Body: "page not found"}
	data, _ := json.MarshalIndent(resp, "", "  ")
	fmt.Println("\n" + string(data))
}
```

---

### Problem 2 — State Machine with Enum Transitions

**Problem:** Model an order lifecycle as a state machine using enums. States: Created → Paid → Shipped → Delivered (or Cancelled from Created/Paid). Enforce valid transitions — attempting an invalid transition returns an error.

```
┌───────────────────────────────────────────────────────┐
│  Created ──► Paid ──► Shipped ──► Delivered           │
│     │          │                                      │
│     └──────────┴──────► Cancelled                     │
│                                                       │
│  Valid transitions:                                   │
│  Created   → Paid, Cancelled                          │
│  Paid      → Shipped, Cancelled                       │
│  Shipped   → Delivered                                │
│  Delivered → (none — terminal)                        │
│  Cancelled → (none — terminal)                        │
└───────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type OrderState int

const (
	orderStateBegin OrderState = iota
	StateCreated
	StatePaid
	StateShipped
	StateDelivered
	StateCancelled
	orderStateEnd
)

var stateNames = [...]string{
	StateCreated:   "Created",
	StatePaid:      "Paid",
	StateShipped:   "Shipped",
	StateDelivered: "Delivered",
	StateCancelled: "Cancelled",
}

func (s OrderState) String() string {
	if s > orderStateBegin && s < orderStateEnd {
		return stateNames[s]
	}
	return fmt.Sprintf("OrderState(%d)", int(s))
}

// Valid transitions: from → set of valid destinations
var validTransitions = map[OrderState][]OrderState{
	StateCreated: {StatePaid, StateCancelled},
	StatePaid:    {StateShipped, StateCancelled},
	StateShipped: {StateDelivered},
	// Delivered and Cancelled are terminal — no transitions
}

type Order struct {
	ID    string
	State OrderState
}

func NewOrder(id string) *Order {
	return &Order{ID: id, State: StateCreated}
}

func (o *Order) Transition(to OrderState) error {
	allowed, ok := validTransitions[o.State]
	if !ok {
		return fmt.Errorf("order %s: state %s is terminal, cannot transition", o.ID, o.State)
	}

	for _, valid := range allowed {
		if valid == to {
			fmt.Printf("Order %s: %s → %s\n", o.ID, o.State, to)
			o.State = to
			return nil
		}
	}

	return fmt.Errorf("order %s: invalid transition %s → %s", o.ID, o.State, to)
}

func main() {
	order := NewOrder("ORD-001")

	// Valid transitions
	order.Transition(StatePaid)      // Created → Paid ✓
	order.Transition(StateShipped)   // Paid → Shipped ✓
	order.Transition(StateDelivered) // Shipped → Delivered ✓

	// Terminal state — error
	err := order.Transition(StatePaid) // Delivered → Paid ✗
	fmt.Println("Error:", err)

	// Invalid transition
	order2 := NewOrder("ORD-002")
	err = order2.Transition(StateShipped) // Created → Shipped ✗
	fmt.Println("Error:", err)

	// Cancellation
	order3 := NewOrder("ORD-003")
	order3.Transition(StatePaid)
	order3.Transition(StateCancelled) // Paid → Cancelled ✓
}
```

**Key Points:**
- Enum + transition map enforces valid state changes at runtime
- Terminal states have no entries in the transition map
- Adding a new state requires updating the enum and transition map
- Pattern scales to complex workflows (approval chains, ticket systems)
