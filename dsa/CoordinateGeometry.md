# Phase 21: Math & Geometry — Coordinate Geometry

## Overview

Coordinate geometry solves problems involving points, lines, and shapes in 2D/3D space. Key operations include distance, area, convex hull, and intersection detection.

| Operation | Formula | Complexity |
|-----------|---------|------------|
| Distance | √((x₂-x₁)² + (y₂-y₁)²) | O(1) |
| Cross product | (b-a) × (c-a) | O(1) |
| Triangle area | ½|cross product| | O(1) |
| Polygon area | Shoelace formula | O(n) |
| Convex hull | Graham/Andrew's | O(n log n) |
| Point in polygon | Ray casting | O(n) |

---

## Example 1: Distance and Basic Operations

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

func distSq(a, b Point) float64 {
	dx, dy := a.X-b.X, a.Y-b.Y
	return dx*dx + dy*dy
}

func midpoint(a, b Point) Point {
	return Point{(a.X + b.X) / 2, (a.Y + b.Y) / 2}
}

func slope(a, b Point) float64 {
	if a.X == b.X { return math.Inf(1) }
	return (b.Y - a.Y) / (b.X - a.X)
}

func main() {
	a := Point{1, 2}
	b := Point{4, 6}

	fmt.Printf("A=%v, B=%v\n", a, b)
	fmt.Printf("  Distance:     %.4f\n", dist(a, b))
	fmt.Printf("  Distance²:    %.0f  (avoid sqrt)\n", distSq(a, b))
	fmt.Printf("  Midpoint:     %v\n", midpoint(a, b))
	fmt.Printf("  Slope:        %.4f\n", slope(a, b))
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Distance & Operations: A(1,2) B(4,6)            │
├─────────────────────────────────────────────────┤
│  Y                                               │
│  6 │            • B(4,6)                         │
│  5 │          /                                   │
│  4 │    M(2.5,4)   ← midpoint                    │
│  3 │      /                                       │
│  2 │ • A(1,2)                                    │
│  1 │                                               │
│    └──────────────── X                           │
│      1  2  3  4  5                                │
│                                                 │
│  dist = √((4-1)²+(6-2)²) = √(9+16) = 5.0       │
│  dist² = 25  (avoid sqrt when possible)          │
│  slope = (6-2)/(4-1) = 4/3 ≈ 1.3333             │
└─────────────────────────────────────────────────┘
```

---

## Example 2: Cross Product and Orientation

```go
package main

import "fmt"

type Point struct{ X, Y int }

// Cross product of vectors (OA × OB)
func cross(o, a, b Point) int {
	return (a.X-o.X)*(b.Y-o.Y) - (a.Y-o.Y)*(b.X-o.X)
}

// Returns: 0 = collinear, 1 = clockwise, 2 = counterclockwise
func orientation(p, q, r Point) int {
	v := cross(p, q, r)
	if v == 0 { return 0 }
	if v > 0 { return 2 } // counterclockwise
	return 1               // clockwise
}

func main() {
	tests := []struct {
		p, q, r Point
	}{
		{Point{0, 0}, Point{4, 4}, Point{1, 2}},  // CCW (left turn)
		{Point{0, 0}, Point{4, 4}, Point{2, 1}},  // CW (right turn)
		{Point{0, 0}, Point{2, 2}, Point{4, 4}},  // Collinear
	}

	names := []string{"Collinear", "Clockwise", "Counterclockwise"}
	for _, t := range tests {
		o := orientation(t.p, t.q, t.r)
		fmt.Printf("  P%v → Q%v → R%v: %s (cross=%d)\n",
			t.p, t.q, t.r, names[o], cross(t.p, t.q, t.r))
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Cross Product & Orientation                     │
├─────────────────────────────────────────────────┤
│  cross(P,Q,R) = (Q-P) × (R-P)                     │
│                                                 │
│   R(1,2)         > 0: CCW (left turn)           │
│    ∙              cross = (4-0)(2-0)-(4-0)(1-0) │
│  P ─────── Q             = 8-4 = 4 > 0          │
│  (0,0)   (4,4)                                  │
│                                                 │
│  P ─────── Q     < 0: CW (right turn)           │
│  (0,0)   (4,4)   cross = (4-0)(1-0)-(4-0)(2-0) │
│    ∙                    = 4-8 = -4 < 0          │
│   R(2,1)                                        │
│                                                 │
│  P ──── Q ──── R  = 0: Collinear               │
│  (0,0) (2,2) (4,4)                              │
└─────────────────────────────────────────────────┘
```

---

## Example 3: Triangle Area

```go
package main

import (
	"fmt"
	"math"
)

type Point struct{ X, Y float64 }

// Signed area * 2
func signedArea2(a, b, c Point) float64 {
	return (b.X-a.X)*(c.Y-a.Y) - (c.X-a.X)*(b.Y-a.Y)
}

func triangleArea(a, b, c Point) float64 {
	return math.Abs(signedArea2(a, b, c)) / 2
}

func collinear(a, b, c Point) bool {
	return math.Abs(signedArea2(a, b, c)) < 1e-9
}

func main() {
	tests := []struct {
		a, b, c Point
	}{
		{Point{0, 0}, Point{4, 0}, Point{0, 3}},  // 3-4-5 right triangle
		{Point{1, 1}, Point{4, 1}, Point{4, 5}},
		{Point{0, 0}, Point{2, 2}, Point{4, 4}},  // collinear
	}

	for _, t := range tests {
		area := triangleArea(t.a, t.b, t.c)
		fmt.Printf("  Triangle %v-%v-%v: area=%.2f, collinear=%v\n",
			t.a, t.b, t.c, area, collinear(t.a, t.b, t.c))
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Triangle Area via Cross Product                 │
├─────────────────────────────────────────────────┤
│  3-4-5 right triangle: (0,0)(4,0)(0,3)           │
│                                                 │
│  Y                                               │
│  3 │ C(0,3)                                     │
│    │ │╲                                          │
│  2 │ │  ╲                                        │
│    │ │   ╲                                       │
│  1 │ │    ╲                                      │
│    │ │     ╲                                     │
│  0 │ A─────B(4,0)                                │
│    └────────── X                               │
│                                                 │
│  Area = ½|cross| = ½|(4)(3) - (0)(0)| = 6.0     │
│  Collinear check: cross == 0 means area = 0     │
└─────────────────────────────────────────────────┘
```

---

## Example 4: Shoelace Formula (Polygon Area)

```go
package main

import (
	"fmt"
	"math"
)

type Point struct{ X, Y float64 }

func polygonArea(pts []Point) float64 {
	n := len(pts)
	area := 0.0
	for i := 0; i < n; i++ {
		j := (i + 1) % n
		area += pts[i].X * pts[j].Y
		area -= pts[j].X * pts[i].Y
	}
	return math.Abs(area) / 2
}

func main() {
	// Square
	square := []Point{{0, 0}, {4, 0}, {4, 4}, {0, 4}}
	fmt.Printf("  Square area:   %.1f\n", polygonArea(square))

	// Triangle
	tri := []Point{{0, 0}, {6, 0}, {3, 4}}
	fmt.Printf("  Triangle area: %.1f\n", polygonArea(tri))

	// Pentagon
	pent := []Point{{1, 0}, {3, 0}, {4, 2}, {2, 4}, {0, 2}}
	fmt.Printf("  Pentagon area: %.1f\n", polygonArea(pent))

	// Irregular
	irreg := []Point{{0, 0}, {5, 0}, {5, 3}, {3, 5}, {0, 3}}
	fmt.Printf("  Irregular area: %.1f\n", polygonArea(irreg))
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Shoelace Formula: Polygon Area                  │
├─────────────────────────────────────────────────┤
│  Square (0,0)(4,0)(4,4)(0,4):                    │
│  (0,4)──────(4,4)                                │
│    │          │                                    │
│    │          │    A = ½|Σ(xᵢyᵢ₊₁ - xᵢ₊₁yᵢ)|    │
│    │          │                                    │
│  (0,0)──────(4,0)                                │
│                                                 │
│  = ½|0×0-4×0 + 4×4-4×0 + 4×4-0×4 + 0×0-0×4| │
│  = ½|0 + 16 + 16 + 0| = ½×32 = 16.0             │
│                                                 │
│  Triangle (0,0)(6,0)(3,4): area = 12.0           │
│  Works for any simple polygon, O(n)              │
└─────────────────────────────────────────────────┘
```

---

## Example 5: Convex Hull (Andrew's Monotone Chain)

```go
package main

import (
	"fmt"
	"sort"
)

type Point struct{ X, Y int }

func cross(o, a, b Point) int {
	return (a.X-o.X)*(b.Y-o.Y) - (a.Y-o.Y)*(b.X-o.X)
}

func convexHull(points []Point) []Point {
	n := len(points)
	if n < 3 { return points }

	sort.Slice(points, func(i, j int) bool {
		if points[i].X != points[j].X { return points[i].X < points[j].X }
		return points[i].Y < points[j].Y
	})

	// Build lower hull
	lower := []Point{}
	for _, p := range points {
		for len(lower) >= 2 && cross(lower[len(lower)-2], lower[len(lower)-1], p) <= 0 {
			lower = lower[:len(lower)-1]
		}
		lower = append(lower, p)
	}

	// Build upper hull
	upper := []Point{}
	for i := n - 1; i >= 0; i-- {
		p := points[i]
		for len(upper) >= 2 && cross(upper[len(upper)-2], upper[len(upper)-1], p) <= 0 {
			upper = upper[:len(upper)-1]
		}
		upper = append(upper, p)
	}

	// Remove last point of each (it's repeated)
	return append(lower[:len(lower)-1], upper[:len(upper)-1]...)
}

func main() {
	points := []Point{
		{0, 0}, {1, 1}, {2, 2}, {4, 4}, {0, 4},
		{4, 0}, {2, 1}, {1, 3}, {3, 3},
	}

	hull := convexHull(points)
	fmt.Println("Points:", points)
	fmt.Println("Convex Hull:", hull)
	fmt.Printf("Hull size: %d out of %d points\n", len(hull), len(points))
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Convex Hull (Andrew's Monotone Chain)            │
├─────────────────────────────────────────────────┤
│  Y                                                │
│  4 │ •(0,4)────┬────(4,4)                       │
│    │ │       ∙(3,3)  │                            │
│  3 │ │   (1,3)       │                            │
│    │ │  ∙       ∙    │                            │
│  2 │ │      (2,2)    │  ← points inside hull      │
│    │ │    ∙ (2,1)   │     are excluded            │
│  1 │ │  (1,1)       │                            │
│    │ │             │                             │
│  0 │ •(0,0)────┴────(4,0)                       │
│    └─────────────────── X                      │
│                                                 │
│  Hull: (0,0)→(4,0)→(4,4)→(0,4) (4 of 9 pts)   │
│  Lower hull: sort by x, keep CCW turns only      │
│  Upper hull: reverse, same rule                  │
└─────────────────────────────────────────────────┘
```

---

## Example 6: Line Segment Intersection

```go
package main

import "fmt"

type Point struct{ X, Y int }

func cross(o, a, b Point) int {
	return (a.X-o.X)*(b.Y-o.Y) - (a.Y-o.Y)*(b.X-o.X)
}

func onSegment(p, q, r Point) bool {
	return min(p.X, r.X) <= q.X && q.X <= max(p.X, r.X) &&
		min(p.Y, r.Y) <= q.Y && q.Y <= max(p.Y, r.Y)
}

func segmentsIntersect(p1, q1, p2, q2 Point) bool {
	d1 := cross(p2, q2, p1)
	d2 := cross(p2, q2, q1)
	d3 := cross(p1, q1, p2)
	d4 := cross(p1, q1, q2)

	if ((d1 > 0 && d2 < 0) || (d1 < 0 && d2 > 0)) &&
		((d3 > 0 && d4 < 0) || (d3 < 0 && d4 > 0)) {
		return true
	}

	if d1 == 0 && onSegment(p2, p1, q2) { return true }
	if d2 == 0 && onSegment(p2, q1, q2) { return true }
	if d3 == 0 && onSegment(p1, p2, q1) { return true }
	if d4 == 0 && onSegment(p1, q2, q1) { return true }

	return false
}

func min(a, b int) int { if a < b { return a }; return b }
func max(a, b int) int { if a > b { return a }; return b }

func main() {
	tests := []struct {
		p1, q1, p2, q2 Point
		desc           string
	}{
		{Point{1, 1}, Point{10, 1}, Point{1, 2}, Point{10, 2}, "Parallel"},
		{Point{10, 0}, Point{0, 10}, Point{0, 0}, Point{10, 10}, "X-shape"},
		{Point{-5, -5}, Point{0, 0}, Point{1, 1}, Point{10, 10}, "Collinear no overlap"},
		{Point{0, 0}, Point{5, 5}, Point{3, 3}, Point{10, 10}, "Collinear overlap"},
	}

	for _, t := range tests {
		result := segmentsIntersect(t.p1, t.q1, t.p2, t.q2)
		fmt.Printf("  %-25s: intersect=%v\n", t.desc, result)
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Line Segment Intersection Tests                 │
├─────────────────────────────────────────────────┤
│  Parallel (no intersect):                        │
│    ──────────  seg1                              │
│    ──────────  seg2                              │
│                                                 │
│  X-shape (intersect ✓):                          │
│       ╲ /                                        │
│        X    cross products have opposite signs   │
│       / ╲                                        │
│                                                 │
│  Method: check orientation of 4 triples          │
│    d1=cross(p2,q2,p1)  d2=cross(p2,q2,q1)       │
│    d3=cross(p1,q1,p2)  d4=cross(p1,q1,q2)       │
│    Intersect if (d1,d2) diff sign & (d3,d4) diff │
└─────────────────────────────────────────────────┘
```

---

## Example 7: Point in Polygon (Ray Casting)

```go
package main

import "fmt"

type Point struct{ X, Y float64 }

func pointInPolygon(p Point, poly []Point) bool {
	n := len(poly)
	inside := false

	j := n - 1
	for i := 0; i < n; i++ {
		if (poly[i].Y > p.Y) != (poly[j].Y > p.Y) &&
			p.X < (poly[j].X-poly[i].X)*(p.Y-poly[i].Y)/(poly[j].Y-poly[i].Y)+poly[i].X {
			inside = !inside
		}
		j = i
	}
	return inside
}

func main() {
	polygon := []Point{{0, 0}, {10, 0}, {10, 10}, {0, 10}}

	tests := []Point{
		{5, 5},     // inside
		{11, 5},    // outside
		{0, 0},     // vertex
		{5, 0},     // edge
		{-1, -1},   // outside
	}

	for _, p := range tests {
		fmt.Printf("  Point (%.0f,%.0f): inside=%v\n", p.X, p.Y, pointInPolygon(p, polygon))
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Point in Polygon: Ray Casting                   │
├─────────────────────────────────────────────────┤
│  Polygon: (0,0)(10,0)(10,10)(0,10) — square      │
│                                                 │
│  (0,10)────────(10,10)                           │
│    │              │                                │
│    │   (5,5)∙─────────→ ray hits 1 edge = inside  │
│    │              │                                │
│    │              │  (11,5)∙──→ ray hits 0 = out  │
│    │              │                                │
│  (0,0)─────────(10,0)                            │
│                                                 │
│  Rule: cast horizontal ray → count crossings    │
│    odd crossings = inside, even = outside        │
└─────────────────────────────────────────────────┘
```

---

## Example 8: Closest Pair of Points

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

func closestPairBrute(pts []Point) float64 {
	minD := math.Inf(1)
	for i := 0; i < len(pts); i++ {
		for j := i + 1; j < len(pts); j++ {
			d := dist(pts[i], pts[j])
			if d < minD { minD = d }
		}
	}
	return minD
}

func closestStrip(strip []Point, d float64) float64 {
	sort.Slice(strip, func(i, j int) bool { return strip[i].Y < strip[j].Y })
	for i := 0; i < len(strip); i++ {
		for j := i + 1; j < len(strip) && strip[j].Y-strip[i].Y < d; j++ {
			dd := dist(strip[i], strip[j])
			if dd < d { d = dd }
		}
	}
	return d
}

func closestPair(pts []Point) float64 {
	n := len(pts)
	if n <= 3 { return closestPairBrute(pts) }

	mid := n / 2
	midX := pts[mid].X

	dl := closestPair(pts[:mid])
	dr := closestPair(pts[mid:])
	d := math.Min(dl, dr)

	strip := []Point{}
	for _, p := range pts {
		if math.Abs(p.X-midX) < d { strip = append(strip, p) }
	}
	return closestStrip(strip, d)
}

func main() {
	pts := []Point{{2, 3}, {12, 30}, {40, 50}, {5, 1}, {12, 10}, {3, 4}}
	sort.Slice(pts, func(i, j int) bool { return pts[i].X < pts[j].X })

	fmt.Printf("Points: %v\n", pts)
	fmt.Printf("Closest pair distance: %.4f\n", closestPair(pts))
	fmt.Printf("Brute force verify:    %.4f\n", closestPairBrute(pts))
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Closest Pair: Divide & Conquer                  │
├─────────────────────────────────────────────────┤
│    •(2,3) •(3,4)  │  •(5,1)      •(12,10)       │
│       ←───→       │                              │
│      d=1.41      │          •(12,30)           │
│     closest!     │                      •(40,50)│
│                   │                              │
│    Left half     midline    Right half           │
│                                                 │
│  1. Sort by X                                    │
│  2. Split at median                              │
│  3. Solve each half recursively                  │
│  4. Check strip (width 2δ) near midline          │
│  5. Each point checks ≤7 neighbors in strip      │
└─────────────────────────────────────────────────┘
```

---

## Example 9: Rotate Point / Reflect Point

```go
package main

import (
	"fmt"
	"math"
)

type Point struct{ X, Y float64 }

func rotateAroundOrigin(p Point, angle float64) Point {
	rad := angle * math.Pi / 180
	cos, sin := math.Cos(rad), math.Sin(rad)
	return Point{
		X: p.X*cos - p.Y*sin,
		Y: p.X*sin + p.Y*cos,
	}
}

func rotateAroundCenter(p, center Point, angle float64) Point {
	translated := Point{p.X - center.X, p.Y - center.Y}
	rotated := rotateAroundOrigin(translated, angle)
	return Point{rotated.X + center.X, rotated.Y + center.Y}
}

func reflectOverX(p Point) Point { return Point{p.X, -p.Y} }
func reflectOverY(p Point) Point { return Point{-p.X, p.Y} }
func reflectOverLine(p Point) Point { return Point{p.Y, p.X} } // y=x

func main() {
	p := Point{3, 4}
	fmt.Printf("Original: %v\n", p)
	fmt.Printf("  Rotate 90°:  (%.2f, %.2f)\n", rotateAroundOrigin(p, 90).X, rotateAroundOrigin(p, 90).Y)
	fmt.Printf("  Rotate 180°: (%.2f, %.2f)\n", rotateAroundOrigin(p, 180).X, rotateAroundOrigin(p, 180).Y)

	center := Point{1, 1}
	r := rotateAroundCenter(p, center, 90)
	fmt.Printf("  Rotate 90° around %v: (%.2f, %.2f)\n", center, r.X, r.Y)

	fmt.Printf("  Reflect over X-axis: %v\n", reflectOverX(p))
	fmt.Printf("  Reflect over Y-axis: %v\n", reflectOverY(p))
	fmt.Printf("  Reflect over y=x:    %v\n", reflectOverLine(p))
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Point Rotation & Reflection                     │
├─────────────────────────────────────────────────┤
│  P(3,4) rotated around origin:                   │
│                                                 │
│  Y      (-4,3)←─90°                              │
│  4 │      ∙    ∙ P(3,4)                          │
│  3 │                                             │
│  2 │                                             │
│  1 │                                             │
│  0 ┴─────────── X                              │
│ -3 │    ∙ (-3,-4)  ← 180°                       │
│ -4 │                                             │
│                                                 │
│  Rotation matrix: [cosθ  -sinθ]                │
│                   [sinθ   cosθ]                 │
│  Reflect X-axis: (3,-4)   Y-axis: (-3,4)        │
│  Reflect y=x: (4,3)  ← swap coordinates          │
└─────────────────────────────────────────────────┘
```

---

## Example 10: Coordinate Geometry Patterns

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Coordinate Geometry Patterns ===")
	fmt.Println()

	patterns := []struct{ technique, use, complexity string }{
		{"Euclidean distance", "Distance between points", "O(1)"},
		{"Cross product", "Orientation, area, turns", "O(1)"},
		{"Shoelace formula", "Polygon area from vertices", "O(n)"},
		{"Convex hull", "Smallest enclosing convex polygon", "O(n log n)"},
		{"Line intersection", "Check if segments cross", "O(1)"},
		{"Ray casting", "Point in polygon test", "O(n)"},
		{"Closest pair", "Minimum distance in point set", "O(n log n)"},
		{"Rotation matrix", "Rotate points by angle", "O(1)"},
		{"Manhattan distance", "|x1-x2| + |y1-y2|, grids", "O(1)"},
		{"Pick's theorem", "A = I + B/2 - 1 (lattice)", "O(n)"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-22s %-38s %s\n", p.technique, p.use, p.complexity)
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Coordinate Geometry Toolkit                     │
├─────────────────────────────────────────────────┤
│  Point ops     → distance, midpoint, slope O(1) │
│  Cross product → orientation, area         O(1) │
│  Shoelace      → polygon area              O(n) │
│  Convex hull   → enclosing polygon     O(nlogn) │
│  Intersection  → segment crossing          O(1) │
│  Ray casting   → point-in-polygon          O(n) │
│  Closest pair  → min distance D&C      O(nlogn) │
│  Rotation      → transform by angle        O(1) │
│                                                 │
│  Tip: use squared distances to avoid floats!    │
└─────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Cross product** determines turn direction (CW/CCW/collinear)
2. **Shoelace formula** computes polygon area in O(n)
3. **Convex hull** via Andrew's monotone chain — O(n log n)
4. **Closest pair** uses divide-and-conquer — O(n log n)
5. **Use squared distances** when possible to avoid floating-point issues

> **Next up:** Matrix Multiplication →
