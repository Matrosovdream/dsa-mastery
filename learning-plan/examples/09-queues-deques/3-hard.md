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

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

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

**Rust:**

```rust
use std::collections::VecDeque;
use std::fmt;

// Two BFS variations that come up constantly and are each one small change to
// example 6's loop:
//
//	LEVEL ORDER      process the queue one LAYER at a time
//	MULTI-SOURCE     seed the queue with every source at distance 0
//
// Both are about what you put in the queue and when, not about the traversal.

#[derive(Clone, Copy)]
struct Point {
    r: usize,
    c: usize,
}

/// Go's `%v` of a struct: {0 0}
impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{{{} {}}}", self.r, self.c)
    }
}

// Signed steps: up, down, left, right. Indexes are usize, so a step cannot be
// added with `+` (the delta may be -1) and `r - 1` at row 0 would overflow.
const DIRS: [(isize, isize); 4] = [(-1, 0), (1, 0), (0, -1), (0, 1)];

/// p moved by one step, or None when the step leaves the grid. Go's
/// `nr < 0 || nr >= rows || ...` check, folded into the type: checked_add_signed
/// returns None below zero, and the filter catches the far edge.
fn step(p: Point, (dr, dc): (isize, isize), rows: usize, cols: usize) -> Option<Point> {
    let r = p.r.checked_add_signed(dr).filter(|&r| r < rows)?;
    let c = p.c.checked_add_signed(dc).filter(|&c| c < cols)?;
    Some(Point { r, c })
}

// Go's plain queue here is a slice (`q = q[1:]`). An owned Vec cannot be
// re-sliced from the front, and the two ways to write it on a Vec are
// remove(0) (shifts everything, Theta(n) per pop) or a head index (keeps every
// element ever pushed). bfs_levels pops from the front in the middle of the
// loop, so it uses std's VecDeque -- a ring buffer, the structure example 5
// built by hand: O(1) pop_front, and the same FIFO order as Go's slice. The
// distances and layers are identical either way; only the container differs.

/// bfs_levels is the level-order idiom: snapshot the queue length BEFORE the
/// inner loop, and that count is exactly one layer. It is the only reliable way
/// to know where a layer ends without storing the depth in every entry.
fn bfs_levels(g: &[Vec<u8>], start: Point) -> Vec<Vec<Point>> {
    let (rows, cols) = (g.len(), g[0].len());
    let mut seen = vec![vec![false; cols]; rows];
    let mut levels = Vec::new();
    let mut q = VecDeque::from([start]);
    seen[start.r][start.c] = true;

    while !q.is_empty() {
        let n = q.len(); // <- the snapshot. Everything after this is the next layer.
        let mut layer = Vec::with_capacity(n);
        for _ in 0..n {
            let p = q.pop_front().expect("the snapshot counted n entries");
            layer.push(p);
            for d in DIRS {
                let Some(nb) = step(p, d, rows, cols) else {
                    continue;
                };
                if seen[nb.r][nb.c] || g[nb.r][nb.c] == b'#' {
                    continue;
                }
                seen[nb.r][nb.c] = true;
                q.push_back(nb);
            }
        }
        levels.push(layer);
    }
    levels
}

type Dist = Vec<Vec<Option<usize>>>; // None: not reached (Go: -1)

/// bfs_multi seeds EVERY source at distance 0 before the loop starts. The result
/// is the distance to the NEAREST source, for every cell, in one pass.
///
/// The naive alternative -- run one BFS per source and take the minimum -- is
/// Theta(S * V). This is Theta(V), regardless of how many sources there are.
fn bfs_multi(g: &[Vec<u8>], sources: &[Point]) -> Dist {
    let (rows, cols) = (g.len(), g[0].len());
    let mut dist: Dist = vec![vec![None; cols]; rows];
    // Each cell is queued at most once, so this never reallocates.
    let mut q: Vec<Point> = Vec::with_capacity(rows * cols);
    for &s in sources {
        if dist[s.r][s.c].is_none() {
            dist[s.r][s.c] = Some(0);
            q.push(s);
        }
    }
    let mut head = 0; // head-index queue, example 2: nothing is ever removed
    while let Some(&p) = q.get(head) {
        head += 1;
        let d = dist[p.r][p.c].expect("every queued cell has a distance") + 1;
        for dir in DIRS {
            let Some(nb) = step(p, dir, rows, cols) else {
                continue;
            };
            if dist[nb.r][nb.c].is_some() || g[nb.r][nb.c] == b'#' {
                continue;
            }
            dist[nb.r][nb.c] = Some(d);
            q.push(nb);
        }
    }
    dist
}

/// single_source_min is the naive version, kept so the cost can be measured.
fn single_source_min(g: &[Vec<u8>], sources: &[Point]) -> (Dist, usize) {
    let (rows, cols) = (g.len(), g[0].len());
    let mut dist: Dist = vec![vec![None; cols]; rows];
    let mut visits = 0;
    for &s in sources {
        let d = bfs_multi(g, &[s]);
        for (from_row, row) in d.iter().zip(dist.iter_mut()) {
            for (&x, best) in from_row.iter().zip(row.iter_mut()) {
                visits += 1;
                let Some(x) = x else { continue };
                if best.is_none_or(|b| x < b) {
                    *best = Some(x);
                }
            }
        }
    }
    (dist, visits)
}

fn parse(s: &str) -> Vec<Vec<u8>> {
    s.trim()
        .lines()
        .map(|line| line.trim().as_bytes().to_vec())
        .collect()
}

fn render(g: &[Vec<u8>], dist: &Dist) -> String {
    let mut out = String::new();
    for (row, drow) in g.iter().zip(dist) {
        out.push_str("    ");
        for (&cell, d) in row.iter().zip(drow) {
            match (cell, d) {
                (b'#', _) => out.push_str("  #"),
                (_, None) => out.push_str("  ."),
                (_, Some(d)) => out.push_str(&format!("{d:>3}")),
            }
        }
        out.push('\n');
    }
    out
}

/// Go's `%v` of a []point: [{1 0} {0 1}]
fn show(ps: &[Point]) -> String {
    let parts: Vec<String> = ps.iter().map(ToString::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    let grid = parse(
        "
        ..........
        ..####....
        ..#..#..#.
        ..#..#..#.
        ..####..#.
        ..........
    ",
    );

    println!("the grid (# is a wall):");
    println!();
    for row in &grid {
        println!("    {}", std::str::from_utf8(row).expect("ascii grid"));
    }

    println!();
    println!("LEVEL ORDER from the top-left. The trick is one line:");
    println!();
    println!("      let n = q.len();       // snapshot BEFORE the inner loop");
    println!("      for _ in 0..n {{ ... }}  // exactly one layer");
    println!();
    let levels = bfs_levels(&grid, Point { r: 0, c: 0 });
    println!("  {:<8} {:>8}  first few", "layer", "cells");
    for (i, l) in levels.iter().enumerate() {
        if i > 6 && i < levels.len() - 2 {
            if i == 7 {
                println!("  {:<8} {:>8}  ...", "...", "...");
            }
            continue;
        }
        println!("  {:<8} {:>8}  {}", i, l.len(), show(&l[..l.len().min(4)]));
    }
    println!(
        "\n  {} layers, {} cells reached.",
        levels.len(),
        levels.iter().map(Vec::len).sum::<usize>()
    );
    println!();
    println!("  everything already in the queue when the layer starts is at the");
    println!("  current distance; everything pushed during the layer is one");
    println!("  further. That is the whole justification, and it fails the moment");
    println!("  you read q.len() INSIDE the loop -- which is the usual bug.");
    println!();
    println!("  this is the same idiom as binary-tree level order (lesson 17) and");
    println!("  as Kahn's topological sort by layers (lesson 23).");

    println!();
    println!("MULTI-SOURCE: seed the queue with ALL sources at distance 0.");
    println!("Distance to the nearest of three sources, one pass:");
    println!();
    let p = |r, c| Point { r, c };
    let sources = [p(0, 0), p(5, 9), p(2, 3)];
    let d = bfs_multi(&grid, &sources);
    print!("{}", render(&grid, &d));
    println!();
    println!("  the 0s are the three sources. Every other cell holds its distance");
    println!("  to the CLOSEST one, and the region boundaries between them fall");
    println!("  out for free -- that is a Voronoi diagram under the grid metric.");
    println!();
    println!("  the 2x2 pocket at rows 2-3, columns 3-4 is sealed off by walls.");
    println!("  It shows real distances only because one of the three sources is");
    println!("  inside it. Drop that source and it becomes unreachable:");

    let d2 = bfs_multi(&grid, &[p(0, 0), p(5, 9)]);
    println!();
    print!("{}", render(&grid, &d2));

    println!();
    println!("why it is one pass and not one pass per source:");
    println!();
    println!(
        "  {:>10} {:>10} {:>16} {:>18} {:>10}",
        "grid", "sources", "multi-source", "one BFS per source", "ratio"
    );
    for (n, s) in [(100, 4), (100, 64), (300, 4), (300, 64)] {
        let g = vec![vec![b'.'; n]; n];
        let srcs: Vec<Point> = (0..s)
            .map(|i| p((i * 7919) % n, (i * 104729) % n))
            .collect();
        let multi = bfs_multi(&g, &srcs);
        let (naive, visits) = single_source_min(&g, &srcs);
        // prove they agree before comparing cost
        assert!(
            multi == naive,
            "multi-source disagrees with per-source minimum"
        );
        let cells = n * n;
        println!(
            "  {:>10} {:>10} {:>16} {:>18} {:>9.0}x",
            format!("{n}x{n}"),
            s,
            cells,
            visits,
            visits as f64 / cells as f64
        );
    }
    println!();
    println!("  identical answers -- checked, not assumed -- at 1/S of the work.");
    println!("  The queue does not care where its entries came from, so seeding it");
    println!("  with S sources costs the same as seeding it with one.");
    println!();
    println!("  reach for this whenever a problem says 'nearest' and there is more");
    println!("  than one target: rotting oranges, walls and gates, distance to the");
    println!("  nearest 0, fire spreading from several points. They are all this.");
}
```

**Rust output:**

```
the grid (# is a wall):

    ..........
    ..####....
    ..#..#..#.
    ..#..#..#.
    ..####..#.
    ..........

LEVEL ORDER from the top-left. The trick is one line:

      let n = q.len();       // snapshot BEFORE the inner loop
      for _ in 0..n { ... }  // exactly one layer

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
  you read q.len() INSIDE the loop -- which is the usual bug.

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

> **Go vs Rust:** Layers, distances, grids and the S-times-the-work table are identical to Go's (diffed): the only output lines that differ are the three that quote Go syntax (`n := len(q)` becomes `let n = q.len();`). The bounds check shrinks into the type: `usize` cannot go below zero, so `checked_add_signed` returns `None` where Go tests `nr < 0`, and `Option<usize>` replaces the `-1` distance. Go's plain queue is the re-sliced `q = q[1:]`; an owned `Vec` cannot do that (`remove(0)` shifts everything, a head index keeps every element), so the level-order loop uses std's `VecDeque` ring, and the multi-source loop keeps Go's own head index.

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

**Rust:**

```rust
use std::cmp::Reverse;
use std::collections::BinaryHeap;
use std::hint::black_box;
use std::time::{Duration, Instant};

// ---- ns/op: the kit's stand-in for testing.Benchmark / b.Loop ----
fn ns_per_op(mut f: impl FnMut()) -> f64 {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            f();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return el.as_nanos() as f64 / n as f64;
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

// ---- RNG: the kit's stand-in for math/rand (splitmix64; seeded, reproducible) ----
// It is NOT Go's generator, so every random grid below is a different grid from
// the Go run's, and every number that depends on a random grid is Rust's own.
struct Rng(u64);
impl Rng {
    fn new(seed: u64) -> Self {
        Rng(seed)
    }
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
    /// like rand.Intn(n): uniform-enough in [0, n)
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
}

// ---- Deque<T> from example 5, trimmed to the methods this program calls ----
// (clippy -D warnings rejects unused methods; std's VecDeque has all of these.)
struct Deque<T> {
    buf: Vec<Option<T>>,
    mask: usize,
    head: usize,
    count: usize,
}

impl<T> Deque<T> {
    fn new(capacity: usize) -> Self {
        let n = capacity.next_power_of_two();
        Deque {
            buf: std::iter::repeat_with(|| None).take(n).collect(),
            mask: n - 1,
            head: 0,
            count: 0,
        }
    }

    fn grow(&mut self) {
        let doubled = self.buf.len() * 2;
        let mut old = std::mem::take(&mut self.buf);
        let mut next = Vec::with_capacity(doubled);
        next.extend(old.drain(self.head..)); // the piece before the wrap
        next.append(&mut old); // the piece after it
        next.resize_with(doubled, || None);
        self.buf = next;
        self.mask = doubled - 1;
        self.head = 0;
    }

    fn push_back(&mut self, v: T) {
        if self.count == self.buf.len() {
            self.grow();
        }
        let i = (self.head + self.count) & self.mask;
        self.buf[i] = Some(v);
        self.count += 1;
    }

    fn push_front(&mut self, v: T) {
        if self.count == self.buf.len() {
            self.grow();
        }
        self.head = self.head.wrapping_sub(1) & self.mask; // wraps to the end when head is 0
        self.buf[self.head] = Some(v);
        self.count += 1;
    }

    fn pop_front(&mut self) -> Option<T> {
        if self.count == 0 {
            return None;
        }
        let v = self.buf[self.head].take();
        self.head = (self.head + 1) & self.mask;
        self.count -= 1;
        v
    }
}

// 0-1 BFS is the payoff for having built a DEQUE rather than a queue.
//
// Setting: a graph whose edge weights are all 0 or 1. Dijkstra solves it in
// Theta(E log V). Plain BFS does not solve it at all -- BFS assumes every edge
// costs the same. But with a deque you get Theta(V + E):
//
//	weight 0 edge -> push_front   (same distance, so it belongs at the front)
//	weight 1 edge -> push_back    (one further, so it belongs at the back)
//
// The deque is then always sorted by distance and holds at most two distinct
// values -- d and d+1. That is a priority queue with two priorities, and a
// deque is the fastest possible implementation of one.
//
// Go's plain queue (bfs_wrong, bfs_relax) is a slice re-sliced with `q = q[1:]`.
// An owned Vec cannot be re-sliced from the front, so here every version uses
// the same Deque, and the plain queues touch only push_back and pop_front. Same
// FIFO order and the same pop counts as Go; the point of sharing one container
// is that the timing columns below then differ only in the algorithm.

#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
struct Point {
    r: usize,
    c: usize,
}

// Signed steps: up, down, left, right.
const DIRS: [(isize, isize); 4] = [(-1, 0), (1, 0), (0, -1), (0, 1)];

/// p moved by one step, or None when the step leaves the grid (Go's bounds
/// check: checked_add_signed is None below zero, the filter catches the far edge).
fn step(p: Point, (dr, dc): (isize, isize), rows: usize, cols: usize) -> Option<Point> {
    let r = p.r.checked_add_signed(dr).filter(|&r| r < rows)?;
    let c = p.c.checked_add_signed(dc).filter(|&c| c < cols)?;
    Some(Point { r, c })
}

/// "Not reached yet". Go's 1<<30. It is a plain usize rather than an Option
/// because the algorithms compare `nd < dist`, and Option orders None as the
/// SMALLEST value -- exactly backwards for "no distance yet".
const INF: usize = usize::MAX;

/// Grid: '.' costs 0 to enter, '#' costs 1 (you break the wall).
fn cost(b: u8) -> usize {
    usize::from(b == b'#')
}

fn bfs01(g: &[Vec<u8>], src: Point, dst: Point) -> (usize, usize) {
    let (rows, cols) = (g.len(), g[0].len());
    let mut dist = vec![vec![INF; cols]; rows];
    let mut d = Deque::new(64);
    dist[src.r][src.c] = 0;
    d.push_back(src);
    let mut pops = 0;

    while let Some(p) = d.pop_front() {
        pops += 1;
        for dir in DIRS {
            let Some(nb) = step(p, dir, rows, cols) else {
                continue;
            };
            let w = cost(g[nb.r][nb.c]);
            let nd = dist[p.r][p.c] + w;
            if nd < dist[nb.r][nb.c] {
                dist[nb.r][nb.c] = nd;
                if w == 0 {
                    d.push_front(nb); // same layer -- must come out first
                } else {
                    d.push_back(nb); // next layer
                }
            }
        }
    }
    (dist[dst.r][dst.c], pops)
}

// The Dijkstra reference, so the answers can be checked rather than trusted.
// Go needs a node type and five heap.Interface methods; BinaryHeap is a MAX-heap
// of anything Ord, so a tuple (distance, point) wrapped in Reverse is a min-heap
// on distance. (Ties break on the point; Go's heap breaks them by layout. Each
// cell is pushed exactly once either way, so the pop count does not change.)
fn dijkstra(g: &[Vec<u8>], src: Point, dst: Point) -> (usize, usize) {
    let (rows, cols) = (g.len(), g[0].len());
    let mut dist = vec![vec![INF; cols]; rows];
    let mut h = BinaryHeap::from([Reverse((0, src))]);
    dist[src.r][src.c] = 0;
    let mut pops = 0;
    while let Some(Reverse((d, p))) = h.pop() {
        pops += 1;
        if d > dist[p.r][p.c] {
            continue;
        }
        for dir in DIRS {
            let Some(nb) = step(p, dir, rows, cols) else {
                continue;
            };
            let nd = d + cost(g[nb.r][nb.c]);
            if nd < dist[nb.r][nb.c] {
                dist[nb.r][nb.c] = nd;
                h.push(Reverse((nd, nb)));
            }
        }
    }
    (dist[dst.r][dst.c], pops)
}

/// bfs_wrong is the trap: a plain queue, push_back for everything, and a vertex
/// settled the FIRST time it is reached -- which is exactly the correct BFS from
/// example 6. On a 0-1 graph that assumption is false, so it is now a bug.
fn bfs_wrong(g: &[Vec<u8>], src: Point, dst: Point) -> usize {
    let (rows, cols) = (g.len(), g[0].len());
    let mut dist = vec![vec![0; cols]; rows];
    let mut seen = vec![vec![false; cols]; rows];
    let mut q = Deque::new(64);
    q.push_back(src);
    seen[src.r][src.c] = true;
    while let Some(p) = q.pop_front() {
        for dir in DIRS {
            let Some(nb) = step(p, dir, rows, cols) else {
                continue;
            };
            if seen[nb.r][nb.c] {
                continue;
            }
            seen[nb.r][nb.c] = true; // settled on first arrival -- the bug
            dist[nb.r][nb.c] = dist[p.r][p.c] + cost(g[nb.r][nb.c]);
            q.push_back(nb); // always the back
        }
    }
    dist[dst.r][dst.c]
}

/// bfs_relax is the OTHER plain-queue version, and it is correct: it re-queues a
/// vertex whenever it finds a cheaper route. That is SPFA / Bellman-Ford, not
/// BFS, and the cost is that a vertex can be processed many times.
fn bfs_relax(g: &[Vec<u8>], src: Point, dst: Point) -> (usize, usize) {
    let (rows, cols) = (g.len(), g[0].len());
    let mut dist = vec![vec![INF; cols]; rows];
    let mut q = Deque::new(64);
    q.push_back(src);
    dist[src.r][src.c] = 0;
    let mut pops = 0;
    while let Some(p) = q.pop_front() {
        pops += 1;
        for dir in DIRS {
            let Some(nb) = step(p, dir, rows, cols) else {
                continue;
            };
            let nd = dist[p.r][p.c] + cost(g[nb.r][nb.c]);
            if nd < dist[nb.r][nb.c] {
                dist[nb.r][nb.c] = nd;
                q.push_back(nb);
            }
        }
    }
    (dist[dst.r][dst.c], pops)
}

fn parse(s: &str) -> Vec<Vec<u8>> {
    s.trim()
        .lines()
        .map(|line| line.trim().as_bytes().to_vec())
        .collect()
}

fn rand_grid(n: usize, wall_pct: u64, seed: u64) -> Vec<Vec<u8>> {
    let mut rng = Rng::new(seed);
    let mut g: Vec<Vec<u8>> = (0..n)
        .map(|_| {
            (0..n)
                .map(|_| {
                    if rng.below(100) < wall_pct {
                        b'#'
                    } else {
                        b'.'
                    }
                })
                .collect()
        })
        .collect();
    g[0][0] = b'.';
    g[n - 1][n - 1] = b'.';
    g
}

/// The 3,000-grid check against Dijkstra: how many grids each version gets wrong.
/// Run first in main only so the prose above its table can quote the result.
fn cross_check() -> (usize, usize, usize) {
    let (mut m01, mut m_bfs, mut m_relax) = (0, 0, 0);
    for i in 0..3000u64 {
        let rg = rand_grid(6 + i as usize % 10, 20 + i % 50, i);
        let (s, d) = (
            Point { r: 0, c: 0 },
            Point {
                r: rg.len() - 1,
                c: rg.len() - 1,
            },
        );
        let want = dijkstra(&rg, s, d).0;
        m01 += usize::from(bfs01(&rg, s, d).0 != want);
        m_bfs += usize::from(bfs_wrong(&rg, s, d) != want);
        m_relax += usize::from(bfs_relax(&rg, s, d).0 != want);
    }
    (m01, m_bfs, m_relax)
}

/// 10139058 -> "10,139,058" (Go's prose writes these by hand)
fn commas(n: usize) -> String {
    let digits = n.to_string();
    let mut out = String::new();
    for (i, ch) in digits.chars().enumerate() {
        if i > 0 && (digits.len() - i).is_multiple_of(3) {
            out.push(',');
        }
        out.push(ch);
    }
    out
}

fn main() {
    let (mismatch01, mismatch_bfs, mismatch_relax) = cross_check();

    let g = parse(
        "
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
    ",
    );
    println!("walking through '.' is free; breaking a '#' costs 1. Find the");
    println!("cheapest route from the top-left to the bottom-right:");
    println!();
    for row in &g {
        println!("    {}", std::str::from_utf8(row).expect("ascii grid"));
    }

    let (src, dst) = (
        Point { r: 0, c: 0 },
        Point {
            r: g.len() - 1,
            c: g[0].len() - 1,
        },
    );
    let (got, pops01) = bfs01(&g, src, dst);
    let (want, pops_dij) = dijkstra(&g, src, dst);
    let bad = bfs_wrong(&g, src, dst);

    println!();
    println!(
        "  {:<34} {:>6} walls broken   ({} pops)",
        "0-1 BFS with a deque", got, pops01
    );
    println!(
        "  {:<34} {:>6} walls broken   ({} pops)",
        "Dijkstra, as a reference", want, pops_dij
    );
    let (relax, pops_relax) = bfs_relax(&g, src, dst);
    println!(
        "  {:<34} {:>6} walls broken",
        "plain BFS, settle on arrival", bad
    );
    println!(
        "  {:<34} {:>6} walls broken   ({} pops)",
        "plain queue, re-relaxing (SPFA)", relax, pops_relax
    );
    println!();
    println!("  all four agree here, which is precisely the problem. Plain BFS");
    println!("  settles a cell the FIRST time it is reached, and with 0-weight");
    println!("  edges the first arrival need not be the cheapest -- but on this");
    println!(
        "  grid it happens to be. It fails on {:.0}% of random grids (below).",
        100.0 * mismatch_bfs as f64 / 3000.0
    );
    println!();
    println!("  the fourth row is the other plain-queue version, and it is correct:");
    println!("  it re-queues a cell whenever it finds a cheaper route. That is not");
    println!("  BFS, it is SPFA, and it pays for correctness in repeated work --");
    println!("  {pops_relax} pops against {pops01} here, and far worse below. The 3,000-grid");
    println!("  check below reports zero disagreements for it, which is how you");
    println!("  find out it was never the bug.");

    println!();
    println!("why push_front is the whole algorithm:");
    println!();
    println!("  the deque's invariant is that it holds at most TWO distinct");
    println!("  distances at any moment -- d at the front and d+1 at the back --");
    println!("  and is sorted. A 0-edge produces a d, which must go in front of");
    println!("  every d+1 already queued. A 1-edge produces a d+1, which goes");
    println!("  behind them.");
    println!();
    println!("  that IS a priority queue. It just happens to have two priorities,");
    println!("  so it needs no heap, no comparisons and no log factor.");

    println!();
    println!("checked against Dijkstra on 3,000 random grids before it is timed:");
    println!();
    println!("  0-1 BFS vs Dijkstra:     {mismatch01:>4} disagreements");
    println!("  plain BFS vs Dijkstra:   {mismatch_bfs:>4} disagreements");
    println!("  SPFA vs Dijkstra:        {mismatch_relax:>4} disagreements");
    println!();
    println!("  three of the four rows are correct; only the one that settles on");
    println!("  arrival is not. It is wrong on a large fraction of inputs and");
    println!("  right on the rest -- the failure mode that survives a hand-written");
    println!("  test suite and is caught in one line by a reference implementation.");

    println!();
    println!("and the cost, against the heap it replaces:");
    println!();
    println!(
        "  {:>6} {:>7} {:>13} {:>13} {:>13} {:>9} {:>16}",
        "n", "walls", "0-1 BFS ns", "SPFA ns", "Dijkstra ns", "vs heap", "pops (01/spfa)"
    );
    let mut last = (0, 0);
    for (n, w) in [(200, 30), (200, 60), (600, 30), (600, 60)] {
        let rg = rand_grid(n, w, 42);
        let d = Point { r: n - 1, c: n - 1 };
        let o = Point { r: 0, c: 0 };
        let (a, pa) = bfs01(&rg, o, d);
        let (b, pd) = dijkstra(&rg, o, d);
        assert_eq!(a, b, "disagreement on the benchmark grid");
        assert_eq!(
            pa, pd,
            "0-1 BFS and Dijkstra pop a different number of cells"
        );
        let (rr, pr) = bfs_relax(&rg, o, d);
        assert_eq!(rr, b, "SPFA disagrees on the benchmark grid");
        let t1 = ns_per_op(|| {
            black_box(bfs01(black_box(&rg), o, d));
        });
        let t3 = ns_per_op(|| {
            black_box(bfs_relax(black_box(&rg), o, d));
        });
        let t2 = ns_per_op(|| {
            black_box(dijkstra(black_box(&rg), o, d));
        });
        println!(
            "  {:>6} {:>6}% {:>13.0} {:>13.0} {:>13.0} {:>8.1}x {:>9}/{}",
            n,
            w,
            t1,
            t3,
            t2,
            t2 / t1,
            pa,
            pr
        );
        last = (pa, pr);
    }
    println!();
    let (pa, pr) = last;
    println!(
        "  0-1 BFS pops each cell exactly once -- {} on the 600x600",
        commas(pa)
    );
    println!("  grid, the same as Dijkstra (asserted) -- and it was ahead of the");
    println!("  heap on every row of every run, by 4x to 15x (one row moved by more");
    println!("  than 2x between runs: the machine was shared). The log factor is");
    println!("  real; a plausible reason for the constant, not measured, is that a");
    println!("  heap push or pop does O(log V) compares and swaps of 24-byte entries");
    println!("  where push_front is a masked index and one store.");
    println!();
    println!("  and look at the SPFA column. It is correct, it uses only a plain");
    println!(
        "  queue, and at 60% walls it pops {} times to 0-1 BFS's",
        commas(pr)
    );
    println!(
        "  {} -- {:.0}x the work. It was slower than the heap",
        commas(pa),
        pr as f64 / pa as f64
    );
    println!("  it was meant to avoid in every run at 60% walls (about 2x at 200x200,");
    println!("  4x to 5x at 600x600); at 30% walls the two traded places from run");
    println!("  to run, SPFA anywhere from 1.8x ahead to 3.4x behind.");
    println!("  Correct-but-unbounded is its own failure mode.");
    println!();
    println!("  the generalisation is Dial's algorithm: with weights in 0..C, use");
    println!("  C+1 buckets instead of a heap. 0-1 BFS is the C=1 case, and the");
    println!("  two buckets happen to be the two ends of one deque.");
}
```

**Rust sample output:**

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

  the fourth row is the other plain-queue version, and it is correct:
  it re-queues a cell whenever it finds a cheaper route. That is not
  BFS, it is SPFA, and it pays for correctness in repeated work --
  107 pops against 100 here, and far worse below. The 3,000-grid
  check below reports zero disagreements for it, which is how you
  find out it was never the bug.

why push_front is the whole algorithm:

  the deque's invariant is that it holds at most TWO distinct
  distances at any moment -- d at the front and d+1 at the back --
  and is sorted. A 0-edge produces a d, which must go in front of
  every d+1 already queued. A 1-edge produces a d+1, which goes
  behind them.

  that IS a priority queue. It just happens to have two priorities,
  so it needs no heap, no comparisons and no log factor.

checked against Dijkstra on 3,000 random grids before it is timed:

  0-1 BFS vs Dijkstra:        0 disagreements
  plain BFS vs Dijkstra:   2943 disagreements
  SPFA vs Dijkstra:           0 disagreements

  three of the four rows are correct; only the one that settles on
  arrival is not. It is wrong on a large fraction of inputs and
  right on the rest -- the failure mode that survives a hand-written
  test suite and is caught in one line by a reference implementation.

and the cost, against the heap it replaces:

       n   walls    0-1 BFS ns       SPFA ns   Dijkstra ns   vs heap   pops (01/spfa)
     200     30%        420201       3137469       4150405      9.9x     40000/235077
     200     60%        785029       5787849       2801355      3.6x     40000/381331
     600     30%       6082732      73987389      43649028      7.2x    360000/5371007
     600     60%       5874747     136181938      27770792      4.7x    360000/9620007

  0-1 BFS pops each cell exactly once -- 360,000 on the 600x600
  grid, the same as Dijkstra (asserted) -- and it was ahead of the
  heap on every row of every run, by 4x to 15x (one row moved by more
  than 2x between runs: the machine was shared). The log factor is
  real; a plausible reason for the constant, not measured, is that a
  heap push or pop does O(log V) compares and swaps of 24-byte entries
  where push_front is a masked index and one store.

  and look at the SPFA column. It is correct, it uses only a plain
  queue, and at 60% walls it pops 9,620,007 times to 0-1 BFS's
  360,000 -- 27x the work. It was slower than the heap
  it was meant to avoid in every run at 60% walls (about 2x at 200x200,
  4x to 5x at 600x600); at 30% walls the two traded places from run
  to run, SPFA anywhere from 1.8x ahead to 3.4x behind.
  Correct-but-unbounded is its own failure mode.

  the generalisation is Dial's algorithm: with weights in 0..C, use
  C+1 buckets instead of a heap. 0-1 BFS is the C=1 case, and the
  two buckets happen to be the two ends of one deque.
```

> **Go vs Rust:** The algorithm is Go's, move for move: the demo grid gives the same four answers and pop counts (2 walls, 100 / 100 / 107 pops), and with Go's `math/rand` generator fed in (a throwaway build, not shipped, since its 607-entry seed table would swamp the example) this program printed Go's 2,939 plain-BFS failures and its exact SPFA pop counts (223,556 ... 10,139,058). The shipped kit `Rng` is splitmix64, so its random grids differ and the counts are Rust's own (2,943 failures, SPFA 9,620,007 pops at 600x600 and 60%); the 0-1 BFS pop counts (40,000 and 360,000) are properties of the algorithm and equal Go's. Two baselines are not Go's: the plain queues use the same ring `Deque` (push_back/pop_front only) because an owned `Vec` cannot do Go's `q = q[1:]`, and Dijkstra is a `BinaryHeap<Reverse<(usize, Point)>>` (no boxing into `any`, ties broken by point, same pop count, asserted). Timings differ from Go's: 0-1 BFS was ahead of the heap by 4x to 15x across my runs where Go measured 6.2x to 9.9x, and SPFA stayed slower than the heap at 60% walls.

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

**Rust:**

```rust
use std::collections::VecDeque;
use std::hint::black_box;
use std::time::{Duration, Instant};

// ---- ns/op: the kit's stand-in for testing.Benchmark / b.Loop ----
fn ns_per_op(mut f: impl FnMut()) -> f64 {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            f();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return el.as_nanos() as f64 / n as f64;
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

// ---- RNG: the kit's stand-in for math/rand (splitmix64; seeded, reproducible) ----
// It is NOT Go's generator: the random operation sequences below are different
// sequences from the Go run's, so the numbers that depend on them are Rust's own.
struct Rng(u64);
impl Rng {
    fn new(seed: u64) -> Self {
        Rng(seed)
    }
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
    /// like rand.Intn(n): uniform-enough in [0, n)
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
}

// Two structures that look unrelated and are the same idea:
//
//	QUEUE FROM TWO STACKS   push onto `in_`; when `out` is empty, tip `in_` into
//	                        it. Amortized Theta(1), worst case Theta(n).
//	MIN QUEUE               a queue that also reports its minimum in Theta(1).
//
// The second falls out of the first for free, because lesson 08 already built a
// stack that tracks its own minimum -- and a queue's minimum is just the
// smaller of its two stacks' minima. That composition is the point of this
// example: you do not need a new algorithm, you need the right two pieces.
//
// (`in` is a keyword in Rust, so that stack is the field `in_`. A Vec is the
// stack: push and pop at the end are Theta(1), and pop returns an Option, so
// "the stack is empty" is a value, not a bounds check.)

// ---- queue from two stacks -------------------------------------------------

#[derive(Default)]
struct TwoStack {
    in_: Vec<i64>,
    out: Vec<i64>,
    moved: usize, // total elements ever tipped, for the amortized argument
}

impl TwoStack {
    fn push(&mut self, v: i64) {
        self.in_.push(v);
    }

    fn pop(&mut self) -> Option<i64> {
        if self.out.is_empty() {
            self.tip(); // a no-op when `in_` is empty too: out.pop() is then None
        }
        self.out.pop()
    }

    /// tip is the Theta(n) step -- and it is the ONLY place elements move.
    fn tip(&mut self) {
        while let Some(v) = self.in_.pop() {
            self.out.push(v);
            self.moved += 1;
        }
    }

    fn len(&self) -> usize {
        self.in_.len() + self.out.len()
    }

    fn is_empty(&self) -> bool {
        self.len() == 0
    }
}

// ---- min stack (lesson 08) and the min queue built from two of them --------

#[derive(Default)]
struct MinStack {
    vals: Vec<i64>,
    mins: Vec<i64>, // mins[i] is the minimum of vals[..=i]
}

impl MinStack {
    fn push(&mut self, v: i64) {
        self.vals.push(v);
        let m = self.mins.last().map_or(v, |&m| m.min(v));
        self.mins.push(m);
    }

    fn pop(&mut self) -> Option<i64> {
        self.mins.pop();
        self.vals.pop()
    }

    fn min(&self) -> Option<i64> {
        self.mins.last().copied()
    }

    fn is_empty(&self) -> bool {
        self.vals.is_empty()
    }
}

#[derive(Default)]
struct MinQueue {
    in_: MinStack,
    out: MinStack,
}

impl MinQueue {
    fn push(&mut self, v: i64) {
        self.in_.push(v);
    }

    fn pop(&mut self) -> Option<i64> {
        if self.out.is_empty() {
            while let Some(v) = self.in_.pop() {
                self.out.push(v);
            }
        }
        self.out.pop()
    }

    /// Min is Theta(1): each stack knows its own minimum, and the queue's minimum
    /// is whichever is smaller. No scan, no recomputation.
    fn min(&self) -> Option<i64> {
        match (self.in_.min(), self.out.min()) {
            (Some(a), Some(b)) => Some(a.min(b)),
            (a, b) => a.or(b),
        }
    }

    fn len(&self) -> usize {
        self.in_.vals.len() + self.out.vals.len()
    }
}

// scan_min is the honest baseline: keep the elements, scan them for the minimum.
// Go's queue here is a slice re-sliced with `s.q = s.q[1:]`, an O(1) pop. An owned
// Vec has no such pop: remove(0) would shift everything on every Pop and add an
// O(n) cost of its own to the baseline. A VecDeque pops the front in O(1), which
// is the same cost Go's baseline pays -- so the baseline is a VecDeque.
#[derive(Default)]
struct ScanQueue {
    q: VecDeque<i64>,
}

impl ScanQueue {
    fn push(&mut self, v: i64) {
        self.q.push_back(v);
    }

    fn pop(&mut self) -> Option<i64> {
        self.q.pop_front()
    }

    fn min(&self) -> Option<i64> {
        self.q.iter().copied().min()
    }
}

/// Go's `%v` of a slice: [1 2 3]
fn show(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(ToString::to_string).collect();
    format!("[{}]", parts.join(" "))
}

/// One line of the trace: run `f` on the queue, then print both stacks and its result.
fn step(q: &mut TwoStack, label: &str, f: impl FnOnce(&mut TwoStack) -> String) {
    let r = f(q);
    println!(
        "  {:<10} in={:<16} out={:<16} {}",
        label,
        show(&q.in_),
        show(&q.out),
        r
    );
}

fn main() {
    println!("a queue from two stacks. Push goes on `in`; Pop takes from `out`,");
    println!("and when `out` runs dry the whole of `in` is tipped into it --");
    println!("which reverses the order, which is exactly what a queue needs:");
    println!();
    let mut q = TwoStack::default();
    for v in [1, 2, 3] {
        step(&mut q, &format!("Push({v})"), |q| {
            q.push(v);
            String::new()
        });
    }
    let popped = |q: &mut TwoStack, note: &str| {
        let v = q.pop().expect("the queue is not empty");
        format!("-> {v}{note}")
    };
    step(&mut q, "Pop()", |q| popped(q, "  (tipped)"));
    step(&mut q, "Push(4)", |q| {
        q.push(4);
        String::new()
    });
    step(&mut q, "Pop()", |q| popped(q, ""));
    step(&mut q, "Pop()", |q| popped(q, ""));
    step(&mut q, "Pop()", |q| popped(q, "  (tipped)"));
    println!();
    println!("  note Push(4) landed on `in` while 2 and 3 were still on `out`.");
    println!("  The two stacks are never merged and never sorted; the ordering is");
    println!("  entirely a consequence of tipping ONE at a time.");

    println!();
    println!("the amortized argument, and it is the same one as the monotonic");
    println!("stack in lesson 08 -- each element moves at most twice:");
    println!();
    println!(
        "  {:>10} {:>14} {:>14} {:>16}",
        "operations", "elements tipped", "worst tip", "tips/operation"
    );
    let mut rng = Rng::new(3);
    let mut worsts = Vec::new();
    for n in [1000, 10_000, 100_000, 1_000_000] {
        let mut t = TwoStack::default();
        let mut worst = 0;
        for i in 0..n {
            if rng.below(2) == 0 || t.is_empty() {
                t.push(i);
                continue;
            }
            let before = t.moved;
            t.pop();
            worst = worst.max(t.moved - before);
        }
        println!(
            "  {:>10} {:>14} {:>14} {:>16.2}",
            n,
            t.moved,
            worst,
            t.moved as f64 / n as f64
        );
        worsts.push(worst);
    }
    println!();
    println!("  about 0.50 tips per operation in every row (0.49 at n=1,000, 0.50");
    println!("  after): each element is tipped at most once and about half the");
    println!(
        "  operations are pushes. Yet the WORST single Pop moved {} elements",
        worsts[3]
    );
    println!(
        "  at n=1,000,000 against {} at n=1,000, so that number grows with n",
        worsts[0]
    );
    println!("  while the average does not.");
    println!();
    println!("  that is what amortized means, and why it is not worst case: this");
    println!("  queue is a bad choice for a latency-sensitive path and a fine");
    println!("  choice for total throughput.");
    println!();
    println!("  (a ring buffer is Theta(1) WORST case, so if the tail latency");
    println!("  matters, example 3 is the answer and this one is a curiosity.)");

    println!();
    println!("but the two-stack shape buys something a ring cannot: because each");
    println!("half is a STACK, and a stack can track its own minimum in Theta(1)");
    println!("(lesson 08), the QUEUE gets a Theta(1) minimum for free:");
    println!();
    let mut mq = MinQueue::default();
    let mut sq = ScanQueue::default();
    for v in [5, 2, 8, 1, 9, 3] {
        mq.push(v);
        sq.push(v);
    }
    println!("  {:<24} {:<10} scan", "after", "MinQueue");
    for i in 0..6 {
        let a = mq.min().expect("not empty");
        let b = sq.min().expect("not empty");
        println!("  {:<24} {:<10} {}", format!("{i} pops"), a, b);
        mq.pop();
        sq.pop();
    }
    println!();
    println!("  identical answers. The MinQueue never looks at more than two");
    println!("  numbers to produce them.");

    println!();
    println!("verified on 200,000 random operations before it is timed:");
    println!();
    let mut m = MinQueue::default();
    let mut s = ScanQueue::default();
    let (mut mismatch, mut checks) = (0, 0);
    for _ in 0..200_000 {
        if rng.below(3) > 0 || m.len() == 0 {
            let v = rng.below(1000) as i64;
            m.push(v);
            s.push(v);
        } else if m.pop() != s.pop() {
            mismatch += 1;
        }
        checks += 1;
        if m.min() != s.min() {
            mismatch += 1;
        }
    }
    println!(
        "  {} checks, {} mismatches, final Len {}",
        checks,
        mismatch,
        m.len()
    );

    println!();
    println!("and what the augmentation is worth:");
    println!();
    println!(
        "  {:>10} {:>16} {:>16} {:>10}",
        "live", "MinQueue ns/op", "scan ns/op", "speedup"
    );
    for live in [4, 8, 16, 256, 4096, 65536] {
        let mut mm = MinQueue::default();
        let mut ss = ScanQueue::default();
        for _ in 0..live {
            let v = rng.below(1 << 20) as i64;
            mm.push(v);
            ss.push(v);
        }
        let t1 = ns_per_op(|| {
            mm.push(rng.below(1 << 20) as i64);
            black_box(mm.min());
            black_box(mm.pop());
        });
        let t2 = ns_per_op(|| {
            ss.push(rng.below(1 << 20) as i64);
            black_box(ss.min());
            black_box(ss.pop());
        });
        println!("  {:>10} {:>16.1} {:>16.1} {:>9.1}x", live, t1, t2, t2 / t1);
    }
    println!();
    println!("  (the rows for 4 and 8 live elements are not in the Go example; Go's");
    println!("  claim was about 16, and Rust's crossover turned out to be lower.)");
    println!();
    println!("  the MinQueue is flat and the scan is a straight line: across my runs the");
    println!("  MinQueue stayed at 3 to 14 ns/op at every size with no trend, while the");
    println!("  scan went from about 3 to 9 ns at 4 live elements to 8 to 18 us at 65,536");
    println!("  (roughly 0.1 to 0.3 ns per element, against about 0.5 in Go). They cross");
    println!("  lower than Go's 16: at 4 live the two were level within noise (0.5x to");
    println!("  1.4x), at 8 the MinQueue was level to 1.3x ahead, and at 16 it was ahead");
    println!("  in every run, by 1.1x to 1.5x. From 256 up the gap is 5x to 9x, then 70x");
    println!("  to 145x, then 800x to 2,400x. Plausible reason, not measured: the scan is");
    println!("  a tight loop over contiguous i64 that the compiler can vectorise, so it");
    println!("  is cheap per element and the MinQueue's four Vecs overtake it within a");
    println!("  few elements, not sixteen.");
    println!();
    println!("  this is lesson 08's min-stack finding repeated one level up, and");
    println!("  the general principle behind it is worth naming: AUGMENT A");
    println!("  STRUCTURE WITH THE ANSWER, do not recompute it. The same move");
    println!("  gives you order statistics on a BST (lesson 18), range sums on a");
    println!("  segment tree (lesson 36) and subtree sizes for rank queries.");
    println!();
    println!("  the sliding-window maximum of example 10 is the same problem with");
    println!("  a fixed window instead of an explicit Pop -- and a deque solves it");
    println!("  with ONE array instead of four. Prefer that when the window is");
    println!("  what you actually have.");
}
```

**Rust sample output:**

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
        1000            492             29             0.49
       10000           5031            107             0.50
      100000          49941            542             0.50
     1000000         500020           1074             0.50

  about 0.50 tips per operation in every row (0.49 at n=1,000, 0.50
  after): each element is tipped at most once and about half the
  operations are pushes. Yet the WORST single Pop moved 1074 elements
  at n=1,000,000 against 29 at n=1,000, so that number grows with n
  while the average does not.

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

  200000 checks, 0 mismatches, final Len 65522

and what the augmentation is worth:

        live   MinQueue ns/op       scan ns/op    speedup
           4              3.1              3.5       1.1x
           8              3.4              4.3       1.2x
          16              3.9              4.2       1.1x
         256              3.6             29.1       8.1x
        4096              4.0            523.5     130.8x
       65536              3.8           8405.3    2230.4x

  (the rows for 4 and 8 live elements are not in the Go example; Go's
  claim was about 16, and Rust's crossover turned out to be lower.)

  the MinQueue is flat and the scan is a straight line: across my runs the
  MinQueue stayed at 3 to 14 ns/op at every size with no trend, while the
  scan went from about 3 to 9 ns at 4 live elements to 8 to 18 us at 65,536
  (roughly 0.1 to 0.3 ns per element, against about 0.5 in Go). They cross
  lower than Go's 16: at 4 live the two were level within noise (0.5x to
  1.4x), at 8 the MinQueue was level to 1.3x ahead, and at 16 it was ahead
  in every run, by 1.1x to 1.5x. From 256 up the gap is 5x to 9x, then 70x
  to 145x, then 800x to 2,400x. Plausible reason, not measured: the scan is
  a tight loop over contiguous i64 that the compiler can vectorise, so it
  is cheap per element and the MinQueue's four Vecs overtake it within a
  few elements, not sixteen.

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

> **Go vs Rust:** The push/pop/tip trace and the min table are identical to Go's. With Go's `math/rand` generator fed in (a throwaway build, not shipped) this program reproduced Go's tip table (498 / 21, 4,967 / 143, 49,997 / 493, 499,980 / 577) and its `final Len 66406` exactly; the shipped kit `Rng` is splitmix64, so its sequence differs (492 tips and a worst of 29 at n=1,000, 1,074 at n=1,000,000, `final Len 65522`). Go's "exactly 0.50" does not carry over to 0.49 at n=1,000, and Go's "they cross at about 16 live elements, below that the scan wins" does not either: here the MinQueue was already ahead at 16 (1.1x to 1.5x) and level at 4 to 8, so rows 4 and 8 were added; at 65,536 the speedup was 800x to 2,400x against Go's 3,993x, because Rust's scan costs roughly 0.1 to 0.3 ns per element against Go's 0.5 (vectorisation is a plausible reason, not measured). `in` is a keyword (the field is `in_`), `Vec::pop` returns an `Option` so the empty checks vanish, and the scan baseline is a `VecDeque` rather than Go's re-sliced `q = q[1:]` (`remove(0)` would add an O(n) shift to every pop).

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

**Rust:**

```rust
use std::collections::VecDeque;
use std::hint::black_box;
use std::sync::atomic::{
    AtomicI64, AtomicU64,
    Ordering::{Acquire, Relaxed, Release},
};
use std::sync::{Mutex, MutexGuard};
use std::thread;
use std::time::{Duration, Instant};

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
//
// Rust differences, in the order they show up below:
//   * the Mutex OWNS the deque. In Go `mu` and `buf` are two fields and nothing
//     makes you lock before you touch `buf`; here the only way to reach the
//     VecDeque is through the guard that lock() returns.
//   * the buffer is std's VecDeque, itself a ring buffer. Go's popFront is
//     `d.buf = d.buf[1:]`, which abandons the front of the array (example 1);
//     VecDeque reuses it.
//   * workers are std::thread::scope threads that BORROW the pool. No Arc, no
//     'static: the scope guarantees every thread has joined before the pool is
//     touched again.
//   * `go run -race` has no counterpart. Safe Rust rejects the data race at
//     compile time: a `let mut sum = 0i64;` that several spawned closures do
//     `sum += x` on is error E0499 (second mutable borrow; with `move` the i64
//     is merely copied into each closure and the total never changes, which
//     rustc warns about). So the shared counters below have to be atomics (or
//     sit behind a Mutex).

#[derive(Clone, Copy)]
struct Task {
    lo: i64,
    hi: i64,
} // sum the range [lo, hi)

struct WorkDeque {
    buf: Mutex<VecDeque<Task>>,
}

impl WorkDeque {
    fn new() -> Self {
        WorkDeque {
            buf: Mutex::new(VecDeque::new()),
        }
    }

    // The guard is a temporary: the lock is released at the end of each
    // statement below, which is what Go's `defer d.mu.Unlock()` is for.
    fn lock(&self) -> MutexGuard<'_, VecDeque<Task>> {
        self.buf
            .lock()
            .expect("a worker panicked while holding the lock")
    }

    // push_back / pop_back: the owner's end.
    fn push_back(&self, t: Task) {
        self.lock().push_back(t);
    }
    fn pop_back(&self) -> Option<Task> {
        self.lock().pop_back()
    }

    // pop_front: the thief's end.
    fn pop_front(&self) -> Option<Task> {
        self.lock().pop_front()
    }
    // Go also defines len(); nothing calls it, and clippy -D warnings rejects
    // dead code, so it is omitted.
}

const THRESHOLD: i64 = 4096; // below this, do the work rather than splitting

type StealFn = fn(&Pool, usize) -> Option<Task>;

struct Pool {
    deques: Vec<WorkDeque>,
    // Memory orderings, stated once:
    //  * sum, steals, locals: Relaxed. They are only totals; nobody makes a
    //    decision from them while the workers run. main reads them after
    //    thread::scope returns, and joining a thread synchronises with
    //    everything that thread did, so the final loads see every update.
    //  * active: the termination counter (tasks created and not yet finished).
    //    Decrement with Release, read with Acquire, so a worker that sees 0
    //    also sees the effects of the work that brought it to 0.
    //    The increment can be Relaxed: it is followed by a push (mutex unlock)
    //    and whoever pops that task locks the same mutex, so the increment
    //    happens-before any decrement for the tasks it created. active can
    //    therefore never read 0 while a split is half-done.
    sum: AtomicI64,
    active: AtomicI64,
    steals: AtomicU64,
    locals: AtomicU64,
    // not in the Go example: how big the stolen tasks were -- total elements
    // covered, and how many steals took a task that was already a leaf (small
    // enough to run, not split). Relaxed, like the other statistics.
    stolen_elems: AtomicU64,
    stolen_leaves: AtomicU64,
    steal_fn: StealFn,
}

fn work(t: Task) -> i64 {
    let mut s: i64 = 0;
    for i in t.lo..t.hi {
        // black_box: LLVM turns `s += i` over a range into the closed form
        // (lo+hi-1)*(hi-lo)/2 and the loop vanishes (measured: 2 ns for all
        // 40,000,000 additions). Go's compiler does not, so to keep the same
        // amount of real work per leaf the addend is hidden from the optimiser.
        s += black_box(i);
    }
    s
}

impl Pool {
    fn run(&self, id: usize) {
        let d = &self.deques[id];
        loop {
            let task = match d.pop_back() {
                // 1. own work first, LIFO
                Some(t) => {
                    self.locals.fetch_add(1, Relaxed);
                    Some(t)
                }
                // 2. otherwise steal
                None => {
                    let t = (self.steal_fn)(self, id);
                    if let Some(t) = t {
                        self.steals.fetch_add(1, Relaxed);
                        self.stolen_elems.fetch_add((t.hi - t.lo) as u64, Relaxed);
                        if t.hi - t.lo <= THRESHOLD {
                            self.stolen_leaves.fetch_add(1, Relaxed);
                        }
                    }
                    t
                }
            };
            let Some(t) = task else {
                if self.active.load(Acquire) == 0 {
                    return;
                }
                thread::yield_now();
                continue;
            };
            if t.hi - t.lo <= THRESHOLD {
                self.sum.fetch_add(work(t), Relaxed);
                self.active.fetch_sub(1, Release);
                continue;
            }
            let mid = (t.lo + t.hi) / 2;
            self.active.fetch_add(1, Relaxed);
            d.push_back(Task { lo: t.lo, hi: mid });
            d.push_back(Task { lo: mid, hi: t.hi });
        }
    }
}

// steal_front takes the OLDEST task from a victim -- the correct end.
fn steal_front(p: &Pool, id: usize) -> Option<Task> {
    let n = p.deques.len();
    (1..n).find_map(|i| p.deques[(id + i) % n].pop_front())
}

// steal_back takes the NEWEST task -- the same end the owner is using. Still
// correct, and measurably worse.
fn steal_back(p: &Pool, id: usize) -> Option<Task> {
    let n = p.deques.len();
    (1..n).find_map(|i| p.deques[(id + i) % n].pop_back())
}

fn run_pool(workers: usize, n: i64, steal: StealFn) -> Pool {
    let pool = Pool {
        deques: (0..workers).map(|_| WorkDeque::new()).collect(),
        sum: AtomicI64::new(0),
        active: AtomicI64::new(1),
        steals: AtomicU64::new(0),
        locals: AtomicU64::new(0),
        stolen_elems: AtomicU64::new(0),
        stolen_leaves: AtomicU64::new(0),
        steal_fn: steal,
    };
    pool.deques[0].push_back(Task { lo: 0, hi: n });

    // Go: wg.Add / go func / wg.Wait. The scope joins every spawned thread
    // before it returns, and lets them borrow `pool` from this stack frame.
    thread::scope(|s| {
        for id in 0..workers {
            let pool = &pool;
            s.spawn(move || pool.run(id));
        }
    });
    pool
}

/// run_pool plus the correctness check, so that every run -- the measured ones
/// too -- is checked, not just the first. (Go checks the first run of each
/// configuration and then benchmarks unchecked.)
fn run_checked(workers: usize, n: i64, steal: StealFn) -> Pool {
    let pool = run_pool(workers, n, steal);
    // Relaxed is enough: thread::scope has joined every worker.
    assert_eq!(pool.sum.load(Relaxed), n * (n - 1) / 2, "wrong sum");
    pool
}

/// ns per call of `run`, measured for about 200 ms (kit: stand-in for testing.Benchmark).
fn ns_per_op(mut run: impl FnMut()) -> f64 {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            run();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return el.as_nanos() as f64 / n as f64;
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

fn main() {
    const N: i64 = 40_000_000;

    println!("summing 0..{N} across workers, each owning a deque.");
    println!("The task is split in half and both halves pushed back, so the");
    println!("deque holds a tree of pending subranges.");
    println!();
    println!(
        "  {:>8} {:>14} {:>14} {:>14} {:>10}",
        "workers", "local pops", "steals", "steal rate", "ns"
    );
    for w in [1, 2, 4, 8] {
        let p = run_checked(w, N, steal_front);
        let t = ns_per_op(|| {
            black_box(run_checked(w, N, steal_front));
        });
        let (locals, steals) = (p.locals.load(Relaxed), p.steals.load(Relaxed));
        println!(
            "  {:>8} {:>14} {:>14} {:>13.2}% {:>10.0}",
            w,
            locals,
            steals,
            100.0 * steals as f64 / (locals + steals) as f64,
            t
        );
    }
    println!();
    println!("  every row adds up to 32,767 tasks (a full binary tree: 16,384 leaves");
    println!("  of about 2,441 elements, plus the 16,383 splits above them), and the");
    println!("  split between local pops and steals is the number that matters.");
    println!("  A steal is the one moment two workers touch the same deque (and,");
    println!("  plausibly, the same cache line; not measured), and it stays a small");
    println!("  fraction of operations. Across 32 runs the steals were: 0 at 1");
    println!("  worker, 3 to 11 at 2 workers (0.03% at most), 13 to 41 at 4 (0.13%)");
    println!("  and 25 to 180 at 8 (0.55%). The third table below measures why: each");
    println!("  steal hands the thief a large subtree.");
    println!();
    println!("that only works because the thief takes the FRONT. Stealing from");
    println!("the back -- the same end the owner uses -- is still correct:");
    println!();
    println!(
        "  {:>8} {:>15} {:>15} {:>13} {:>13} {:>8}",
        "workers", "steal FRONT ns", "steal BACK ns", "front steals", "back steals", "ratio"
    );
    let mut sizes = Vec::new();
    for w in [2, 4, 8] {
        let pf = run_checked(w, N, steal_front);
        let pb = run_checked(w, N, steal_back);
        let tf = ns_per_op(|| {
            black_box(run_checked(w, N, steal_front));
        });
        let tb = ns_per_op(|| {
            black_box(run_checked(w, N, steal_back));
        });
        let (front, back) = (pf.steals.load(Relaxed), pb.steals.load(Relaxed));
        println!(
            "  {:>8} {:>15.0} {:>15.0} {:>13} {:>13} {:>7.0}x",
            w,
            tf,
            tb,
            front,
            back,
            back as f64 / front.max(1) as f64
        );
        // (average elements per stolen task, % of steals that took a leaf)
        let size = |p: &Pool| {
            let steals = p.steals.load(Relaxed).max(1) as f64;
            (
                p.stolen_elems.load(Relaxed) as f64 / steals,
                100.0 * p.stolen_leaves.load(Relaxed) as f64 / steals,
            )
        };
        sizes.push((w, size(&pf), size(&pb)));
    }
    println!();
    println!("  both produce the correct sum -- checked on every run, the timed ones");
    println!("  too (run_checked asserts inside the benchmark closure). Read the two");
    println!("  right-hand columns: across 32 runs back-stealing needed 10x to 815x");
    println!("  more steals, between one and three orders of magnitude (the ratio");
    println!("  shrinks as workers are added: 47x-815x at 2 workers, 42x-249x at 4,");
    println!("  10x-163x at 8). The table after this one shows what a steal took.");
    println!();
    println!("  and the clock moves, modestly. Back-stealing was slower in 74 of the");
    println!("  96 (workers, run) cells, with a median penalty of 7% to 12% at each");
    println!("  worker count; single cells ranged from 24% faster to 53% slower (the");
    println!("  machine is shared), so read it as 'a few percent to about a tenth'.");
    println!("  That is small next to a 10x-815x difference in steals because each");
    println!("  leaf task does only 2,441 additions (about 1 us: the 1-worker row,");
    println!("  13 to 17 ms, over 16,384 leaves), so a few thousand extra steals are");
    println!("  milliseconds at worst. Making the leaves cheaper should make the");
    println!("  steal count what you pay; that is a prediction, not measured here.");
    println!();
    println!("what one steal buys (not in the Go example): the average number of");
    println!("elements covered by a stolen task, from the same runs as above:");
    println!();
    println!(
        "  {:>8} {:>21} {:>8} {:>21} {:>8}",
        "workers", "FRONT avg elements", "% leaf", "BACK avg elements", "% leaf"
    );
    for (w, (fe, fl), (be, bl)) in sizes {
        println!("  {w:>8} {fe:>21.0} {fl:>7.1}% {be:>21.0} {bl:>7.1}%");
    }
    println!();
    println!("  that is the mechanism, measured. Across 32 runs a front steal took");
    println!("  0.6 to 9 million elements on average, between 10x and 860x what a");
    println!("  back steal took in the same cell (8 thousand to 100 thousand). A task");
    println!("  is a leaf when it is small enough to run without splitting. At 4 and");
    println!("  8 workers, where there are enough steals for a percentage to mean");
    println!("  something, 0% to 24% of front steals took a leaf against 45% to 64%");
    println!("  of back steals: a thief that took the back often got work it");
    println!("  finished in about a microsecond, and was back at once.");
    println!();
    println!("why LIFO for the owner and FIFO for the thief, stated plainly:");
    println!();
    let row = |role: &str, end: &str, gets: &str, because: &str| {
        println!("  {role:<12} {end:<10} {gets:<30} {because}");
    };
    row("role", "end", "gets", "because");
    row(
        "owner",
        "back",
        "the task it just created",
        "it is still in L1",
    );
    row(
        "thief",
        "front",
        "the oldest, largest subtree",
        "one steal lasts",
    );
    println!();
    println!("  in a divide-and-conquer workload the oldest task is the one that");
    println!("  has been split the fewest times, so it is the biggest. The deque");
    println!("  is sorted by size for free, purely as a side effect of the order");
    println!("  things were pushed in -- no priority queue required.");
    println!();
    println!("  that is the deepest reason this lesson exists. A queue and a stack");
    println!("  are not two structures to choose between: OPENING BOTH ENDS lets");
    println!("  two different consumers each get the order that suits them, from");
    println!("  one array, without either one having to coordinate with the other.");

    println!();
    println!("  (Go's own scheduler does exactly this: each P has a 256-slot");
    println!("  run queue, runs it LIFO-ish, and idle Ps steal half of another");
    println!("  P's queue from the front. `runtime/proc.go`, runqsteal.)");
}
```

**Rust sample output:**

```
summing 0..40000000 across workers, each owning a deque.
The task is split in half and both halves pushed back, so the
deque holds a tree of pending subranges.

   workers     local pops         steals     steal rate         ns
         1          32767              0          0.00%   14744439
         2          32756             11          0.03%    8929476
         4          32744             23          0.07%    5774131
         8          32656            111          0.34%    4263034

  every row adds up to 32,767 tasks (a full binary tree: 16,384 leaves
  of about 2,441 elements, plus the 16,383 splits above them), and the
  split between local pops and steals is the number that matters.
  A steal is the one moment two workers touch the same deque (and,
  plausibly, the same cache line; not measured), and it stays a small
  fraction of operations. Across 32 runs the steals were: 0 at 1
  worker, 3 to 11 at 2 workers (0.03% at most), 13 to 41 at 4 (0.13%)
  and 25 to 180 at 8 (0.55%). The third table below measures why: each
  steal hands the thief a large subtree.

that only works because the thief takes the FRONT. Stealing from
the back -- the same end the owner uses -- is still correct:

   workers  steal FRONT ns   steal BACK ns  front steals   back steals    ratio
         2         8564326         9600373             5          1519     304x
         4         5872483         7049831            19          2327     122x
         8         4766923         4489665            50          2383      48x

  both produce the correct sum -- checked on every run, the timed ones
  too (run_checked asserts inside the benchmark closure). Read the two
  right-hand columns: across 32 runs back-stealing needed 10x to 815x
  more steals, between one and three orders of magnitude (the ratio
  shrinks as workers are added: 47x-815x at 2 workers, 42x-249x at 4,
  10x-163x at 8). The table after this one shows what a steal took.

  and the clock moves, modestly. Back-stealing was slower in 74 of the
  96 (workers, run) cells, with a median penalty of 7% to 12% at each
  worker count; single cells ranged from 24% faster to 53% slower (the
  machine is shared), so read it as 'a few percent to about a tenth'.
  That is small next to a 10x-815x difference in steals because each
  leaf task does only 2,441 additions (about 1 us: the 1-worker row,
  13 to 17 ms, over 16,384 leaves), so a few thousand extra steals are
  milliseconds at worst. Making the leaves cheaper should make the
  steal count what you pay; that is a prediction, not measured here.

what one steal buys (not in the Go example): the average number of
elements covered by a stolen task, from the same runs as above:

   workers    FRONT avg elements   % leaf     BACK avg elements   % leaf
         2               4021972    20.0%                 38316    58.6%
         4               2817897     0.0%                 19345    58.5%
         8               1405176     4.0%                 25871    57.2%

  that is the mechanism, measured. Across 32 runs a front steal took
  0.6 to 9 million elements on average, between 10x and 860x what a
  back steal took in the same cell (8 thousand to 100 thousand). A task
  is a leaf when it is small enough to run without splitting. At 4 and
  8 workers, where there are enough steals for a percentage to mean
  something, 0% to 24% of front steals took a leaf against 45% to 64%
  of back steals: a thief that took the back often got work it
  finished in about a microsecond, and was back at once.

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

> **Go vs Rust:** The task count (32,767) and the steal counts are the same order as Go's (Go: 0 / 4-8 / 18-20 / 110-116 steals at 1 / 2 / 4 / 8 workers; Rust over 32 runs: 0 / 3-11 / 13-41 / 25-180), and `std::sync::Mutex` owns its `VecDeque`, so the deque cannot be touched without the lock, while `go run -race` has no counterpart because a shared `sum += x` across spawned threads is compile error E0499. LLVM replaces `s += i` over a range by its closed form (2 ns for all 40,000,000 additions, measured in a scratch program), so the addend goes through `black_box` to keep Go's real work per leaf; with it one worker took 13-17 ms against Go's 10.6 ms. One Go claim does not carry over: "the clock barely moves, a few percent" -- back-stealing was slower in 74 of 96 cells here, median 7-12% (cells from 24% faster to 53% slower), mechanism not measured; the third table, which measures the size of a stolen task, is not in the Go example.

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

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::VecDeque;
use std::fmt::Display;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// ---- the kit's allocation counter: stand-in for testing.AllocsPerRun and
// runtime.ReadMemStats. Go reads HeapAlloc after runtime.GC(); Rust has no GC,
// so the allocator itself counts allocation events and the bytes currently
// live -- exact and deterministic.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)
static LIVE: AtomicUsize = AtomicUsize::new(0); // bytes currently live

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        LIVE.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        LIVE.fetch_add(new, Relaxed);
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// allocation events while running `f`.
fn allocs_of(f: impl FnOnce()) -> usize {
    let before = ALLOCS.load(Relaxed);
    f();
    ALLOCS.load(Relaxed) - before
}
/// bytes currently live on the heap (a deterministic MemStats.HeapAlloc).
fn live_bytes() -> usize {
    LIVE.load(Relaxed)
}

/// ns per call of `run`, measured for about 200 ms (kit: stand-in for testing.Benchmark).
fn ns_per_op(mut run: impl FnMut()) -> f64 {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            run();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return el.as_nanos() as f64 / n as f64;
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

// ---- the kit's RNG: stand-in for math/rand (splitmix64; seeded, so runs are
// reproducible -- but its digits are NOT Go's, so anything RNG-dependent
// below differs from the Go output).
struct Rng(u64);
impl Rng {
    fn new(seed: u64) -> Self {
        Rng(seed)
    }
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
    /// like rand.Intn(n): uniform-enough in [0, n)
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
}

// The capstone, and the three-pass method end to end:
//
//	BUILD IT    Deque<T> -- one array, both ends, generic, allocation-free
//	            in the steady state
//	PROVE IT    a model oracle over 200,000 random operations, an invariant
//	            check after every one, and a rotation sweep that reaches the
//	            wrapped states no hand-written test ever will
//	MEASURE IT  allocation budget, worst-case latency, and the two consumers
//	            from this lesson (BFS and sliding-window maximum)
//
// Everything below is the lesson's findings, assembled.

// ---- BUILD IT --------------------------------------------------------------

// Same type as example 5: the slots are Option<T>, because Rust has no universal
// zero value to park in an empty slot and a Vec<T> cannot hold holes. None is
// the hole, and take() is how a pop leaves one behind.
struct Deque<T> {
    buf: Vec<Option<T>>,
    mask: usize,
    head: usize,
    count: usize,
    grows: usize,
}

impl<T> Deque<T> {
    /// new rounds the capacity UP to a power of two, which is not an
    /// optimisation -- (head-1)&mask depends on it (example 5). Go loops
    /// `n <<= 1`; Rust has it built in (and 0.next_power_of_two() is 1, which
    /// is what Go's max(capacity, 1) is for).
    fn new(capacity: usize) -> Self {
        let n = capacity.next_power_of_two();
        Deque {
            buf: std::iter::repeat_with(|| None).take(n).collect(),
            mask: n - 1,
            head: 0,
            count: 0,
            grows: 0,
        }
    }

    fn len(&self) -> usize {
        self.count
    }
    fn cap(&self) -> usize {
        self.buf.len()
    }

    /// grow doubles and UNWRAPS. Two copies, because a wrapped live region is
    /// at most two contiguous pieces (example 4). T need not be Copy, so the
    /// pieces are MOVED out of the old array with drain/append, not copied.
    fn grow(&mut self) {
        let doubled = self.buf.len() * 2;
        let mut old = std::mem::take(&mut self.buf);
        let mut next = Vec::with_capacity(doubled);
        next.extend(old.drain(self.head..)); // the piece before the wrap
        next.append(&mut old); // the piece after it
        next.resize_with(doubled, || None);
        self.buf = next;
        self.mask = doubled - 1;
        self.head = 0;
        self.grows += 1;
    }

    fn push_back(&mut self, v: T) {
        if self.count == self.buf.len() {
            self.grow();
        }
        let i = (self.head + self.count) & self.mask;
        self.buf[i] = Some(v);
        self.count += 1;
    }

    fn push_front(&mut self, v: T) {
        if self.count == self.buf.len() {
            self.grow();
        }
        // head is a usize: wrapping_sub makes the wrap past zero explicit.
        self.head = self.head.wrapping_sub(1) & self.mask;
        self.buf[self.head] = Some(v);
        self.count += 1;
    }

    fn pop_front(&mut self) -> Option<T> {
        if self.count == 0 {
            return None;
        }
        // take() moves the value out AND leaves None behind: Go's
        // `v := buf[head]; buf[head] = zero` is one operation here (example 5).
        let v = self.buf[self.head].take();
        self.head = (self.head + 1) & self.mask;
        self.count -= 1;
        v
    }

    fn pop_back(&mut self) -> Option<T> {
        if self.count == 0 {
            return None;
        }
        let i = (self.head + self.count - 1) & self.mask;
        self.count -= 1;
        self.buf[i].take()
    }

    /// Indexes logically; None when i is not a live position (Go: `ok` false).
    fn at(&self, i: usize) -> Option<&T> {
        if i >= self.count {
            return None;
        }
        self.buf[(self.head + i) & self.mask].as_ref()
    }

    /// Go writes `d.At(d.Len()-1)`, which on an empty deque is At(-1) -> false.
    /// With usize, `len() - 1` would underflow, so the subtraction is checked.
    fn back(&self) -> Option<&T> {
        self.count.checked_sub(1).and_then(|i| self.at(i))
    }

    /// The live region in order, copied out. Two pieces, as in grow: the
    /// physical array from head to its end, then from its start to the wrap.
    fn slice(&self) -> Vec<T>
    where
        T: Clone,
    {
        let first_end = (self.head + self.count).min(self.buf.len());
        let wrapped = self.count - (first_end - self.head);
        self.buf[self.head..first_end]
            .iter()
            .chain(&self.buf[..wrapped])
            .flatten()
            .cloned()
            .collect()
    }

    // ---- PROVE IT: the invariants ----------------------------------------------

    /// invariants must hold after EVERY operation. They are cheap, so there is
    /// no excuse for not checking them in a test.
    fn invariants(&self) -> Result<(), String> {
        let n = self.buf.len();
        if !n.is_power_of_two() {
            return Err(format!("capacity {n} is not a power of two"));
        }
        if self.mask != n - 1 {
            return Err(format!("mask {} does not match capacity {n}", self.mask));
        }
        // Go also checks `head < 0` and `count < 0`; a usize cannot be negative,
        // so those two checks are impossible to fail and are not written.
        if self.head >= n {
            return Err(format!("head {} out of range [0,{n})", self.head));
        }
        if self.count > n {
            return Err(format!("count {} out of range [0,{n}]", self.count));
        }
        Ok(())
    }

    /// Not in the Go example, and O(capacity) rather than cheap, so it is run
    /// on selected states, not after every operation: exactly the `count`
    /// live positions hold Some and every other slot is None. In Go this is
    /// the invisible half of "Pop writes the zero value" -- a stale element
    /// left in a dead slot is undetectable from outside; here it can be
    /// checked, and take() makes it impossible to get wrong by forgetting.
    fn dead_slots_empty(&self) -> bool {
        (0..self.buf.len()).all(|p| {
            let live = (p.wrapping_sub(self.head) & self.mask) < self.count;
            self.buf[p].is_some() == live
        })
    }
}

// model is the obviously-correct reference: a plain Vec, no cleverness. Go's
// PushFront allocates a new slice; Vec::insert(0, v) shifts in place -- a
// different cost, the same answers, and only the answers are used here.
struct Model<T> {
    s: Vec<T>,
}

impl<T> Model<T> {
    fn push_back(&mut self, v: T) {
        self.s.push(v);
    }
    fn push_front(&mut self, v: T) {
        self.s.insert(0, v);
    }
    fn pop_front(&mut self) -> Option<T> {
        if self.s.is_empty() {
            None
        } else {
            Some(self.s.remove(0))
        }
    }
    fn pop_back(&mut self) -> Option<T> {
        self.s.pop()
    }
}

// Go needs an equal([]int, []int) helper; Rust's slices and Vecs have ==.

// ---- the two consumers, rebuilt on the finished type ----------------------

/// Sliding-window maximum with the deque of indices. An empty Vec stands for
/// Go's nil (k <= 0, or a window longer than the input).
fn max_sliding(a: &[i64], k: usize) -> Vec<i64> {
    if k == 0 || a.len() < k {
        return Vec::new();
    }
    let mut out = Vec::with_capacity(a.len() - k + 1);
    let mut d: Deque<usize> = Deque::new(k + 1);
    for (i, &v) in a.iter().enumerate() {
        // Go: `f <= i-k`. i-k would underflow a usize while i < k.
        if d.at(0).is_some_and(|&f| f + k <= i) {
            d.pop_front();
        }
        while let Some(&b) = d.back() {
            if a[b] > v {
                break;
            }
            d.pop_back();
        }
        d.push_back(i);
        if i + 1 >= k
            && let Some(&f) = d.at(0)
        {
            out.push(a[f]);
        }
    }
    out
}

/// The O(n*k) reference: Go's two nested loops, the inner one over a subslice.
/// Not `windows(k).map(|w| w.iter().max())`: that spelling measured 4x to 5x
/// slower than this plain loop at k >= 128 in a scratch program (why was not
/// investigated), and a baseline that slow would inflate the speedup column.
fn max_sliding_brute(a: &[i64], k: usize) -> Vec<i64> {
    if k == 0 || a.len() < k {
        return Vec::new();
    }
    let mut out = Vec::with_capacity(a.len() - k + 1);
    for i in 0..=a.len() - k {
        let mut m = a[i];
        for &x in &a[i + 1..i + k] {
            if x > m {
                m = x;
            }
        }
        out.push(m);
    }
    out
}

/// Go's `%v` of a slice: [1 2 3]
fn show<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(ToString::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    println!("PASS 1 -- BUILD IT");
    println!();
    let mut d: Deque<&str> = Deque::new(2);
    d.push_back("b");
    d.push_back("c");
    d.push_front("a");
    d.push_back("d");
    println!(
        "  {}   Len={} Cap={} (grew {} time(s) from 2)",
        show(&d.slice()),
        d.len(),
        d.cap(),
        d.grows
    );
    let f = d.pop_front().expect("deque has 4 elements");
    let bk = d.pop_back().expect("deque has 3 elements");
    println!("  PopFront={f:?} PopBack={bk:?} -> {}", show(&d.slice()));

    println!();
    println!("PASS 2 -- PROVE IT");
    println!();

    // 2a. model oracle, with the invariants checked after every operation
    let mut rng = Rng::new(9);
    let mut dq: Deque<i64> = Deque::new(1);
    let mut md = Model { s: Vec::new() };
    const OPS: usize = 200_000;
    let (mut mismatches, mut inv_fails) = (0, 0);
    for _ in 0..OPS {
        match rng.below(4) {
            0 => {
                let v = rng.below(1000) as i64;
                dq.push_back(v);
                md.push_back(v);
            }
            1 => {
                let v = rng.below(1000) as i64;
                dq.push_front(v);
                md.push_front(v);
            }
            2 => {
                // one comparison covers Go's `okA != okB || a != b`
                if dq.pop_front() != md.pop_front() {
                    mismatches += 1;
                }
            }
            _ => {
                if dq.pop_back() != md.pop_back() {
                    mismatches += 1;
                }
            }
        }
        if dq.invariants().is_err() {
            inv_fails += 1;
        }
        if dq.len() != md.s.len() {
            mismatches += 1;
        }
    }
    if dq.slice() != md.s {
        mismatches += 1;
    }
    println!(
        "  {:<46} {} mismatches, {} invariant failures",
        format!("{OPS} random ops against a slice model:"),
        mismatches,
        inv_fails
    );

    // 2b. the rotation sweep -- the states no hand-written test reaches
    let mut rot_fails = 0;
    let mut slot_fails = usize::from(!dq.dead_slots_empty()); // 2a's final state
    for rot in 0..64 {
        let mut r: Deque<i64> = Deque::new(8);
        for _ in 0..rot {
            // drive head to an arbitrary offset
            r.push_back(-1);
            r.pop_front();
        }
        for i in 0..20 {
            // now force two grows from a wrapped state
            r.push_back(i);
        }
        let want: Vec<i64> = (0..20).collect();
        if r.slice() != want {
            rot_fails += 1;
        }
        if !r.dead_slots_empty() {
            slot_fails += 1;
        }
    }
    println!(
        "  {:<46} {} failures",
        "64 starting rotations, then grow:", rot_fails
    );
    println!(
        "  {:<46} {} failures",
        "empty slots hold None (not in Go; 65 states):", slot_fails
    );

    // 2c. the consumer, proved against brute force
    let (mut sw_fails, mut sw_cases) = (0, 0);
    for _ in 0..2000 {
        let n = 1 + rng.below(24) as usize;
        // small range: forces ties
        let arr: Vec<i64> = (0..n).map(|_| rng.below(15) as i64 - 7).collect();
        for k in 1..=n {
            sw_cases += 1;
            if max_sliding(&arr, k) != max_sliding_brute(&arr, k) {
                sw_fails += 1;
            }
        }
    }
    println!(
        "  {:<46} {} failures",
        format!("sliding-window max, {sw_cases} (array,k) pairs:"),
        sw_fails
    );

    println!();
    println!("  three layers, and each catches what the others cannot: the model");
    println!("  finds wrong ANSWERS, the invariants find corrupt STATE the model");
    println!("  has not surfaced yet, and the rotation sweep reaches the wrapped");
    println!("  configurations that both would otherwise never visit.");

    println!();
    println!("PASS 3 -- MEASURE IT");
    println!();

    // 3a. allocation budget
    let mut warm: Deque<i64> = Deque::new(4096);
    for i in 0..1000 {
        warm.push_back(i);
    }
    let allocs = allocs_of(|| {
        for i in 0..100_000 {
            warm.push_back(i);
            black_box(warm.pop_front());
        }
    });
    println!(
        "  {:<46} {}",
        "allocations, 100,000 steady-state ops:", allocs
    );

    // 3b. steady state; Go's comment says "vs the naive queue", but it only
    // times the deque, and so does this. std's VecDeque is added for scale.
    const STEADY: usize = 500_000;
    let t_deque = ns_per_op(|| {
        let mut q: Deque<i64> = Deque::new(4096);
        for i in 0..1000 {
            q.push_back(i);
        }
        for i in 0..STEADY as i64 {
            q.push_back(i);
            black_box(q.pop_front());
        }
    });
    let t_std = ns_per_op(|| {
        let mut q: VecDeque<i64> = VecDeque::with_capacity(4096);
        for i in 0..1000 {
            q.push_back(i);
        }
        for i in 0..STEADY as i64 {
            q.push_back(i);
            black_box(q.pop_front());
        }
    });
    // Go's label says "per operation" but divides by the iterations, and an
    // iteration is a push AND a pop; the label here says what is measured.
    println!(
        "  {:<46} {:.2} ns",
        "steady state, per push+pop pair:",
        t_deque / STEADY as f64
    );
    println!(
        "  {:<46} {:.2} ns",
        "  the same, std VecDeque (not in Go):",
        t_std / STEADY as f64
    );

    // 3c. worst-case latency -- the thing amortized analysis hides
    let mut q: Deque<i64> = Deque::new(1);
    let (mut worst_grow, mut worst_at) = (0, 0);
    for i in 0..1_000_000 {
        let before = q.grows;
        q.push_back(i);
        if q.grows > before && q.cap() > worst_grow {
            worst_grow = q.cap();
            worst_at = i;
        }
    }
    println!(
        "  {:<46} {} elements copied at push {}",
        "worst single PushBack:",
        worst_grow / 2,
        worst_at
    );

    // 3d. memory held after a burst and a full drain. Slots are Option<i64>,
    // which has no niche to hide its tag in: 16 bytes against Go's 8.
    let mb = |bytes: usize| bytes as f64 / (1 << 20) as f64;
    let base = live_bytes();
    let mut burst: Deque<i64> = Deque::new(8);
    for i in 0..4_000_000 {
        burst.push_back(i);
    }
    let peak = live_bytes() - base;
    while burst.pop_front().is_some() {}
    let held = live_bytes() - base;
    black_box(&burst); // Go's runtime.KeepAlive: measured while the deque is alive
    println!(
        "  {:<46} {:.1} MB peak, {:.1} MB still held at Len 0",
        "burst of 4,000,000 then drained:",
        mb(peak),
        mb(held)
    );
    println!(
        "  {:<46} {} bytes per slot (Option<i64>); Go's int slot is 8",
        "",
        size_of::<Option<i64>>()
    );

    println!();
    println!("  that last line is example 1's finding, in the finished type, and");
    println!("  it is deliberate: this deque grows and never shrinks. If your");
    println!("  workload has rare huge bursts, add lesson 06's hysteresis (shrink");
    println!("  at 25% full, halve only) -- and measure that it does not thrash.");
    println!();
    println!("  two Rust-specific readings of the lines above. The slot is an");
    println!("  Option<i64>, 16 bytes, so the same 4,194,304-slot array that is 32 MB");
    println!("  in Go is 64 MB here, and the worst push (524,288 elements) moves");
    println!("  8 MB of slots, not 4. The element counts and the grow history are");
    println!("  identical to Go's. And 'peak' is the live bytes right after the burst:");
    println!("  by construction grow holds the old and the new array at once, so the");
    println!("  true high-water mark was 32 + 64 MB at the last doubling (arithmetic");
    println!("  from the sizes, not measured; Go's 'peak' is the same kind of figure).");
    println!();
    println!("  the steady-state lines compare two different programs, so read them as");
    println!("  measurements, not a verdict. Across more than 30 runs this ring took");
    println!("  0.8 to 1.5 ns per push+pop pair and std's VecDeque 2.7 to 4.2 ns (2.5x");
    println!("  to 3.8x more); Go's version of this ring took 2.2 ns. A plausible");
    println!("  reason for std's cost, not measured: it supports any capacity, where");
    println!("  this ring gets to mask.");

    // 3e. the two consumers
    println!();
    println!("  and what it does for example 10's consumer:");
    println!();
    let arr: Vec<i64> = (0..200_000).map(|_| rng.below(1 << 20) as i64).collect();
    println!(
        "  {:<24} {:>14} {:>14} {:>10}",
        "sliding-window max", "brute ns", "deque ns", "speedup"
    );
    for k in [8, 128, 2048] {
        assert_eq!(
            max_sliding(&arr[..20_000], k),
            max_sliding_brute(&arr[..20_000], k),
            "consumer disagrees with brute force"
        );
        let tb = ns_per_op(|| {
            black_box(max_sliding_brute(black_box(&arr), k));
        });
        let td = ns_per_op(|| {
            black_box(max_sliding(black_box(&arr), k));
        });
        println!(
            "  {:<24} {:>14.0} {:>14.0} {:>9.1}x",
            format!("k = {k}"),
            tb,
            td,
            tb / td
        );
    }
    println!();
    println!("  brute force wins at k = 8 in every run (0.3x to 0.5x). At k = 128 the");
    println!("  two are close (1.0x to 2.5x) and at k = 2048 the deque wins by 20x to");
    println!("  46x, across more than 30 runs. The deque's time barely moves with k");
    println!("  (1.3 to 3.1 ms for all three: 7 to 15 ns per element, why not");
    println!("  measured); brute force grows 13x to 26x between k = 128 and k = 2048,");
    println!("  where O(n*k) predicts 16x. So the crossover sits near k = 128, later");
    println!("  than 'O(1) beats O(k)' suggests: at k = 2048 the plain loop costs");
    println!("  about 0.12 ns per element at best (51 ms for 200,000 x 2048 compares),");
    println!("  under one comparison per cycle, so it is almost certainly vectorised");
    println!("  (assembly not inspected).");
    println!();
    println!("THE CHECKLIST, which is the actual deliverable of this lesson:");
    println!();
    for line in [
        "capacity is a power of two            (& instead of %, and PushFront works)",
        "count is stored, not derived          (head == tail is ambiguous)",
        "grow unwraps with TWO copies          (the live region is two pieces)",
        "Pop leaves None behind                (take() moves out and clears: Go's zero write)",
        "tests rotate before they assert       (head == 0 hides every real bug)",
        "an overflow policy is chosen          (block / drop / error -- pick one)",
        "the steady state allocates nothing    (measure it; do not assume it)",
    ] {
        println!("  [x] {line}");
    }
    println!();
    println!("  seven lines. The compiler enforces one of them outright: a pop that");
    println!("  moves the value out cannot leave a stale copy in a dead slot, and the");
    println!("  dead-slot check above found none in 65 states. It also removes two of");
    println!("  the invariant checks (head < 0 and count < 0 cannot happen to a");
    println!("  usize). The other six it does not check -- they still need a test or");
    println!("  a measurement.");
}
```

**Rust sample output:**

```
PASS 1 -- BUILD IT

  [a b c d]   Len=4 Cap=4 (grew 1 time(s) from 2)
  PopFront="a" PopBack="d" -> [b c]

PASS 2 -- PROVE IT

  200000 random ops against a slice model:       0 mismatches, 0 invariant failures
  64 starting rotations, then grow:              0 failures
  empty slots hold None (not in Go; 65 states):  0 failures
  sliding-window max, 25009 (array,k) pairs:     0 failures

  three layers, and each catches what the others cannot: the model
  finds wrong ANSWERS, the invariants find corrupt STATE the model
  has not surfaced yet, and the rotation sweep reaches the wrapped
  configurations that both would otherwise never visit.

PASS 3 -- MEASURE IT

  allocations, 100,000 steady-state ops:         0
  steady state, per push+pop pair:               0.95 ns
    the same, std VecDeque (not in Go):          2.82 ns
  worst single PushBack:                         524288 elements copied at push 524288
  burst of 4,000,000 then drained:               64.0 MB peak, 64.0 MB still held at Len 0
                                                 16 bytes per slot (Option<i64>); Go's int slot is 8

  that last line is example 1's finding, in the finished type, and
  it is deliberate: this deque grows and never shrinks. If your
  workload has rare huge bursts, add lesson 06's hysteresis (shrink
  at 25% full, halve only) -- and measure that it does not thrash.

  two Rust-specific readings of the lines above. The slot is an
  Option<i64>, 16 bytes, so the same 4,194,304-slot array that is 32 MB
  in Go is 64 MB here, and the worst push (524,288 elements) moves
  8 MB of slots, not 4. The element counts and the grow history are
  identical to Go's. And 'peak' is the live bytes right after the burst:
  by construction grow holds the old and the new array at once, so the
  true high-water mark was 32 + 64 MB at the last doubling (arithmetic
  from the sizes, not measured; Go's 'peak' is the same kind of figure).

  the steady-state lines compare two different programs, so read them as
  measurements, not a verdict. Across more than 30 runs this ring took
  0.8 to 1.5 ns per push+pop pair and std's VecDeque 2.7 to 4.2 ns (2.5x
  to 3.8x more); Go's version of this ring took 2.2 ns. A plausible
  reason for std's cost, not measured: it supports any capacity, where
  this ring gets to mask.

  and what it does for example 10's consumer:

  sliding-window max             brute ns       deque ns    speedup
  k = 8                            982260        1526094       0.6x
  k = 128                         2780361        1646203       1.7x
  k = 2048                       53050562        1791652      29.6x

  brute force wins at k = 8 in every run (0.3x to 0.5x). At k = 128 the
  two are close (1.0x to 2.5x) and at k = 2048 the deque wins by 20x to
  46x, across more than 30 runs. The deque's time barely moves with k
  (1.3 to 3.1 ms for all three: 7 to 15 ns per element, why not
  measured); brute force grows 13x to 26x between k = 128 and k = 2048,
  where O(n*k) predicts 16x. So the crossover sits near k = 128, later
  than 'O(1) beats O(k)' suggests: at k = 2048 the plain loop costs
  about 0.12 ns per element at best (51 ms for 200,000 x 2048 compares),
  under one comparison per cycle, so it is almost certainly vectorised
  (assembly not inspected).

THE CHECKLIST, which is the actual deliverable of this lesson:

  [x] capacity is a power of two            (& instead of %, and PushFront works)
  [x] count is stored, not derived          (head == tail is ambiguous)
  [x] grow unwraps with TWO copies          (the live region is two pieces)
  [x] Pop leaves None behind                (take() moves out and clears: Go's zero write)
  [x] tests rotate before they assert       (head == 0 hides every real bug)
  [x] an overflow policy is chosen          (block / drop / error -- pick one)
  [x] the steady state allocates nothing    (measure it; do not assume it)

  seven lines. The compiler enforces one of them outright: a pop that
  moves the value out cannot leave a stale copy in a dead slot, and the
  dead-slot check above found none in 65 states. It also removes two of
  the invariant checks (head < 0 and count < 0 cannot happen to a
  usize). The other six it does not check -- they still need a test or
  a measurement.
```

> **Go vs Rust:** Every deterministic line matches Go's (the 4-element trace, 0 mismatches / 0 invariant / 0 rotation failures, 0 allocations, 524,288 elements copied at push 524,288) except the sliding-window pair count (25,009 here, 24,792 in Go: the kit's splitmix64 is not `math/rand`) and the burst, which holds 64.0 MB against Go's 32.0 MB: the same 4,194,304 slots, but `Option<i64>` is 16 bytes. `usize` changes three spellings (`checked_sub` for `At(Len()-1)`, `f + k <= i` for `f <= i-k`, no `head < 0` / `count < 0` checks) and `take()` makes Go's zero write part of the pop, so the Rust adds a dead-slot check and a `std::collections::VecDeque` row that Go lacks, and labels the steady-state line per push+pop pair (Go's "per operation" divides by iterations, each of which is a pair). The consumer table does not carry over: Go's speedup column read 0.6x / 7.4x / 71x at k = 8 / 128 / 2048, but with a plain Rust loop as the baseline it reads 0.3-0.5x / 1.0-2.5x / 20-46x over 30+ runs, and the first baseline I wrote, `windows(k).map(|w| w.iter().max())`, was 4-5x slower than the plain loop (scratch measurement) and showed 7-11x at k = 128 and 95-170x at k = 2048.

**Complexity:** all six operations Θ(1) amortized, **0 allocations** in the steady state · the worst
single `PushBack` copied **524,288** elements — which is what "amortized Θ(1)" costs you at the tail ·
after a 4,000,000 burst and a full drain it still holds **32.0 MB**, deliberately

---

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Lesson: [09-queues-deques.md](../../09-queues-deques.md)
