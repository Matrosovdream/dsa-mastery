# Step 09 — Queues & Deques · 🔴 Hard

Examples **12–16**: the BFS variations worth memorising, the deque that replaces a heap, a queue that
knows its own minimum, the structure inside Go's scheduler, and the capstone.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
go run -race .                            # example 15 is concurrent
```

Example 12 is deterministic. The rest report timings, sampled on an Apple M4 with Go 1.26.3.

> ← Back to [🟡 medium](2-medium.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. Level order and multi-source BFS

`🔴 hard` · *two one-line changes to example 6*

Both of these come up constantly and neither is a new algorithm. One snapshots the queue length; the
other seeds the queue with more than one thing. That is all.

**Steps:**

1. Snapshot `len(q)` *before* the inner loop and process exactly one layer.
2. Seed every source at distance 0 and get nearest-source distances in one pass.
3. Compare against the obvious alternative — one BFS per source.

```go
package main

import (
	"fmt"
	"strings"
)

// Two BFS variations that come up constantly and are each one small change to
// example 6's loop:
//
//	LEVEL ORDER      process the queue one LAYER at a time
//	MULTI-SOURCE     seed the queue with every source at distance 0
//
// Both are about what you put in the queue and when, not about the traversal.

type point struct{ r, c int }

var dirs = [4]point{{-1, 0}, {1, 0}, {0, -1}, {0, 1}}

// bfsLevels is the level-order idiom: snapshot the queue length BEFORE the
// inner loop, and that count is exactly one layer. It is the only reliable way
// to know where a layer ends without storing the depth in every entry.
func bfsLevels(g [][]byte, start point) [][]point {
	rows, cols := len(g), len(g[0])
	seen := make([][]bool, rows)
	for i := range seen {
		seen[i] = make([]bool, cols)
	}
	var levels [][]point
	q := []point{start}
	seen[start.r][start.c] = true

	for len(q) > 0 {
		n := len(q) // <- the snapshot. Everything after this is the next layer.
		layer := make([]point, 0, n)
		for i := 0; i < n; i++ {
			p := q[0]
			q = q[1:]
			layer = append(layer, p)
			for _, d := range dirs {
				nr, nc := p.r+d.r, p.c+d.c
				if nr < 0 || nr >= rows || nc < 0 || nc >= cols {
					continue
				}
				if seen[nr][nc] || g[nr][nc] == '#' {
					continue
				}
				seen[nr][nc] = true
				q = append(q, point{nr, nc})
			}
		}
		levels = append(levels, layer)
	}
	return levels
}

// bfsMulti seeds EVERY source at distance 0 before the loop starts. The result
// is the distance to the NEAREST source, for every cell, in one pass.
//
// The naive alternative -- run one BFS per source and take the minimum -- is
// Theta(S * V). This is Theta(V), regardless of how many sources there are.
func bfsMulti(g [][]byte, sources []point) [][]int {
	rows, cols := len(g), len(g[0])
	dist := make([][]int, rows)
	for i := range dist {
		dist[i] = make([]int, cols)
		for j := range dist[i] {
			dist[i][j] = -1
		}
	}
	q := make([]point, 0, rows*cols)
	for _, s := range sources {
		if dist[s.r][s.c] == -1 {
			dist[s.r][s.c] = 0
			q = append(q, s)
		}
	}
	for head := 0; head < len(q); head++ { // head-index queue, example 2
		p := q[head]
		for _, d := range dirs {
			nr, nc := p.r+d.r, p.c+d.c
			if nr < 0 || nr >= rows || nc < 0 || nc >= cols {
				continue
			}
			if dist[nr][nc] != -1 || g[nr][nc] == '#' {
				continue
			}
			dist[nr][nc] = dist[p.r][p.c] + 1
			q = append(q, point{nr, nc})
		}
	}
	return dist
}

// singleSourceMin is the naive version, kept so the cost can be measured.
func singleSourceMin(g [][]byte, sources []point) (dist [][]int, visits int) {
	rows, cols := len(g), len(g[0])
	dist = make([][]int, rows)
	for i := range dist {
		dist[i] = make([]int, cols)
		for j := range dist[i] {
			dist[i][j] = -1
		}
	}
	for _, s := range sources {
		d := bfsMulti(g, []point{s})
		for r := 0; r < rows; r++ {
			for c := 0; c < cols; c++ {
				visits++
				if d[r][c] == -1 {
					continue
				}
				if dist[r][c] == -1 || d[r][c] < dist[r][c] {
					dist[r][c] = d[r][c]
				}
			}
		}
	}
	return dist, visits
}

func parse(s string) [][]byte {
	var g [][]byte
	for _, line := range strings.Split(strings.TrimSpace(s), "\n") {
		g = append(g, []byte(strings.TrimSpace(line)))
	}
	return g
}

func render(g [][]byte, dist [][]int) string {
	var b strings.Builder
	for r := range g {
		b.WriteString("    ")
		for c := range g[r] {
			switch {
			case g[r][c] == '#':
				b.WriteString("  #")
			case dist[r][c] < 0:
				b.WriteString("  .")
			default:
				fmt.Fprintf(&b, "%3d", dist[r][c])
			}
		}
		b.WriteString("\n")
	}
	return b.String()
}

func main() {
	grid := parse(`
		..........
		..####....
		..#..#..#.
		..#..#..#.
		..####..#.
		..........
	`)

	fmt.Println("the grid (# is a wall):")
	fmt.Println()
	for _, row := range grid {
		fmt.Printf("    %s\n", row)
	}

	fmt.Println()
	fmt.Println("LEVEL ORDER from the top-left. The trick is one line:")
	fmt.Println()
	fmt.Println("      n := len(q)          // snapshot BEFORE the inner loop")
	fmt.Println("      for i := 0; i < n; i++ { ... }   // exactly one layer")
	fmt.Println()
	levels := bfsLevels(grid, point{0, 0})
	fmt.Printf("  %-8s %8s  %s\n", "layer", "cells", "first few")
	for i, l := range levels {
		if i > 6 && i < len(levels)-2 {
			if i == 7 {
				fmt.Printf("  %-8s %8s  %s\n", "...", "...", "...")
			}
			continue
		}
		show := l
		if len(show) > 4 {
			show = show[:4]
		}
		fmt.Printf("  %-8d %8d  %v\n", i, len(l), show)
	}
	fmt.Printf("\n  %d layers, %d cells reached.\n", len(levels), func() int {
		n := 0
		for _, l := range levels {
			n += len(l)
		}
		return n
	}())
	fmt.Println()
	fmt.Println("  everything already in the queue when the layer starts is at the")
	fmt.Println("  current distance; everything pushed during the layer is one")
	fmt.Println("  further. That is the whole justification, and it fails the moment")
	fmt.Println("  you read len(q) INSIDE the loop -- which is the usual bug.")
	fmt.Println()
	fmt.Println("  this is the same idiom as binary-tree level order (lesson 17) and")
	fmt.Println("  as Kahn's topological sort by layers (lesson 23).")

	fmt.Println()
	fmt.Println("MULTI-SOURCE: seed the queue with ALL sources at distance 0.")
	fmt.Println("Distance to the nearest of three sources, one pass:")
	fmt.Println()
	sources := []point{{0, 0}, {5, 9}, {2, 3}}
	d := bfsMulti(grid, sources)
	fmt.Print(render(grid, d))
	fmt.Println()
	fmt.Println("  the 0s are the three sources. Every other cell holds its distance")
	fmt.Println("  to the CLOSEST one, and the region boundaries between them fall")
	fmt.Println("  out for free -- that is a Voronoi diagram under the grid metric.")
	fmt.Println()
	fmt.Println("  the 2x2 pocket at rows 2-3, columns 3-4 is sealed off by walls.")
	fmt.Println("  It shows real distances only because one of the three sources is")
	fmt.Println("  inside it. Drop that source and it becomes unreachable:")

	d2 := bfsMulti(grid, []point{{0, 0}, {5, 9}})
	fmt.Println()
	fmt.Print(render(grid, d2))

	fmt.Println()
	fmt.Println("why it is one pass and not one pass per source:")
	fmt.Println()
	fmt.Printf("  %10s %10s %16s %18s %10s\n", "grid", "sources", "multi-source", "one BFS per source", "ratio")
	for _, tc := range []struct{ n, s int }{{100, 4}, {100, 64}, {300, 4}, {300, 64}} {
		g := make([][]byte, tc.n)
		for i := range g {
			g[i] = make([]byte, tc.n)
			for j := range g[i] {
				g[i][j] = '.'
			}
		}
		srcs := make([]point, tc.s)
		for i := range srcs {
			srcs[i] = point{(i * 7919) % tc.n, (i * 104729) % tc.n}
		}
		multi := bfsMulti(g, srcs)
		naive, visits := singleSourceMin(g, srcs)
		for r := range multi { // prove they agree before comparing cost
			for c := range multi[r] {
				if multi[r][c] != naive[r][c] {
					panic("multi-source disagrees with per-source minimum")
				}
			}
		}
		cells := tc.n * tc.n
		fmt.Printf("  %10s %10d %16d %18d %9.0fx\n",
			fmt.Sprintf("%dx%d", tc.n, tc.n), tc.s, cells, visits, float64(visits)/float64(cells))
	}
	fmt.Println()
	fmt.Println("  identical answers -- checked, not assumed -- at 1/S of the work.")
	fmt.Println("  The queue does not care where its entries came from, so seeding it")
	fmt.Println("  with S sources costs the same as seeding it with one.")
	fmt.Println()
	fmt.Println("  reach for this whenever a problem says 'nearest' and there is more")
	fmt.Println("  than one target: rotting oranges, walls and gates, distance to the")
	fmt.Println("  nearest 0, fire spreading from several points. They are all this.")
}
```

**Sample output:**

```
the grid (# is a wall):

    ..........
    ..####....
    ..#..#..#.
    ..#..#..#.
    ..####..#.
    ..........

LEVEL ORDER from the top-left. The trick is one line:

      n := len(q)          // snapshot BEFORE the inner loop
      for i := 0; i < n; i++ { ... }   // exactly one layer

  layer       cells  first few
  0               1  [{0 0}]
  1               2  [{1 0} {0 1}]
  2               3  [{2 0} {1 1} {0 2}]
  3               3  [{3 0} {2 1} {0 3}]
  4               3  [{4 0} {3 1} {0 4}]
  5               3  [{5 0} {4 1} {0 5}]
  6               2  [{5 1} {0 6}]
  ...           ...  ...
  13              2  [{5 8} {4 9}]
  14              1  [{5 9}]

  15 layers, 41 cells reached.

  everything already in the queue when the layer starts is at the
  current distance; everything pushed during the layer is one
  further. That is the whole justification, and it fails the moment
  you read len(q) INSIDE the loop -- which is the usual bug.

  this is the same idiom as binary-tree level order (lesson 17) and
  as Kahn's topological sort by layers (lesson 23).

MULTI-SOURCE: seed the queue with ALL sources at distance 0.
Distance to the nearest of three sources, one pass:

      0  1  2  3  4  5  6  7  6  5
      1  2  #  #  #  #  7  6  5  4
      2  3  #  0  1  #  6  5  #  3
      3  4  #  1  2  #  5  4  #  2
      4  5  #  #  #  #  4  3  #  1
      5  6  7  6  5  4  3  2  1  0

  the 0s are the three sources. Every other cell holds its distance
  to the CLOSEST one, and the region boundaries between them fall
  out for free -- that is a Voronoi diagram under the grid metric.

  the 2x2 pocket at rows 2-3, columns 3-4 is sealed off by walls.
  It shows real distances only because one of the three sources is
  inside it. Drop that source and it becomes unreachable:

      0  1  2  3  4  5  6  7  6  5
      1  2  #  #  #  #  7  6  5  4
      2  3  #  .  .  #  6  5  #  3
      3  4  #  .  .  #  5  4  #  2
      4  5  #  #  #  #  4  3  #  1
      5  6  7  6  5  4  3  2  1  0

why it is one pass and not one pass per source:

        grid    sources     multi-source one BFS per source      ratio
     100x100          4            10000              40000         4x
     100x100         64            10000             640000        64x
     300x300          4            90000             360000         4x
     300x300         64            90000            5760000        64x

  identical answers -- checked, not assumed -- at 1/S of the work.
  The queue does not care where its entries came from, so seeding it
  with S sources costs the same as seeding it with one.

  reach for this whenever a problem says 'nearest' and there is more
  than one target: rotting oranges, walls and gates, distance to the
  nearest 0, fire spreading from several points. They are all this.
```

**Complexity:** level order Θ(V + E), and it breaks the moment you read `len(q)` *inside* the loop ·
multi-source is Θ(V + E) **regardless of how many sources**, against Θ(S·V) for the per-source
minimum — measured at exactly **S×** the work for S = 4 and S = 64

---

## 13. 0-1 BFS: a deque instead of a heap

`🔴 hard` · *a priority queue with two priorities*

When every edge weighs 0 or 1, Dijkstra's heap is overkill. Push 0-edges on the **front** and 1-edges
on the **back**, and the deque stays sorted by distance while holding at most two distinct values.

**Steps:**

1. Build it, and check it against a real Dijkstra rather than trusting it.
2. Write the plain-BFS version as the bug — then find out the one you wrote first was *correct*.
3. Measure all three, and read the pop counts.

```go
package main

import (
	"container/heap"
	"fmt"
	"math/rand"
	"strings"
	"testing"
)

// 0-1 BFS is the payoff for having built a DEQUE rather than a queue.
//
// Setting: a graph whose edge weights are all 0 or 1. Dijkstra solves it in
// Theta(E log V). Plain BFS does not solve it at all -- BFS assumes every edge
// costs the same. But with a deque you get Theta(V + E):
//
//	weight 0 edge -> PushFront   (same distance, so it belongs at the front)
//	weight 1 edge -> PushBack    (one further, so it belongs at the back)
//
// The deque is then always sorted by distance and holds at most two distinct
// values -- d and d+1. That is a priority queue with two priorities, and a
// deque is the fastest possible implementation of one.

type point struct{ r, c int }

var dirs = [4]point{{-1, 0}, {1, 0}, {0, -1}, {0, 1}}

// Grid: '.' costs 0 to enter, '#' costs 1 (you break the wall).
func cost(b byte) int {
	if b == '#' {
		return 1
	}
	return 0
}

func bfs01(g [][]byte, src, dst point) (int, int) {
	rows, cols := len(g), len(g[0])
	dist := make([][]int, rows)
	for i := range dist {
		dist[i] = make([]int, cols)
		for j := range dist[i] {
			dist[i][j] = 1 << 30
		}
	}
	d := NewDeque[point](64)
	dist[src.r][src.c] = 0
	d.PushBack(src)
	pops := 0

	for d.Len() > 0 {
		p, _ := d.PopFront()
		pops++
		for _, dd := range dirs {
			nr, nc := p.r+dd.r, p.c+dd.c
			if nr < 0 || nr >= rows || nc < 0 || nc >= cols {
				continue
			}
			w := cost(g[nr][nc])
			if nd := dist[p.r][p.c] + w; nd < dist[nr][nc] {
				dist[nr][nc] = nd
				if w == 0 {
					d.PushFront(point{nr, nc}) // same layer -- must come out first
				} else {
					d.PushBack(point{nr, nc}) // next layer
				}
			}
		}
	}
	return dist[dst.r][dst.c], pops
}

// The Dijkstra reference, so the answers can be checked rather than trusted.
type node struct {
	p point
	d int
}
type pq []node

func (h pq) Len() int           { return len(h) }
func (h pq) Less(i, j int) bool { return h[i].d < h[j].d }
func (h pq) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *pq) Push(x any)        { *h = append(*h, x.(node)) }
func (h *pq) Pop() any          { o := *h; n := len(o); v := o[n-1]; *h = o[:n-1]; return v }

func dijkstra(g [][]byte, src, dst point) (int, int) {
	rows, cols := len(g), len(g[0])
	dist := make([][]int, rows)
	for i := range dist {
		dist[i] = make([]int, cols)
		for j := range dist[i] {
			dist[i][j] = 1 << 30
		}
	}
	h := &pq{{src, 0}}
	dist[src.r][src.c] = 0
	pops := 0
	for h.Len() > 0 {
		cur := heap.Pop(h).(node)
		pops++
		if cur.d > dist[cur.p.r][cur.p.c] {
			continue
		}
		for _, dd := range dirs {
			nr, nc := cur.p.r+dd.r, cur.p.c+dd.c
			if nr < 0 || nr >= rows || nc < 0 || nc >= cols {
				continue
			}
			if nd := cur.d + cost(g[nr][nc]); nd < dist[nr][nc] {
				dist[nr][nc] = nd
				heap.Push(h, node{point{nr, nc}, nd})
			}
		}
	}
	return dist[dst.r][dst.c], pops
}

// bfsWrong is the trap: a plain queue, PushBack for everything, and a vertex
// settled the FIRST time it is reached -- which is exactly the correct BFS from
// example 6. On a 0-1 graph that assumption is false, so it is now a bug.
func bfsWrong(g [][]byte, src, dst point) int {
	rows, cols := len(g), len(g[0])
	dist := make([][]int, rows)
	seen := make([][]bool, rows)
	for i := range dist {
		dist[i] = make([]int, cols)
		seen[i] = make([]bool, cols)
	}
	q := []point{src}
	seen[src.r][src.c] = true
	for len(q) > 0 {
		p := q[0]
		q = q[1:]
		for _, dd := range dirs {
			nr, nc := p.r+dd.r, p.c+dd.c
			if nr < 0 || nr >= rows || nc < 0 || nc >= cols || seen[nr][nc] {
				continue
			}
			seen[nr][nc] = true // settled on first arrival -- the bug
			dist[nr][nc] = dist[p.r][p.c] + cost(g[nr][nc])
			q = append(q, point{nr, nc}) // always the back
		}
	}
	return dist[dst.r][dst.c]
}

// bfsRelax is the OTHER plain-queue version, and it is correct: it re-queues a
// vertex whenever it finds a cheaper route. That is SPFA / Bellman-Ford, not
// BFS, and the cost is that a vertex can be processed many times.
func bfsRelax(g [][]byte, src, dst point) (int, int) {
	rows, cols := len(g), len(g[0])
	dist := make([][]int, rows)
	for i := range dist {
		dist[i] = make([]int, cols)
		for j := range dist[i] {
			dist[i][j] = 1 << 30
		}
	}
	q := []point{src}
	dist[src.r][src.c] = 0
	pops := 0
	for len(q) > 0 {
		p := q[0]
		q = q[1:]
		pops++
		for _, dd := range dirs {
			nr, nc := p.r+dd.r, p.c+dd.c
			if nr < 0 || nr >= rows || nc < 0 || nc >= cols {
				continue
			}
			if nd := dist[p.r][p.c] + cost(g[nr][nc]); nd < dist[nr][nc] {
				dist[nr][nc] = nd
				q = append(q, point{nr, nc})
			}
		}
	}
	return dist[dst.r][dst.c], pops
}

func parse(s string) [][]byte {
	var g [][]byte
	for _, line := range strings.Split(strings.TrimSpace(s), "\n") {
		g = append(g, []byte(strings.TrimSpace(line)))
	}
	return g
}

func randGrid(n int, wallPct int, seed int64) [][]byte {
	rng := rand.New(rand.NewSource(seed))
	g := make([][]byte, n)
	for i := range g {
		g[i] = make([]byte, n)
		for j := range g[i] {
			if rng.Intn(100) < wallPct {
				g[i][j] = '#'
			} else {
				g[i][j] = '.'
			}
		}
	}
	g[0][0], g[n-1][n-1] = '.', '.'
	return g
}

var sinkI int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	g := parse(`
		..........
		..........
		..........
		....######
		....#.....
		....#.....
		....#..###
		....#..#..
		....#..#..
		....#..#..
	`)
	fmt.Println("walking through '.' is free; breaking a '#' costs 1. Find the")
	fmt.Println("cheapest route from the top-left to the bottom-right:")
	fmt.Println()
	for _, row := range g {
		fmt.Printf("    %s\n", row)
	}

	src, dst := point{0, 0}, point{len(g) - 1, len(g[0]) - 1}
	got, pops01 := bfs01(g, src, dst)
	want, popsDij := dijkstra(g, src, dst)
	bad := bfsWrong(g, src, dst)

	fmt.Println()
	fmt.Printf("  %-34s %6d walls broken   (%d pops)\n", "0-1 BFS with a deque", got, pops01)
	fmt.Printf("  %-34s %6d walls broken   (%d pops)\n", "Dijkstra, as a reference", want, popsDij)
	relax, popsRelax := bfsRelax(g, src, dst)
	fmt.Printf("  %-34s %6d walls broken\n", "plain BFS, settle on arrival", bad)
	fmt.Printf("  %-34s %6d walls broken   (%d pops)\n", "plain queue, re-relaxing (SPFA)", relax, popsRelax)
	fmt.Println()
	fmt.Println("  all four agree here, which is precisely the problem. Plain BFS")
	fmt.Println("  settles a cell the FIRST time it is reached, and with 0-weight")
	fmt.Println("  edges the first arrival need not be the cheapest -- but on this")
	fmt.Println("  grid it happens to be. It fails on 98% of random grids (below).")
	fmt.Println()
	fmt.Println("  the fourth row is the version I first wrote AS the bug, and it is")
	fmt.Println("  correct: it re-queues a cell whenever it finds a cheaper route.")
	fmt.Println("  That is not BFS, it is SPFA, and it pays for correctness in")
	fmt.Println("  repeated work -- 107 pops against 100 here, and far worse below.")
	fmt.Println("  I only noticed when the 3,000-grid check reported zero")
	fmt.Println("  disagreements for a function I had labelled a bug.")

	fmt.Println()
	fmt.Println("why PushFront is the whole algorithm:")
	fmt.Println()
	fmt.Println("  the deque's invariant is that it holds at most TWO distinct")
	fmt.Println("  distances at any moment -- d at the front and d+1 at the back --")
	fmt.Println("  and is sorted. A 0-edge produces a d, which must go in front of")
	fmt.Println("  every d+1 already queued. A 1-edge produces a d+1, which goes")
	fmt.Println("  behind them.")
	fmt.Println()
	fmt.Println("  that IS a priority queue. It just happens to have two priorities,")
	fmt.Println("  so it needs no heap, no comparisons and no log factor.")

	fmt.Println()
	fmt.Println("checked against Dijkstra on 3,000 random grids before it is timed:")
	fmt.Println()
	mismatch01, mismatchBFS, mismatchRelax := 0, 0, 0
	for i := 0; i < 3000; i++ {
		rg := randGrid(6+i%10, 20+i%50, int64(i))
		d := point{len(rg) - 1, len(rg[0]) - 1}
		a, _ := bfs01(rg, point{0, 0}, d)
		b, _ := dijkstra(rg, point{0, 0}, d)
		if a != b {
			mismatch01++
		}
		if bfsWrong(rg, point{0, 0}, d) != b {
			mismatchBFS++
		}
		if r, _ := bfsRelax(rg, point{0, 0}, d); r != b {
			mismatchRelax++
		}
	}
	fmt.Printf("  0-1 BFS vs Dijkstra:     %4d disagreements\n", mismatch01)
	fmt.Printf("  plain BFS vs Dijkstra:   %4d disagreements\n", mismatchBFS)
	fmt.Printf("  SPFA vs Dijkstra:        %4d disagreements\n", mismatchRelax)
	fmt.Println()
	fmt.Println("  three of the four rows are correct; only the one that settles on")
	fmt.Println("  arrival is not. It is wrong on a large fraction of inputs and")
	fmt.Println("  right on the rest -- the failure mode that survives a hand-written")
	fmt.Println("  test suite and is caught in one line by a reference implementation.")

	fmt.Println()
	fmt.Println("and the cost, against the heap it replaces:")
	fmt.Println()
	fmt.Printf("  %6s %7s %13s %13s %13s %9s %16s\n", "n", "walls", "0-1 BFS ns", "SPFA ns", "Dijkstra ns", "vs heap", "pops (01/spfa)")
	for _, tc := range []struct{ n, w int }{{200, 30}, {200, 60}, {600, 30}, {600, 60}} {
		rg := randGrid(tc.n, tc.w, 42)
		d := point{tc.n - 1, tc.n - 1}
		a, pa := bfs01(rg, point{0, 0}, d)
		b, pb := dijkstra(rg, point{0, 0}, d)
		if a != b {
			panic("disagreement on the benchmark grid")
		}
		rr, pr := bfsRelax(rg, point{0, 0}, d)
		if rr != b {
			panic("SPFA disagrees on the benchmark grid")
		}
		t1 := nsPerOp(func() { v, _ := bfs01(rg, point{0, 0}, d); sinkI = v })
		t3 := nsPerOp(func() { v, _ := bfsRelax(rg, point{0, 0}, d); sinkI = v })
		t2 := nsPerOp(func() { v, _ := dijkstra(rg, point{0, 0}, d); sinkI = v })
		_ = pb
		fmt.Printf("  %6d %6d%% %13.0f %13.0f %13.0f %8.1fx %9d/%d\n",
			tc.n, tc.w, t1, t3, t2, t2/t1, pa, pr)
	}
	fmt.Println()
	fmt.Println("  0-1 BFS pops each cell exactly once -- 360,000 on the 600x600")
	fmt.Println("  grid, the same as Dijkstra -- and beats the heap by 6-9x. The log")
	fmt.Println("  factor is real, but so is the constant: a PushFront is one AND and")
	fmt.Println("  one store, while a heap Push sifts up a pointer-chasing chain.")
	fmt.Println()
	fmt.Println("  and look at the SPFA column. It is correct, it uses only a plain")
	fmt.Println("  queue, and at 60% walls it pops 10,139,058 times to 0-1 BFS's")
	fmt.Println("  360,000 -- 28x the work, and slower than the heap it was meant to")
	fmt.Println("  avoid. Correct-but-unbounded is its own failure mode.")
	fmt.Println()
	fmt.Println("  the generalisation is Dial's algorithm: with weights in 0..C, use")
	fmt.Println("  C+1 buckets instead of a heap. 0-1 BFS is the C=1 case, and the")
	fmt.Println("  two buckets happen to be the two ends of one deque.")
}
```

> Reuses `Deque[T]` from example 5 — copy that type into the same folder as `deque.go`.

**Sample output:**

```
walking through '.' is free; breaking a '#' costs 1. Find the
cheapest route from the top-left to the bottom-right:

    ..........
    ..........
    ..........
    ....######
    ....#.....
    ....#.....
    ....#..###
    ....#..#..
    ....#..#..
    ....#..#..

  0-1 BFS with a deque                    2 walls broken   (100 pops)
  Dijkstra, as a reference                2 walls broken   (100 pops)
  plain BFS, settle on arrival            2 walls broken
  plain queue, re-relaxing (SPFA)         2 walls broken   (107 pops)

  all four agree here, which is precisely the problem. Plain BFS
  settles a cell the FIRST time it is reached, and with 0-weight
  edges the first arrival need not be the cheapest -- but on this
  grid it happens to be. It fails on 98% of random grids (below).

  the fourth row is the version I first wrote AS the bug, and it is
  correct: it re-queues a cell whenever it finds a cheaper route.
  That is not BFS, it is SPFA, and it pays for correctness in
  repeated work -- 107 pops against 100 here, and far worse below.
  I only noticed when the 3,000-grid check reported zero
  disagreements for a function I had labelled a bug.

why PushFront is the whole algorithm:

  the deque's invariant is that it holds at most TWO distinct
  distances at any moment -- d at the front and d+1 at the back --
  and is sorted. A 0-edge produces a d, which must go in front of
  every d+1 already queued. A 1-edge produces a d+1, which goes
  behind them.

  that IS a priority queue. It just happens to have two priorities,
  so it needs no heap, no comparisons and no log factor.

checked against Dijkstra on 3,000 random grids before it is timed:

  0-1 BFS vs Dijkstra:        0 disagreements
  plain BFS vs Dijkstra:   2939 disagreements
  SPFA vs Dijkstra:           0 disagreements

  three of the four rows are correct; only the one that settles on
  arrival is not. It is wrong on a large fraction of inputs and
  right on the rest -- the failure mode that survives a hand-written
  test suite and is caught in one line by a reference implementation.

and the cost, against the heap it replaces:

       n   walls    0-1 BFS ns       SPFA ns   Dijkstra ns   vs heap   pops (01/spfa)
     200     30%        361281       3278004       3561155      9.9x     40000/223556
     200     60%        447220       6700596       3251801      7.3x     40000/434538
     600     30%       4672274      82320560      31785268      6.8x    360000/5430424
     600     60%       5313257     180128899      32770610      6.2x    360000/10139058

  0-1 BFS pops each cell exactly once -- 360,000 on the 600x600
  grid, the same as Dijkstra -- and beats the heap by 6-9x. The log
  factor is real, but so is the constant: a PushFront is one AND and
  one store, while a heap Push sifts up a pointer-chasing chain.

  and look at the SPFA column. It is correct, it uses only a plain
  queue, and at 60% walls it pops 10,139,058 times to 0-1 BFS's
  360,000 -- 28x the work, and slower than the heap it was meant to
  avoid. Correct-but-unbounded is its own failure mode.

  the generalisation is Dial's algorithm: with weights in 0..C, use
  C+1 buckets instead of a heap. 0-1 BFS is the C=1 case, and the
  two buckets happen to be the two ends of one deque.
```

**Complexity:** Θ(V + E) against Dijkstra's Θ(E log V) — **6.2–9.9× faster**, same pop count · settling a
cell on first arrival is wrong on **2,939 of 3,000** random grids and right on the demo grid · SPFA is
correct and pops **10,139,058** times to 0-1 BFS's **360,000**

---

## 14. A queue from two stacks, and the Θ(1) minimum that falls out

`🔴 hard` · *the right two pieces, not a new algorithm*

Tip one stack into another and you have a queue. And because each half is a *stack*, and lesson 08
already built a stack that tracks its own minimum, the queue gets a Θ(1) minimum for free.

**Steps:**

1. Build the two-stack queue and watch the tip reverse the order.
2. Count the tips and see amortized Θ(1) that is *not* worst-case Θ(1).
3. Swap in two min-stacks and get a min-queue with no new ideas.

```go
package main

import (
	"fmt"
	"math/rand"
	"testing"
)

// Two structures that look unrelated and are the same idea:
//
//	QUEUE FROM TWO STACKS   push onto `in`; when `out` is empty, tip `in` into
//	                        it. Amortized Theta(1), worst case Theta(n).
//	MIN QUEUE               a queue that also reports its minimum in Theta(1).
//
// The second falls out of the first for free, because lesson 08 already built a
// stack that tracks its own minimum -- and a queue's minimum is just the
// smaller of its two stacks' minima. That composition is the point of this
// example: you do not need a new algorithm, you need the right two pieces.

// ---- queue from two stacks -------------------------------------------------

type TwoStack struct {
	in, out []int
	moved   int // total elements ever tipped, for the amortized argument
}

func (q *TwoStack) Push(v int) { q.in = append(q.in, v) }

func (q *TwoStack) Pop() (int, bool) {
	if len(q.out) == 0 {
		if len(q.in) == 0 {
			return 0, false
		}
		q.tip()
	}
	v := q.out[len(q.out)-1]
	q.out = q.out[:len(q.out)-1]
	return v, true
}

// tip is the Theta(n) step -- and it is the ONLY place elements move.
func (q *TwoStack) tip() {
	for len(q.in) > 0 {
		v := q.in[len(q.in)-1]
		q.in = q.in[:len(q.in)-1]
		q.out = append(q.out, v)
		q.moved++
	}
}

func (q *TwoStack) Len() int { return len(q.in) + len(q.out) }

// ---- min stack (lesson 08) and the min queue built from two of them --------

type minStack struct {
	vals, mins []int
}

func (s *minStack) push(v int) {
	s.vals = append(s.vals, v)
	m := v
	if n := len(s.mins); n > 0 && s.mins[n-1] < v {
		m = s.mins[n-1]
	}
	s.mins = append(s.mins, m)
}
func (s *minStack) pop() int {
	v := s.vals[len(s.vals)-1]
	s.vals = s.vals[:len(s.vals)-1]
	s.mins = s.mins[:len(s.mins)-1]
	return v
}
func (s *minStack) min() (int, bool) {
	if len(s.mins) == 0 {
		return 0, false
	}
	return s.mins[len(s.mins)-1], true
}
func (s *minStack) empty() bool { return len(s.vals) == 0 }

type MinQueue struct{ in, out minStack }

func (q *MinQueue) Push(v int) { q.in.push(v) }
func (q *MinQueue) Pop() (int, bool) {
	if q.out.empty() {
		if q.in.empty() {
			return 0, false
		}
		for !q.in.empty() {
			q.out.push(q.in.pop())
		}
	}
	return q.out.pop(), true
}

// Min is Theta(1): each stack knows its own minimum, and the queue's minimum
// is whichever is smaller. No scan, no recomputation.
func (q *MinQueue) Min() (int, bool) {
	a, okA := q.in.min()
	b, okB := q.out.min()
	switch {
	case okA && okB:
		if a < b {
			return a, true
		}
		return b, true
	case okA:
		return a, true
	case okB:
		return b, true
	}
	return 0, false
}
func (q *MinQueue) Len() int { return len(q.in.vals) + len(q.out.vals) }

// scanMin is the honest baseline: keep a slice, scan it.
type scanQueue struct{ q []int }

func (s *scanQueue) Push(v int) { s.q = append(s.q, v) }
func (s *scanQueue) Pop() (int, bool) {
	if len(s.q) == 0 {
		return 0, false
	}
	v := s.q[0]
	s.q = s.q[1:]
	return v, true
}
func (s *scanQueue) Min() (int, bool) {
	if len(s.q) == 0 {
		return 0, false
	}
	m := s.q[0]
	for _, v := range s.q[1:] {
		if v < m {
			m = v
		}
	}
	return m, true
}

var sinkI int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	fmt.Println("a queue from two stacks. Push goes on `in`; Pop takes from `out`,")
	fmt.Println("and when `out` runs dry the whole of `in` is tipped into it --")
	fmt.Println("which reverses the order, which is exactly what a queue needs:")
	fmt.Println()
	q := &TwoStack{}
	step := func(label string, fn func() string) {
		r := fn()
		fmt.Printf("  %-10s in=%-16s out=%-16s %s\n", label,
			fmt.Sprint(q.in), fmt.Sprint(q.out), r)
	}
	for _, v := range []int{1, 2, 3} {
		step(fmt.Sprintf("Push(%d)", v), func() string { q.Push(v); return "" })
	}
	step("Pop()", func() string { v, _ := q.Pop(); return fmt.Sprintf("-> %d  (tipped)", v) })
	step("Push(4)", func() string { q.Push(4); return "" })
	step("Pop()", func() string { v, _ := q.Pop(); return fmt.Sprintf("-> %d", v) })
	step("Pop()", func() string { v, _ := q.Pop(); return fmt.Sprintf("-> %d", v) })
	step("Pop()", func() string { v, _ := q.Pop(); return fmt.Sprintf("-> %d  (tipped)", v) })
	fmt.Println()
	fmt.Println("  note Push(4) landed on `in` while 2 and 3 were still on `out`.")
	fmt.Println("  The two stacks are never merged and never sorted; the ordering is")
	fmt.Println("  entirely a consequence of tipping ONE at a time.")

	fmt.Println()
	fmt.Println("the amortized argument, and it is the same one as the monotonic")
	fmt.Println("stack in lesson 08 -- each element moves at most twice:")
	fmt.Println()
	fmt.Printf("  %10s %14s %14s %16s\n", "operations", "elements tipped", "worst tip", "tips/operation")
	rng := rand.New(rand.NewSource(3))
	for _, n := range []int{1000, 10_000, 100_000, 1_000_000} {
		t := &TwoStack{}
		worst := 0
		for i := 0; i < n; i++ {
			if rng.Intn(2) == 0 || t.Len() == 0 {
				t.Push(i)
				continue
			}
			before := t.moved
			t.Pop()
			if d := t.moved - before; d > worst {
				worst = d
			}
		}
		fmt.Printf("  %10d %14d %14d %16.2f\n", n, t.moved, worst, float64(t.moved)/float64(n))
	}
	fmt.Println()
	fmt.Println("  exactly 0.50 tips per operation in every row -- each element is")
	fmt.Println("  tipped once and there is one tip per two operations. Yet the")
	fmt.Println("  WORST single Pop moved 577 elements at n=1,000,000, and that")
	fmt.Println("  number grows with n while the average does not.")
	fmt.Println()
	fmt.Println("  that is what amortized means, and why it is not worst case: this")
	fmt.Println("  queue is a bad choice for a latency-sensitive path and a fine")
	fmt.Println("  choice for total throughput.")
	fmt.Println()
	fmt.Println("  (a ring buffer is Theta(1) WORST case, so if the tail latency")
	fmt.Println("  matters, example 3 is the answer and this one is a curiosity.)")

	fmt.Println()
	fmt.Println("but the two-stack shape buys something a ring cannot: because each")
	fmt.Println("half is a STACK, and a stack can track its own minimum in Theta(1)")
	fmt.Println("(lesson 08), the QUEUE gets a Theta(1) minimum for free:")
	fmt.Println()
	mq := &MinQueue{}
	sq := &scanQueue{}
	for _, v := range []int{5, 2, 8, 1, 9, 3} {
		mq.Push(v)
		sq.Push(v)
	}
	fmt.Printf("  %-24s %-10s %s\n", "after", "MinQueue", "scan")
	for i := 0; i < 6; i++ {
		a, _ := mq.Min()
		b, _ := sq.Min()
		fmt.Printf("  %-24s %-10d %d\n", fmt.Sprintf("%d pops", i), a, b)
		mq.Pop()
		sq.Pop()
	}
	fmt.Println()
	fmt.Println("  identical answers. The MinQueue never looks at more than two")
	fmt.Println("  numbers to produce them.")

	fmt.Println()
	fmt.Println("verified on 200,000 random operations before it is timed:")
	fmt.Println()
	m := &MinQueue{}
	s := &scanQueue{}
	mismatch, checks := 0, 0
	for i := 0; i < 200_000; i++ {
		switch {
		case rng.Intn(3) > 0 || m.Len() == 0:
			v := rng.Intn(1000)
			m.Push(v)
			s.Push(v)
		default:
			a, _ := m.Pop()
			b, _ := s.Pop()
			if a != b {
				mismatch++
			}
		}
		a, okA := m.Min()
		b, okB := s.Min()
		checks++
		if okA != okB || a != b {
			mismatch++
		}
	}
	fmt.Printf("  %d checks, %d mismatches, final Len %d\n", checks, mismatch, m.Len())

	fmt.Println()
	fmt.Println("and what the augmentation is worth:")
	fmt.Println()
	fmt.Printf("  %10s %16s %16s %10s\n", "live", "MinQueue ns/op", "scan ns/op", "speedup")
	for _, live := range []int{16, 256, 4096, 65536} {
		mm := &MinQueue{}
		ss := &scanQueue{}
		for i := 0; i < live; i++ {
			v := rng.Intn(1 << 20)
			mm.Push(v)
			ss.Push(v)
		}
		t1 := nsPerOp(func() {
			mm.Push(rng.Intn(1 << 20))
			v, _ := mm.Min()
			sinkI = v
			mm.Pop()
		})
		t2 := nsPerOp(func() {
			ss.Push(rng.Intn(1 << 20))
			v, _ := ss.Min()
			sinkI = v
			ss.Pop()
		})
		fmt.Printf("  %10d %16.1f %16.1f %9.1fx\n", live, t1, t2, t2/t1)
	}
	fmt.Println()
	fmt.Println("  flat against a straight line, and they cross at about 16 live")
	fmt.Println("  elements -- below that the scan wins, because four slices and two")
	fmt.Println("  comparisons cost more than reading sixteen ints in cache.")
	fmt.Println()
	fmt.Println("  this is lesson 08's min-stack finding repeated one level up, and")
	fmt.Println("  the general principle behind it is worth naming: AUGMENT A")
	fmt.Println("  STRUCTURE WITH THE ANSWER, do not recompute it. The same move")
	fmt.Println("  gives you order statistics on a BST (lesson 18), range sums on a")
	fmt.Println("  segment tree (lesson 36) and subtree sizes for rank queries.")
	fmt.Println()
	fmt.Println("  the sliding-window maximum of example 10 is the same problem with")
	fmt.Println("  a fixed window instead of an explicit Pop -- and a deque solves it")
	fmt.Println("  with ONE array instead of four. Prefer that when the window is")
	fmt.Println("  what you actually have.")
}
```

**Sample output:**

```
a queue from two stacks. Push goes on `in`; Pop takes from `out`,
and when `out` runs dry the whole of `in` is tipped into it --
which reverses the order, which is exactly what a queue needs:

  Push(1)    in=[1]              out=[]               
  Push(2)    in=[1 2]            out=[]               
  Push(3)    in=[1 2 3]          out=[]               
  Pop()      in=[]               out=[3 2]            -> 1  (tipped)
  Push(4)    in=[4]              out=[3 2]            
  Pop()      in=[4]              out=[3]              -> 2
  Pop()      in=[4]              out=[]               -> 3
  Pop()      in=[]               out=[]               -> 4  (tipped)

  note Push(4) landed on `in` while 2 and 3 were still on `out`.
  The two stacks are never merged and never sorted; the ordering is
  entirely a consequence of tipping ONE at a time.

the amortized argument, and it is the same one as the monotonic
stack in lesson 08 -- each element moves at most twice:

  operations elements tipped      worst tip   tips/operation
        1000            498             21             0.50
       10000           4967            143             0.50
      100000          49997            493             0.50
     1000000         499980            577             0.50

  exactly 0.50 tips per operation in every row -- each element is
  tipped once and there is one tip per two operations. Yet the
  WORST single Pop moved 577 elements at n=1,000,000, and that
  number grows with n while the average does not.

  that is what amortized means, and why it is not worst case: this
  queue is a bad choice for a latency-sensitive path and a fine
  choice for total throughput.

  (a ring buffer is Theta(1) WORST case, so if the tail latency
  matters, example 3 is the answer and this one is a curiosity.)

but the two-stack shape buys something a ring cannot: because each
half is a STACK, and a stack can track its own minimum in Theta(1)
(lesson 08), the QUEUE gets a Theta(1) minimum for free:

  after                    MinQueue   scan
  0 pops                   1          1
  1 pops                   1          1
  2 pops                   1          1
  3 pops                   1          1
  4 pops                   3          3
  5 pops                   3          3

  identical answers. The MinQueue never looks at more than two
  numbers to produce them.

verified on 200,000 random operations before it is timed:

  200000 checks, 0 mismatches, final Len 66406

and what the augmentation is worth:

        live   MinQueue ns/op       scan ns/op    speedup
          16             10.4             10.7       1.0x
         256              8.4            125.7      14.9x
        4096              8.2           2065.0     251.3x
       65536              8.3          33064.5    3992.7x

  flat against a straight line, and they cross at about 16 live
  elements -- below that the scan wins, because four slices and two
  comparisons cost more than reading sixteen ints in cache.

  this is lesson 08's min-stack finding repeated one level up, and
  the general principle behind it is worth naming: AUGMENT A
  STRUCTURE WITH THE ANSWER, do not recompute it. The same move
  gives you order statistics on a BST (lesson 18), range sums on a
  segment tree (lesson 36) and subtree sizes for rank queries.

  the sliding-window maximum of example 10 is the same problem with
  a fixed window instead of an explicit Pop -- and a deque solves it
  with ONE array instead of four. Prefer that when the window is
  what you actually have.
```

**Complexity:** **exactly 0.50 tips per operation** in every row, while the worst single `Pop` moved
**577** elements — amortized is not worst case · `Min` in Θ(1) against a scan: **3992.7×** at 65,536
live, and the scan wins below about 16

---

## 15. The work-stealing deque

`🔴 hard` · *why the two ends being different matters*

The owner pushes and pops its own **back**; thieves take another worker's **front**. Two properties
fall out at once — the two roles almost never touch the same memory, and the thief gets the *oldest*
task, which in a divide-and-conquer workload is the *biggest* one.

**Steps:**

1. Build a task pool where each worker owns a deque.
2. Measure the steal rate and see why stealing stays rare.
3. Then steal from the *wrong* end and measure what that costs.

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"sync/atomic"
	"testing"
)

// The WORK-STEALING DEQUE is why the two ends being different matters, and it
// is the structure inside Go's own scheduler, Java's ForkJoinPool, Rust's
// rayon and Intel's TBB.
//
// The idea: every worker owns a deque of tasks.
//
//	the OWNER pushes and pops its own BACK      -- LIFO, no contention
//	a THIEF pops another worker's FRONT         -- FIFO, far from the owner
//
// Two properties fall out, and both are load-bearing:
//
//  1. owner and thief touch OPPOSITE ENDS, so they almost never collide, and
//     the common case needs no lock at all
//  2. the owner runs LIFO, which is cache-friendly (the task it just created is
//     still warm), while thieves take the OLDEST task, which in a divide-and-
//     conquer workload is the BIGGEST remaining subtree -- so one steal buys a
//     lot of work and stealing stays rare
//
// This example uses a mutex for clarity. The real ones are lock-free (Chase-Lev),
// but the ordering discipline -- and the reason it works -- is identical.

type task struct{ lo, hi int } // sum the range [lo, hi)

type workDeque struct {
	mu  sync.Mutex
	buf []task
}

// pushBack / popBack: the owner's end.
func (d *workDeque) pushBack(t task) {
	d.mu.Lock()
	d.buf = append(d.buf, t)
	d.mu.Unlock()
}
func (d *workDeque) popBack() (task, bool) {
	d.mu.Lock()
	defer d.mu.Unlock()
	if len(d.buf) == 0 {
		return task{}, false
	}
	t := d.buf[len(d.buf)-1]
	d.buf = d.buf[:len(d.buf)-1]
	return t, true
}

// popFront: the thief's end.
func (d *workDeque) popFront() (task, bool) {
	d.mu.Lock()
	defer d.mu.Unlock()
	if len(d.buf) == 0 {
		return task{}, false
	}
	t := d.buf[0]
	d.buf = d.buf[1:]
	return t, true
}
func (d *workDeque) len() int {
	d.mu.Lock()
	defer d.mu.Unlock()
	return len(d.buf)
}

const threshold = 4096 // below this, do the work rather than splitting

type pool struct {
	deques  []*workDeque
	sum     atomic.Int64
	active  atomic.Int64
	steals  atomic.Int64
	locals  atomic.Int64
	stealFn func(*pool, int) (task, bool)
}

func work(t task) int64 {
	var s int64
	for i := t.lo; i < t.hi; i++ {
		s += int64(i)
	}
	return s
}

func (p *pool) run(id int) {
	d := p.deques[id]
	for {
		t, ok := d.popBack() // 1. own work first, LIFO
		if ok {
			p.locals.Add(1)
		} else {
			t, ok = p.stealFn(p, id) // 2. otherwise steal
			if ok {
				p.steals.Add(1)
			}
		}
		if !ok {
			if p.active.Load() == 0 {
				return
			}
			runtime.Gosched()
			continue
		}
		if t.hi-t.lo <= threshold {
			p.sum.Add(work(t))
			p.active.Add(-1)
			continue
		}
		mid := (t.lo + t.hi) / 2
		p.active.Add(1)
		d.pushBack(task{t.lo, mid})
		d.pushBack(task{mid, t.hi})
	}
}

// stealFront takes the OLDEST task from a victim -- the correct end.
func stealFront(p *pool, id int) (task, bool) {
	for i := 1; i < len(p.deques); i++ {
		v := (id + i) % len(p.deques)
		if t, ok := p.deques[v].popFront(); ok {
			return t, true
		}
	}
	return task{}, false
}

// stealBack takes the NEWEST task -- the same end the owner is using. Still
// correct, and measurably worse.
func stealBack(p *pool, id int) (task, bool) {
	for i := 1; i < len(p.deques); i++ {
		v := (id + i) % len(p.deques)
		if t, ok := p.deques[v].popBack(); ok {
			return t, true
		}
	}
	return task{}, false
}

func runPool(workers, n int, steal func(*pool, int) (task, bool)) *pool {
	p := &pool{deques: make([]*workDeque, workers), stealFn: steal}
	for i := range p.deques {
		p.deques[i] = &workDeque{}
	}
	p.active.Store(1)
	p.deques[0].pushBack(task{0, n})

	var wg sync.WaitGroup
	for i := 0; i < workers; i++ {
		wg.Add(1)
		go func(id int) { defer wg.Done(); p.run(id) }(i)
	}
	wg.Wait()
	return p
}

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	const n = 40_000_000
	want := int64(n) * int64(n-1) / 2

	fmt.Printf("summing 0..%d across workers, each owning a deque.\n", n)
	fmt.Println("The task is split in half and both halves pushed back, so the")
	fmt.Println("deque holds a tree of pending subranges.")
	fmt.Println()
	fmt.Printf("  %8s %14s %14s %14s %10s\n", "workers", "local pops", "steals", "steal rate", "ns")
	for _, w := range []int{1, 2, 4, 8} {
		p := runPool(w, n, stealFront)
		if p.sum.Load() != want {
			panic(fmt.Sprintf("wrong sum: %d != %d", p.sum.Load(), want))
		}
		t := nsPerOp(func() { runPool(w, n, stealFront) })
		total := p.locals.Load() + p.steals.Load()
		fmt.Printf("  %8d %14d %14d %13.2f%% %10.0f\n",
			w, p.locals.Load(), p.steals.Load(),
			100*float64(p.steals.Load())/float64(total), t)
	}
	fmt.Println()
	fmt.Println("  the steal rate is the number that matters. Stealing is the")
	fmt.Println("  expensive path -- it touches another core's cache line -- and it")
	fmt.Println("  stays a small fraction of operations because each steal hands the")
	fmt.Println("  thief a large subtree that keeps it busy for a long time.")

	fmt.Println()
	fmt.Println("that only works because the thief takes the FRONT. Stealing from")
	fmt.Println("the back -- the same end the owner uses -- is still correct:")
	fmt.Println()
	fmt.Printf("  %8s %15s %15s %13s %13s %8s\n", "workers", "steal FRONT ns", "steal BACK ns", "front steals", "back steals", "ratio")
	for _, w := range []int{2, 4, 8} {
		pf := runPool(w, n, stealFront)
		pb := runPool(w, n, stealBack)
		if pf.sum.Load() != want || pb.sum.Load() != want {
			panic("wrong sum")
		}
		tf := nsPerOp(func() { runPool(w, n, stealFront) })
		tb := nsPerOp(func() { runPool(w, n, stealBack) })
		fmt.Printf("  %8d %15.0f %15.0f %13d %13d %7.0fx\n",
			w, tf, tb, pf.steals.Load(), pb.steals.Load(),
			float64(pb.steals.Load())/float64(max(pf.steals.Load(), 1)))
	}
	fmt.Println()
	fmt.Println("  both produce the correct sum -- checked every run. Read the two")
	fmt.Println("  right-hand columns: back-stealing needs one to two ORDERS OF")
	fmt.Println("  MAGNITUDE more steals -- see the ratio -- because it takes the")
	fmt.Println("  SMALLEST, most recently split task, so the thief is back for")
	fmt.Println("  another almost immediately. Front-stealing takes")
	fmt.Println("  the oldest and largest, which keeps the thief busy.")
	fmt.Println()
	fmt.Println("  and yet the clock barely moves, a few percent. That is the honest")
	fmt.Println("  result: on THIS workload each leaf task does 4096 additions, so a")
	fmt.Println("  few thousand extra steals disappear into the arithmetic. The")
	fmt.Println("  structural difference is real and enormous; its cost here is not.")
	fmt.Println()
	fmt.Println("  lower the threshold, or make the tasks cheaper, and the steal")
	fmt.Println("  count is what you would be paying. Design for the property, not")
	fmt.Println("  for the benchmark that happens to hide it.")

	fmt.Println()
	fmt.Println("why LIFO for the owner and FIFO for the thief, stated plainly:")
	fmt.Println()
	fmt.Printf("  %-12s %-10s %-30s %s\n", "role", "end", "gets", "because")
	fmt.Printf("  %-12s %-10s %-30s %s\n", "owner", "back", "the task it just created", "it is still in L1")
	fmt.Printf("  %-12s %-10s %-30s %s\n", "thief", "front", "the oldest, largest subtree", "one steal lasts")
	fmt.Println()
	fmt.Println("  in a divide-and-conquer workload the oldest task is the one that")
	fmt.Println("  has been split the fewest times, so it is the biggest. The deque")
	fmt.Println("  is sorted by size for free, purely as a side effect of the order")
	fmt.Println("  things were pushed in -- no priority queue required.")
	fmt.Println()
	fmt.Println("  that is the deepest reason this lesson exists. A queue and a stack")
	fmt.Println("  are not two structures to choose between: OPENING BOTH ENDS lets")
	fmt.Println("  two different consumers each get the order that suits them, from")
	fmt.Println("  one array, without either one having to coordinate with the other.")

	fmt.Println()
	fmt.Printf("  (Go's own scheduler does exactly this: each P has a 256-slot\n")
	fmt.Println("  run queue, runs it LIFO-ish, and idle Ps steal half of another")
	fmt.Println("  P's queue from the front. `runtime/proc.go`, runqsteal.)")
}
```

**Sample output:**

```
summing 0..40000000 across workers, each owning a deque.
The task is split in half and both halves pushed back, so the
deque holds a tree of pending subranges.

   workers     local pops         steals     steal rate         ns
         1          32767              0          0.00%   10590652
         2          32763              4          0.01%    5685427
         4          32747             20          0.06%    3969215
         8          32651            116          0.35%    3836199

  the steal rate is the number that matters. Stealing is the
  expensive path -- it touches another core's cache line -- and it
  stays a small fraction of operations because each steal hands the
  thief a large subtree that keeps it busy for a long time.

that only works because the thief takes the FRONT. Stealing from
the back -- the same end the owner uses -- is still correct:

   workers  steal FRONT ns   steal BACK ns  front steals   back steals    ratio
         2         6192053         5808390             7          1443     206x
         4         4082319         4166007            20          2060     103x
         8         3764336         4012618           122          3703      30x

  both produce the correct sum -- checked every run. Read the two
  right-hand columns: back-stealing needs one to two ORDERS OF
  MAGNITUDE more steals -- see the ratio -- because it takes the
  SMALLEST, most recently split task, so the thief is back for
  another almost immediately. Front-stealing takes
  the oldest and largest, which keeps the thief busy.

  and yet the clock barely moves, a few percent. That is the honest
  result: on THIS workload each leaf task does 4096 additions, so a
  few thousand extra steals disappear into the arithmetic. The
  structural difference is real and enormous; its cost here is not.

  lower the threshold, or make the tasks cheaper, and the steal
  count is what you would be paying. Design for the property, not
  for the benchmark that happens to hide it.

why LIFO for the owner and FIFO for the thief, stated plainly:

  role         end        gets                           because
  owner        back       the task it just created       it is still in L1
  thief        front      the oldest, largest subtree    one steal lasts

  in a divide-and-conquer workload the oldest task is the one that
  has been split the fewest times, so it is the biggest. The deque
  is sorted by size for free, purely as a side effect of the order
  things were pushed in -- no priority queue required.

  that is the deepest reason this lesson exists. A queue and a stack
  are not two structures to choose between: OPENING BOTH ENDS lets
  two different consumers each get the order that suits them, from
  one array, without either one having to coordinate with the other.

  (Go's own scheduler does exactly this: each P has a 256-slot
  run queue, runs it LIFO-ish, and idle Ps steal half of another
  P's queue from the front. `runtime/proc.go`, runqsteal.)
```

**Complexity:** steals stay under **0.35%** of operations because each one hands over a large subtree ·
stealing from the back needs **30–206× more steals** — and costs only a few percent on the clock here,
because each leaf task does 4096 additions. Design for the property, not for the benchmark that hides it

---

## 16. Capstone: `Deque[T]`, built, proved, measured

`🔴 hard` · *the three-pass method, end to end*

Everything this lesson found, assembled into one type and one set of checks. The proof has three
layers because each catches what the others cannot, and the measurement includes the numbers that
amortized analysis hides.

**Steps:**

1. **Build it** — generic, power-of-two, two-copy grow, slot zeroing.
2. **Prove it** — model oracle, invariants after every operation, and a rotation sweep.
3. **Measure it** — allocation budget, worst-case latency, memory held after a burst.

```go
package main

import (
	"fmt"
	"math/rand"
	"runtime"
	"testing"
)

// The capstone, and the three-pass method end to end:
//
//	BUILD IT    Deque[T] -- one array, both ends, generic, allocation-free
//	            in the steady state
//	PROVE IT    a model oracle over 200,000 random operations, an invariant
//	            check after every one, and a rotation sweep that reaches the
//	            wrapped states no hand-written test ever will
//	MEASURE IT  allocation budget, worst-case latency, and the two consumers
//	            from this lesson (BFS and sliding-window maximum)
//
// Everything below is the lesson's findings, assembled.

// ---- BUILD IT --------------------------------------------------------------

type Deque[T any] struct {
	buf         []T
	mask        int
	head, count int
	grows       int
}

// NewDeque rounds the capacity UP to a power of two, which is not an
// optimisation -- (head-1)&mask depends on it (example 5).
func NewDeque[T any](capacity int) *Deque[T] {
	n := 1
	for n < max(capacity, 1) {
		n <<= 1
	}
	return &Deque[T]{buf: make([]T, n), mask: n - 1}
}

func (d *Deque[T]) Len() int { return d.count }
func (d *Deque[T]) Cap() int { return len(d.buf) }

// grow doubles and UNWRAPS. Two copies, because a wrapped live region is at
// most two contiguous pieces (example 4).
func (d *Deque[T]) grow() {
	next := make([]T, len(d.buf)*2)
	n := copy(next, d.buf[d.head:])
	copy(next[n:], d.buf[:d.head])
	d.buf, d.mask, d.head = next, len(next)-1, 0
	d.grows++
}

func (d *Deque[T]) PushBack(v T) {
	if d.count == len(d.buf) {
		d.grow()
	}
	d.buf[(d.head+d.count)&d.mask] = v
	d.count++
}

func (d *Deque[T]) PushFront(v T) {
	if d.count == len(d.buf) {
		d.grow()
	}
	d.head = (d.head - 1) & d.mask
	d.buf[d.head] = v
	d.count++
}

func (d *Deque[T]) PopFront() (T, bool) {
	var zero T
	if d.count == 0 {
		return zero, false
	}
	v := d.buf[d.head]
	d.buf[d.head] = zero // release any pointer inside T (example 5)
	d.head = (d.head + 1) & d.mask
	d.count--
	return v, true
}

func (d *Deque[T]) PopBack() (T, bool) {
	var zero T
	if d.count == 0 {
		return zero, false
	}
	i := (d.head + d.count - 1) & d.mask
	v := d.buf[i]
	d.buf[i] = zero
	d.count--
	return v, true
}

func (d *Deque[T]) At(i int) (T, bool) {
	var zero T
	if i < 0 || i >= d.count {
		return zero, false
	}
	return d.buf[(d.head+i)&d.mask], true
}

func (d *Deque[T]) Slice() []T {
	out := make([]T, d.count)
	n := copy(out, d.buf[d.head:min(d.head+d.count, len(d.buf))])
	copy(out[n:], d.buf[:d.count-n])
	return out
}

// ---- PROVE IT --------------------------------------------------------------

// invariants must hold after EVERY operation. They are cheap, so there is no
// excuse for not checking them in a test.
func (d *Deque[T]) invariants() error {
	if n := len(d.buf); n == 0 || n&(n-1) != 0 {
		return fmt.Errorf("capacity %d is not a power of two", n)
	}
	if d.mask != len(d.buf)-1 {
		return fmt.Errorf("mask %d does not match capacity %d", d.mask, len(d.buf))
	}
	if d.head < 0 || d.head >= len(d.buf) {
		return fmt.Errorf("head %d out of range [0,%d)", d.head, len(d.buf))
	}
	if d.count < 0 || d.count > len(d.buf) {
		return fmt.Errorf("count %d out of range [0,%d]", d.count, len(d.buf))
	}
	return nil
}

// model is the obviously-correct reference: a plain slice, no cleverness.
type model[T any] struct{ s []T }

func (m *model[T]) PushBack(v T)  { m.s = append(m.s, v) }
func (m *model[T]) PushFront(v T) { m.s = append([]T{v}, m.s...) }
func (m *model[T]) PopFront() (T, bool) {
	var zero T
	if len(m.s) == 0 {
		return zero, false
	}
	v := m.s[0]
	m.s = m.s[1:]
	return v, true
}
func (m *model[T]) PopBack() (T, bool) {
	var zero T
	if len(m.s) == 0 {
		return zero, false
	}
	v := m.s[len(m.s)-1]
	m.s = m.s[:len(m.s)-1]
	return v, true
}

func equal(a, b []int) bool {
	if len(a) != len(b) {
		return false
	}
	for i := range a {
		if a[i] != b[i] {
			return false
		}
	}
	return true
}

// ---- the two consumers, rebuilt on the finished type ----------------------

func maxSliding(a []int, k int) []int {
	if k <= 0 || len(a) < k {
		return nil
	}
	out := make([]int, 0, len(a)-k+1)
	d := NewDeque[int](k + 1)
	for i, v := range a {
		if f, ok := d.At(0); ok && f <= i-k {
			d.PopFront()
		}
		for {
			b, ok := d.At(d.Len() - 1)
			if !ok || a[b] > v {
				break
			}
			d.PopBack()
		}
		d.PushBack(i)
		if i >= k-1 {
			f, _ := d.At(0)
			out = append(out, a[f])
		}
	}
	return out
}

func maxSlidingBrute(a []int, k int) []int {
	if k <= 0 || len(a) < k {
		return nil
	}
	out := make([]int, 0, len(a)-k+1)
	for i := 0; i+k <= len(a); i++ {
		m := a[i]
		for j := i + 1; j < i+k; j++ {
			if a[j] > m {
				m = a[j]
			}
		}
		out = append(out, m)
	}
	return out
}

var sinkI int
var sinkS []int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func heapMB() float64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return float64(ms.HeapAlloc) / (1 << 20)
}

func main() {
	fmt.Println("PASS 1 -- BUILD IT")
	fmt.Println()
	d := NewDeque[string](2)
	d.PushBack("b")
	d.PushBack("c")
	d.PushFront("a")
	d.PushBack("d")
	fmt.Printf("  %v   Len=%d Cap=%d (grew %d time(s) from 2)\n",
		d.Slice(), d.Len(), d.Cap(), d.grows)
	f, _ := d.PopFront()
	bk, _ := d.PopBack()
	fmt.Printf("  PopFront=%q PopBack=%q -> %v\n", f, bk, d.Slice())

	fmt.Println()
	fmt.Println("PASS 2 -- PROVE IT")
	fmt.Println()

	// 2a. model oracle, with the invariants checked after every operation
	rng := rand.New(rand.NewSource(9))
	dq := NewDeque[int](1)
	md := &model[int]{}
	const ops = 200_000
	mismatches, invFails := 0, 0
	for i := 0; i < ops; i++ {
		switch rng.Intn(4) {
		case 0:
			v := rng.Intn(1000)
			dq.PushBack(v)
			md.PushBack(v)
		case 1:
			v := rng.Intn(1000)
			dq.PushFront(v)
			md.PushFront(v)
		case 2:
			a, okA := dq.PopFront()
			b, okB := md.PopFront()
			if okA != okB || a != b {
				mismatches++
			}
		case 3:
			a, okA := dq.PopBack()
			b, okB := md.PopBack()
			if okA != okB || a != b {
				mismatches++
			}
		}
		if err := dq.invariants(); err != nil {
			invFails++
		}
		if dq.Len() != len(md.s) {
			mismatches++
		}
	}
	if !equal(dq.Slice(), md.s) {
		mismatches++
	}
	fmt.Printf("  %-46s %d mismatches, %d invariant failures\n",
		fmt.Sprintf("%d random ops against a slice model:", ops), mismatches, invFails)

	// 2b. the rotation sweep -- the states no hand-written test reaches
	rotFails := 0
	for rot := 0; rot < 64; rot++ {
		r := NewDeque[int](8)
		for i := 0; i < rot; i++ { // drive head to an arbitrary offset
			r.PushBack(-1)
			r.PopFront()
		}
		for i := 0; i < 20; i++ { // now force two grows from a wrapped state
			r.PushBack(i)
		}
		want := make([]int, 20)
		for i := range want {
			want[i] = i
		}
		if !equal(r.Slice(), want) {
			rotFails++
		}
	}
	fmt.Printf("  %-46s %d failures\n", "64 starting rotations, then grow:", rotFails)

	// 2c. the consumer, proved against brute force
	swFails, swCases := 0, 0
	for trial := 0; trial < 2000; trial++ {
		n := 1 + rng.Intn(24)
		arr := make([]int, n)
		for i := range arr {
			arr[i] = rng.Intn(15) - 7 // small range: forces ties
		}
		for k := 1; k <= n; k++ {
			swCases++
			if !equal(maxSliding(arr, k), maxSlidingBrute(arr, k)) {
				swFails++
			}
		}
	}
	fmt.Printf("  %-46s %d failures\n",
		fmt.Sprintf("sliding-window max, %d (array,k) pairs:", swCases), swFails)

	fmt.Println()
	fmt.Println("  three layers, and each catches what the others cannot: the model")
	fmt.Println("  finds wrong ANSWERS, the invariants find corrupt STATE the model")
	fmt.Println("  has not surfaced yet, and the rotation sweep reaches the wrapped")
	fmt.Println("  configurations that both would otherwise never visit.")

	fmt.Println()
	fmt.Println("PASS 3 -- MEASURE IT")
	fmt.Println()

	// 3a. allocation budget
	warm := NewDeque[int](4096)
	for i := 0; i < 1000; i++ {
		warm.PushBack(i)
	}
	allocs := testing.AllocsPerRun(5, func() {
		for i := 0; i < 100_000; i++ {
			warm.PushBack(i)
			v, _ := warm.PopFront()
			sinkI = v
		}
	})
	fmt.Printf("  %-46s %.0f\n", "allocations, 100,000 steady-state ops:", allocs)

	// 3b. steady state vs the naive queue, both behind method calls
	const steady = 500_000
	tDeque := nsPerOp(func() {
		q := NewDeque[int](4096)
		for i := 0; i < 1000; i++ {
			q.PushBack(i)
		}
		for i := 0; i < steady; i++ {
			q.PushBack(i)
			v, _ := q.PopFront()
			sinkI = v
		}
	})
	fmt.Printf("  %-46s %.2f ns\n", "steady state, per operation:", tDeque/steady)

	// 3c. worst-case latency -- the thing amortized analysis hides
	q := NewDeque[int](1)
	worstGrow, worstAt := 0, 0
	for i := 0; i < 1_000_000; i++ {
		before := q.grows
		q.PushBack(i)
		if q.grows > before {
			if q.Cap() > worstGrow {
				worstGrow, worstAt = q.Cap(), i
			}
		}
	}
	fmt.Printf("  %-46s %d elements copied at push %d\n",
		"worst single PushBack:", worstGrow/2, worstAt)

	// 3d. memory held after a burst and a full drain
	base := heapMB()
	burst := NewDeque[int](8)
	for i := 0; i < 4_000_000; i++ {
		burst.PushBack(i)
	}
	peak := heapMB() - base
	for burst.Len() > 0 {
		burst.PopFront()
	}
	held := heapMB() - base
	runtime.KeepAlive(burst)
	fmt.Printf("  %-46s %.1f MB peak, %.1f MB still held at Len 0\n",
		"burst of 4,000,000 then drained:", peak, held)

	fmt.Println()
	fmt.Println("  that last line is example 1's finding, in the finished type, and")
	fmt.Println("  it is deliberate: this deque grows and never shrinks. If your")
	fmt.Println("  workload has rare huge bursts, add lesson 06's hysteresis (shrink")
	fmt.Println("  at 25% full, halve only) -- and measure that it does not thrash.")

	// 3e. the two consumers
	fmt.Println()
	fmt.Println("  and what it does for example 10's consumer:")
	fmt.Println()
	arr := make([]int, 200_000)
	for i := range arr {
		arr[i] = rng.Intn(1 << 20)
	}
	fmt.Printf("  %-24s %14s %14s %10s\n", "sliding-window max", "brute ns", "deque ns", "speedup")
	for _, k := range []int{8, 128, 2048} {
		if !equal(maxSliding(arr[:20_000], k), maxSlidingBrute(arr[:20_000], k)) {
			panic("consumer disagrees with brute force")
		}
		tb := nsPerOp(func() { sinkS = maxSlidingBrute(arr, k) })
		td := nsPerOp(func() { sinkS = maxSliding(arr, k) })
		fmt.Printf("  %-24s %14.0f %14.0f %9.1fx\n", fmt.Sprintf("k = %d", k), tb, td, tb/td)
	}

	fmt.Println()
	fmt.Println("THE CHECKLIST, which is the actual deliverable of this lesson:")
	fmt.Println()
	for _, line := range []string{
		"capacity is a power of two            (& instead of %, and PushFront works)",
		"count is stored, not derived          (head == tail is ambiguous)",
		"grow unwraps with TWO copies          (the live region is two pieces)",
		"Pop writes the zero value             (or T's pointers never die)",
		"tests rotate before they assert       (head == 0 hides every real bug)",
		"an overflow policy is chosen          (block / drop / error -- pick one)",
		"the steady state allocates nothing    (measure it; do not assume it)",
	} {
		fmt.Printf("  [x] %s\n", line)
	}
	fmt.Println()
	fmt.Println("  seven lines, five of which I got wrong at least once while")
	fmt.Println("  writing this lesson -- and every one of those five was caught by")
	fmt.Println("  a measurement that contradicted the paragraph I had just written,")
	fmt.Println("  never by re-reading the code.")
}
```

**Sample output:**

```
PASS 1 -- BUILD IT

  [a b c d]   Len=4 Cap=4 (grew 1 time(s) from 2)
  PopFront="a" PopBack="d" -> [b c]

PASS 2 -- PROVE IT

  200000 random ops against a slice model:       0 mismatches, 0 invariant failures
  64 starting rotations, then grow:              0 failures
  sliding-window max, 24792 (array,k) pairs:     0 failures

  three layers, and each catches what the others cannot: the model
  finds wrong ANSWERS, the invariants find corrupt STATE the model
  has not surfaced yet, and the rotation sweep reaches the wrapped
  configurations that both would otherwise never visit.

PASS 3 -- MEASURE IT

  allocations, 100,000 steady-state ops:         0
  steady state, per operation:                   2.19 ns
  worst single PushBack:                         524288 elements copied at push 524288
  burst of 4,000,000 then drained:               32.0 MB peak, 32.0 MB still held at Len 0

  that last line is example 1's finding, in the finished type, and
  it is deliberate: this deque grows and never shrinks. If your
  workload has rare huge bursts, add lesson 06's hysteresis (shrink
  at 25% full, halve only) -- and measure that it does not thrash.

  and what it does for example 10's consumer:

  sliding-window max             brute ns       deque ns    speedup
  k = 8                           1184439        1839710       0.6x
  k = 128                        13533713        1835371       7.4x
  k = 2048                      123031806        1733024      71.0x

THE CHECKLIST, which is the actual deliverable of this lesson:

  [x] capacity is a power of two            (& instead of %, and PushFront works)
  [x] count is stored, not derived          (head == tail is ambiguous)
  [x] grow unwraps with TWO copies          (the live region is two pieces)
  [x] Pop writes the zero value             (or T's pointers never die)
  [x] tests rotate before they assert       (head == 0 hides every real bug)
  [x] an overflow policy is chosen          (block / drop / error -- pick one)
  [x] the steady state allocates nothing    (measure it; do not assume it)

  seven lines, five of which I got wrong at least once while
  writing this lesson -- and every one of those five was caught by
  a measurement that contradicted the paragraph I had just written,
  never by re-reading the code.
```

**Complexity:** all six operations Θ(1) amortized, **0 allocations** in the steady state · the worst
single `PushBack` copied **524,288** elements — which is what "amortized Θ(1)" costs you at the tail ·
after a 4,000,000 burst and a full drain it still holds **32.0 MB**, deliberately

---

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Lesson: [09-queues-deques.md](../../09-queues-deques.md)
