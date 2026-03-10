# Phase 24: Divide & Conquer — Closest Pair of Points

## Overview

Given n points in 2D, find the pair with the smallest Euclidean distance. The naive O(n²) approach checks all pairs. The **D&C approach** achieves O(n log n) by:
1. **Divide**: Sort by x, split into left/right halves
2. **Conquer**: Find closest pair in each half
3. **Combine**: Check the strip near the dividing line (the critical step)

The key insight: in the strip of width 2δ, each point needs to compare with at most 7 neighbors (sorted by y).

---

## Example 1: Brute Force Closest Pair

```go
package main

import (
	"fmt"
	"math"
)

type Point struct{ X, Y float64 }

func dist(a, b Point) float64 {
	dx, dy := a.X-b.X, a.Y-b.Y
	return math.Sqrt(dx*dx + dy*dy)
}

func closestBrute(points []Point) (float64, Point, Point) {
	minDist := math.Inf(1)
	var p1, p2 Point
	for i := 0; i < len(points); i++ {
		for j := i + 1; j < len(points); j++ {
			d := dist(points[i], points[j])
			if d < minDist {
				minDist = d
				p1, p2 = points[i], points[j]
			}
		}
	}
	return minDist, p1, p2
}

func main() {
	points := []Point{{2, 3}, {12, 30}, {40, 50}, {5, 1}, {12, 10}, {3, 4}}

	d, p1, p2 := closestBrute(points)
	fmt.Printf("Points: %v\n", points)
	fmt.Printf("Closest pair: (%.0f,%.0f) and (%.0f,%.0f)\n", p1.X, p1.Y, p2.X, p2.Y)
	fmt.Printf("Distance: %.4f\n", d)
	fmt.Println("\nBrute force: O(n²)")
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Brute Force Closest Pair                       │
├─────────────────────────────────────────────────┤
│  Points: (2,3)(12,30)(40,50)(5,1)(12,10)(3,4)  │
│                                                 │
│  Check ALL C(6,2) = 15 pairs:                   │
│    (2,3)-(12,30)   = 28.79                      │
│    (2,3)-(40,50)   = 58.52                      │
│    (2,3)-(5,1)     = 3.61                       │
│    (2,3)-(12,10)   = 12.21                      │
│    (2,3)-(3,4)  →  = 1.41  ★ closest!          │
│    (12,30)-(40,50) = 33.53                      │
│    ...  (10 more pairs)                         │
│                                                 │
│  Answer: (2,3)-(3,4), dist = 1.4142             │
│  Time: O(n²) — checks every pair                │
└─────────────────────────────────────────────────┘
```

---

## Example 2: Divide & Conquer O(n log n)

```go
package main

import (
	"fmt"
	"math"
	"sort"
)

type Point struct{ X, Y float64 }

func dist(a, b Point) float64 {
	dx, dy := a.X-b.X, a.Y-b.Y
	return math.Sqrt(dx*dx + dy*dy)
}

func closestStrip(strip []Point, delta float64) float64 {
	// Sort strip by Y coordinate
	sort.Slice(strip, func(i, j int) bool { return strip[i].Y < strip[j].Y })

	minDist := delta
	for i := 0; i < len(strip); i++ {
		// Only check next ~7 points (proven sufficient)
		for j := i + 1; j < len(strip) && strip[j].Y-strip[i].Y < minDist; j++ {
			d := dist(strip[i], strip[j])
			if d < minDist { minDist = d }
		}
	}
	return minDist
}

func closestDC(points []Point) float64 {
	n := len(points)
	if n <= 3 {
		minD := math.Inf(1)
		for i := 0; i < n; i++ {
			for j := i + 1; j < n; j++ {
				d := dist(points[i], points[j])
				if d < minD { minD = d }
			}
		}
		return minD
	}

	mid := n / 2
	midPoint := points[mid]

	dl := closestDC(points[:mid])
	dr := closestDC(points[mid:])
	delta := math.Min(dl, dr)

	// Build strip: points within delta of midline
	var strip []Point
	for _, p := range points {
		if math.Abs(p.X-midPoint.X) < delta {
			strip = append(strip, p)
		}
	}

	return math.Min(delta, closestStrip(strip, delta))
}

func ClosestPair(points []Point) float64 {
	// Sort by X coordinate once
	sort.Slice(points, func(i, j int) bool { return points[i].X < points[j].X })
	return closestDC(points)
}

func main() {
	points := []Point{{2, 3}, {12, 30}, {40, 50}, {5, 1}, {12, 10}, {3, 4}}

	d := ClosestPair(points)
	fmt.Printf("Closest pair distance: %.4f\n", d)
	fmt.Println("\nD&C: O(n log n) with O(n log n) sort + O(n) strip check")
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Divide & Conquer Closest Pair                  │
├─────────────────────────────────────────────────┤
│  Sorted by X: (2,3)(3,4)(5,1)(12,10)(12,30)(40,50)│
│                                                 │
│  ┌── Left ──┐ mid ┌─── Right ───┐              │
│  (2,3)(3,4)(5,1)  (12,10)(12,30)(40,50)        │
│     δL=1.41          δR=20.0                    │
│                                                 │
│  δ = min(1.41, 20.0) = 1.41                     │
│                                                 │
│  Strip: points within 1.41 of midline x=5       │
│    → (3,4)(5,1) — check pairs in strip          │
│    → no pair < 1.41 in strip                    │
│                                                 │
│  Final: δ = 1.4142  → (2,3)-(3,4)              │
└─────────────────────────────────────────────────┘
```

---

## Example 3: Closest Pair with Pair Tracking

```go
package main

import (
	"fmt"
	"math"
	"sort"
)

type Point struct{ X, Y float64 }

func dist(a, b Point) float64 {
	dx, dy := a.X-b.X, a.Y-b.Y
	return math.Sqrt(dx*dx + dy*dy)
}

type Result struct {
	dist float64
	p1, p2 Point
}

func closestPairDC(pts []Point) Result {
	n := len(pts)
	if n <= 3 {
		best := Result{dist: math.Inf(1)}
		for i := 0; i < n; i++ {
			for j := i + 1; j < n; j++ {
				d := dist(pts[i], pts[j])
				if d < best.dist { best = Result{d, pts[i], pts[j]} }
			}
		}
		return best
	}

	mid := n / 2
	midX := pts[mid].X

	leftResult := closestPairDC(pts[:mid])
	rightResult := closestPairDC(pts[mid:])

	best := leftResult
	if rightResult.dist < best.dist { best = rightResult }

	// Strip
	var strip []Point
	for _, p := range pts {
		if math.Abs(p.X-midX) < best.dist { strip = append(strip, p) }
	}
	sort.Slice(strip, func(i, j int) bool { return strip[i].Y < strip[j].Y })

	for i := 0; i < len(strip); i++ {
		for j := i + 1; j < len(strip) && strip[j].Y-strip[i].Y < best.dist; j++ {
			d := dist(strip[i], strip[j])
			if d < best.dist { best = Result{d, strip[i], strip[j]} }
		}
	}
	return best
}

func main() {
	points := []Point{
		{0, 0}, {5, 5}, {1, 1}, {10, 10},
		{3, 3}, {7, 7}, {2, 1.5}, {8, 9},
	}
	sort.Slice(points, func(i, j int) bool { return points[i].X < points[j].X })

	result := closestPairDC(points)
	fmt.Printf("Closest pair: (%.1f, %.1f) - (%.1f, %.1f)\n", result.p1.X, result.p1.Y, result.p2.X, result.p2.Y)
	fmt.Printf("Distance: %.4f\n", result.dist)
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Closest Pair with Result Tracking              │
├─────────────────────────────────────────────────┤
│  Points sorted by X:                            │
│  (0,0)(1,1)(2,1.5)(3,3)(5,5)(7,7)(8,9)(10,10)  │
│                                                 │
│  Recursive trace:                               │
│    Left [0..3]: best=(1,1)-(2,1.5) d=1.12      │
│    Right[4..7]: best=(7,7)-(8,9)   d=2.24      │
│    δ = min(1.12, 2.24) = 1.12                   │
│                                                 │
│  Strip near x=3:                                │
│    Check cross-boundary pairs near midline      │
│    (2,1.5)-(3,3) = 1.80 > 1.12, skip           │
│                                                 │
│  Returns: Result{1.12, (1,1), (2,1.5)}          │
└─────────────────────────────────────────────────┘
```

---

## Example 4: Closest Pair in 1D (Simpler Case)

```go
package main

import (
	"fmt"
	"math"
	"sort"
)

// In 1D, closest pair = sort + check adjacent

func closestPair1D(arr []int) (int, int, int) {
	sorted := append([]int{}, arr...)
	sort.Ints(sorted)

	minDiff := math.MaxInt64
	p1, p2 := 0, 0
	for i := 1; i < len(sorted); i++ {
		diff := sorted[i] - sorted[i-1]
		if diff < minDiff {
			minDiff = diff
			p1, p2 = sorted[i-1], sorted[i]
		}
	}
	return minDiff, p1, p2
}

func main() {
	arr := []int{7, 1, 4, 9, 2, 15, 3}
	diff, p1, p2 := closestPair1D(arr)
	fmt.Printf("Array: %v\n", arr)
	fmt.Printf("Closest pair: %d and %d (diff=%d)\n", p1, p2, diff)

	fmt.Println("\n1D: sort + adjacent check = O(n log n)")
	fmt.Println("2D: D&C with strip check = O(n log n)")
	fmt.Println("3D+: same D&C extends but gets more complex")
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  1D Closest Pair: Sort + Adjacent Check         │
├─────────────────────────────────────────────────┤
│  arr = [7, 1, 4, 9, 2, 15, 3]                  │
│                                                 │
│  After sorting:                                 │
│  ┌───┬───┬───┬───┬───┬───┬───┐                 │
│  │ 1 │ 2 │ 3 │ 4 │ 7 │ 9 │15 │                 │
│  └───┴───┴───┴───┴───┴───┴───┘                 │
│    ↕1  ↕1  ↕1  ↕3  ↕2  ↕6   ← adjacent diffs  │
│                                                 │
│  min diff = 1 → pair (1,2) or (2,3) or (3,4)   │
│                                                 │
│  Key insight: in 1D, closest pair is always     │
│  adjacent after sorting → O(n log n)            │
└─────────────────────────────────────────────────┘
```

---

## Example 5: Closest Pair with Integer Coordinates

```go
package main

import (
	"fmt"
	"math"
	"sort"
)

// Integer version — uses squared distance to avoid floating point

type IPoint struct{ X, Y int }

func distSq(a, b IPoint) int64 {
	dx, dy := int64(a.X-b.X), int64(a.Y-b.Y)
	return dx*dx + dy*dy
}

func closestPairInt(pts []IPoint) (int64, IPoint, IPoint) {
	sort.Slice(pts, func(i, j int) bool {
		if pts[i].X != pts[j].X { return pts[i].X < pts[j].X }
		return pts[i].Y < pts[j].Y
	})
	return closestDCInt(pts)
}

func closestDCInt(pts []IPoint) (int64, IPoint, IPoint) {
	n := len(pts)
	if n <= 3 {
		best := int64(math.MaxInt64)
		var bp1, bp2 IPoint
		for i := 0; i < n; i++ {
			for j := i + 1; j < n; j++ {
				d := distSq(pts[i], pts[j])
				if d < best { best = d; bp1, bp2 = pts[i], pts[j] }
			}
		}
		return best, bp1, bp2
	}

	mid := n / 2
	midX := pts[mid].X
	bestD, bp1, bp2 := closestDCInt(pts[:mid])
	d2, p1, p2 := closestDCInt(pts[mid:])
	if d2 < bestD { bestD, bp1, bp2 = d2, p1, p2 }

	delta := int(math.Ceil(math.Sqrt(float64(bestD))))
	var strip []IPoint
	for _, p := range pts {
		if abs(p.X-midX) < delta { strip = append(strip, p) }
	}
	sort.Slice(strip, func(i, j int) bool { return strip[i].Y < strip[j].Y })

	for i := 0; i < len(strip); i++ {
		for j := i + 1; j < len(strip) && strip[j].Y-strip[i].Y < delta; j++ {
			d := distSq(strip[i], strip[j])
			if d < bestD { bestD, bp1, bp2 = d, strip[i], strip[j] }
		}
	}
	return bestD, bp1, bp2
}

func abs(x int) int { if x < 0 { return -x }; return x }

func main() {
	points := []IPoint{{0, 0}, {3, 4}, {1, 1}, {5, 2}, {7, 8}, {2, 1}}
	dSq, p1, p2 := closestPairInt(points)

	fmt.Printf("Points: %v\n", points)
	fmt.Printf("Closest: (%d,%d) and (%d,%d)\n", p1.X, p1.Y, p2.X, p2.Y)
	fmt.Printf("Distance² = %d, Distance = %.4f\n", dSq, math.Sqrt(float64(dSq)))
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Integer Coordinates — Squared Distance         │
├─────────────────────────────────────────────────┤
│  Points: (0,0)(3,4)(1,1)(5,2)(7,8)(2,1)        │
│                                                 │
│  Using distSq (avoids sqrt!):                   │
│    distSq((1,1),(2,1)) = (2-1)²+(1-1)² = 1     │
│    distSq((0,0),(1,1)) = 1²+1²         = 2     │
│    distSq((1,1),(3,4)) = 2²+3²         = 13    │
│                                                 │
│  Compare squared distances directly:            │
│    1 < 2 < 13 ...                               │
│                                                 │
│  Closest: (1,1)-(2,1)                           │
│    dist² = 1, dist = 1.0000                     │
│                                                 │
│  Tip: int64 for overflow safety on large coords │
└─────────────────────────────────────────────────┘
```

---

## Example 6: Why Strip Has at Most 7 Comparisons

```go
package main

import "fmt"

// Proof visualization: in a δ×2δ strip, at most 8 points can exist

func main() {
	fmt.Println("=== Why Strip Check is O(n) ===\n")

	fmt.Println("Key insight: within the strip of width 2δ centered on midline:")
	fmt.Println()
	fmt.Println("  +---------+---------+")
	fmt.Println("  |         |         |")
	fmt.Println("  |  Left   | Right   |  ← height = δ")
	fmt.Println("  |  half   |  half   |")
	fmt.Println("  +---------+---------+")
	fmt.Println("    ← δ →     ← δ →")
	fmt.Println()
	fmt.Println("Each δ × δ square can contain at most 4 points")
	fmt.Println("(otherwise two points within half would be < δ apart)")
	fmt.Println()
	fmt.Println("So in the δ × 2δ rectangle: at most 8 points")
	fmt.Println("For any point, check at most 7 others (sorted by Y)")
	fmt.Println()
	fmt.Println("This makes the strip check O(n) instead of O(n²)")
	fmt.Println()
	fmt.Println("Total: T(n) = 2T(n/2) + O(n log n) = O(n log²n)")
	fmt.Println("With pre-sorted Y: T(n) = 2T(n/2) + O(n) = O(n log n)")
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Strip Analysis: Why ≤7 Comparisons              │
├─────────────────────────────────────────────────┤
│  The δ×2δ rectangle around the midline:            │
│                                                 │
│   ┌─────────┬─────────┐                          │
│   │  ∙   ∙  │  ∙   ∙  │  4 points max per half │
│   │ L-half │ R-half │  (else d < δ in that   │
│   │  ∙   ∙  │  ∙   ∙  │   half — contradiction)  │
│   └─────────┴─────────┘                          │
│     ←─ δ ─→ ←─ δ ─→   ↑ height = δ            │
│                                                 │
│  At most 8 pts in δ×2δ box → 7 comparisons     │
│  Sort strip by Y, check only next 7 neighbors  │
│  Strip check total: O(7n) = O(n)                │
└─────────────────────────────────────────────────┘
```

---

## Example 7: Optimized Version with Pre-Sorted Y

```go
package main

import (
	"fmt"
	"math"
	"sort"
)

type Point struct{ X, Y float64 }

func dist(a, b Point) float64 {
	dx, dy := a.X-b.X, a.Y-b.Y
	return math.Sqrt(dx*dx + dy*dy)
}

// O(n log n): maintain Y-sorted order throughout recursion

func closestOptimal(points []Point) float64 {
	px := append([]Point{}, points...)
	sort.Slice(px, func(i, j int) bool { return px[i].X < px[j].X })

	py := append([]Point{}, points...)
	sort.Slice(py, func(i, j int) bool { return py[i].Y < py[j].Y })

	return closestRec(px, py)
}

func closestRec(px, py []Point) float64 {
	n := len(px)
	if n <= 3 {
		best := math.Inf(1)
		for i := 0; i < n; i++ {
			for j := i + 1; j < n; j++ {
				d := dist(px[i], px[j])
				if d < best { best = d }
			}
		}
		return best
	}

	mid := n / 2
	midPoint := px[mid]

	// Split py into left and right (maintaining Y-sort)
	leftSet := make(map[Point]bool)
	for _, p := range px[:mid] { leftSet[p] = true }

	var pyLeft, pyRight []Point
	for _, p := range py {
		if leftSet[p] { pyLeft = append(pyLeft, p) } else { pyRight = append(pyRight, p) }
	}

	dl := closestRec(px[:mid], pyLeft)
	dr := closestRec(px[mid:], pyRight)
	delta := math.Min(dl, dr)

	// Build strip from py (already Y-sorted!)
	var strip []Point
	for _, p := range py {
		if math.Abs(p.X-midPoint.X) < delta { strip = append(strip, p) }
	}

	// No need to re-sort strip — it's already sorted by Y from py
	for i := 0; i < len(strip); i++ {
		for j := i + 1; j < len(strip) && strip[j].Y-strip[i].Y < delta; j++ {
			d := dist(strip[i], strip[j])
			if d < delta { delta = d }
		}
	}
	return delta
}

func main() {
	points := []Point{
		{2, 3}, {12, 30}, {40, 50}, {5, 1},
		{12, 10}, {3, 4}, {1, 2}, {7, 8},
	}

	d := closestOptimal(points)
	fmt.Printf("Closest pair distance: %.4f\n", d)
	fmt.Println("\nOptimized: O(n log n) with pre-sorted Y")
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Optimized: Pre-Sorted Y Coordinates            │
├─────────────────────────────────────────────────┤
│  Naive: sort strip by Y each recursion          │
│    → O(n log n) per level → O(n log²n) total   │
│                                                 │
│  Optimized: maintain PY (Y-sorted) throughout   │
│    ┌───────────────────────────────────┐      │
│    │ PX (sorted by X)              │      │
│    │ PY (sorted by Y)              │      │
│    │                               │      │
│    │ Split PY into PYleft, PYright  │      │
│    │ using set membership in O(n)   │      │
│    └───────────────────────────────────┘      │
│    Strip from PY: already sorted by Y!         │
│    T(n) = 2T(n/2) + O(n) = O(n log n)          │
└─────────────────────────────────────────────────┘
```

---

## Example 8: Application — Nearest Neighbor Search

```go
package main

import (
	"fmt"
	"math"
	"sort"
)

type Point struct{ X, Y float64 }

func dist(a, b Point) float64 {
	dx, dy := a.X-b.X, a.Y-b.Y
	return math.Sqrt(dx*dx + dy*dy)
}

// Find nearest neighbor for a query point from a set of points
// Simple approach: sort by distance from query → O(n log n)
// Better for many queries: use KD-tree (not D&C topic)

func nearestNeighbor(points []Point, query Point) (Point, float64) {
	best := points[0]
	bestD := dist(points[0], query)
	for _, p := range points[1:] {
		d := dist(p, query)
		if d < bestD { bestD = d; best = p }
	}
	return best, bestD
}

// K nearest neighbors
func kNearestNeighbors(points []Point, query Point, k int) []Point {
	type pd struct {
		p Point
		d float64
	}
	dists := make([]pd, len(points))
	for i, p := range points { dists[i] = pd{p, dist(p, query)} }
	sort.Slice(dists, func(i, j int) bool { return dists[i].d < dists[j].d })

	result := make([]Point, k)
	for i := 0; i < k; i++ { result[i] = dists[i].p }
	return result
}

func main() {
	points := []Point{{1, 1}, {3, 4}, {5, 2}, {7, 8}, {2, 6}, {9, 3}}
	query := Point{4, 3}

	nn, d := nearestNeighbor(points, query)
	fmt.Printf("Query: (%.0f, %.0f)\n", query.X, query.Y)
	fmt.Printf("Nearest: (%.0f, %.0f), dist=%.2f\n\n", nn.X, nn.Y, d)

	knn := kNearestNeighbors(points, query, 3)
	fmt.Printf("3 nearest neighbors: %v\n", knn)
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Nearest Neighbor Search                        │
├─────────────────────────────────────────────────┤
│  Points: (1,1)(3,4)(5,2)(7,8)(2,6)(9,3)         │
│  Query: Q(4,3)                                  │
│                                                 │
│   8 │       ∙(7,8)                               │
│   6 │  ∙(2,6)                                    │
│   4 │    ∙(3,4)                                  │
│   3 │      Q───→1.41                              │
│   2 │        ∙(5,2)              ∙(9,3)          │
│   1 │ ∙(1,1)                                     │
│   0 ┴───────────────────────                     │
│                                                 │
│  NN: (3,4) at d=1.41                            │
│  3-NN: {(3,4),(5,2),(1,1)} by distance          │
└─────────────────────────────────────────────────┘
```

---

## Example 9: Closest Pair on Collinear Points

```go
package main

import (
	"fmt"
	"math"
	"sort"
)

// Special case: all points on a line
// Reduces to 1D closest pair

func closestCollinear(points []float64) (float64, float64, float64) {
	sort.Float64s(points)
	minDist := math.Inf(1)
	var p1, p2 float64
	for i := 1; i < len(points); i++ {
		d := points[i] - points[i-1]
		if d < minDist { minDist = d; p1, p2 = points[i-1], points[i] }
	}
	return minDist, p1, p2
}

func main() {
	// Points on y = 2x + 1
	xs := []float64{1, 5, 2, 8, 3, 7, 4}
	fmt.Println("X coordinates:", xs)

	d, p1, p2 := closestCollinear(xs)
	fmt.Printf("Closest along x-axis: %.0f and %.0f (dx=%.0f)\n", p1, p2, d)

	// Actual distance on line y=2x+1: d * sqrt(1 + slope²) = d * sqrt(5)
	slope := 2.0
	actualDist := d * math.Sqrt(1+slope*slope)
	fmt.Printf("Actual Euclidean distance: %.4f\n", actualDist)

	fmt.Println("\nCollinear case: O(n log n) by sorting")
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Collinear Points: y = 2x + 1                   │
├─────────────────────────────────────────────────┤
│  X coords: [1, 5, 2, 8, 3, 7, 4]               │
│                                                 │
│  Sorted: 1  2  3  4  5  7  8                    │
│          ∙──∙──∙──∙──∙────∙──∙  on number line  │
│          ↕1  ↕1  ↕1  ↕1  ↕2  ↕1                   │
│                                                 │
│  Min Δx = 1 → between consecutive points        │
│  Actual distance = Δx × √(1+slope²)             │
│                   = 1 × √(1+4) = √5 ≈ 2.236    │
│                                                 │
│  Reduces 2D → 1D: sort + adjacent check O(nlogn)│
└─────────────────────────────────────────────────┘
```

---

## Example 10: Complexity Analysis & Patterns

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Closest Pair Patterns & Complexity ===\n")

	fmt.Println("--- Algorithm Steps ---")
	steps := []string{
		"1. Sort points by X: O(n log n)",
		"2. (Optional) Pre-sort by Y: O(n log n)",
		"3. Divide: split at median X",
		"4. Conquer: solve left and right halves",
		"5. Combine: check strip of width 2δ",
		"   - Each point checks ≤ 7 neighbors",
		"   - Strip check is O(n)",
	}
	for _, s := range steps { fmt.Println("  " + s) }

	fmt.Println("\n--- Complexity ---")
	variants := []struct {
		variant, time string
	}{
		{"Brute force", "O(n²)"},
		{"D&C (sort strip each time)", "O(n log²n)"},
		{"D&C (pre-sorted Y)", "O(n log n)"},
		{"1D case (just sort)", "O(n log n)"},
		{"Randomized (grid hashing)", "O(n) expected"},
	}
	for _, v := range variants {
		fmt.Printf("  %-35s %s\n", v.variant, v.time)
	}

	fmt.Println("\n--- Related Problems ---")
	related := []string{
		"Nearest neighbor search → KD-tree",
		"K closest points to origin → Quick select",
		"Closest pair in 3D → Same D&C extends",
		"All nearest neighbors → Voronoi diagram",
	}
	for _, r := range related { fmt.Println("  • " + r) }
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Closest Pair: Complexity Comparison             │
├─────────────────────────────────────────────────┤
│  Approach            Time        Strip Sort      │
│  ────────────────────────────────────────         │
│  Brute force         O(n²)      N/A             │
│  D&C sort-each-time  O(nlog²n)  O(nlogn)/level  │
│  D&C pre-sorted Y   O(nlogn)   O(n)/level      │
│  Randomized grid     O(n) exp   N/A             │
│  1D (just sort)      O(nlogn)   N/A             │
│                                                 │
│  T(n) recurrence:                               │
│    T(n) = 2T(n/2) + [strip work per level]      │
│    O(nlogn) strip → O(nlog²n) total             │
│    O(n) strip     → O(nlogn)  total  ★          │
└─────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Strip check is O(n)**: each point checks ≤ 7 neighbors — the non-obvious key insight
2. **Sort by X once** at the start; maintain Y-order through recursion for optimal O(n log n)
3. **Integer distances**: use squared distance to avoid floating point errors
4. **1D reduces to sorting** + checking adjacent elements
5. **Extends to higher dimensions**: same D&C idea, but strip analysis gets more complex
6. **Practical**: randomized grid approach gives O(n) expected time

> **Next up:** Strassen's Matrix Multiplication →
