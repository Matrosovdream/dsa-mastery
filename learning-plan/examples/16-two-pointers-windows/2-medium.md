# Step 16 — Two Pointers, Sliding Window & Prefix Sums · 🟡 Medium

Examples **7–11**: the container with most water and why the shorter wall goes, the Dutch national
flag with its invariant checked and its one-line bug found, the two shapes of variable window
(shortest and longest), and 2-D prefix sums with their inclusion-exclusion.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples marked **Output:** are deterministic and should match character for character; **Sample
output:** ones print timings.

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

> ← Back to [🟢 easy](1-easy.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🔴 hard](3-hard.md)

---

## 7. Container with most water: why the shorter wall goes

`🟡 medium` · *Two pointers, greedy discard*

Choose two walls `i < j` to maximise the water between them, `(j − i) · min(h[i], h[j])`. Start at the widest pair and walk inward: from any pair, every narrower pair that keeps the **shorter** wall holds no more water, because the width shrank and the height is capped by that same wall. So the shorter wall can never be part of a better pair, and its pointer moves. That is the whole technique, and it is a discard argument rather than a trick: n − 1 evaluations replace n(n−1)/2 pairs, Θ(n) time and Θ(1) space. The program turns the argument into an executable check, then shows that the tempting wrong rules fail on small arrays.

**Steps:**

1. Trace the walk on `[1 8 6 2 5 4 8 3 7]` and compare it with brute force.
2. Check the discard argument at every step of 20,000 random arrays, then run "move the taller wall" and "move both" against the same oracle and read off the shortest counterexample.
3. Count evaluations for n up to 100,000 (areas near 10¹⁴ force a 64-bit type), then try skipping walls that cannot win and watch which counter drops and which does not.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
)

// Container with most water. Pick two walls i < j; the water between them holds
//
//	area(i, j) = (j - i) * min(h[i], h[j])
//
// Brute force tries all n(n-1)/2 pairs. Two pointers start at the WIDEST pair
// and walk inward, one step at a time, and evaluate only n-1 pairs. The whole
// technique is one discard argument:
//
//	From (lo, hi), every pair that keeps the SHORTER wall but is narrower holds
//	no more water: its width is smaller and its height is capped by that same
//	shorter wall. So the shorter wall can never be part of a better pair --
//	throw it away (move its pointer) and never look at it again.

// rule says which pointer moves after each evaluation.
type rule int

const (
	moveShorter rule = iota // the correct one
	moveTaller              // wrong: throws away the wall that could still win
	moveBoth                // wrong: also throws away the shorter wall's partners
)

func (r rule) String() string {
	return [...]string{"move the shorter wall", "move the taller wall", "move both walls"}[r]
}

// area is the water held between walls i < j. It is int64 because areas get
// big: see section (d).
func area(h []int64, i, j int) int64 {
	return int64(j-i) * min(h[i], h[j])
}

// step is one evaluation of a two-pointer run: the pair evaluated, its area,
// the best so far (this pair included) and which pointer(s) moved afterwards.
type step struct {
	lo, hi     int
	area, best int64
	moved      string
}

// result is the best pair a run found and what it cost.
type result struct {
	best  int64 // 0 when there are fewer than two walls: no pair, no water
	i, j  int
	evals int // pairs whose area was computed
	moves int // pointer steps taken
}

// run walks lo and hi inward under rule r. onStep (may be nil) sees every
// evaluation, with lo and hi as they were BEFORE the move.
func run(h []int64, r rule, onStep func(step)) result {
	var res result
	lo, hi := 0, len(h)-1 // for len(h) < 2 the loop below does not run
	for lo < hi {
		a := area(h, lo, hi)
		res.evals++
		if a > res.best { // strict: the first pair with this area is kept
			res.best, res.i, res.j = a, lo, hi
		}
		s := step{lo: lo, hi: hi, area: a, best: res.best}
		var goLo, goHi bool // which pointer(s) move
		switch r {
		case moveShorter: // tie: either wall may go, take the left one
			goLo = h[lo] <= h[hi]
			goHi = !goLo
		case moveTaller: // tie: neither is taller, take the right one
			goLo = h[lo] > h[hi]
			goHi = !goLo
		case moveBoth:
			goLo, goHi = true, true
		}
		if goLo {
			lo++
			res.moves++
			s.moved = "lo"
		}
		if goHi {
			hi--
			res.moves++
			s.moved = "hi"
		}
		if goLo && goHi {
			s.moved = "both"
		}
		if onStep != nil {
			onStep(s)
		}
	}
	return res
}

// runSkip is the same walk with one optimisation: after discarding a wall,
// keep discarding walls that are not taller than it WITHOUT computing their
// area. A pair that keeps such a wall is narrower and no taller, so it cannot
// beat the pair just evaluated. The pointers follow exactly the same path as
// run(moveShorter); only the multiplications are saved.
func runSkip(h []int64) result {
	var res result
	lo, hi := 0, len(h)-1
	for lo < hi {
		a := area(h, lo, hi)
		res.evals++
		if a > res.best {
			res.best, res.i, res.j = a, lo, hi
		}
		if h[lo] <= h[hi] {
			old := h[lo]
			lo++
			res.moves++
			for lo < hi && h[lo] <= old {
				lo++
				res.moves++
			}
		} else {
			old := h[hi]
			hi--
			res.moves++
			for lo < hi && h[hi] <= old {
				hi--
				res.moves++
			}
		}
	}
	return res
}

// brute is the oracle: every pair, n(n-1)/2 of them.
func brute(h []int64) result {
	var res result
	for i := range h {
		for j := i + 1; j < len(h); j++ {
			res.evals++
			if a := area(h, i, j); a > res.best {
				res.best, res.i, res.j = a, i, j
			}
		}
	}
	return res
}

// proofHolds checks the discard argument for ONE step of the correct rule: it
// takes the wall that is about to be discarded, pairs it with every index that
// is still inside the window afterwards, and asserts none of those pairs holds
// more water than the pair just evaluated. It returns how many pairs it tried
// and how many broke the claim.
func proofHolds(h []int64, s step) (checked, violations int) {
	if s.moved == "lo" { // wall lo is discarded; the window becomes [lo+1, hi]
		for k := s.lo + 1; k <= s.hi; k++ {
			checked++
			if area(h, s.lo, k) > s.area {
				violations++
			}
		}
	} else { // wall hi is discarded; the window becomes [lo, hi-1]
		for k := s.lo; k < s.hi; k++ {
			checked++
			if area(h, k, s.hi) > s.area {
				violations++
			}
		}
	}
	return checked, violations
}

func randomHeights(rng *rand.Rand, n int, bound int) []int64 {
	h := make([]int64, n)
	for i := range h {
		h[i] = int64(rng.IntN(bound))
	}
	return h
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))

	fmt.Println("(a) the classic example: walls [1 8 6 2 5 4 8 3 7]")
	h := []int64{1, 8, 6, 2, 5, 4, 8, 3, 7}
	fmt.Printf("  %4s %3s %3s %6s %6s %6s %6s %5s  %s\n", "step", "lo", "hi", "h[lo]", "h[hi]", "width", "area", "best", "moved")
	n := 0
	res := run(h, moveShorter, func(s step) {
		n++
		fmt.Printf("  %4d %3d %3d %6d %6d %6d %6d %5d  %s\n",
			n, s.lo, s.hi, h[s.lo], h[s.hi], s.hi-s.lo, s.area, s.best, s.moved)
	})
	fmt.Printf("  best %d between walls %d and %d (heights %d and %d), after %d evaluations;\n",
		res.best, res.i, res.j, h[res.i], h[res.j], res.evals)
	oracle := brute(h)
	fmt.Printf("  brute force looks at %d pairs and agrees: %d\n", oracle.evals, oracle.best)

	// 20,000 random small cases, shared by sections (b), (c) and (e).
	const cases = 20000
	arrays := make([][]int64, cases)
	for c := range arrays {
		n := 2 + rng.IntN(11) // length 2..12
		arrays[c] = randomHeights(rng, n, 10)
		for i := range arrays[c] {
			arrays[c][i]++ // heights 1..10
		}
	}

	fmt.Println()
	fmt.Printf("(b) the proof, as a check: %d random arrays (length 2-12, heights 1-10)\n", cases)
	steps, pairChecks, violations := 0, 0, 0
	for _, a := range arrays {
		run(a, moveShorter, func(s step) {
			steps++
			c, v := proofHolds(a, s)
			pairChecks += c
			violations += v
		})
	}
	fmt.Printf("  %d steps checked, %d (discarded wall, wall still in the window) pairs compared with the\n", steps, pairChecks)
	fmt.Printf("  pair just evaluated, %d violations\n", violations)
	wrong := 0
	for _, a := range arrays {
		if run(a, moveShorter, nil).best != brute(a).best {
			wrong++
		}
	}
	fmt.Printf("  and the final answer differs from brute force on %d of %d arrays\n", wrong, cases)

	fmt.Println()
	fmt.Println("(c) the wrong rules, against the same oracle on the same arrays")
	for _, r := range []rule{moveShorter, moveTaller, moveBoth} {
		wrong := 0
		var first []int64 // shortest failing array, earliest among equals
		for _, a := range arrays {
			if run(a, r, nil).best != brute(a).best {
				wrong++
				if first == nil || len(a) < len(first) {
					first = a
				}
			}
		}
		fmt.Printf("  %-22s wrong maximum on %5d of %d arrays\n", r, wrong, cases)
		if first == nil {
			continue
		}
		want := brute(first)
		var visited []string
		got := run(first, r, func(s step) { visited = append(visited, fmt.Sprintf("(%d,%d)", s.lo, s.hi)) })
		fmt.Printf("    shortest counterexample %v: best is %d at (%d,%d), this rule says %d\n",
			first, want.best, want.i, want.j, got.best)
		fmt.Printf("    it evaluated %v and never looked at (%d,%d)\n", visited, want.i, want.j)
	}

	fmt.Println()
	fmt.Println("(d) n-1 evaluations instead of n(n-1)/2, heights = IntN(1_000_000_000)")
	fmt.Printf("  %7s %9s %15s %11s %17s  %s\n", "n", "evals", "n(n-1)/2", "pairs/eval", "best area", "brute force")
	var lastBest int64
	for _, n := range []int{10, 100, 1000, 10000, 100000} {
		h := randomHeights(rng, n, 1_000_000_000)
		res := run(h, moveShorter, nil)
		pairs := n * (n - 1) / 2
		check := "not run"
		if n <= 10000 {
			check = "agrees"
			if brute(h).best != res.best {
				check = "MISMATCH"
			}
		}
		fmt.Printf("  %7d %9d %15d %10.1fx %17d  %s\n", n, res.evals, pairs, float64(pairs)/float64(res.evals), res.best, check)
		lastBest = res.best
	}
	fmt.Printf("  the n=100000 area is %d; int32 tops out at %d, so int32(area) would print %d\n",
		lastBest, math.MaxInt32, int32(lastBest))
	fmt.Println("  (width up to 99999 times height up to 999999999 is near 1e14: int64 holds it, int32 cannot)")
	fmt.Println("  the ratio is exactly n/2, so the gap between the two columns widens linearly with n")

	fmt.Println()
	fmt.Println("(e) skip walls that are not taller than the one just discarded")
	same := 0
	for _, a := range arrays {
		p, s := run(a, moveShorter, nil), runSkip(a)
		if p.best == s.best && p.i == s.i && p.j == s.j {
			same++
		}
	}
	fmt.Printf("  on the %d small arrays, same best AND same pair as plain two pointers: %d\n", cases, same)
	fmt.Printf("  %7s %12s %12s %12s %12s\n", "n", "plain evals", "skip evals", "plain moves", "skip moves")
	var p, s result
	for _, n := range []int{100, 1000, 10000, 100000} {
		h := randomHeights(rng, n, 1_000_000_000)
		p, s = run(h, moveShorter, nil), runSkip(h)
		note := ""
		if p.best != s.best {
			note = "  DIFFERENT ANSWER"
		}
		fmt.Printf("  %7d %12d %12d %12d %12d%s\n", n, p.evals, s.evals, p.moves, s.moves, note)
	}
	fmt.Printf("  at n=100000 only %d of %d pairs still need an area computed, but the pointers still take\n", s.evals, p.evals)
	fmt.Printf("  %d steps either way: the skip removes area evaluations, not pointer moves, so it is\n", s.moves)
	fmt.Println("  still Theta(n) (only evaluations and moves were counted here, not time)")

	fmt.Println()
	fmt.Println("lesson 08 example 15 (trapping rain water) sums the water over EVERY index of a")
	fmt.Println("histogram; this example picks the ONE best pair of walls. Same picture, different question.")
}
```

**Output:**

```
(a) the classic example: walls [1 8 6 2 5 4 8 3 7]
  step  lo  hi  h[lo]  h[hi]  width   area  best  moved
     1   0   8      1      7      8      8     8  lo
     2   1   8      8      7      7     49    49  hi
     3   1   7      8      3      6     18    49  hi
     4   1   6      8      8      5     40    49  lo
     5   2   6      6      8      4     24    49  lo
     6   3   6      2      8      3      6    49  lo
     7   4   6      5      8      2     10    49  lo
     8   5   6      4      8      1      4    49  lo
  best 49 between walls 1 and 8 (heights 8 and 7), after 8 evaluations;
  brute force looks at 36 pairs and agrees: 49

(b) the proof, as a check: 20000 random arrays (length 2-12, heights 1-10)
  119436 steps checked, 517124 (discarded wall, wall still in the window) pairs compared with the
  pair just evaluated, 0 violations
  and the final answer differs from brute force on 0 of 20000 arrays

(c) the wrong rules, against the same oracle on the same arrays
  move the shorter wall  wrong maximum on     0 of 20000 arrays
  move the taller wall   wrong maximum on 11097 of 20000 arrays
    shortest counterexample [2 8 10]: best is 8 at (1,2), this rule says 4
    it evaluated [(0,2) (0,1)] and never looked at (1,2)
  move both walls        wrong maximum on  9613 of 20000 arrays
    shortest counterexample [2 8 10]: best is 8 at (1,2), this rule says 4
    it evaluated [(0,2)] and never looked at (1,2)

(d) n-1 evaluations instead of n(n-1)/2, heights = IntN(1_000_000_000)
        n     evals        n(n-1)/2  pairs/eval         best area  brute force
       10         9              45        5.0x        3644325251  agrees
      100        99            4950       50.0x       78436453844  agrees
     1000       999          499500      500.0x      965057658658  agrees
    10000      9999        49995000     5000.0x     9809481522475  agrees
   100000     99999      4999950000    50000.0x    99132042211588  not run
  the n=100000 area is 99132042211588; int32 tops out at 2147483647, so int32(area) would print -97947388
  (width up to 99999 times height up to 999999999 is near 1e14: int64 holds it, int32 cannot)
  the ratio is exactly n/2, so the gap between the two columns widens linearly with n

(e) skip walls that are not taller than the one just discarded
  on the 20000 small arrays, same best AND same pair as plain two pointers: 20000
        n  plain evals   skip evals  plain moves   skip moves
      100           99            9           99           99
     1000          999           14          999          999
    10000         9999           13         9999         9999
   100000        99999           18        99999        99999
  at n=100000 only 18 of 99999 pairs still need an area computed, but the pointers still take
  99999 steps either way: the skip removes area evaluations, not pointer moves, so it is
  still Theta(n) (only evaluations and moves were counted here, not time)

lesson 08 example 15 (trapping rain water) sums the water over EVERY index of a
histogram; this example picks the ONE best pair of walls. Same picture, different question.
```

**Rust:**

```rust
// Container with most water. Pick two walls i < j; the water between them holds
//
//     area(i, j) = (j - i) * min(h[i], h[j])
//
// Brute force tries all n(n-1)/2 pairs. Two pointers start at the WIDEST pair
// and walk inward, one step at a time, and evaluate only n-1 pairs. The whole
// technique is one discard argument:
//
//     From (lo, hi), every pair that keeps the SHORTER wall but is narrower holds
//     no more water: its width is smaller and its height is capped by that same
//     shorter wall. So the shorter wall can never be part of a better pair --
//     throw it away (move its pointer) and never look at it again.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so that seed (1, 2) gives Rust exactly the arrays Go sees and every
// count below comes out identical. Only next_u64 and int_n are needed.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        // state = state * MUL + INC, with explicit 128-bit wraparound
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;

        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

/// Which pointer moves after each evaluation.
#[derive(Clone, Copy)]
enum Rule {
    Shorter, // the correct one
    Taller,  // wrong: throws away the wall that could still win
    Both,    // wrong: also throws away the shorter wall's partners
}

impl Rule {
    fn name(self) -> &'static str {
        match self {
            Rule::Shorter => "move the shorter wall",
            Rule::Taller => "move the taller wall",
            Rule::Both => "move both walls",
        }
    }
}

/// Water held between walls i < j. It is u64 because areas get big (see
/// section (d)). `j - i` cannot underflow: every caller has i < j.
fn area(h: &[u64], i: usize, j: usize) -> u64 {
    (j - i) as u64 * h[i].min(h[j])
}

/// One evaluation of a two-pointer run: the pair evaluated, its area, the best
/// so far (this pair included) and which pointer(s) moved afterwards.
struct Step {
    lo: usize,
    hi: usize,
    area: u64,
    best: u64,
    moved: &'static str,
}

/// The best pair a run found and what it cost.
struct Res {
    best: u64,
    i: usize,
    j: usize,
    evals: usize, // pairs whose area was computed
    moves: usize, // pointer steps taken
}

/// Walk lo and hi inward under `rule`. `on_step` sees every evaluation, with lo
/// and hi as they were BEFORE the move. Fewer than two walls: there is no pair,
/// and `None` says so (Go returns best = 0, which also means "no water").
fn run(h: &[u64], rule: Rule, mut on_step: impl FnMut(Step)) -> Option<Res> {
    if h.len() < 2 {
        return None; // also keeps h.len() - 1 from underflowing
    }
    let (mut lo, mut hi) = (0, h.len() - 1);
    let mut res = Res {
        best: 0,
        i: 0,
        j: 0,
        evals: 0,
        moves: 0,
    };
    while lo < hi {
        let a = area(h, lo, hi);
        res.evals += 1;
        if a > res.best {
            // strict: the first pair with this area is kept
            (res.best, res.i, res.j) = (a, lo, hi);
        }
        let mut s = Step {
            lo,
            hi,
            area: a,
            best: res.best,
            moved: "",
        };
        let (go_lo, go_hi) = match rule {
            // tie: either wall may go, take the left one
            Rule::Shorter => (h[lo] <= h[hi], h[lo] > h[hi]),
            // tie: neither is taller, take the right one
            Rule::Taller => (h[lo] > h[hi], h[lo] <= h[hi]),
            Rule::Both => (true, true),
        };
        if go_lo {
            lo += 1;
            res.moves += 1;
            s.moved = "lo";
        }
        if go_hi {
            // hi > lo >= 0 here, so hi - 1 cannot underflow
            hi -= 1;
            res.moves += 1;
            s.moved = "hi";
        }
        if go_lo && go_hi {
            s.moved = "both";
        }
        on_step(s);
    }
    Some(res)
}

/// The same walk with one optimisation: after discarding a wall, keep
/// discarding walls that are not taller than it WITHOUT computing their area.
/// A pair that keeps such a wall is narrower and no taller, so it cannot beat
/// the pair just evaluated. The pointers follow exactly the same path as
/// run(Shorter); only the multiplications are saved.
fn run_skip(h: &[u64]) -> Option<Res> {
    if h.len() < 2 {
        return None;
    }
    let (mut lo, mut hi) = (0, h.len() - 1);
    let mut res = Res {
        best: 0,
        i: 0,
        j: 0,
        evals: 0,
        moves: 0,
    };
    while lo < hi {
        let a = area(h, lo, hi);
        res.evals += 1;
        if a > res.best {
            (res.best, res.i, res.j) = (a, lo, hi);
        }
        if h[lo] <= h[hi] {
            let old = h[lo];
            lo += 1;
            res.moves += 1;
            while lo < hi && h[lo] <= old {
                lo += 1;
                res.moves += 1;
            }
        } else {
            let old = h[hi];
            hi -= 1;
            res.moves += 1;
            while lo < hi && h[hi] <= old {
                hi -= 1;
                res.moves += 1;
            }
        }
    }
    Some(res)
}

/// The oracle: every pair, n(n-1)/2 of them.
fn brute(h: &[u64]) -> Option<Res> {
    if h.len() < 2 {
        return None;
    }
    let mut res = Res {
        best: 0,
        i: 0,
        j: 0,
        evals: 0,
        moves: 0,
    };
    for i in 0..h.len() {
        for j in i + 1..h.len() {
            res.evals += 1;
            let a = area(h, i, j);
            if a > res.best {
                (res.best, res.i, res.j) = (a, i, j);
            }
        }
    }
    Some(res)
}

/// Check the discard argument for ONE step of the correct rule: take the wall
/// that is about to be discarded, pair it with every index still inside the
/// window afterwards, and assert none of those pairs holds more water than the
/// pair just evaluated. Returns (pairs tried, pairs that broke the claim).
fn proof_holds(h: &[u64], s: &Step) -> (usize, usize) {
    let (mut checked, mut violations) = (0, 0);
    if s.moved == "lo" {
        // wall lo is discarded; the window becomes [lo+1, hi]
        for k in s.lo + 1..=s.hi {
            checked += 1;
            if area(h, s.lo, k) > s.area {
                violations += 1;
            }
        }
    } else {
        // wall hi is discarded; the window becomes [lo, hi-1]
        for k in s.lo..s.hi {
            checked += 1;
            if area(h, k, s.hi) > s.area {
                violations += 1;
            }
        }
    }
    (checked, violations)
}

fn random_heights(rng: &mut Pcg, n: usize, bound: u64) -> Vec<u64> {
    (0..n).map(|_| rng.int_n(bound)).collect()
}

// Go prints a slice as [1 8 6]; Rust's {:?} prints [1, 8, 6]. Format by hand
// so the output reads the same in both languages.
fn go_slice(xs: &[u64]) -> String {
    let items: Vec<String> = xs.iter().map(u64::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn main() {
    let mut rng = Pcg::new(1, 2);

    println!("(a) the classic example: walls [1 8 6 2 5 4 8 3 7]");
    let h: Vec<u64> = vec![1, 8, 6, 2, 5, 4, 8, 3, 7];
    println!(
        "  {:>4} {:>3} {:>3} {:>6} {:>6} {:>6} {:>6} {:>5}  moved",
        "step", "lo", "hi", "h[lo]", "h[hi]", "width", "area", "best"
    );
    let mut n = 0;
    let res = run(&h, Rule::Shorter, |s| {
        n += 1;
        println!(
            "  {:>4} {:>3} {:>3} {:>6} {:>6} {:>6} {:>6} {:>5}  {}",
            n,
            s.lo,
            s.hi,
            h[s.lo],
            h[s.hi],
            s.hi - s.lo,
            s.area,
            s.best,
            s.moved
        );
    })
    .expect("nine walls make pairs");
    println!(
        "  best {} between walls {} and {} (heights {} and {}), after {} evaluations;",
        res.best, res.i, res.j, h[res.i], h[res.j], res.evals
    );
    let oracle = brute(&h).expect("nine walls make pairs");
    println!(
        "  brute force looks at {} pairs and agrees: {}",
        oracle.evals, oracle.best
    );

    // 20,000 random small cases, shared by sections (b), (c) and (e).
    const CASES: usize = 20000;
    let arrays: Vec<Vec<u64>> = (0..CASES)
        .map(|_| {
            let n = 2 + rng.int_n(11) as usize; // length 2..12
            random_heights(&mut rng, n, 10)
                .into_iter()
                .map(|x| x + 1) // heights 1..10
                .collect()
        })
        .collect();

    // The answer of a run, 0 when there is no pair (every array here has two walls).
    let best_of = |a: &[u64], rule: Rule| run(a, rule, |_| {}).map_or(0, |r| r.best);

    println!();
    println!("(b) the proof, as a check: {CASES} random arrays (length 2-12, heights 1-10)");
    let (mut steps, mut pair_checks, mut violations) = (0, 0, 0);
    for a in &arrays {
        run(a, Rule::Shorter, |s| {
            steps += 1;
            let (c, v) = proof_holds(a, &s);
            pair_checks += c;
            violations += v;
        });
    }
    println!(
        "  {steps} steps checked, {pair_checks} (discarded wall, wall still in the window) pairs compared with the"
    );
    println!("  pair just evaluated, {violations} violations");
    let wrong = arrays
        .iter()
        .filter(|a| best_of(a, Rule::Shorter) != brute(a).map_or(0, |r| r.best))
        .count();
    println!("  and the final answer differs from brute force on {wrong} of {CASES} arrays");

    println!();
    println!("(c) the wrong rules, against the same oracle on the same arrays");
    for rule in [Rule::Shorter, Rule::Taller, Rule::Both] {
        let mut wrong = 0;
        let mut first: Option<&Vec<u64>> = None; // shortest failing array, earliest among equals
        for a in &arrays {
            if best_of(a, rule) != brute(a).map_or(0, |r| r.best) {
                wrong += 1;
                if first.is_none_or(|f| a.len() < f.len()) {
                    first = Some(a);
                }
            }
        }
        println!(
            "  {:<22} wrong maximum on {:>5} of {CASES} arrays",
            rule.name(),
            wrong
        );
        let Some(first) = first else { continue };
        let want = brute(first).expect("two or more walls");
        let mut visited: Vec<String> = Vec::new();
        let got = run(first, rule, |s| {
            visited.push(format!("({},{})", s.lo, s.hi))
        })
        .expect("two or more walls");
        println!(
            "    shortest counterexample {}: best is {} at ({},{}), this rule says {}",
            go_slice(first),
            want.best,
            want.i,
            want.j,
            got.best
        );
        println!(
            "    it evaluated [{}] and never looked at ({},{})",
            visited.join(" "),
            want.i,
            want.j
        );
    }

    println!();
    println!("(d) n-1 evaluations instead of n(n-1)/2, heights = IntN(1_000_000_000)");
    println!(
        "  {:>7} {:>9} {:>15} {:>11} {:>17}  brute force",
        "n", "evals", "n(n-1)/2", "pairs/eval", "best area"
    );
    let mut last_best = 0;
    for n in [10, 100, 1000, 10000, 100000] {
        let h = random_heights(&mut rng, n, 1_000_000_000);
        let res = run(&h, Rule::Shorter, |_| {}).expect("n >= 10");
        let pairs = n * (n - 1) / 2;
        let check = if n <= 10000 {
            if brute(&h).expect("n >= 10").best == res.best {
                "agrees"
            } else {
                "MISMATCH"
            }
        } else {
            "not run"
        };
        println!(
            "  {:>7} {:>9} {:>15} {:>10.1}x {:>17}  {}",
            n,
            res.evals,
            pairs,
            pairs as f64 / res.evals as f64,
            res.best,
            check
        );
        last_best = res.best;
    }
    // `as i32` truncates (wraps) silently in Rust, exactly like Go's int32(x)
    // conversion; a u32::try_from(...) would be the checked way to ask.
    println!(
        "  the n=100000 area is {}; int32 tops out at {}, so area as i32 would print {}",
        last_best,
        i32::MAX,
        last_best as i32
    );
    println!(
        "  (width up to 99999 times height up to 999999999 is near 1e14: u64 holds it, 32 bits cannot)"
    );
    println!(
        "  the ratio is exactly n/2, so the gap between the two columns widens linearly with n"
    );

    println!();
    println!("(e) skip walls that are not taller than the one just discarded");
    let same = arrays
        .iter()
        .filter(|a| {
            let (p, s) = (
                run(a, Rule::Shorter, |_| {}).expect("two walls"),
                run_skip(a).expect("two walls"),
            );
            p.best == s.best && p.i == s.i && p.j == s.j
        })
        .count();
    println!(
        "  on the {CASES} small arrays, same best AND same pair as plain two pointers: {same}"
    );
    println!(
        "  {:>7} {:>12} {:>12} {:>12} {:>12}",
        "n", "plain evals", "skip evals", "plain moves", "skip moves"
    );
    let (mut plain_evals, mut skip_evals, mut skip_moves) = (0, 0, 0);
    for n in [100, 1000, 10000, 100000] {
        let h = random_heights(&mut rng, n, 1_000_000_000);
        let (p, s) = (
            run(&h, Rule::Shorter, |_| {}).expect("n >= 100"),
            run_skip(&h).expect("n >= 100"),
        );
        let note = if p.best != s.best {
            "  DIFFERENT ANSWER"
        } else {
            ""
        };
        println!(
            "  {:>7} {:>12} {:>12} {:>12} {:>12}{}",
            n, p.evals, s.evals, p.moves, s.moves, note
        );
        (plain_evals, skip_evals, skip_moves) = (p.evals, s.evals, s.moves);
    }
    println!(
        "  at n=100000 only {skip_evals} of {plain_evals} pairs still need an area computed, but the pointers still take"
    );
    println!(
        "  {skip_moves} steps either way: the skip removes area evaluations, not pointer moves, so it is"
    );
    println!("  still Theta(n) (only evaluations and moves were counted here, not time)");

    println!();
    println!("lesson 08 example 15 (trapping rain water) sums the water over EVERY index of a");
    println!(
        "histogram; this example picks the ONE best pair of walls. Same picture, different question."
    );
}
```

**Rust output:**

```
(a) the classic example: walls [1 8 6 2 5 4 8 3 7]
  step  lo  hi  h[lo]  h[hi]  width   area  best  moved
     1   0   8      1      7      8      8     8  lo
     2   1   8      8      7      7     49    49  hi
     3   1   7      8      3      6     18    49  hi
     4   1   6      8      8      5     40    49  lo
     5   2   6      6      8      4     24    49  lo
     6   3   6      2      8      3      6    49  lo
     7   4   6      5      8      2     10    49  lo
     8   5   6      4      8      1      4    49  lo
  best 49 between walls 1 and 8 (heights 8 and 7), after 8 evaluations;
  brute force looks at 36 pairs and agrees: 49

(b) the proof, as a check: 20000 random arrays (length 2-12, heights 1-10)
  119436 steps checked, 517124 (discarded wall, wall still in the window) pairs compared with the
  pair just evaluated, 0 violations
  and the final answer differs from brute force on 0 of 20000 arrays

(c) the wrong rules, against the same oracle on the same arrays
  move the shorter wall  wrong maximum on     0 of 20000 arrays
  move the taller wall   wrong maximum on 11097 of 20000 arrays
    shortest counterexample [2 8 10]: best is 8 at (1,2), this rule says 4
    it evaluated [(0,2) (0,1)] and never looked at (1,2)
  move both walls        wrong maximum on  9613 of 20000 arrays
    shortest counterexample [2 8 10]: best is 8 at (1,2), this rule says 4
    it evaluated [(0,2)] and never looked at (1,2)

(d) n-1 evaluations instead of n(n-1)/2, heights = IntN(1_000_000_000)
        n     evals        n(n-1)/2  pairs/eval         best area  brute force
       10         9              45        5.0x        3644325251  agrees
      100        99            4950       50.0x       78436453844  agrees
     1000       999          499500      500.0x      965057658658  agrees
    10000      9999        49995000     5000.0x     9809481522475  agrees
   100000     99999      4999950000    50000.0x    99132042211588  not run
  the n=100000 area is 99132042211588; int32 tops out at 2147483647, so area as i32 would print -97947388
  (width up to 99999 times height up to 999999999 is near 1e14: u64 holds it, 32 bits cannot)
  the ratio is exactly n/2, so the gap between the two columns widens linearly with n

(e) skip walls that are not taller than the one just discarded
  on the 20000 small arrays, same best AND same pair as plain two pointers: 20000
        n  plain evals   skip evals  plain moves   skip moves
      100           99            9           99           99
     1000          999           14          999          999
    10000         9999           13         9999         9999
   100000        99999           18        99999        99999
  at n=100000 only 18 of 99999 pairs still need an area computed, but the pointers still take
  99999 steps either way: the skip removes area evaluations, not pointer moves, so it is
  still Theta(n) (only evaluations and moves were counted here, not time)

lesson 08 example 15 (trapping rain water) sums the water over EVERY index of a
histogram; this example picks the ONE best pair of walls. Same picture, different question.
```

> **Go vs Rust:** Same PCG-DXSM stream, so every count matches Go; the only differing lines are the two that name the integer type (`u64` here, `int64`/`int32(area)` there). `run` returns `Option<Res>`, so "fewer than two walls" is `None` rather than a best of 0, and the early return also keeps `h.len() - 1` from underflowing a `usize`; `hi -= 1` is safe because the loop guard gives `hi > lo >= 0`. `area as i32` truncates silently, exactly like Go's `int32(x)`: the -97947388 is the same in both.

**Complexity:** Θ(n) time, Θ(1) space. The two-pointer walk evaluates exactly n − 1 pairs against n(n−1)/2 for brute force (99,999 against 4,999,950,000 at n = 100,000, a ratio of exactly n/2), and the discard argument held on every one of the 119,436 steps checked over 20,000 arrays with 0 violations. Moving the taller wall gave a wrong maximum on 11,097 of the same 20,000 arrays and moving both on 9,613, and the shortest counterexample for both is `[2 8 10]`. Areas reach about 10¹⁴, so they need 64-bit integers. Skipping walls that are not taller than the one just discarded cut the evaluations at n = 100,000 from 99,999 to 18 while the pointer moves stayed at 99,999.

---

## 8. The Dutch national flag, and the one-line bug

`🟡 medium` · *Three regions, one pass*

Earlier lessons *used* a three-way partition; this one **proves** it. Four regions cover the array at every moment: `[0,lo)` zeros, `[lo,mid)` ones, `[mid,hi)` unknown and `[hi,n)` twos. Each iteration examines `a[mid]` and shrinks the unknown region by exactly one (`mid++` or `hi--`), so the loop runs **exactly n times** and does at most one swap per iteration: Θ(n) time, Θ(1) space. The one-line bug is to advance `mid` after swapping with `a[hi-1]`: the element that arrived there has not been examined. Because the input space is tiny (3ⁿ arrays), the program does not sample, it enumerates every array up to length 10 and finds the bug on the shortest length that can show it.

**Steps:**

1. Trace the partition on `[2 0 2 1 1 0]`, drawing the three marks and checking the invariant after every iteration.
2. Run all 88,573 arrays of length 0 to 10 through the correct and the buggy version, counting unsorted results and iterations, then replay the first failure.
3. Sort one million random 0/1/2 values three ways (the flag, two two-way partitions, counting sort) and compare reads, writes and swaps.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
)

// The Dutch national flag sorts three classes (0, 1, 2) in ONE pass, in place.
// Four regions partition the array at every moment:
//
//	[0, lo)    all 0   -- done
//	[lo, mid)  all 1   -- done
//	[mid, hi)  unknown -- not examined yet
//	[hi, n)    all 2   -- done
//
// Start with lo = mid = 0 and hi = n: everything is unknown. Each iteration
// looks at a[mid] and shrinks the unknown region by EXACTLY one:
//
//	a[mid] == 0: swap it with a[lo]; lo++, mid++   (what comes back to a[mid] is a 1,
//	             because [lo,mid) holds only 1s -- or a[mid] itself when lo == mid --
//	             so mid may advance)
//	a[mid] == 1: mid++                              (already in the right region)
//	a[mid] == 2: hi--, swap a[mid] with a[hi]       (mid must NOT advance: the value
//	             that arrived from the unknown region has not been examined)
//
// So the loop runs exactly n times, whatever the input. This example PROVES the
// partition instead of trusting it: the invariant is checked after every step,
// every input of length <= 10 is tried, and the one-line bug is found.

// counted is a slice that counts its own accesses. A swap is 2 reads and 2
// writes, even when i == j (the swap statement still runs).
type counted struct {
	a                         []int
	reads, writes, swaps, its int // its: loop iterations
}

func (c *counted) get(i int) int { c.reads++; return c.a[i] }

func (c *counted) set(i, v int) { c.writes++; c.a[i] = v }

func (c *counted) swap(i, j int) {
	c.reads += 2
	c.writes += 2
	c.swaps++
	c.a[i], c.a[j] = c.a[j], c.a[i]
}

// dutch is the one-pass partition. With buggy set it advances mid after the
// swap with a[hi-1] -- the one-line bug. hook (may be nil) is called after
// every iteration with the value that was examined, a description of the
// action, and the new lo, mid, hi.
func dutch(c *counted, buggy bool, hook func(examined int, action string, lo, mid, hi int)) {
	lo, mid, hi := 0, 0, len(c.a)
	for mid < hi {
		c.its++
		v := c.get(mid)
		var action string
		switch {
		case v < 1:
			c.swap(lo, mid)
			lo++
			mid++
			action = "swap(lo,mid) lo++ mid++"
		case v == 1:
			mid++
			action = "mid++"
		default: // v > 1: the third class
			hi--
			c.swap(mid, hi)
			action = "hi-- swap(mid,hi)"
			if buggy {
				mid++
				action += " mid++ BUG"
			}
		}
		if hook != nil {
			hook(v, action, lo, mid, hi)
		}
	}
}

// invariant verifies the contents of the three known regions.
func invariant(a []int, lo, mid, hi int) bool {
	if lo < 0 || lo > mid || mid > hi || hi > len(a) {
		return false
	}
	for i, v := range a {
		switch {
		case i < lo && v != 0, i >= lo && i < mid && v != 1, i >= hi && v != 2:
			return false
		}
	}
	return true
}

// render prints the array with a | at the gaps lo, mid and hi (left to right
// that is lo, mid, hi whenever lo <= mid <= hi).
func render(a []int, lo, mid, hi int) string {
	var tokens []string
	for p := 0; p <= len(a); p++ {
		if bars := strings.Repeat("|", count(p, lo, mid, hi)); bars != "" {
			tokens = append(tokens, bars)
		}
		if p < len(a) {
			tokens = append(tokens, fmt.Sprint(a[p]))
		}
	}
	return strings.Join(tokens, " ")
}

func count(p int, marks ...int) int {
	n := 0
	for _, m := range marks {
		if m == p {
			n++
		}
	}
	return n
}

// forEachInput calls f on all 3^n arrays over {0,1,2}, in lexicographic order
// (a[0] is the most significant base-3 digit). f must not keep the slice.
func forEachInput(n int, f func(a []int)) {
	a := make([]int, n)
	for {
		f(a)
		i := n - 1
		for i >= 0 && a[i] == 2 {
			a[i] = 0
			i--
		}
		if i < 0 {
			return
		}
		a[i]++
	}
}

// twoPasses is the alternative: partition "is 0" to the front, then partition
// "is 1" over what is left.
func twoPasses(c *counted) {
	w := 0
	for i := range c.a {
		if c.get(i) == 0 {
			c.swap(i, w)
			w++
		}
	}
	w2 := w
	for i := w; i < len(c.a); i++ {
		if c.get(i) == 1 {
			c.swap(i, w2)
			w2++
		}
	}
}

// countingSort counts the three values, then rewrites the array. It cannot
// move records, only rebuild keys, but for bare 0/1/2 values it is a valid rival.
func countingSort(c *counted) {
	var cnt [3]int
	for i := range c.a {
		cnt[c.get(i)]++
	}
	k := 0
	for v := range cnt {
		for ; cnt[v] > 0; cnt[v]-- {
			c.set(k, v)
			k++
		}
	}
}

func main() {
	fmt.Println("(a) trace on [2 0 2 1 1 0]: | marks lo, mid, hi (left to right)")
	a := []int{2, 0, 2, 1, 1, 0}
	c := &counted{a: slices.Clone(a)}
	fmt.Printf("  %4s %8s  %-25s %2s %3s %2s  %-18s %s\n", "iter", "examined", "action", "lo", "mid", "hi", "array", "invariant")
	fmt.Printf("  %4d %8s  %-25s %2d %3d %2d  %-18s %v\n", 0, "-", "start", 0, 0, len(a), render(c.a, 0, 0, len(a)), invariant(c.a, 0, 0, len(a)))
	held := 0
	dutch(c, false, func(examined int, action string, lo, mid, hi int) {
		ok := invariant(c.a, lo, mid, hi)
		if ok {
			held++
		}
		fmt.Printf("  %4d %8d  %-25s %2d %3d %2d  %-18s %v\n", c.its, examined, action, lo, mid, hi, render(c.a, lo, mid, hi), ok)
	})
	fmt.Printf("  invariant held after %d of %d iterations; loop ran %d times for n = %d; result %v\n",
		held, c.its, c.its, len(a), c.a)

	fmt.Println()
	fmt.Println("(b) exhaustive: every array over {0,1,2} of length 0..10")
	fmt.Printf("  %2s %6s | %-39s | %-27s\n", "n", "inputs", "correct version", "buggy (mid++ after hi swap)")
	fmt.Printf("  %2s %6s | %8s %10s %9s %10s | %8s %10s\n", "", "", "unsorted", "iterations", "max swaps", "inv. broke", "unsorted", "iterations")
	total, steps, stepViolations := 0, 0, 0
	var firstBad []int
	for n := 0; n <= 10; n++ {
		var inputs, unsorted, buggyUnsorted, maxSwaps, broke int
		minIts, maxIts, bugMinIts, bugMaxIts := n, 0, n, 0
		forEachInput(n, func(in []int) {
			inputs++
			good := &counted{a: slices.Clone(in)}
			dutch(good, false, func(_ int, _ string, lo, mid, hi int) {
				steps++
				if !invariant(good.a, lo, mid, hi) {
					stepViolations++
					broke++
				}
			})
			if !slices.IsSorted(good.a) {
				unsorted++
			}
			minIts, maxIts, maxSwaps = min(minIts, good.its), max(maxIts, good.its), max(maxSwaps, good.swaps)

			bad := &counted{a: slices.Clone(in)}
			dutch(bad, true, nil)
			if !slices.IsSorted(bad.a) {
				buggyUnsorted++
				if firstBad == nil {
					firstBad = slices.Clone(in)
				}
			}
			bugMinIts, bugMaxIts = min(bugMinIts, bad.its), max(bugMaxIts, bad.its)
		})
		total += inputs
		fmt.Printf("  %2d %6d | %8d %10s %9d %10d | %8d %10s\n", n, inputs, unsorted,
			fmt.Sprintf("%d..%d", minIts, maxIts), maxSwaps, broke, buggyUnsorted, fmt.Sprintf("%d..%d", bugMinIts, bugMaxIts))
	}
	fmt.Printf("  %d inputs in all; the invariant was checked after each of %d iterations: %d violations\n", total, steps, stepViolations)
	fmt.Printf("  first failing input of the buggy version (shortest length, then lexicographic): %v\n", firstBad)
	c = &counted{a: slices.Clone(firstBad)}
	fmt.Printf("  %4s %8s  %-30s %2s %3s %2s  %-18s %s\n", "iter", "examined", "action", "lo", "mid", "hi", "array", "invariant")
	arrived, at := -1, -1 // the value the bug skipped over, and where it sits
	dutch(c, true, func(examined int, action string, lo, mid, hi int) {
		fmt.Printf("  %4d %8d  %-30s %2d %3d %2d  %-18s %v\n", c.its, examined, action, lo, mid, hi, render(c.a, lo, mid, hi), invariant(c.a, lo, mid, hi))
		if strings.HasSuffix(action, "BUG") && arrived < 0 {
			arrived, at = c.a[mid-1], mid-1
		}
	})
	fmt.Printf("  the %d that arrived at index %d was never examined: mid stepped past it, and the result is %v\n", arrived, at, c.a)

	fmt.Println()
	const n = 1_000_000
	fmt.Printf("(c) the same sort three ways, n = %d values from IntN(3)\n", n)
	rng := rand.New(rand.NewPCG(1, 2))
	data := make([]int, n)
	var kinds [3]int
	for i := range data {
		data[i] = rng.IntN(3)
		kinds[data[i]]++
	}
	fmt.Printf("  %d zeros, %d ones, %d twos; a swap costs 2 reads + 2 writes\n", kinds[0], kinds[1], kinds[2])
	type algo struct {
		name   string
		passes int
		run    func(*counted)
		c      *counted
	}
	algos := []*algo{
		{name: "Dutch flag", passes: 1, run: func(c *counted) { dutch(c, false, nil) }},
		{name: "two-way partition twice", passes: 2, run: twoPasses},
		{name: "counting sort", passes: 2, run: countingSort},
	}
	fmt.Printf("  %-24s %6s %10s %10s %10s %9s %9s\n", "algorithm", "passes", "reads", "writes", "swaps", "reads/n", "writes/n")
	var reads, writes []int
	var names []string
	var ref []int
	for _, al := range algos {
		al.c = &counted{a: slices.Clone(data)}
		al.run(al.c)
		if ref == nil {
			ref = al.c.a
		}
		if !slices.Equal(al.c.a, ref) || !slices.IsSorted(al.c.a) {
			fmt.Println("  RESULTS DIFFER")
		}
		fmt.Printf("  %-24s %6d %10d %10d %10d %9.3f %9.3f\n", al.name, al.passes, al.c.reads, al.c.writes, al.c.swaps,
			float64(al.c.reads)/n, float64(al.c.writes)/n)
		reads, writes, names = append(reads, al.c.reads), append(writes, al.c.writes), append(names, al.name)
	}
	fmt.Printf("  all three produce the same sorted array; swaps: flag = zeros + twos is %v, two passes = zeros + ones is %v\n",
		algos[0].c.swaps == kinds[0]+kinds[2], algos[1].c.swaps == kinds[0]+kinds[1])
	fmt.Printf("  fewest reads: %s; fewest writes: %s\n", argmins(names, reads), argmins(names, writes))

	fmt.Println()
	fmt.Println("this three-region partition is the engine of lesson 13 example 9: its quicksort keeps")
	fmt.Println("(< pivot | == pivot | unknown | > pivot) with the pivot's value playing the role of 1.")
}

// argmins names the algorithm(s) with the smallest value.
func argmins(names []string, vals []int) string {
	best := slices.Min(vals)
	var out []string
	for i, v := range vals {
		if v == best {
			out = append(out, fmt.Sprintf("%s (%d)", names[i], v))
		}
	}
	return strings.Join(out, " and ")
}
```

**Output:**

```
(a) trace on [2 0 2 1 1 0]: | marks lo, mid, hi (left to right)
  iter examined  action                    lo mid hi  array              invariant
     0        -  start                      0   0  6  || 2 0 2 1 1 0 |   true
     1        2  hi-- swap(mid,hi)          0   0  5  || 0 0 2 1 1 | 2   true
     2        0  swap(lo,mid) lo++ mid++    1   1  5  0 || 0 2 1 1 | 2   true
     3        0  swap(lo,mid) lo++ mid++    2   2  5  0 0 || 2 1 1 | 2   true
     4        2  hi-- swap(mid,hi)          2   2  4  0 0 || 1 1 | 2 2   true
     5        1  mid++                      2   3  4  0 0 | 1 | 1 | 2 2  true
     6        1  mid++                      2   4  4  0 0 | 1 1 || 2 2   true
  invariant held after 6 of 6 iterations; loop ran 6 times for n = 6; result [0 0 1 1 2 2]

(b) exhaustive: every array over {0,1,2} of length 0..10
   n inputs | correct version                         | buggy (mid++ after hi swap)
            | unsorted iterations max swaps inv. broke | unsorted iterations
   0      1 |        0       0..0         0          0 |        0       0..0
   1      3 |        0       1..1         1          0 |        0       1..1
   2      9 |        0       2..2         2          0 |        0       1..2
   3     27 |        0       3..3         3          0 |        2       2..3
   4     81 |        0       4..4         4          0 |       15       2..4
   5    243 |        0       5..5         5          0 |       73       3..5
   6    729 |        0       6..6         6          0 |      300       3..6
   7   2187 |        0       7..7         7          0 |     1118       4..7
   8   6561 |        0       8..8         8          0 |     3922       4..8
   9  19683 |        0       9..9         9          0 |    13210       5..9
  10  59049 |        0     10..10        10          0 |    43245      5..10
  88573 inputs in all; the invariant was checked after each of 841449 iterations: 0 violations
  first failing input of the buggy version (shortest length, then lexicographic): [1 2 0]
  iter examined  action                         lo mid hi  array              invariant
     1        1  mid++                           0   1  3  | 1 | 2 0 |        true
     2        2  hi-- swap(mid,hi) mid++ BUG     0   2  2  | 1 0 || 2         false
  the 0 that arrived at index 1 was never examined: mid stepped past it, and the result is [1 0 2]

(c) the same sort three ways, n = 1000000 values from IntN(3)
  333755 zeros, 332849 ones, 333396 twos; a swap costs 2 reads + 2 writes
  algorithm                passes      reads     writes      swaps   reads/n  writes/n
  Dutch flag                    1    2334302    1334302     667151     2.334     1.334
  two-way partition twice       2    2999453    1333208     666604     2.999     1.333
  counting sort                 2    1000000    1000000          0     1.000     1.000
  all three produce the same sorted array; swaps: flag = zeros + twos is true, two passes = zeros + ones is true
  fewest reads: counting sort (1000000); fewest writes: counting sort (1000000)

this three-region partition is the engine of lesson 13 example 9: its quicksort keeps
(< pivot | == pivot | unknown | > pivot) with the pivot's value playing the role of 1.
```

**Rust:**

```rust
use std::cmp::Ordering;

// The Dutch national flag sorts three classes (0, 1, 2) in ONE pass, in place.
// Four regions partition the array at every moment:
//
//     [0, lo)    all 0   -- done
//     [lo, mid)  all 1   -- done
//     [mid, hi)  unknown -- not examined yet
//     [hi, n)    all 2   -- done
//
// Start with lo = mid = 0 and hi = n: everything is unknown. Each iteration
// looks at a[mid] and shrinks the unknown region by EXACTLY one:
//
//     a[mid] == 0: swap it with a[lo]; lo++, mid++   (what comes back to a[mid] is a 1,
//                  because [lo,mid) holds only 1s -- or a[mid] itself when lo == mid --
//                  so mid may advance)
//     a[mid] == 1: mid++                              (already in the right region)
//     a[mid] == 2: hi--, swap a[mid] with a[hi]       (mid must NOT advance: the value
//                  that arrived from the unknown region has not been examined)
//
// So the loop runs exactly n times, whatever the input. This example PROVES the
// partition instead of trusting it: the invariant is checked after every step,
// every input of length <= 10 is tried, and the one-line bug is found.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so that seed (1, 2) gives Rust exactly the arrays Go sees and every
// count below comes out identical. Only next_u64 and int_n are needed.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        // state = state * MUL + INC, with explicit 128-bit wraparound
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;

        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

/// A vector that counts its own accesses. A swap is 2 reads and 2 writes, even
/// when i == j (the swap statement still runs).
struct Counted {
    a: Vec<u8>,
    reads: usize,
    writes: usize,
    swaps: usize,
    its: usize, // loop iterations
}

impl Counted {
    fn new(a: Vec<u8>) -> Self {
        Counted {
            a,
            reads: 0,
            writes: 0,
            swaps: 0,
            its: 0,
        }
    }

    fn get(&mut self, i: usize) -> u8 {
        self.reads += 1;
        self.a[i]
    }

    fn set(&mut self, i: usize, v: u8) {
        self.writes += 1;
        self.a[i] = v;
    }

    fn swap(&mut self, i: usize, j: usize) {
        self.reads += 2;
        self.writes += 2;
        self.swaps += 1;
        self.a.swap(i, j); // slice::swap, where Go writes a[i], a[j] = a[j], a[i]
    }
}

/// What the hook is told after every iteration: the value that was examined, a
/// description of the action, and the new lo, mid, hi.
struct Event {
    examined: u8,
    action: &'static str,
    lo: usize,
    mid: usize,
    hi: usize,
}

/// The one-pass partition. With `buggy` set it advances mid after the swap with
/// a[hi-1] -- the one-line bug. The hook gets the counted vector back, because
/// the closure cannot borrow it while `dutch` holds it mutably.
fn dutch(c: &mut Counted, buggy: bool, mut hook: impl FnMut(&Counted, Event)) {
    let (mut lo, mut mid, mut hi) = (0, 0, c.a.len());
    while mid < hi {
        c.its += 1;
        let v = c.get(mid);
        // Go uses a switch; Rust matches on how the value compares with 1.
        let action = match v.cmp(&1) {
            Ordering::Less => {
                c.swap(lo, mid);
                lo += 1;
                mid += 1;
                "swap(lo,mid) lo++ mid++"
            }
            Ordering::Equal => {
                mid += 1;
                "mid++"
            }
            Ordering::Greater => {
                // mid < hi, so hi >= 1 and this cannot underflow the usize
                hi -= 1;
                c.swap(mid, hi);
                if buggy {
                    mid += 1;
                    "hi-- swap(mid,hi) mid++ BUG"
                } else {
                    "hi-- swap(mid,hi)"
                }
            }
        };
        hook(
            c,
            Event {
                examined: v,
                action,
                lo,
                mid,
                hi,
            },
        );
    }
}

/// Verify the contents of the three known regions.
fn invariant(a: &[u8], lo: usize, mid: usize, hi: usize) -> bool {
    if lo > mid || mid > hi || hi > a.len() {
        return false;
    }
    a.iter()
        .enumerate()
        .all(|(i, &v)| !(i < lo && v != 0 || (lo..mid).contains(&i) && v != 1 || i >= hi && v != 2))
}

/// Print the array with a | at the gaps lo, mid and hi (left to right that is
/// lo, mid, hi whenever lo <= mid <= hi).
fn render(a: &[u8], lo: usize, mid: usize, hi: usize) -> String {
    let mut tokens: Vec<String> = Vec::new();
    for p in 0..=a.len() {
        let bars = [lo, mid, hi].iter().filter(|&&m| m == p).count();
        if bars > 0 {
            tokens.push("|".repeat(bars));
        }
        if p < a.len() {
            tokens.push(a[p].to_string());
        }
    }
    tokens.join(" ")
}

// Go prints a slice as [0 0 1]; Rust's {:?} prints [0, 0, 1]. Format by hand
// so the output reads the same in both languages.
fn go_slice(xs: &[u8]) -> String {
    let items: Vec<String> = xs.iter().map(u8::to_string).collect();
    format!("[{}]", items.join(" "))
}

/// Call f on all 3^n arrays over {0,1,2}, in lexicographic order (a[0] is the
/// most significant base-3 digit). `digits` counts the digits not yet known to
/// be 2, which avoids a `usize` going below zero.
fn for_each_input(n: usize, mut f: impl FnMut(&[u8])) {
    let mut a = vec![0u8; n];
    loop {
        f(&a);
        let mut digits = n;
        while digits > 0 && a[digits - 1] == 2 {
            a[digits - 1] = 0;
            digits -= 1;
        }
        if digits == 0 {
            return;
        }
        a[digits - 1] += 1;
    }
}

/// The alternative: partition "is 0" to the front, then partition "is 1" over
/// what is left.
fn two_passes(c: &mut Counted) {
    let mut w = 0;
    for i in 0..c.a.len() {
        if c.get(i) == 0 {
            c.swap(i, w);
            w += 1;
        }
    }
    let mut w2 = w;
    for i in w..c.a.len() {
        if c.get(i) == 1 {
            c.swap(i, w2);
            w2 += 1;
        }
    }
}

/// Count the three values, then rewrite the array. It cannot move records,
/// only rebuild keys, but for bare 0/1/2 values it is a valid rival.
fn counting_sort(c: &mut Counted) {
    let mut cnt = [0usize; 3];
    for i in 0..c.a.len() {
        cnt[c.get(i) as usize] += 1;
    }
    let mut k = 0;
    for (v, count) in cnt.iter().enumerate() {
        for _ in 0..*count {
            c.set(k, v as u8);
            k += 1;
        }
    }
}

type Sorter = fn(&mut Counted);

fn flag(c: &mut Counted) {
    dutch(c, false, |_, _| {});
}

/// Name the algorithm(s) with the smallest value.
fn argmins(names: &[&str], vals: &[usize]) -> String {
    let best = vals.iter().min().expect("at least one algorithm");
    let out: Vec<String> = names
        .iter()
        .zip(vals)
        .filter(|&(_, v)| v == best)
        .map(|(name, v)| format!("{name} ({v})"))
        .collect();
    out.join(" and ")
}

fn main() {
    println!("(a) trace on [2 0 2 1 1 0]: | marks lo, mid, hi (left to right)");
    let a: Vec<u8> = vec![2, 0, 2, 1, 1, 0];
    let mut c = Counted::new(a.clone());
    println!(
        "  {:>4} {:>8}  {:<25} {:>2} {:>3} {:>2}  {:<18} invariant",
        "iter", "examined", "action", "lo", "mid", "hi", "array"
    );
    println!(
        "  {:>4} {:>8}  {:<25} {:>2} {:>3} {:>2}  {:<18} {}",
        0,
        "-",
        "start",
        0,
        0,
        a.len(),
        render(&c.a, 0, 0, a.len()),
        invariant(&c.a, 0, 0, a.len())
    );
    let mut held = 0;
    dutch(&mut c, false, |c, ev| {
        let ok = invariant(&c.a, ev.lo, ev.mid, ev.hi);
        if ok {
            held += 1;
        }
        println!(
            "  {:>4} {:>8}  {:<25} {:>2} {:>3} {:>2}  {:<18} {}",
            c.its,
            ev.examined,
            ev.action,
            ev.lo,
            ev.mid,
            ev.hi,
            render(&c.a, ev.lo, ev.mid, ev.hi),
            ok
        );
    });
    println!(
        "  invariant held after {held} of {} iterations; loop ran {} times for n = {}; result {}",
        c.its,
        c.its,
        a.len(),
        go_slice(&c.a)
    );

    println!();
    println!("(b) exhaustive: every array over {{0,1,2}} of length 0..10");
    println!(
        "  {:>2} {:>6} | {:<39} | {:<27}",
        "n", "inputs", "correct version", "buggy (mid++ after hi swap)"
    );
    println!(
        "  {:>2} {:>6} | {:>8} {:>10} {:>9} {:>10} | {:>8} {:>10}",
        "", "", "unsorted", "iterations", "max swaps", "inv. broke", "unsorted", "iterations"
    );
    let (mut total, mut steps, mut step_violations) = (0, 0, 0);
    let mut first_bad: Option<Vec<u8>> = None;
    for n in 0..=10usize {
        let (mut inputs, mut unsorted, mut buggy_unsorted, mut max_swaps, mut broke) =
            (0, 0, 0, 0, 0);
        let (mut min_its, mut max_its, mut bug_min_its, mut bug_max_its) = (n, 0, n, 0);
        for_each_input(n, |input| {
            inputs += 1;
            let mut good = Counted::new(input.to_vec());
            dutch(&mut good, false, |c, ev| {
                steps += 1;
                if !invariant(&c.a, ev.lo, ev.mid, ev.hi) {
                    step_violations += 1;
                    broke += 1;
                }
            });
            if !good.a.is_sorted() {
                unsorted += 1;
            }
            min_its = min_its.min(good.its);
            max_its = max_its.max(good.its);
            max_swaps = max_swaps.max(good.swaps);

            let mut bad = Counted::new(input.to_vec());
            dutch(&mut bad, true, |_, _| {});
            if !bad.a.is_sorted() {
                buggy_unsorted += 1;
                if first_bad.is_none() {
                    first_bad = Some(input.to_vec());
                }
            }
            bug_min_its = bug_min_its.min(bad.its);
            bug_max_its = bug_max_its.max(bad.its);
        });
        total += inputs;
        println!(
            "  {:>2} {:>6} | {:>8} {:>10} {:>9} {:>10} | {:>8} {:>10}",
            n,
            inputs,
            unsorted,
            format!("{min_its}..{max_its}"),
            max_swaps,
            broke,
            buggy_unsorted,
            format!("{bug_min_its}..{bug_max_its}")
        );
    }
    println!(
        "  {total} inputs in all; the invariant was checked after each of {steps} iterations: {step_violations} violations"
    );
    let first_bad = first_bad.expect("the buggy version fails somewhere");
    println!(
        "  first failing input of the buggy version (shortest length, then lexicographic): {}",
        go_slice(&first_bad)
    );
    let mut c = Counted::new(first_bad.clone());
    println!(
        "  {:>4} {:>8}  {:<30} {:>2} {:>3} {:>2}  {:<18} invariant",
        "iter", "examined", "action", "lo", "mid", "hi", "array"
    );
    let mut arrived: Option<(u8, usize)> = None; // the value the bug skipped over, and where it sits
    dutch(&mut c, true, |c, ev| {
        println!(
            "  {:>4} {:>8}  {:<30} {:>2} {:>3} {:>2}  {:<18} {}",
            c.its,
            ev.examined,
            ev.action,
            ev.lo,
            ev.mid,
            ev.hi,
            render(&c.a, ev.lo, ev.mid, ev.hi),
            invariant(&c.a, ev.lo, ev.mid, ev.hi)
        );
        if ev.action.ends_with("BUG") && arrived.is_none() {
            arrived = Some((c.a[ev.mid - 1], ev.mid - 1));
        }
    });
    let (value, at) = arrived.expect("the failing input triggers the bug");
    println!(
        "  the {value} that arrived at index {at} was never examined: mid stepped past it, and the result is {}",
        go_slice(&c.a)
    );

    println!();
    const N: usize = 1_000_000;
    println!("(c) the same sort three ways, n = {N} values from IntN(3)");
    let mut rng = Pcg::new(1, 2);
    let mut kinds = [0usize; 3];
    let data: Vec<u8> = (0..N)
        .map(|_| {
            let v = rng.int_n(3) as u8;
            kinds[v as usize] += 1;
            v
        })
        .collect();
    println!(
        "  {} zeros, {} ones, {} twos; a swap costs 2 reads + 2 writes",
        kinds[0], kinds[1], kinds[2]
    );
    // (name, passes, the sort); a plain `fn` pointer lets one array hold all three
    let algos: [(&str, usize, Sorter); 3] = [
        ("Dutch flag", 1, flag),
        ("two-way partition twice", 2, two_passes),
        ("counting sort", 2, counting_sort),
    ];
    println!(
        "  {:<24} {:>6} {:>10} {:>10} {:>10} {:>9} {:>9}",
        "algorithm", "passes", "reads", "writes", "swaps", "reads/n", "writes/n"
    );
    let (mut reads, mut writes, mut names) = (Vec::new(), Vec::new(), Vec::new());
    let mut results: Vec<Counted> = Vec::new();
    for (name, passes, run) in algos {
        let mut c = Counted::new(data.clone());
        run(&mut c);
        if results.first().is_some_and(|r| r.a != c.a) || !c.a.is_sorted() {
            println!("  RESULTS DIFFER");
        }
        println!(
            "  {:<24} {:>6} {:>10} {:>10} {:>10} {:>9.3} {:>9.3}",
            name,
            passes,
            c.reads,
            c.writes,
            c.swaps,
            c.reads as f64 / N as f64,
            c.writes as f64 / N as f64
        );
        reads.push(c.reads);
        writes.push(c.writes);
        names.push(name);
        results.push(c);
    }
    println!(
        "  all three produce the same sorted array; swaps: flag = zeros + twos is {}, two passes = zeros + ones is {}",
        results[0].swaps == kinds[0] + kinds[2],
        results[1].swaps == kinds[0] + kinds[1]
    );
    println!(
        "  fewest reads: {}; fewest writes: {}",
        argmins(&names, &reads),
        argmins(&names, &writes)
    );

    println!();
    println!(
        "this three-region partition is the engine of lesson 13 example 9: its quicksort keeps"
    );
    println!(
        "(< pivot | == pivot | unknown | > pivot) with the pivot's value playing the role of 1."
    );
}
```

**Rust output:**

```
(a) trace on [2 0 2 1 1 0]: | marks lo, mid, hi (left to right)
  iter examined  action                    lo mid hi  array              invariant
     0        -  start                      0   0  6  || 2 0 2 1 1 0 |   true
     1        2  hi-- swap(mid,hi)          0   0  5  || 0 0 2 1 1 | 2   true
     2        0  swap(lo,mid) lo++ mid++    1   1  5  0 || 0 2 1 1 | 2   true
     3        0  swap(lo,mid) lo++ mid++    2   2  5  0 0 || 2 1 1 | 2   true
     4        2  hi-- swap(mid,hi)          2   2  4  0 0 || 1 1 | 2 2   true
     5        1  mid++                      2   3  4  0 0 | 1 | 1 | 2 2  true
     6        1  mid++                      2   4  4  0 0 | 1 1 || 2 2   true
  invariant held after 6 of 6 iterations; loop ran 6 times for n = 6; result [0 0 1 1 2 2]

(b) exhaustive: every array over {0,1,2} of length 0..10
   n inputs | correct version                         | buggy (mid++ after hi swap)
            | unsorted iterations max swaps inv. broke | unsorted iterations
   0      1 |        0       0..0         0          0 |        0       0..0
   1      3 |        0       1..1         1          0 |        0       1..1
   2      9 |        0       2..2         2          0 |        0       1..2
   3     27 |        0       3..3         3          0 |        2       2..3
   4     81 |        0       4..4         4          0 |       15       2..4
   5    243 |        0       5..5         5          0 |       73       3..5
   6    729 |        0       6..6         6          0 |      300       3..6
   7   2187 |        0       7..7         7          0 |     1118       4..7
   8   6561 |        0       8..8         8          0 |     3922       4..8
   9  19683 |        0       9..9         9          0 |    13210       5..9
  10  59049 |        0     10..10        10          0 |    43245      5..10
  88573 inputs in all; the invariant was checked after each of 841449 iterations: 0 violations
  first failing input of the buggy version (shortest length, then lexicographic): [1 2 0]
  iter examined  action                         lo mid hi  array              invariant
     1        1  mid++                           0   1  3  | 1 | 2 0 |        true
     2        2  hi-- swap(mid,hi) mid++ BUG     0   2  2  | 1 0 || 2         false
  the 0 that arrived at index 1 was never examined: mid stepped past it, and the result is [1 0 2]

(c) the same sort three ways, n = 1000000 values from IntN(3)
  333755 zeros, 332849 ones, 333396 twos; a swap costs 2 reads + 2 writes
  algorithm                passes      reads     writes      swaps   reads/n  writes/n
  Dutch flag                    1    2334302    1334302     667151     2.334     1.334
  two-way partition twice       2    2999453    1333208     666604     2.999     1.333
  counting sort                 2    1000000    1000000          0     1.000     1.000
  all three produce the same sorted array; swaps: flag = zeros + twos is true, two passes = zeros + ones is true
  fewest reads: counting sort (1000000); fewest writes: counting sort (1000000)

this three-region partition is the engine of lesson 13 example 9: its quicksort keeps
(< pivot | == pivot | unknown | > pivot) with the pivot's value playing the role of 1.
```

> **Go vs Rust:** The output is identical to Go's, line for line. `match v.cmp(&1)` on `Ordering` replaces the `switch`, and in the `Greater` arm `hi -= 1` cannot underflow because the loop guard `mid < hi` gives `hi >= 1`. The trace hook receives the `Counted` vector as an argument: a closure cannot borrow it while `dutch` holds it mutably, which is the borrow checker doing what the Go closure over `c` silently allowed.

**Complexity:** Θ(n) time, Θ(1) space. The loop ran exactly n iterations on all 88,573 arrays of length 0 to 10 (the minimum equals the maximum at every length), did at most n swaps, left 0 unsorted results, and the invariant held after each of 841,449 iterations. The buggy version (`mid++` after the swap with `hi`) sorts wrongly on 2 of the 27 arrays of length 3 and on 43,245 of the 59,049 of length 10; the first failure is `[1 2 0]`, the result `[1 0 2]`. On 1,000,000 random values the flag read 2.334 n elements and wrote 1.334 n, the two-pass version read 2.999 n and wrote 1.333 n, and counting sort did 1.000 n of each but only rebuilds keys, it cannot move records.

---

## 9. Minimum window substring

`🟡 medium` · *Variable window, shortest valid*

Find the shortest substring of `s` that contains every character of `t`, with multiplicity. Grow `right` until the window is valid, then shrink `left` *while it is still valid*, recording the best inside that loop. The test must be O(1), so keep `need[c]` and a `missing` counter updated on every add and remove and never recount. Each index enters the window once and leaves at most once, so there are at most 2n moves, Θ(n + m) time and Θ(σ) space for an alphabet of σ symbols. Shrinking while valid is correct because the predicate is *expand-closed*: a valid window stays valid when it grows. The program also decides what a "character" is. It counts **runes**, not bytes, and shows on multi-byte text what the byte version gets wrong.

**Steps:**

1. Trace `"ADOBECODEBANC"` with `t = "ABC"`: every right move, the moment `missing` hits 0, each shrink step and each record.
2. Count moves against a version that recounts the window on every check and against all n(n+1)/2 substrings, check 20,000 random cases against an independent oracle, then break the multiplicity with a set-of-characters bug.
3. Run the edge cases, then feed UTF-8 text to the byte version and the rune version and compare both with a rune oracle.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"strings"
	"unicode/utf8"
)

// Minimum window substring: the SHORTEST substring of s that contains every
// character of t, with multiplicity (t = "aab" needs two a's and a b).
//
// The window s[left:right+1] grows on the right until it is valid, then shrinks
// from the left WHILE it is still valid, recording the best inside that loop.
// Validity is O(1): need[c] says how many more copies of c the window wants
// (negative = surplus) and missing counts the characters of t not yet covered;
// both change on every add and remove, so nothing is ever recounted.
//
// Why shrinking while valid is right: validity is EXPAND-CLOSED -- a valid
// window stays valid when it grows -- so once s[left:right+1] is valid, every
// window with the same left and a larger right is valid and longer. For this
// left, the shortest is the one we have; drop the left character and move on.
//
// BYTES OR RUNES? The core works on symbol ids, so it serves both. This
// program's text is ASCII almost everywhere, where a byte IS a character. Where
// it is not (section e), the unit of "character" is the RUNE (a Unicode code
// point): the answer is a substring of runes, and every byte offset must fall on
// a rune boundary. A rune is still not what a reader sees: e + U+0301 is two
// runes (lesson 11, example 15).

// stats counts what each version did.
type stats struct {
	rightMoves, leftMoves int
	reads                 int // reads of s by the recounting and brute-force versions
	substrings            int // (left, right) pairs a brute-force version examined
}

// event kinds for the optional trace of minWindow.
const (
	evAdd    = "add"    // idx = right; missing goes before -> after
	evDrop   = "drop"   // idx = left being dropped
	evRecord = "record" // idx = lo, before = hi (exclusive)
)

// minWindow returns the shortest, then leftmost, window s[lo:hi] covering t.
// s and t are symbol ids in [0, sigma). An empty t has no window by convention.
func minWindow(s, t []int, sigma int, st *stats, on func(kind string, idx, before, after int)) (lo, hi int, ok bool) {
	if len(t) == 0 {
		return 0, 0, false
	}
	need := make([]int, sigma) // need[c] > 0: still wanted; <= 0: covered (with surplus)
	for _, c := range t {
		need[c]++
	}
	missing := len(t) // characters of t, with multiplicity, not yet covered
	left := 0
	bestLen := len(s) + 1
	for right, c := range s {
		st.rightMoves++
		before := missing
		if need[c] > 0 {
			missing--
		}
		need[c]--
		if on != nil {
			on(evAdd, right, before, missing)
		}
		for missing == 0 { // shrink while the window is still valid
			if right+1-left < bestLen { // record INSIDE the loop; strict < keeps the leftmost
				lo, hi, bestLen, ok = left, right+1, right+1-left, true
				if on != nil {
					on(evRecord, lo, hi, 0)
				}
			}
			d := s[left]
			before := missing
			need[d]++
			if need[d] > 0 {
				missing++
			}
			if on != nil {
				on(evDrop, left, before, missing)
			}
			left++
			st.leftMoves++
		}
	}
	return lo, hi, ok
}

// covers recounts a window from scratch: len(window) reads, no shortcuts.
func covers(window, want, have []int, reads *int) bool {
	clear(have)
	for _, c := range window {
		have[c]++
		*reads++
	}
	for c := range want {
		if have[c] < want[c] {
			return false
		}
	}
	return true
}

func counts(t []int, sigma int) []int {
	want := make([]int, sigma)
	for _, c := range t {
		want[c]++
	}
	return want
}

// minWindowRecount is the same two-pointer walk with the validity test written
// the obvious way: recount the window on every check.
func minWindowRecount(s, t []int, sigma int, st *stats) (lo, hi int, ok bool) {
	if len(t) == 0 {
		return 0, 0, false
	}
	want, have := counts(t, sigma), make([]int, sigma)
	left, bestLen := 0, len(s)+1
	for right := range s {
		st.rightMoves++
		for covers(s[left:right+1], want, have, &st.reads) {
			if right+1-left < bestLen {
				lo, hi, bestLen, ok = left, right+1, right+1-left, true
			}
			left++
			st.leftMoves++
		}
	}
	return lo, hi, ok
}

// minWindowAll is brute force with incremental counts: for every left, extend
// right one character at a time -- all n(n+1)/2 substrings, O(1) per substring.
func minWindowAll(s, t []int, sigma int, st *stats) (lo, hi int, ok bool) {
	if len(t) == 0 {
		return 0, 0, false
	}
	base, need := counts(t, sigma), make([]int, sigma)
	bestLen := len(s) + 1
	for l := range s {
		copy(need, base)
		missing := len(t)
		for r := l; r < len(s); r++ {
			st.reads++
			st.substrings++
			if need[s[r]] > 0 {
				missing--
			}
			need[s[r]]--
			if missing == 0 && r+1-l < bestLen {
				lo, hi, bestLen, ok = l, r+1, r+1-l, true
			}
		}
	}
	return lo, hi, ok
}

// oracle shares nothing with the versions above: every window, shortest first
// and leftmost first, tested by recounting it from scratch.
func oracle(s, t []int, sigma int) (lo, hi int, ok bool) {
	if len(t) == 0 {
		return 0, 0, false
	}
	want, have, reads := counts(t, sigma), make([]int, sigma), 0
	for size := 1; size <= len(s); size++ {
		for l := 0; l+size <= len(s); l++ {
			if covers(s[l:l+size], want, have, &reads) {
				return l, l + size, true
			}
		}
	}
	return 0, 0, false
}

// minWindowSet is the bug: it remembers WHICH characters t needs, not how many
// of each, so "aa" is satisfied by a single a.
func minWindowSet(s, t []int, sigma int) (lo, hi int, ok bool) {
	if len(t) == 0 {
		return 0, 0, false
	}
	wanted, have := make([]bool, sigma), make([]int, sigma)
	missing := 0 // distinct characters of t not yet present
	for _, c := range t {
		if !wanted[c] {
			wanted[c] = true
			missing++
		}
	}
	left, bestLen := 0, len(s)+1
	for right, c := range s {
		if wanted[c] {
			have[c]++
			if have[c] == 1 {
				missing--
			}
		}
		for missing == 0 {
			if right+1-left < bestLen {
				lo, hi, bestLen, ok = left, right+1, right+1-left, true
			}
			if d := s[left]; wanted[d] {
				have[d]--
				if have[d] == 0 {
					missing++
				}
			}
			left++
		}
	}
	return lo, hi, ok
}

// ---- bytes and runes ----

// found is a window: [lo, hi) in the unit of the version (bytes or runes) and
// [blo, bhi) as byte offsets into the string.
type found struct {
	lo, hi, blo, bhi int
	ok               bool
}

func byteIDs(s string) []int {
	ids := make([]int, len(s))
	for i := 0; i < len(s); i++ {
		ids[i] = int(s[i])
	}
	return ids
}

// windowBytes treats every BYTE as a character: sigma is 256 and the window
// is a byte range. Correct for ASCII; on other UTF-8 it answers about bytes.
func windowBytes(s, t string, st *stats) found {
	lo, hi, ok := minWindow(byteIDs(s), byteIDs(t), 256, st, nil)
	return found{lo, hi, lo, hi, ok}
}

// runeIDs maps every distinct rune of s and t to a small id and remembers the
// byte offset where each rune of s starts (plus len(s) at the end).
func runeIDs(s, t string) (sIDs, tIDs, offs []int, sigma int) {
	ids := map[rune]int{}
	id := func(r rune) int {
		if _, seen := ids[r]; !seen {
			ids[r] = len(ids)
		}
		return ids[r]
	}
	for i, r := range s {
		sIDs, offs = append(sIDs, id(r)), append(offs, i)
	}
	offs = append(offs, len(s))
	for _, r := range t {
		tIDs = append(tIDs, id(r))
	}
	return sIDs, tIDs, offs, len(ids)
}

// windowRunes treats every RUNE as a character. The window is a rune range and
// its byte offsets come from offs, so they are always on rune boundaries.
func windowRunes(s, t string, st *stats) found {
	sIDs, tIDs, offs, sigma := runeIDs(s, t)
	lo, hi, ok := minWindow(sIDs, tIDs, sigma, st, nil)
	if !ok {
		return found{}
	}
	return found{lo, hi, offs[lo], offs[hi], true}
}

func oracleRunes(s, t string) found {
	sIDs, tIDs, offs, sigma := runeIDs(s, t)
	lo, hi, ok := oracle(sIDs, tIDs, sigma)
	if !ok {
		return found{}
	}
	return found{lo, hi, offs[lo], offs[hi], true}
}

// show prints a result for humans: the quoted window, or "none".
func show(s string, f found) string {
	if !f.ok {
		return "none"
	}
	return fmt.Sprintf("%q", s[f.blo:f.bhi])
}

func randomString(rng *rand.Rand, n int, alphabet []string) string {
	var b strings.Builder
	for range n {
		b.WriteString(alphabet[rng.IntN(len(alphabet))])
	}
	return b.String()
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))

	fmt.Println(`(a) s = "ADOBECODEBANC", t = "ABC": every event (+x adds s[right], -x drops s[left])`)
	s, t := "ADOBECODEBANC", "ABC"
	var st stats
	lo, hi, _ := minWindow(byteIDs(s), byteIDs(t), 256, &st, func(kind string, idx, before, after int) {
		switch kind {
		case evAdd:
			fmt.Printf("  r=%-2d +%c  missing %d->%d", idx, s[idx], before, after)
			if after == 0 {
				fmt.Print("   <- valid, start shrinking")
			}
			fmt.Println()
		case evRecord:
			fmt.Printf("        record [%d,%d) %q (length %d)\n", idx, before, s[idx:before], before-idx)
		case evDrop:
			fmt.Printf("        l=%-2d -%c  missing %d->%d", idx, s[idx], before, after)
			if after > 0 {
				fmt.Print("   <- invalid, stop shrinking")
			}
			fmt.Println()
		}
	})
	fmt.Printf("  answer %q; %d right moves, %d left moves (n = %d)\n", s[lo:hi], st.rightMoves, st.leftMoves, len(s))

	fmt.Println()
	fmt.Println("(b) cost: random s over abcd, t = 4 random letters; moves vs recounting vs all substrings")
	fmt.Printf("  %7s %6s %6s %7s | %12s %9s | %12s %9s\n", "n", "right", "left", "r + l", "recount", "x moves", "substrings", "x moves")
	for _, n := range []int{100, 1000, 10000} {
		s, t := randomString(rng, n, []string{"a", "b", "c", "d"}), randomString(rng, 4, []string{"a", "b", "c", "d"})
		var fast, rec, all stats
		f := windowBytes(s, t, &fast)
		lo, hi, _ := minWindowRecount(byteIDs(s), byteIDs(t), 256, &rec)
		same := lo == f.lo && hi == f.hi
		substrings, mark := n*(n+1)/2, "*"
		if n <= 1000 {
			lo, hi, _ := minWindowAll(byteIDs(s), byteIDs(t), 256, &all)
			same = same && lo == f.lo && hi == f.hi
			substrings, mark = all.substrings, " "
		}
		moves := fast.rightMoves + fast.leftMoves
		fmt.Printf("  %7d %6d %6d %7d | %12d %8.1fx | %12d%s %8.1fx  all versions agree: %v\n", n, fast.rightMoves, fast.leftMoves, moves,
			rec.reads, float64(rec.reads)/float64(moves), substrings, mark, float64(substrings)/float64(moves), same)
	}
	fmt.Println("  * not run: n(n+1)/2 computed. Moves never exceed 2n. Recounting costs only a small constant")
	fmt.Println("  factor here because the window stays short; the next table makes it long.")
	fmt.Println()
	fmt.Println(`  worst case for recounting: s = "a" x (n-1) + "b", t = "ab" (the window is the whole prefix)`)
	fmt.Printf("  %7s %6s %6s %7s %7s | %12s %9s\n", "n", "right", "left", "r + l", "2n", "recount", "x moves")
	for _, n := range []int{1000, 10000} {
		s, t := strings.Repeat("a", n-1)+"b", "ab"
		var fast, rec stats
		f := windowBytes(s, t, &fast)
		minWindowRecount(byteIDs(s), byteIDs(t), 256, &rec)
		moves := fast.rightMoves + fast.leftMoves
		fmt.Printf("  %7d %6d %6d %7d %7d | %12d %8.1fx  answer %q\n", n, fast.rightMoves, fast.leftMoves, moves, 2*n,
			rec.reads, float64(rec.reads)/float64(moves), s[f.blo:f.bhi])
	}

	fmt.Println()
	fmt.Println("(c) oracle: 20,000 random cases (s: 0-12 chars over abc, t: 0-4 chars over abc)")
	type testCase struct{ s, t string }
	var cases []testCase
	for range 20000 {
		abc := []string{"a", "b", "c"}
		s := randomString(rng, rng.IntN(13), abc)
		cases = append(cases, testCase{s, randomString(rng, rng.IntN(5), abc)})
	}
	mismatches, setWrong, found0, emptyT := 0, 0, 0, 0
	var first *testCase
	for i, c := range cases {
		sIDs, tIDs := byteIDs(c.s), byteIDs(c.t)
		wlo, whi, wok := oracle(sIDs, tIDs, 256)
		var st stats
		lo, hi, ok := minWindow(sIDs, tIDs, 256, &st, nil)
		if lo != wlo || hi != whi || ok != wok {
			mismatches++
		}
		if wok {
			found0++
		}
		if len(c.t) == 0 {
			emptyT++
		}
		lo, hi, ok = minWindowSet(sIDs, tIDs, 256)
		if lo != wlo || hi != whi || ok != wok {
			setWrong++
			if first == nil || len(c.s) < len(first.s) || len(c.s) == len(first.s) && len(c.t) < len(first.t) {
				first = &cases[i]
			}
		}
	}
	fmt.Printf("  two pointers vs oracle (shortest, leftmost on ties): %d mismatches of %d (%d had a window, %d had empty t)\n",
		mismatches, len(cases), found0, emptyT)
	fmt.Printf("  the set-of-characters bug: wrong on %d of %d\n", setWrong, len(cases))
	sIDs, tIDs := byteIDs(first.s), byteIDs(first.t)
	wlo, whi, wok := oracle(sIDs, tIDs, 256)
	blo, bhi, bok := minWindowSet(sIDs, tIDs, 256)
	fmt.Printf("  shortest counterexample s = %q, t = %q: oracle %s, bug %s\n", first.s, first.t,
		show(first.s, found{wlo, whi, wlo, whi, wok}), show(first.s, found{blo, bhi, blo, bhi, bok}))

	fmt.Println()
	fmt.Println("(d) edge cases (right moves / left moves)")
	for _, c := range []testCase{{"ab", "abc"}, {"a", "aa"}, {"abc", ""}, {"", "a"}, {"", ""}, {"abc", "abc"}, {"aab", "aab"}, {"xaay", "aa"}} {
		var st stats
		f := windowBytes(c.s, c.t, &st)
		fmt.Printf("  s = %-6q t = %-6q -> %-7s  (%d / %d)\n", c.s, c.t, show(c.s, f), st.rightMoves, st.leftMoves)
	}
	fmt.Println(`  convention: an empty t has no window; t longer than s, or short of a repeat, has none either`)

	fmt.Println()
	fmt.Println("(e) bytes or runes: this program counts RUNES for text; bytes are only for ASCII")
	utf8Cases := []testCase{{"ADOBECODEBANC", "ABC"}, {"\u00a9\u00e3", "\u00e9"}, {"x\u00e9\u00e3\u00e9y", "\u00e9\u00e9"}, {"a\U0001F600b", "b\U0001F600"}, {"\u65e5\u672c\u8a9e\u65e5", "\u8a9e\u65e5"}}
	for _, c := range utf8Cases {
		var st stats
		b, r := windowBytes(c.s, c.t, &st), windowRunes(c.s, c.t, &st)
		fmt.Printf("  s = %q, t = %q\n", c.s, c.t)
		fmt.Printf("    bytes: %s", show(c.s, b))
		if b.ok {
			fmt.Printf("   valid UTF-8: %v", utf8.ValidString(c.s[b.blo:b.bhi]))
		}
		fmt.Printf("\n    runes: %s", show(c.s, r))
		if r.ok {
			fmt.Printf("   runes [%d,%d) = bytes [%d,%d)", r.lo, r.hi, r.blo, r.bhi)
		}
		fmt.Println()
	}
	alphabet := []string{"a", "\u00e9", "\u00e3", "\u00a9"} // 1, 2, 2, 2 bytes; é and ã share the byte C3, é and © share A9
	runesWrong, bytesWrong, bytesBroken := 0, 0, 0
	const trials = 20000
	var firstBad *testCase
	for range trials {
		c := testCase{randomString(rng, rng.IntN(9), alphabet), randomString(rng, rng.IntN(4), alphabet)}
		want := oracleRunes(c.s, c.t)
		var st stats
		if got := windowRunes(c.s, c.t, &st); show(c.s, got) != show(c.s, want) || got.ok != want.ok {
			runesWrong++
		}
		got := windowBytes(c.s, c.t, &st)
		if show(c.s, got) != show(c.s, want) {
			bytesWrong++
			if firstBad == nil || len(c.s) < len(firstBad.s) {
				cc := c
				firstBad = &cc
			}
		}
		if got.ok && !utf8.ValidString(c.s[got.blo:got.bhi]) {
			bytesBroken++
		}
	}
	fmt.Printf("  %d random cases over a, é, ã, © (1-2 bytes each): the runes version differs from the rune oracle on %d,\n", trials, runesWrong)
	fmt.Printf("  the bytes version on %d, and on %d of those it returned bytes that are not valid UTF-8\n", bytesWrong, bytesBroken)
	want := oracleRunes(firstBad.s, firstBad.t)
	var st2 stats
	fmt.Printf("  shortest bytes failure: s = %q, t = %q: oracle %s, bytes %s\n", firstBad.s, firstBad.t, show(firstBad.s, want), show(firstBad.s, windowBytes(firstBad.s, firstBad.t, &st2)))
}
```

**Output:**

```
(a) s = "ADOBECODEBANC", t = "ABC": every event (+x adds s[right], -x drops s[left])
  r=0  +A  missing 3->2
  r=1  +D  missing 2->2
  r=2  +O  missing 2->2
  r=3  +B  missing 2->1
  r=4  +E  missing 1->1
  r=5  +C  missing 1->0   <- valid, start shrinking
        record [0,6) "ADOBEC" (length 6)
        l=0  -A  missing 0->1   <- invalid, stop shrinking
  r=6  +O  missing 1->1
  r=7  +D  missing 1->1
  r=8  +E  missing 1->1
  r=9  +B  missing 1->1
  r=10 +A  missing 1->0   <- valid, start shrinking
        l=1  -D  missing 0->0
        l=2  -O  missing 0->0
        l=3  -B  missing 0->0
        l=4  -E  missing 0->0
        l=5  -C  missing 0->1   <- invalid, stop shrinking
  r=11 +N  missing 1->1
  r=12 +C  missing 1->0   <- valid, start shrinking
        l=6  -O  missing 0->0
        l=7  -D  missing 0->0
        record [8,13) "EBANC" (length 5)
        l=8  -E  missing 0->0
        record [9,13) "BANC" (length 4)
        l=9  -B  missing 0->1   <- invalid, stop shrinking
  answer "BANC"; 13 right moves, 10 left moves (n = 13)

(b) cost: random s over abcd, t = 4 random letters; moves vs recounting vs all substrings
        n  right   left   r + l |      recount   x moves |   substrings   x moves
      100    100     93     193 |         1545      8.0x |         5050      26.2x  all versions agree: true
     1000   1000    987    1987 |        17764      8.9x |       500500     251.9x  all versions agree: true
    10000  10000   9992   19992 |       236114     11.8x |     50005000*   2501.3x  all versions agree: true
  * not run: n(n+1)/2 computed. Moves never exceed 2n. Recounting costs only a small constant
  factor here because the window stays short; the next table makes it long.

  worst case for recounting: s = "a" x (n-1) + "b", t = "ab" (the window is the whole prefix)
        n  right   left   r + l      2n |      recount   x moves
     1000   1000    999    1999    2000 |      1000000    500.3x  answer "ab"
    10000  10000   9999   19999   20000 |    100000000   5000.3x  answer "ab"

(c) oracle: 20,000 random cases (s: 0-12 chars over abc, t: 0-4 chars over abc)
  two pointers vs oracle (shortest, leftmost on ties): 0 mismatches of 20000 (9010 had a window, 3849 had empty t)
  the set-of-characters bug: wrong on 5634 of 20000
  shortest counterexample s = "a", t = "aa": oracle none, bug "a"

(d) edge cases (right moves / left moves)
  s = "ab"   t = "abc"  -> none     (2 / 0)
  s = "a"    t = "aa"   -> none     (1 / 0)
  s = "abc"  t = ""     -> none     (0 / 0)
  s = ""     t = "a"    -> none     (0 / 0)
  s = ""     t = ""     -> none     (0 / 0)
  s = "abc"  t = "abc"  -> "abc"    (3 / 1)
  s = "aab"  t = "aab"  -> "aab"    (3 / 1)
  s = "xaay" t = "aa"   -> "aa"     (4 / 2)
  convention: an empty t has no window; t longer than s, or short of a repeat, has none either

(e) bytes or runes: this program counts RUNES for text; bytes are only for ASCII
  s = "ADOBECODEBANC", t = "ABC"
    bytes: "BANC"   valid UTF-8: true
    runes: "BANC"   runes [9,13) = bytes [9,13)
  s = "©ã", t = "é"
    bytes: "\xa9\xc3"   valid UTF-8: false
    runes: none
  s = "xéãéy", t = "éé"
    bytes: "\xa9ãé"   valid UTF-8: false
    runes: "éãé"   runes [1,4) = bytes [1,7)
  s = "a😀b", t = "b😀"
    bytes: "😀b"   valid UTF-8: true
    runes: "😀b"   runes [1,3) = bytes [1,6)
  s = "日本語日", t = "語日"
    bytes: "語日"   valid UTF-8: true
    runes: "語日"   runes [2,4) = bytes [6,12)
  20000 random cases over a, é, ã, © (1-2 bytes each): the runes version differs from the rune oracle on 0,
  the bytes version on 1564, and on 1391 of those it returned bytes that are not valid UTF-8
  shortest bytes failure: s = "©é", t = "é": oracle "é", bytes "\xa9\xc3"
```

**Rust:**

```rust
use std::collections::HashMap;

// Minimum window substring: the SHORTEST substring of s that contains every
// character of t, with multiplicity (t = "aab" needs two a's and a b).
//
// The window s[left..=right] grows on the right until it is valid, then shrinks
// from the left WHILE it is still valid, recording the best inside that loop.
// Validity is O(1): need[c] says how many more copies of c the window wants
// (negative = surplus) and missing counts the characters of t not yet covered;
// both change on every add and remove, so nothing is ever recounted.
//
// Why shrinking while valid is right: validity is EXPAND-CLOSED -- a valid
// window stays valid when it grows -- so once s[left..=right] is valid, every
// window with the same left and a larger right is valid and longer. For this
// left, the shortest is the one we have; drop the left character and move on.
//
// BYTES OR RUNES? The core works on symbol ids, so it serves both. This
// program's text is ASCII almost everywhere, where a byte IS a character. Where
// it is not (section e), the unit of "character" is the RUNE (a Unicode code
// point, a Rust `char`): the answer is a substring of runes, and every byte
// offset must fall on a rune boundary. A rune is still not what a reader sees:
// e + U+0301 is two runes (lesson 11, example 15).

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so that seed (1, 2) gives Rust exactly the arrays Go sees and every
// count below comes out identical. Only next_u64 and int_n are needed.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        // state = state * MUL + INC, with explicit 128-bit wraparound
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;

        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

/// What each version did.
#[derive(Default)]
struct Stats {
    right_moves: usize,
    left_moves: usize,
    reads: usize,      // reads of s by the recounting and brute-force versions
    substrings: usize, // (left, right) pairs a brute-force version examined
}

/// The optional trace of `min_window`, one value per event.
enum Event {
    Add {
        right: usize,
        before: usize,
        after: usize,
    }, // missing goes before -> after
    Record {
        lo: usize,
        hi: usize,
    }, // hi is exclusive
    Drop {
        left: usize,
        before: usize,
        after: usize,
    },
}

/// The shortest, then leftmost, window s[lo..hi] covering t, as `Some((lo, hi))`.
/// s and t are symbol ids in 0..sigma. An empty t has no window by convention.
fn min_window(
    s: &[usize],
    t: &[usize],
    sigma: usize,
    st: &mut Stats,
    mut on: impl FnMut(Event),
) -> Option<(usize, usize)> {
    if t.is_empty() {
        return None;
    }
    let mut need = vec![0i32; sigma]; // need[c] > 0: still wanted; <= 0: covered (with surplus)
    for &c in t {
        need[c] += 1;
    }
    let mut missing = t.len(); // characters of t, with multiplicity, not yet covered
    let mut left = 0;
    let mut best = None;
    let mut best_len = s.len() + 1;
    for (right, &c) in s.iter().enumerate() {
        st.right_moves += 1;
        let before = missing;
        if need[c] > 0 {
            missing -= 1;
        }
        need[c] -= 1;
        on(Event::Add {
            right,
            before,
            after: missing,
        });
        while missing == 0 {
            // shrink while the window is still valid
            if right + 1 - left < best_len {
                // record INSIDE the loop; strict < keeps the leftmost
                best_len = right + 1 - left;
                best = Some((left, right + 1));
                on(Event::Record {
                    lo: left,
                    hi: right + 1,
                });
            }
            let d = s[left];
            let before = missing;
            need[d] += 1;
            if need[d] > 0 {
                missing += 1;
            }
            on(Event::Drop {
                left,
                before,
                after: missing,
            });
            left += 1;
            st.left_moves += 1;
        }
    }
    best
}

/// Recount a window from scratch: window.len() reads, no shortcuts.
fn covers(window: &[usize], want: &[i32], have: &mut [i32], reads: &mut usize) -> bool {
    have.fill(0);
    for &c in window {
        have[c] += 1;
        *reads += 1;
    }
    want.iter().zip(have.iter()).all(|(w, h)| h >= w)
}

fn counts(t: &[usize], sigma: usize) -> Vec<i32> {
    let mut want = vec![0; sigma];
    for &c in t {
        want[c] += 1;
    }
    want
}

/// The same two-pointer walk with the validity test written the obvious way:
/// recount the window on every check.
fn min_window_recount(
    s: &[usize],
    t: &[usize],
    sigma: usize,
    st: &mut Stats,
) -> Option<(usize, usize)> {
    if t.is_empty() {
        return None;
    }
    let (want, mut have) = (counts(t, sigma), vec![0; sigma]);
    let (mut left, mut best_len, mut best) = (0, s.len() + 1, None);
    for right in 0..s.len() {
        st.right_moves += 1;
        while covers(&s[left..=right], &want, &mut have, &mut st.reads) {
            if right + 1 - left < best_len {
                best_len = right + 1 - left;
                best = Some((left, right + 1));
            }
            left += 1;
            st.left_moves += 1;
        }
    }
    best
}

/// Brute force with incremental counts: for every left, extend right one
/// character at a time -- all n(n+1)/2 substrings, O(1) per substring.
fn min_window_all(
    s: &[usize],
    t: &[usize],
    sigma: usize,
    st: &mut Stats,
) -> Option<(usize, usize)> {
    if t.is_empty() {
        return None;
    }
    let base = counts(t, sigma);
    let (mut best_len, mut best) = (s.len() + 1, None);
    for l in 0..s.len() {
        let mut need = base.clone();
        let mut missing = t.len();
        for r in l..s.len() {
            st.reads += 1;
            st.substrings += 1;
            if need[s[r]] > 0 {
                missing -= 1;
            }
            need[s[r]] -= 1;
            if missing == 0 && r + 1 - l < best_len {
                best_len = r + 1 - l;
                best = Some((l, r + 1));
            }
        }
    }
    best
}

/// Shares nothing with the versions above: every window, shortest first and
/// leftmost first, tested by recounting it from scratch.
fn oracle(s: &[usize], t: &[usize], sigma: usize) -> Option<(usize, usize)> {
    if t.is_empty() {
        return None;
    }
    let (want, mut have, mut reads) = (counts(t, sigma), vec![0; sigma], 0);
    for size in 1..=s.len() {
        for l in 0..=s.len() - size {
            if covers(&s[l..l + size], &want, &mut have, &mut reads) {
                return Some((l, l + size));
            }
        }
    }
    None
}

/// The bug: it remembers WHICH characters t needs, not how many of each, so
/// "aa" is satisfied by a single a.
fn min_window_set(s: &[usize], t: &[usize], sigma: usize) -> Option<(usize, usize)> {
    if t.is_empty() {
        return None;
    }
    let (mut wanted, mut have) = (vec![false; sigma], vec![0; sigma]);
    let mut missing = 0; // distinct characters of t not yet present
    for &c in t {
        if !wanted[c] {
            wanted[c] = true;
            missing += 1;
        }
    }
    let (mut left, mut best_len, mut best) = (0, s.len() + 1, None);
    for (right, &c) in s.iter().enumerate() {
        if wanted[c] {
            have[c] += 1;
            if have[c] == 1 {
                missing -= 1;
            }
        }
        while missing == 0 {
            if right + 1 - left < best_len {
                best_len = right + 1 - left;
                best = Some((left, right + 1));
            }
            let d = s[left];
            if wanted[d] {
                have[d] -= 1;
                if have[d] == 0 {
                    missing += 1;
                }
            }
            left += 1;
        }
    }
    best
}

// ---- bytes and runes ----

/// A window: [lo, hi) in the unit of the version (bytes or runes) and
/// [blo, bhi) as byte offsets into the string.
#[derive(Clone, Copy)]
struct Found {
    lo: usize,
    hi: usize,
    blo: usize,
    bhi: usize,
}

fn byte_ids(s: &str) -> Vec<usize> {
    s.bytes().map(usize::from).collect()
}

/// Treat every BYTE as a character: sigma is 256 and the window is a byte
/// range. Correct for ASCII; on other UTF-8 it answers about bytes. The result
/// is only a byte range: slicing the `&str` with it (`&s[blo..bhi]`) would
/// PANIC when an end falls inside a rune, so callers slice `s.as_bytes()`.
fn window_bytes(s: &str, t: &str, st: &mut Stats) -> Option<Found> {
    let (lo, hi) = min_window(&byte_ids(s), &byte_ids(t), 256, st, |_| {})?;
    Some(Found {
        lo,
        hi,
        blo: lo,
        bhi: hi,
    })
}

/// Map every distinct rune of s and t to a small id and remember the byte offset
/// where each rune of s starts (plus s.len() at the end).
fn rune_ids(s: &str, t: &str) -> (Vec<usize>, Vec<usize>, Vec<usize>, usize) {
    let mut ids: HashMap<char, usize> = HashMap::new();
    let mut id = |r: char| {
        let next = ids.len();
        *ids.entry(r).or_insert(next)
    };
    let s_ids: Vec<usize> = s.chars().map(&mut id).collect();
    let t_ids: Vec<usize> = t.chars().map(&mut id).collect();
    let mut offs: Vec<usize> = s.char_indices().map(|(i, _)| i).collect();
    offs.push(s.len());
    (s_ids, t_ids, offs, ids.len())
}

/// Treat every RUNE as a character. The window is a rune range and its byte
/// offsets come from `offs`, so they are always on rune boundaries.
fn window_runes(s: &str, t: &str, st: &mut Stats) -> Option<Found> {
    let (s_ids, t_ids, offs, sigma) = rune_ids(s, t);
    let (lo, hi) = min_window(&s_ids, &t_ids, sigma, st, |_| {})?;
    Some(Found {
        lo,
        hi,
        blo: offs[lo],
        bhi: offs[hi],
    })
}

fn oracle_runes(s: &str, t: &str) -> Option<Found> {
    let (s_ids, t_ids, offs, sigma) = rune_ids(s, t);
    let (lo, hi) = oracle(&s_ids, &t_ids, sigma)?;
    Some(Found {
        lo,
        hi,
        blo: offs[lo],
        bhi: offs[hi],
    })
}

/// Quote bytes like Go's %q: valid UTF-8 as is, every invalid byte as \xNN.
fn quote(bytes: &[u8]) -> String {
    let mut out = String::from("\"");
    for chunk in bytes.utf8_chunks() {
        out.push_str(chunk.valid());
        for byte in chunk.invalid() {
            out.push_str(&format!("\\x{byte:02x}"));
        }
    }
    out.push('"');
    out
}

/// A result for humans: the quoted window, or "none".
fn show(s: &str, f: Option<Found>) -> String {
    f.map_or("none".to_string(), |f| quote(&s.as_bytes()[f.blo..f.bhi]))
}

fn is_utf8(s: &str, f: Found) -> bool {
    // Not `&s[blo..bhi]`: that would panic when an end falls inside a rune. The
    // window is bytes first, and from_utf8 is how we ask whether it is also text.
    let window = &s.as_bytes()[f.blo..f.bhi];
    std::str::from_utf8(window).is_ok()
}

fn random_string(rng: &mut Pcg, n: usize, alphabet: &[&str]) -> String {
    (0..n)
        .map(|_| alphabet[rng.int_n(alphabet.len() as u64) as usize])
        .collect()
}

fn span(f: Option<Found>) -> Option<(usize, usize)> {
    f.map(|f| (f.lo, f.hi))
}

fn main() {
    let mut rng = Pcg::new(1, 2);

    println!(
        r#"(a) s = "ADOBECODEBANC", t = "ABC": every event (+x adds s[right], -x drops s[left])"#
    );
    let (s, t) = ("ADOBECODEBANC", "ABC");
    let mut st = Stats::default();
    let (lo, hi) = min_window(&byte_ids(s), &byte_ids(t), 256, &mut st, |ev| match ev {
        Event::Add {
            right,
            before,
            after,
        } => {
            print!(
                "  r={:<2} +{}  missing {}->{}",
                right,
                s.as_bytes()[right] as char,
                before,
                after
            );
            if after == 0 {
                print!("   <- valid, start shrinking");
            }
            println!();
        }
        Event::Record { lo, hi } => {
            println!(
                "        record [{},{}) {} (length {})",
                lo,
                hi,
                quote(&s.as_bytes()[lo..hi]),
                hi - lo
            );
        }
        Event::Drop {
            left,
            before,
            after,
        } => {
            print!(
                "        l={:<2} -{}  missing {}->{}",
                left,
                s.as_bytes()[left] as char,
                before,
                after
            );
            if after > 0 {
                print!("   <- invalid, stop shrinking");
            }
            println!();
        }
    })
    .expect("the example has a window");
    println!(
        "  answer {}; {} right moves, {} left moves (n = {})",
        quote(&s.as_bytes()[lo..hi]),
        st.right_moves,
        st.left_moves,
        s.len()
    );

    println!();
    println!(
        "(b) cost: random s over abcd, t = 4 random letters; moves vs recounting vs all substrings"
    );
    println!(
        "  {:>7} {:>6} {:>6} {:>7} | {:>12} {:>9} | {:>12} {:>9}",
        "n", "right", "left", "r + l", "recount", "x moves", "substrings", "x moves"
    );
    let abcd = ["a", "b", "c", "d"];
    for n in [100, 1000, 10000] {
        let s = random_string(&mut rng, n, &abcd);
        let t = random_string(&mut rng, 4, &abcd);
        let (mut fast, mut rec, mut all) = (Stats::default(), Stats::default(), Stats::default());
        let f = span(window_bytes(&s, &t, &mut fast));
        let mut same = min_window_recount(&byte_ids(&s), &byte_ids(&t), 256, &mut rec) == f;
        let (mut substrings, mut mark) = (n * (n + 1) / 2, "*");
        if n <= 1000 {
            same = same && min_window_all(&byte_ids(&s), &byte_ids(&t), 256, &mut all) == f;
            (substrings, mark) = (all.substrings, " ");
        }
        let moves = fast.right_moves + fast.left_moves;
        println!(
            "  {:>7} {:>6} {:>6} {:>7} | {:>12} {:>8.1}x | {:>12}{} {:>8.1}x  all versions agree: {}",
            n,
            fast.right_moves,
            fast.left_moves,
            moves,
            rec.reads,
            rec.reads as f64 / moves as f64,
            substrings,
            mark,
            substrings as f64 / moves as f64,
            same
        );
    }
    println!(
        "  * not run: n(n+1)/2 computed. Moves never exceed 2n. Recounting costs only a small constant"
    );
    println!("  factor here because the window stays short; the next table makes it long.");
    println!();
    println!(
        r#"  worst case for recounting: s = "a" x (n-1) + "b", t = "ab" (the window is the whole prefix)"#
    );
    println!(
        "  {:>7} {:>6} {:>6} {:>7} {:>7} | {:>12} {:>9}",
        "n", "right", "left", "r + l", "2n", "recount", "x moves"
    );
    for n in [1000, 10000] {
        let s = "a".repeat(n - 1) + "b";
        let t = "ab";
        let (mut fast, mut rec) = (Stats::default(), Stats::default());
        let f = window_bytes(&s, t, &mut fast).expect("the b closes a window");
        min_window_recount(&byte_ids(&s), &byte_ids(t), 256, &mut rec);
        let moves = fast.right_moves + fast.left_moves;
        println!(
            "  {:>7} {:>6} {:>6} {:>7} {:>7} | {:>12} {:>8.1}x  answer {}",
            n,
            fast.right_moves,
            fast.left_moves,
            moves,
            2 * n,
            rec.reads,
            rec.reads as f64 / moves as f64,
            quote(&s.as_bytes()[f.blo..f.bhi])
        );
    }

    println!();
    println!("(c) oracle: 20,000 random cases (s: 0-12 chars over abc, t: 0-4 chars over abc)");
    let abc = ["a", "b", "c"];
    let mut cases: Vec<(String, String)> = Vec::new();
    for _ in 0..20000 {
        let n = rng.int_n(13) as usize;
        let s = random_string(&mut rng, n, &abc);
        let n = rng.int_n(5) as usize;
        cases.push((s, random_string(&mut rng, n, &abc)));
    }
    let (mut mismatches, mut set_wrong, mut found0, mut empty_t) = (0, 0, 0, 0);
    let mut first: Option<usize> = None;
    for (i, (s, t)) in cases.iter().enumerate() {
        let (s_ids, t_ids) = (byte_ids(s), byte_ids(t));
        let want = oracle(&s_ids, &t_ids, 256);
        let got = min_window(&s_ids, &t_ids, 256, &mut Stats::default(), |_| {});
        if got != want {
            mismatches += 1;
        }
        if want.is_some() {
            found0 += 1;
        }
        if t.is_empty() {
            empty_t += 1;
        }
        if min_window_set(&s_ids, &t_ids, 256) != want {
            set_wrong += 1;
            let shorter = |j: usize| {
                let (fs, ft) = (&cases[j].0, &cases[j].1);
                s.len() < fs.len() || s.len() == fs.len() && t.len() < ft.len()
            };
            if first.is_none_or(shorter) {
                first = Some(i);
            }
        }
    }
    println!(
        "  two pointers vs oracle (shortest, leftmost on ties): {} mismatches of {} ({} had a window, {} had empty t)",
        mismatches,
        cases.len(),
        found0,
        empty_t
    );
    println!(
        "  the set-of-characters bug: wrong on {} of {}",
        set_wrong,
        cases.len()
    );
    let (fs, ft) = &cases[first.expect("the set bug fails somewhere")];
    let (s_ids, t_ids) = (byte_ids(fs), byte_ids(ft));
    let as_found = |w: Option<(usize, usize)>| {
        w.map(|(lo, hi)| Found {
            lo,
            hi,
            blo: lo,
            bhi: hi,
        })
    };
    println!(
        "  shortest counterexample s = {}, t = {}: oracle {}, bug {}",
        quote(fs.as_bytes()),
        quote(ft.as_bytes()),
        show(fs, as_found(oracle(&s_ids, &t_ids, 256))),
        show(fs, as_found(min_window_set(&s_ids, &t_ids, 256)))
    );

    println!();
    println!("(d) edge cases (right moves / left moves)");
    let edge = [
        ("ab", "abc"),
        ("a", "aa"),
        ("abc", ""),
        ("", "a"),
        ("", ""),
        ("abc", "abc"),
        ("aab", "aab"),
        ("xaay", "aa"),
    ];
    for (s, t) in edge {
        let mut st = Stats::default();
        let f = window_bytes(s, t, &mut st);
        println!(
            "  s = {:<6} t = {:<6} -> {:<7}  ({} / {})",
            quote(s.as_bytes()),
            quote(t.as_bytes()),
            show(s, f),
            st.right_moves,
            st.left_moves
        );
    }
    println!(
        "  convention: an empty t has no window; t longer than s, or short of a repeat, has none either"
    );

    println!();
    println!("(e) bytes or runes: this program counts RUNES for text; bytes are only for ASCII");
    let utf8_cases = [
        ("ADOBECODEBANC", "ABC"),
        ("\u{a9}\u{e3}", "\u{e9}"),
        ("x\u{e9}\u{e3}\u{e9}y", "\u{e9}\u{e9}"),
        ("a\u{1F600}b", "b\u{1F600}"),
        ("\u{65e5}\u{672c}\u{8a9e}\u{65e5}", "\u{8a9e}\u{65e5}"),
    ];
    for (s, t) in utf8_cases {
        let mut st = Stats::default();
        let (b, r) = (window_bytes(s, t, &mut st), window_runes(s, t, &mut st));
        println!("  s = {}, t = {}", quote(s.as_bytes()), quote(t.as_bytes()));
        print!("    bytes: {}", show(s, b));
        if let Some(b) = b {
            print!("   valid UTF-8: {}", is_utf8(s, b));
        }
        print!("\n    runes: {}", show(s, r));
        if let Some(r) = r {
            print!("   runes [{},{}) = bytes [{},{})", r.lo, r.hi, r.blo, r.bhi);
        }
        println!();
    }
    // 1, 2, 2, 2 bytes; \u{e9} and \u{e3} share the byte C3, \u{e9} and \u{a9} share A9
    let alphabet = ["a", "\u{e9}", "\u{e3}", "\u{a9}"];
    let (mut runes_wrong, mut bytes_wrong, mut bytes_broken) = (0, 0, 0);
    const TRIALS: usize = 20000;
    let mut first_bad: Option<(String, String)> = None;
    for _ in 0..TRIALS {
        let n = rng.int_n(9) as usize;
        let s = random_string(&mut rng, n, &alphabet);
        let n = rng.int_n(4) as usize;
        let t = random_string(&mut rng, n, &alphabet);
        let want = oracle_runes(&s, &t);
        let mut st = Stats::default();
        if show(&s, window_runes(&s, &t, &mut st)) != show(&s, want) {
            runes_wrong += 1;
        }
        let got = window_bytes(&s, &t, &mut st);
        if show(&s, got) != show(&s, want) {
            bytes_wrong += 1;
            if first_bad.as_ref().is_none_or(|(fs, _)| s.len() < fs.len()) {
                first_bad = Some((s.clone(), t.clone()));
            }
        }
        if got.is_some_and(|g| !is_utf8(&s, g)) {
            bytes_broken += 1;
        }
    }
    println!(
        "  {TRIALS} random cases over a, \u{e9}, \u{e3}, \u{a9} (1-2 bytes each): the runes version differs from the rune oracle on {runes_wrong},"
    );
    println!(
        "  the bytes version on {bytes_wrong}, and on {bytes_broken} of those it returned bytes that are not valid UTF-8"
    );
    let (s, t) = first_bad.expect("the byte version fails somewhere");
    println!(
        "  shortest bytes failure: s = {}, t = {}: oracle {}, bytes {}",
        quote(s.as_bytes()),
        quote(t.as_bytes()),
        show(&s, oracle_runes(&s, &t)),
        show(&s, window_bytes(&s, &t, &mut Stats::default()))
    );
}
```

**Rust output:**

```
(a) s = "ADOBECODEBANC", t = "ABC": every event (+x adds s[right], -x drops s[left])
  r=0  +A  missing 3->2
  r=1  +D  missing 2->2
  r=2  +O  missing 2->2
  r=3  +B  missing 2->1
  r=4  +E  missing 1->1
  r=5  +C  missing 1->0   <- valid, start shrinking
        record [0,6) "ADOBEC" (length 6)
        l=0  -A  missing 0->1   <- invalid, stop shrinking
  r=6  +O  missing 1->1
  r=7  +D  missing 1->1
  r=8  +E  missing 1->1
  r=9  +B  missing 1->1
  r=10 +A  missing 1->0   <- valid, start shrinking
        l=1  -D  missing 0->0
        l=2  -O  missing 0->0
        l=3  -B  missing 0->0
        l=4  -E  missing 0->0
        l=5  -C  missing 0->1   <- invalid, stop shrinking
  r=11 +N  missing 1->1
  r=12 +C  missing 1->0   <- valid, start shrinking
        l=6  -O  missing 0->0
        l=7  -D  missing 0->0
        record [8,13) "EBANC" (length 5)
        l=8  -E  missing 0->0
        record [9,13) "BANC" (length 4)
        l=9  -B  missing 0->1   <- invalid, stop shrinking
  answer "BANC"; 13 right moves, 10 left moves (n = 13)

(b) cost: random s over abcd, t = 4 random letters; moves vs recounting vs all substrings
        n  right   left   r + l |      recount   x moves |   substrings   x moves
      100    100     93     193 |         1545      8.0x |         5050      26.2x  all versions agree: true
     1000   1000    987    1987 |        17764      8.9x |       500500     251.9x  all versions agree: true
    10000  10000   9992   19992 |       236114     11.8x |     50005000*   2501.3x  all versions agree: true
  * not run: n(n+1)/2 computed. Moves never exceed 2n. Recounting costs only a small constant
  factor here because the window stays short; the next table makes it long.

  worst case for recounting: s = "a" x (n-1) + "b", t = "ab" (the window is the whole prefix)
        n  right   left   r + l      2n |      recount   x moves
     1000   1000    999    1999    2000 |      1000000    500.3x  answer "ab"
    10000  10000   9999   19999   20000 |    100000000   5000.3x  answer "ab"

(c) oracle: 20,000 random cases (s: 0-12 chars over abc, t: 0-4 chars over abc)
  two pointers vs oracle (shortest, leftmost on ties): 0 mismatches of 20000 (9010 had a window, 3849 had empty t)
  the set-of-characters bug: wrong on 5634 of 20000
  shortest counterexample s = "a", t = "aa": oracle none, bug "a"

(d) edge cases (right moves / left moves)
  s = "ab"   t = "abc"  -> none     (2 / 0)
  s = "a"    t = "aa"   -> none     (1 / 0)
  s = "abc"  t = ""     -> none     (0 / 0)
  s = ""     t = "a"    -> none     (0 / 0)
  s = ""     t = ""     -> none     (0 / 0)
  s = "abc"  t = "abc"  -> "abc"    (3 / 1)
  s = "aab"  t = "aab"  -> "aab"    (3 / 1)
  s = "xaay" t = "aa"   -> "aa"     (4 / 2)
  convention: an empty t has no window; t longer than s, or short of a repeat, has none either

(e) bytes or runes: this program counts RUNES for text; bytes are only for ASCII
  s = "ADOBECODEBANC", t = "ABC"
    bytes: "BANC"   valid UTF-8: true
    runes: "BANC"   runes [9,13) = bytes [9,13)
  s = "©ã", t = "é"
    bytes: "\xa9\xc3"   valid UTF-8: false
    runes: none
  s = "xéãéy", t = "éé"
    bytes: "\xa9ãé"   valid UTF-8: false
    runes: "éãé"   runes [1,4) = bytes [1,7)
  s = "a😀b", t = "b😀"
    bytes: "😀b"   valid UTF-8: true
    runes: "😀b"   runes [1,3) = bytes [1,6)
  s = "日本語日", t = "語日"
    bytes: "語日"   valid UTF-8: true
    runes: "語日"   runes [2,4) = bytes [6,12)
  20000 random cases over a, é, ã, © (1-2 bytes each): the runes version differs from the rune oracle on 0,
  the bytes version on 1564, and on 1391 of those it returned bytes that are not valid UTF-8
  shortest bytes failure: s = "©é", t = "é": oracle "é", bytes "\xa9\xc3"
```

> **Go vs Rust:** The output is identical to Go's, line for line. Go happily returns `s[l:r]` with a cut rune in it (the `"\xa9\xc3"` rows are valid Go strings that are not valid UTF-8); Rust's `&s[l..r]` on a `&str` would panic at a non-boundary, so the byte version hands back a byte range, slices `s.as_bytes()`, and asks `str::from_utf8` whether it is text. `char_indices` gives the rune-to-byte offsets that Go collects with `for i, r := range s`, the result is an `Option` instead of `(lo, hi, ok)`, and `quote` stands in for `%q` using `utf8_chunks` to print invalid bytes as `\xNN`.

**Complexity:** Θ(n + m) time, Θ(σ) space. Right moves equal n and left moves never exceed n: 10,000 right and 9,992 left moves on a random 10,000-character string, and exactly 2n − 1 moves on `"a" x (n−1) + "b"`, so the 2n bound is tight. Brute force looks at n(n+1)/2 substrings (50,005,000 at n = 10,000, 2,501 times the moves). A version that recounts the window on every check reads 236,114 characters on the random string, only about 12 times the moves because the window stays short, but 100,000,000 on the worst case (n² reads for n = 10,000). The two-pointer version matched an independent oracle on all 20,000 random cases, and the set-of-characters bug was wrong on 5,634 of them, first on `s = "a", t = "aa"`. The program counts runes, and the rune version matched the rune oracle on 20,000 random multi-byte cases while the byte version was wrong on 1,564 of them, returning invalid UTF-8 on 1,391.

---

## 10. The longest window: at most k distinct

`🟡 medium` · *Variable window — longest valid*

The mirror image of example 9. "At most k distinct bytes" is **shrink-closed** (a valid window stays valid when it shrinks), so the answer is the *longest* window and the loop changes shape: grow `right` every step, shrink `left` while the window is **invalid**, record *after* the shrink. The shortest-valid shape of example 9 shrinks while valid and records inside the loop. A second form swaps the inner `for` for an `if`, so the window never narrows and the answer is its final width. Both are Θ(n) with at most 2n pointer moves, and the program measures them against a brute-force oracle and against each other.

**Steps:**

1. Trace `"eceba"` with k = 2 through both loop shapes, then check both against brute force on 20,000 random strings, next to a version that records before shrinking.
2. Count pointer moves on 100,000 random bytes over alphabets of 3, 10 and 100: right moves, left moves for each shape, and the totals against 2n.
3. Compare a maintained `distinct` counter with recounting the table at every loop test, and read the `if` form's carrier windows in the trace (invalid, yet exactly as wide as the best window so far).

```go
package main

import (
	"fmt"
	"math/rand/v2"
)

// "Longest substring with at most k distinct bytes" is the mirror image of the
// minimum-window problem. The predicate is SHRINK-CLOSED: if a window has at
// most k distinct bytes, every sub-window does too. So the right shape is
//
//	grow right every step; shrink left while the window is INVALID; record AFTER.
//
// (The shortest-valid shape shrinks while VALID and records INSIDE the loop.)

// win is the window s[left:right+1] plus the bookkeeping that makes every
// step O(1): a count per byte, and `distinct`, kept up to date as a counter.
type win struct {
	s        string
	cnt      [256]int
	distinct int
	left     int
}

func (w *win) add(c byte) {
	if w.cnt[c] == 0 {
		w.distinct++ // 0 -> 1: a new distinct byte entered
	}
	w.cnt[c]++
}

func (w *win) drop() {
	d := w.s[w.left]
	w.cnt[d]--
	if w.cnt[d] == 0 {
		w.distinct-- // 1 -> 0: a distinct byte left
	}
	w.left++
}

// Moves counts pointer advances, the unit of work for a window.
type Moves struct {
	Right, Left int // times right / left advanced
	Invalid     int // iterations that END with an invalid window
}

// longestWhile is the textbook form: an inner `for` shrinks until valid again.
func longestWhile(s string, k int, m *Moves) int {
	w, best := win{s: s}, 0
	for right := 0; right < len(s); right++ {
		w.add(s[right])
		m.Right++
		for w.distinct > k {
			w.drop()
			m.Left++
		}
		best = max(best, right+1-w.left) // record AFTER shrinking
	}
	return best
}

// longestIf replaces the inner `for` by an `if`: left moves at most once per
// step, so the window never narrows. It may be INVALID, but its width is
// still the best width seen, so the answer is the final width.
func longestIf(s string, k int, m *Moves) int {
	w := win{s: s}
	for right := 0; right < len(s); right++ {
		w.add(s[right])
		m.Right++
		if w.distinct > k {
			w.drop()
			m.Left++
		}
		if w.distinct > k {
			m.Invalid++
		}
	}
	return len(s) - w.left // final width = right+1-left with right = len(s)-1
}

// longestBug records BEFORE shrinking: it scores a window that is invalid.
func longestBug(s string, k int, m *Moves) int {
	w, best := win{s: s}, 0
	for right := 0; right < len(s); right++ {
		w.add(s[right])
		m.Right++
		best = max(best, right+1-w.left) // BUG: this window may have k+1 distinct bytes
		for w.distinct > k {
			w.drop()
			m.Left++
		}
	}
	return best
}

// longestBrute is the oracle: every substring, distinct counted from scratch.
func longestBrute(s string, k int) int {
	best := 0
	for i := 0; i < len(s); i++ {
		var seen [256]bool
		distinct := 0
		for j := i; j < len(s); j++ {
			if !seen[s[j]] {
				seen[s[j]] = true
				distinct++
			}
			if distinct <= k {
				best = max(best, j-i+1)
			}
		}
	}
	return best
}

// recountWhile is longestWhile with `distinct` recomputed from the first sigma
// count cells at every loop test instead of being maintained. It returns the
// answer and how many table cells it read.
func recountWhile(s string, k, sigma int) (best, reads int) {
	w := win{s: s}
	recount := func() int {
		d := 0
		for _, v := range w.cnt[:sigma] {
			reads++
			if v > 0 {
				d++
			}
		}
		return d
	}
	for right := 0; right < len(s); right++ {
		w.add(s[right])
		for recount() > k {
			w.drop()
		}
		best = max(best, right+1-w.left)
	}
	return best, reads
}

func randString(rng *rand.Rand, n, sigma int, base byte) string {
	b := make([]byte, n)
	for i := range b {
		b[i] = base + byte(rng.IntN(sigma))
	}
	return string(b)
}

// trace replays both loop shapes on one input, printing the state after each step.
func trace(s string, k int) {
	fmt.Printf("%5s %2s | %-7s %8s %10s %5s | %-7s %8s %5s\n",
		"right", "ch", "window", "distinct", "left moves", "best", "window", "distinct", "valid")
	wi, wf, best := win{s: s}, win{s: s}, 0
	for right := 0; right < len(s); right++ {
		wi.add(s[right])
		moved := 0
		for wi.distinct > k {
			wi.drop()
			moved++
		}
		best = max(best, right+1-wi.left)

		wf.add(s[right])
		if wf.distinct > k {
			wf.drop()
		}
		fmt.Printf("%5d %2c | %-7s %8d %10d %5d | %-7s %8d %5v\n", right, s[right],
			s[wi.left:right+1], wi.distinct, moved, best, s[wf.left:right+1], wf.distinct, wf.distinct <= k)
	}
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))
	var m Moves

	fmt.Println("(a) \"eceba\", k = 2: the while form (valid window) beside the if form (carrier window)")
	trace("eceba", 2)
	fmt.Printf("answers: while form %d, if form %d, brute force %d\n\n",
		longestWhile("eceba", 2, &m), longestIf("eceba", 2, &m), longestBrute("eceba", 2))

	fmt.Println("(b) 20,000 random strings (length 0-14 over abcd, k 0-5) against brute force")
	var bad [3]int
	first := ""
	for t := 0; t < 20000; t++ {
		s := randString(rng, rng.IntN(15), 4, 'a')
		k := rng.IntN(6)
		want := longestBrute(s, k)
		got := [3]int{longestWhile(s, k, &m), longestIf(s, k, &m), longestBug(s, k, &m)}
		for v := range got {
			if got[v] != want {
				bad[v]++
				if v == 2 && first == "" {
					first = fmt.Sprintf("%q k=%d: oracle %d, bug %d", s, k, want, got[v])
				}
			}
		}
	}
	fmt.Printf("  while form (shrink while invalid, record after): %d mismatches\n", bad[0])
	fmt.Printf("  if form (window never narrows):                  %d mismatches\n", bad[1])
	fmt.Printf("  bug (record BEFORE shrinking):                   %d mismatches, first: %s\n", bad[2], first)
	fmt.Printf("  edge cases: k=0 -> %d %d (want 0); k=5 >= sigma=4 on \"eceba\" -> %d %d (want 5); \"\" -> %d %d (want 0)\n\n",
		longestWhile("eceba", 0, &m), longestIf("eceba", 0, &m),
		longestWhile("eceba", 5, &m), longestIf("eceba", 5, &m),
		longestWhile("", 3, &m), longestIf("", 3, &m))

	const n, k = 100_000, 3
	fmt.Printf("(c) pointer moves, n = %d, k = %d\n", n, k)
	fmt.Printf("%6s %7s %10s %12s %10s %12s %10s %12s\n",
		"sigma", "best", "right", "left(while)", "left(if)", "total(while)", "total(if)", "if invalid")
	for _, sigma := range []int{3, 10, 100} {
		s := randString(rng, n, sigma, 0)
		var mw, mi Moves
		bw, bi := longestWhile(s, k, &mw), longestIf(s, k, &mi)
		fmt.Printf("%6d %7d %10d %12d %10d %12d %10d %12d\n",
			sigma, bw, mw.Right, mw.Left, mi.Left, mw.Right+mw.Left, mi.Right+mi.Left, mi.Invalid)
		fmt.Printf("       forms agree: %v; both totals <= 2n: %v; left(if) = n - best: %v; if form moved left %d fewer times\n",
			bw == bi, mw.Right+mw.Left <= 2*n && mi.Right+mi.Left <= 2*n, mi.Left == n-bi, mw.Left-mi.Left)
	}
	fmt.Println()

	fmt.Println("(d) `distinct` kept as a counter vs recounted from the table at every loop test, n = 10,000, k = 3")
	fmt.Printf("%6s %8s %10s %14s %14s\n", "sigma", "answer", "loop tests", "reads(counter)", "reads(recount)")
	for _, sigma := range []int{10, 100} {
		s := randString(rng, 10_000, sigma, 0)
		var mw Moves
		a := longestWhile(s, 3, &mw)
		b, reads := recountWhile(s, 3, sigma)
		tests := mw.Right + mw.Left // one test after each add and one after each removal
		fmt.Printf("%6d %8d %10d %14d %14d   same answer: %v, recount reads = tests * sigma: %v\n",
			sigma, a, tests, 0, reads, a == b, reads == tests*sigma)
	}
}
```

**Output:**

```
(a) "eceba", k = 2: the while form (valid window) beside the if form (carrier window)
right ch | window  distinct left moves  best | window  distinct valid
    0  e | e              1          0     1 | e              1  true
    1  c | ec             2          0     2 | ec             2  true
    2  e | ece            2          0     3 | ece            2  true
    3  b | eb             2          2     3 | ceb            3 false
    4  a | ba             2          1     3 | eba            3 false
answers: while form 3, if form 3, brute force 3

(b) 20,000 random strings (length 0-14 over abcd, k 0-5) against brute force
  while form (shrink while invalid, record after): 0 mismatches
  if form (window never narrows):                  0 mismatches
  bug (record BEFORE shrinking):                   8545 mismatches, first: "cdccbcadbaabaa" k=0: oracle 0, bug 1
  edge cases: k=0 -> 0 0 (want 0); k=5 >= sigma=4 on "eceba" -> 5 5 (want 5); "" -> 0 0 (want 0)

(c) pointer moves, n = 100000, k = 3
 sigma    best      right  left(while)   left(if) total(while)  total(if)   if invalid
     3  100000     100000            0          0       100000     100000            0
       forms agree: true; both totals <= 2n: true; left(if) = n - best: true; if form moved left 0 fewer times
    10      15     100000        99996      99985       199996     199985        99976
       forms agree: true; both totals <= 2n: true; left(if) = n - best: true; if form moved left 11 fewer times
   100       6     100000        99997      99994       199997     199994        99966
       forms agree: true; both totals <= 2n: true; left(if) = n - best: true; if form moved left 3 fewer times

(d) `distinct` kept as a counter vs recounted from the table at every loop test, n = 10,000, k = 3
 sigma   answer loop tests reads(counter) reads(recount)
    10       11      19997              0         199970   same answer: true, recount reads = tests * sigma: true
   100        5      19997              0        1999700   same answer: true, recount reads = tests * sigma: true
```

**Rust:**

```rust
// "Longest substring with at most k distinct bytes" is the mirror image of the
// minimum-window problem. The predicate is SHRINK-CLOSED: if a window has at
// most k distinct bytes, every sub-window does too. So the right shape is
//
//     grow right every step; shrink left while the window is INVALID; record AFTER.
//
// (The shortest-valid shape shrinks while VALID and records INSIDE the loop.)

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so seed (1, 2) feeds this program exactly the strings Go sees and
// every count below comes out identical.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        // state = state * MUL + INC, with explicit 128-bit wraparound
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;

        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

// Win is the window s[left..=right] plus the bookkeeping that makes every
// step O(1): a count per byte, and `distinct`, kept up to date as a counter.
// The window borrows the input (`&'a [u8]`); Go's version holds a string header.
struct Win<'a> {
    s: &'a [u8],
    cnt: [usize; 256],
    distinct: usize,
    left: usize,
}

impl<'a> Win<'a> {
    fn new(s: &'a [u8]) -> Self {
        Win {
            s,
            cnt: [0; 256],
            distinct: 0,
            left: 0,
        }
    }

    fn add(&mut self, c: u8) {
        let slot = &mut self.cnt[usize::from(c)];
        if *slot == 0 {
            self.distinct += 1; // 0 -> 1: a new distinct byte entered
        }
        *slot += 1;
    }

    fn drop_left(&mut self) {
        let slot = &mut self.cnt[usize::from(self.s[self.left])];
        *slot -= 1;
        if *slot == 0 {
            self.distinct -= 1; // 1 -> 0: a distinct byte left
        }
        self.left += 1;
    }
}

/// Moves counts pointer advances, the unit of work for a window.
#[derive(Default)]
struct Moves {
    right: usize,   // times right advanced
    left: usize,    // times left advanced
    invalid: usize, // iterations that END with an invalid window
}

// longest_while is the textbook form: an inner `while` shrinks until valid again.
// `right + 1 - w.left` cannot underflow: the shrink loop stops at the latest
// when the window is empty (distinct = 0 <= k), i.e. left = right + 1.
fn longest_while(s: &[u8], k: usize, m: &mut Moves) -> usize {
    let mut w = Win::new(s);
    let mut best = 0;
    for (right, &c) in s.iter().enumerate() {
        w.add(c);
        m.right += 1;
        while w.distinct > k {
            w.drop_left();
            m.left += 1;
        }
        best = best.max(right + 1 - w.left); // record AFTER shrinking
    }
    best
}

// longest_if replaces the inner `while` by an `if`: left moves at most once per
// step, so the window never narrows. It may be INVALID, but its width is
// still the best width seen, so the answer is the final width.
fn longest_if(s: &[u8], k: usize, m: &mut Moves) -> usize {
    let mut w = Win::new(s);
    for &c in s {
        w.add(c);
        m.right += 1;
        if w.distinct > k {
            w.drop_left();
            m.left += 1;
        }
        if w.distinct > k {
            m.invalid += 1;
        }
    }
    s.len() - w.left // final width = right+1-left with right = len-1
}

// longest_bug records BEFORE shrinking: it scores a window that is invalid.
fn longest_bug(s: &[u8], k: usize, m: &mut Moves) -> usize {
    let mut w = Win::new(s);
    let mut best = 0;
    for (right, &c) in s.iter().enumerate() {
        w.add(c);
        m.right += 1;
        best = best.max(right + 1 - w.left); // BUG: this window may have k+1 distinct bytes
        while w.distinct > k {
            w.drop_left();
            m.left += 1;
        }
    }
    best
}

// longest_brute is the oracle: every substring, distinct counted from scratch.
fn longest_brute(s: &[u8], k: usize) -> usize {
    let mut best = 0;
    for i in 0..s.len() {
        let mut seen = [false; 256];
        let mut distinct = 0;
        for (j, &c) in s.iter().enumerate().skip(i) {
            if !seen[usize::from(c)] {
                seen[usize::from(c)] = true;
                distinct += 1;
            }
            if distinct <= k {
                best = best.max(j - i + 1);
            }
        }
    }
    best
}

// recount reads every cell it is given and counts the non-zero ones. Go used a
// closure over the window and `reads`; here both are plain arguments, so no
// borrow of the window is kept alive across the `w.drop_left()` calls.
fn recount(cells: &[usize], reads: &mut usize) -> usize {
    *reads += cells.len();
    cells.iter().filter(|&&v| v > 0).count()
}

// recount_while is longest_while with `distinct` recomputed from the first
// sigma count cells at every loop test instead of being maintained. It returns
// the answer and how many table cells it read.
fn recount_while(s: &[u8], k: usize, sigma: usize) -> (usize, usize) {
    let mut w = Win::new(s);
    let (mut best, mut reads) = (0, 0);
    for (right, &c) in s.iter().enumerate() {
        w.add(c);
        while recount(&w.cnt[..sigma], &mut reads) > k {
            w.drop_left();
        }
        best = best.max(right + 1 - w.left);
    }
    (best, reads)
}

fn rand_string(rng: &mut Pcg, n: usize, sigma: u64, base: u8) -> Vec<u8> {
    (0..n).map(|_| base + rng.int_n(sigma) as u8).collect()
}

// trace replays both loop shapes on one input, printing the state after each step.
fn trace(s: &str, k: usize) {
    println!(
        "{:>5} {:>2} | {:<7} {:>8} {:>10} {:>5} | {:<7} {:>8} {:>5}",
        "right", "ch", "window", "distinct", "left moves", "best", "window", "distinct", "valid"
    );
    let bytes = s.as_bytes();
    let (mut wi, mut wf, mut best) = (Win::new(bytes), Win::new(bytes), 0);
    for (right, &c) in bytes.iter().enumerate() {
        wi.add(c);
        let mut moved = 0;
        while wi.distinct > k {
            wi.drop_left();
            moved += 1;
        }
        best = best.max(right + 1 - wi.left);

        wf.add(c);
        if wf.distinct > k {
            wf.drop_left();
        }
        println!(
            "{:>5} {:>2} | {:<7} {:>8} {:>10} {:>5} | {:<7} {:>8} {:>5}",
            right,
            char::from(c),
            &s[wi.left..=right],
            wi.distinct,
            moved,
            best,
            &s[wf.left..=right],
            wf.distinct,
            wf.distinct <= k
        );
    }
}

fn main() {
    let mut rng = Pcg::new(1, 2);
    let mut m = Moves::default();

    println!(
        "(a) \"eceba\", k = 2: the while form (valid window) beside the if form (carrier window)"
    );
    trace("eceba", 2);
    println!(
        "answers: while form {}, if form {}, brute force {}\n",
        longest_while(b"eceba", 2, &mut m),
        longest_if(b"eceba", 2, &mut m),
        longest_brute(b"eceba", 2)
    );

    println!("(b) 20,000 random strings (length 0-14 over abcd, k 0-5) against brute force");
    let mut bad = [0usize; 3];
    let mut first = String::new();
    for _ in 0..20000 {
        let len = rng.int_n(15) as usize;
        let s = rand_string(&mut rng, len, 4, b'a');
        let k = rng.int_n(6) as usize;
        let want = longest_brute(&s, k);
        let got = [
            longest_while(&s, k, &mut m),
            longest_if(&s, k, &mut m),
            longest_bug(&s, k, &mut m),
        ];
        for (v, &g) in got.iter().enumerate() {
            if g != want {
                bad[v] += 1;
                if v == 2 && first.is_empty() {
                    first = format!(
                        "{:?} k={k}: oracle {want}, bug {g}",
                        String::from_utf8_lossy(&s)
                    );
                }
            }
        }
    }
    println!(
        "  while form (shrink while invalid, record after): {} mismatches",
        bad[0]
    );
    println!(
        "  if form (window never narrows):                  {} mismatches",
        bad[1]
    );
    println!(
        "  bug (record BEFORE shrinking):                   {} mismatches, first: {first}",
        bad[2]
    );
    println!(
        "  edge cases: k=0 -> {} {} (want 0); k=5 >= sigma=4 on \"eceba\" -> {} {} (want 5); \"\" -> {} {} (want 0)\n",
        longest_while(b"eceba", 0, &mut m),
        longest_if(b"eceba", 0, &mut m),
        longest_while(b"eceba", 5, &mut m),
        longest_if(b"eceba", 5, &mut m),
        longest_while(b"", 3, &mut m),
        longest_if(b"", 3, &mut m)
    );

    const N: usize = 100_000;
    const K: usize = 3;
    println!("(c) pointer moves, n = {N}, k = {K}");
    println!(
        "{:>6} {:>7} {:>10} {:>12} {:>10} {:>12} {:>10} {:>12}",
        "sigma",
        "best",
        "right",
        "left(while)",
        "left(if)",
        "total(while)",
        "total(if)",
        "if invalid"
    );
    for sigma in [3, 10, 100] {
        let s = rand_string(&mut rng, N, sigma, 0);
        let (mut mw, mut mi) = (Moves::default(), Moves::default());
        let (bw, bi) = (longest_while(&s, K, &mut mw), longest_if(&s, K, &mut mi));
        println!(
            "{:>6} {:>7} {:>10} {:>12} {:>10} {:>12} {:>10} {:>12}",
            sigma,
            bw,
            mw.right,
            mw.left,
            mi.left,
            mw.right + mw.left,
            mi.right + mi.left,
            mi.invalid
        );
        // left(if) = n - best <= n - (final while width) = left(while), so this
        // usize subtraction cannot underflow when the identity on this line holds.
        println!(
            "       forms agree: {}; both totals <= 2n: {}; left(if) = n - best: {}; if form moved left {} fewer times",
            bw == bi,
            mw.right + mw.left <= 2 * N && mi.right + mi.left <= 2 * N,
            mi.left == N - bi,
            mw.left - mi.left
        );
    }
    println!();

    println!(
        "(d) `distinct` kept as a counter vs recounted from the table at every loop test, n = 10,000, k = 3"
    );
    println!(
        "{:>6} {:>8} {:>10} {:>14} {:>14}",
        "sigma", "answer", "loop tests", "reads(counter)", "reads(recount)"
    );
    for sigma in [10usize, 100] {
        let s = rand_string(&mut rng, 10_000, sigma as u64, 0);
        let mut mw = Moves::default();
        let a = longest_while(&s, 3, &mut mw);
        let (b, reads) = recount_while(&s, 3, sigma);
        let tests = mw.right + mw.left; // one test after each add and one after each removal
        println!(
            "{:>6} {:>8} {:>10} {:>14} {:>14}   same answer: {}, recount reads = tests * sigma: {}",
            sigma,
            a,
            tests,
            0,
            reads,
            a == b,
            reads == tests * sigma
        );
    }
}
```

**Rust output:**

```
(a) "eceba", k = 2: the while form (valid window) beside the if form (carrier window)
right ch | window  distinct left moves  best | window  distinct valid
    0  e | e              1          0     1 | e              1  true
    1  c | ec             2          0     2 | ec             2  true
    2  e | ece            2          0     3 | ece            2  true
    3  b | eb             2          2     3 | ceb            3 false
    4  a | ba             2          1     3 | eba            3 false
answers: while form 3, if form 3, brute force 3

(b) 20,000 random strings (length 0-14 over abcd, k 0-5) against brute force
  while form (shrink while invalid, record after): 0 mismatches
  if form (window never narrows):                  0 mismatches
  bug (record BEFORE shrinking):                   8545 mismatches, first: "cdccbcadbaabaa" k=0: oracle 0, bug 1
  edge cases: k=0 -> 0 0 (want 0); k=5 >= sigma=4 on "eceba" -> 5 5 (want 5); "" -> 0 0 (want 0)

(c) pointer moves, n = 100000, k = 3
 sigma    best      right  left(while)   left(if) total(while)  total(if)   if invalid
     3  100000     100000            0          0       100000     100000            0
       forms agree: true; both totals <= 2n: true; left(if) = n - best: true; if form moved left 0 fewer times
    10      15     100000        99996      99985       199996     199985        99976
       forms agree: true; both totals <= 2n: true; left(if) = n - best: true; if form moved left 11 fewer times
   100       6     100000        99997      99994       199997     199994        99966
       forms agree: true; both totals <= 2n: true; left(if) = n - best: true; if form moved left 3 fewer times

(d) `distinct` kept as a counter vs recounted from the table at every loop test, n = 10,000, k = 3
 sigma   answer loop tests reads(counter) reads(recount)
    10       11      19997              0         199970   same answer: true, recount reads = tests * sigma: true
   100        5      19997              0        1999700   same answer: true, recount reads = tests * sigma: true
```

> **Go vs Rust:** Identical output, byte for byte (also diffed against a live run of the Go program): the same PCG-DXSM port as in lesson 01 example 14 gives Rust the strings Go sees, so every count is the algorithm's own. `right + 1 - w.left` is a `usize` subtraction and cannot underflow because the shrink loop stops at the latest on an empty window (`left == right + 1`); if that invariant broke, Go's `int` would go negative and `max` would quietly ignore it, while Rust would panic in a debug build and wrap to a huge width under `-O`. `Win` borrows the input as `&'a [u8]` instead of holding a string, and `[usize; 256]` is indexed by `usize::from(byte)`.

**Complexity:** Θ(n) time and Θ(1) extra space for both shapes here (a fixed 256-cell count table; a map of counts would be Θ(min(σ, k+1))), with at most 2n pointer moves: the run shows 199,996 and 199,985 total moves for n = 100,000 and 10 letters, and the `if` form's left count is exactly n − best, so it moves `left` as many times or fewer (11 fewer here, 3 fewer with 100 letters), not asymptotically fewer. Recording before the shrink fails on 8,545 of 20,000 random inputs (about 43%) while both correct shapes make 0 mismatches; a maintained `distinct` counter costs no table reads where recounting costs σ reads per loop test.

---

## 11. 2-D prefix sums and inclusion-exclusion

`🟡 medium` · *2-D prefix sums*

A table `P` with a zero border holds the sum of every top-left block, so any rectangle is four reads: add the big block, subtract the two strips that stick out, add back the corner that was subtracted twice (inclusion-exclusion). The build is Θ(mn), a query is Θ(1) instead of Θ(area), and the table costs Θ(mn) space. The program builds a tiny table by hand, breaks the formula three ways against a brute-force oracle on every rectangle of a 6×6 grid, counts reads on a million-cell grid, and checks what the left-to-right order of the four terms does when the arithmetic is unsigned.

**Steps:**

1. Build `P` for a 4×5 grid, print it, and spell out one build step and one query with the four terms as numbers.
2. Run the correct formula and three broken ones (no corner term, no zero border, inclusive end where exclusive belongs) over all 441 non-empty rectangles of a 6×6 grid.
3. Count cells read by brute force against the prefix table on a 1,000×1,000 grid, then replay a million queries to see where the left-to-right chain goes below zero and why the paired order never does.

```go
package main

import (
	"fmt"
	"math/bits"
	"math/rand/v2"
)

// 2-D prefix sums. P has one more row and column than the grid, zero on the top
// and left edge: P[i][j] is the sum of a[0:i][0:j] (rows < i, cols < j).
//
// Build: P[i+1][j+1] = a[i][j] + P[i][j+1] + P[i+1][j] - P[i][j]
// (the strip above and the strip to the left both contain the block P[i][j]:
// it was added twice, so subtract it once).
//
// Query, half-open rows [r1,r2) x cols [c1,c2):
// P[r2][c2] - P[r1][c2] - P[r2][c1] + P[r1][c1]
// (the two strips cut away share the block P[r1][c1]: it was subtracted twice,
// so add it back once).
//
// Values are int64: every intermediate step is a legal, possibly negative number.

func build(a [][]int64) [][]int64 {
	m, n := len(a), len(a[0])
	P := make([][]int64, m+1)
	for i := range P {
		P[i] = make([]int64, n+1)
	}
	for i := 0; i < m; i++ {
		for j := 0; j < n; j++ {
			P[i+1][j+1] = a[i][j] + P[i][j+1] + P[i+1][j] - P[i][j]
		}
	}
	return P
}

func rect(P [][]int64, r1, c1, r2, c2 int) int64 {
	return P[r2][c2] - P[r1][c2] - P[r2][c1] + P[r1][c1]
}

// brute is the oracle: add up every cell of the rectangle, counting the reads.
func brute(a [][]int64, r1, c1, r2, c2 int, reads *int) int64 {
	var sum int64
	for i := r1; i < r2; i++ {
		for j := c1; j < c2; j++ {
			sum += a[i][j]
		}
	}
	*reads += (r2 - r1) * (c2 - c1)
	return sum
}

// Bug 1: forget to add the corner back.
func rectNoCorner(P [][]int64, r1, c1, r2, c2 int) int64 {
	return P[r2][c2] - P[r1][c2] - P[r2][c1]
}

// Bug 2: a table with no zero border (same shape as the grid, t[i][j] = P[i+1][j+1])
// queried with the bordered formula. at reports reads that fall outside the table.
func at(t [][]int64, i, j int) (int64, bool) {
	if i < len(t) && j < len(t[i]) {
		return t[i][j], true
	}
	return 0, false
}

func rectNoBorder(t [][]int64, r1, c1, r2, c2 int) (int64, bool) {
	a, ok1 := at(t, r2, c2)
	b, ok2 := at(t, r1, c2)
	c, ok3 := at(t, r2, c1)
	d, ok4 := at(t, r1, c1)
	return a - b - c + d, ok1 && ok2 && ok3 && ok4
}

// span draws a non-empty half-open range [lo, hi) of [0, size], uniform over pairs lo < hi.
func span(rng *rand.Rand, size int) (int, int) {
	a, b := rng.IntN(size+1), rng.IntN(size)
	if b >= a {
		b++
	}
	return min(a, b), max(a, b)
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))
	var sink int

	fmt.Println("(a) a 4x5 grid, its 5x6 table with a zero border, one worked query")
	a := [][]int64{{3, 1, 4, 1, 5}, {9, 2, 6, 5, 3}, {5, 8, 9, 7, 9}, {3, 2, 3, 8, 4}}
	P := build(a)
	for _, row := range P {
		for _, v := range row {
			fmt.Printf("%5d", v)
		}
		fmt.Println()
	}
	fmt.Printf("build P[3][4] = a[2][3] + P[2][4] + P[3][3] - P[2][3] = %d + %d + %d - %d = %d\n",
		a[2][3], P[2][4], P[3][3], P[2][3], P[3][4])
	r1, c1, r2, c2 := 1, 1, 3, 4
	fmt.Printf("query rows [%d,%d) x cols [%d,%d): P[%d][%d] - P[%d][%d] - P[%d][%d] + P[%d][%d] = %d - %d - %d + %d = %d\n",
		r1, r2, c1, c2, r2, c2, r1, c2, r2, c1, r1, c1,
		P[r2][c2], P[r1][c2], P[r2][c1], P[r1][c1], rect(P, r1, c1, r2, c2))
	fmt.Printf("brute force over the %d cells: %d\n\n", (r2-r1)*(c2-c1), brute(a, r1, c1, r2, c2, &sink))

	fmt.Println("(b) every non-empty rectangle of a 6x6 grid (entries 1-9) against brute force, three bugs")
	g := make([][]int64, 6)
	for i := range g {
		g[i] = make([]int64, 6)
		for j := range g[i] {
			g[i][j] = int64(1 + rng.IntN(9))
		}
	}
	G := build(g)
	cut := make([][]int64, 6) // drop the zero row and column: the "no border" table
	for i := range cut {
		cut[i] = G[i+1][1:]
	}
	var total, badOK, bad1, hasCorner, bad2, outOfRange, right2 int
	for r1 := 0; r1 < 6; r1++ {
		for r2 := r1 + 1; r2 <= 6; r2++ {
			for c1 := 0; c1 < 6; c1++ {
				for c2 := c1 + 1; c2 <= 6; c2++ {
					want := brute(g, r1, c1, r2, c2, &sink)
					total++
					if rect(G, r1, c1, r2, c2) != want {
						badOK++
					}
					if rectNoCorner(G, r1, c1, r2, c2) != want {
						bad1++
					}
					if r1 > 0 && c1 > 0 {
						hasCorner++
					}
					switch v, ok := rectNoBorder(cut, r1, c1, r2, c2); {
					case !ok:
						outOfRange++
					case v != want:
						bad2++
					default:
						right2++
					}
				}
			}
		}
	}
	// Bug 3: the caller means INCLUSIVE ends, the table wants EXCLUSIVE ones.
	var incl, bad3, smaller int
	for r1 := 0; r1 < 6; r1++ {
		for r2 := r1; r2 < 6; r2++ {
			for c1 := 0; c1 < 6; c1++ {
				for c2 := c1; c2 < 6; c2++ {
					want := brute(g, r1, c1, r2+1, c2+1, &sink)
					got := rect(G, r1, c1, r2, c2) // BUG: r2, c2 passed as inclusive
					incl++
					if got != want {
						bad3++
					}
					if got < want {
						smaller++
					}
				}
			}
		}
	}
	fmt.Printf("  correct formula:            %d wrong of %d\n", badOK, total)
	fmt.Printf("  bug 1, no +P[r1][c1]:       %d wrong of %d; rectangles with r1 > 0 and c1 > 0: %d\n", bad1, total, hasCorner)
	fmt.Printf("  bug 2, no zero border:      %d out of range, %d wrong, %d right by luck, of %d\n", outOfRange, bad2, right2, total)
	fmt.Printf("  bug 3, inclusive/exclusive: %d wrong of %d; too small in %d of them\n\n", bad3, incl, smaller)

	const m, n, q = 1000, 1000, 10_000
	fmt.Printf("(c) a %dx%d grid of digits, %d random rectangles: cells read\n", m, n, q)
	big := make([][]int64, m)
	for i := range big {
		big[i] = make([]int64, n)
		for j := range big[i] {
			big[i][j] = int64(rng.IntN(10))
		}
	}
	BP := build(big)
	buildReads := 4 * m * n // per cell: a[i][j] and three table cells
	var bruteReads, mismatch int
	for t := 0; t < q; t++ {
		r1, r2 := span(rng, m)
		c1, c2 := span(rng, n)
		if brute(big, r1, c1, r2, c2, &bruteReads) != rect(BP, r1, c1, r2, c2) {
			mismatch++
		}
	}
	queryReads := 4 * q
	avg := float64(bruteReads) / q
	fmt.Printf("  brute force: %d reads (%.0f per query); prefix sums: %d to build + %d for the queries = %d\n",
		bruteReads, avg, buildReads, queryReads, buildReads+queryReads)
	fmt.Printf("  ratio %.1fx fewer reads; mismatches %d; break-even at about %.0f queries\n\n",
		float64(bruteReads)/float64(buildReads+queryReads), mismatch, float64(buildReads)/(avg-4))

	const qd = 1_000_000
	fmt.Printf("(d) the left-to-right order on the same table, %d random rectangles\n", qd)
	var step2, step3, anyNeg, wrapBad, borrowed, pairNeg, pairBad int
	for t := 0; t < qd; t++ {
		r1, r2 := span(rng, m)
		c1, c2 := span(rng, n)
		want := rect(BP, r1, c1, r2, c2)
		s2 := BP[r2][c2] - BP[r1][c2] // signed: look at every middle step
		s3 := s2 - BP[r2][c1]
		if s2 < 0 {
			step2++
		}
		if s3 < 0 {
			step3++
		}
		if s2 < 0 || s3 < 0 {
			anyNeg++
		}
		p22, p12, p21, p11 := uint64(BP[r2][c2]), uint64(BP[r1][c2]), uint64(BP[r2][c1]), uint64(BP[r1][c1])
		if p22-p12-p21+p11 != uint64(want) { // unsigned: wraps below zero, then wraps back
			wrapBad++
		}
		d1, b1 := bits.Sub64(p22, p12, 0) // Go has no checked_sub; Sub64 reports the borrow
		_, b2 := bits.Sub64(d1, p21, 0)
		if b1|b2 != 0 {
			borrowed++
		}
		x, y := p22+p11, p12+p21 // paired order: diagonal pair minus the other pair
		if x < y {
			pairNeg++
		}
		if x-y != uint64(want) {
			pairBad++
		}
	}
	fmt.Printf("  signed left-to-right: step 2 negative in %d, step 3 negative in %d; some middle step negative in %d (%.1f%%)\n",
		step2, step3, anyNeg, 100*float64(anyNeg)/qd)
	fmt.Printf("  uint64 left-to-right: %d wrong answers (it wraps below zero and back); a borrow in the chain in %d (same count: %v)\n",
		wrapBad, borrowed, borrowed == anyNeg)
	fmt.Printf("  paired order (P22 + P11) - (P12 + P21): goes negative in %d, wrong answers %d\n", pairNeg, pairBad)
}
```

**Output:**

```
(a) a 4x5 grid, its 5x6 table with a zero border, one worked query
    0    0    0    0    0    0
    0    3    4    8    9   14
    0   12   15   25   31   39
    0   17   28   47   60   77
    0   20   33   55   76   97
build P[3][4] = a[2][3] + P[2][4] + P[3][3] - P[2][3] = 7 + 31 + 47 - 25 = 60
query rows [1,3) x cols [1,4): P[3][4] - P[1][4] - P[3][1] + P[1][1] = 60 - 9 - 17 + 3 = 37
brute force over the 6 cells: 37

(b) every non-empty rectangle of a 6x6 grid (entries 1-9) against brute force, three bugs
  correct formula:            0 wrong of 441
  bug 1, no +P[r1][c1]:       225 wrong of 441; rectangles with r1 > 0 and c1 > 0: 225
  bug 2, no zero border:      216 out of range, 214 wrong, 11 right by luck, of 441
  bug 3, inclusive/exclusive: 441 wrong of 441; too small in 441 of them

(c) a 1000x1000 grid of digits, 10000 random rectangles: cells read
  brute force: 1106882007 reads (110688 per query); prefix sums: 4000000 to build + 40000 for the queries = 4040000
  ratio 274.0x fewer reads; mismatches 0; break-even at about 36 queries

(d) the left-to-right order on the same table, 1000000 random rectangles
  signed left-to-right: step 2 negative in 0, step 3 negative in 498042; some middle step negative in 498042 (49.8%)
  uint64 left-to-right: 0 wrong answers (it wraps below zero and back); a borrow in the chain in 498042 (same count: true)
  paired order (P22 + P11) - (P12 + P21): goes negative in 0, wrong answers 0
```

**Rust:**

```rust
// 2-D prefix sums. P has one more row and column than the grid, zero on the top
// and left edge: P[i][j] is the sum of a[0..i][0..j] (rows < i, cols < j).
//
// Build: P[i+1][j+1] = a[i][j] + P[i][j+1] + P[i+1][j] - P[i][j]
// (the strip above and the strip to the left both contain the block P[i][j]:
// it was added twice, so subtract it once).
//
// Query, half-open rows [r1,r2) x cols [c1,c2):
// P[r2][c2] - P[r1][c2] - P[r2][c1] + P[r1][c1]
// (the two strips cut away share the block P[r1][c1]: it was subtracted twice,
// so add it back once).
//
// Values are i64: every intermediate step is a legal, possibly negative number.
// Indices are usize, and no index is ever decremented: half-open bounds plus the
// zero border mean the formula only ever reads P[r1], P[r2], P[..][c1], P[..][c2].
// So the only subtractions that could underflow a usize are the ones in (d), and
// those are on u64 VALUES, written as explicit wrapping_sub / checked_sub.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so seed (1, 2) feeds this program exactly the grids and rectangles Go
// sees and every count below comes out identical.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        // state = state * MUL + INC, with explicit 128-bit wraparound
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;

        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

// The table is a Vec of rows, like Go's [][]int64; one flat Vec<i64> indexed
// i * (n + 1) + j is the other common layout (same formula, not measured here).
fn build(a: &[Vec<i64>]) -> Vec<Vec<i64>> {
    let (m, n) = (a.len(), a[0].len());
    let mut p = vec![vec![0i64; n + 1]; m + 1];
    // The index arithmetic IS the lesson here: i, j and i + 1, j + 1 together.
    #[allow(clippy::needless_range_loop)]
    for i in 0..m {
        for j in 0..n {
            p[i + 1][j + 1] = a[i][j] + p[i][j + 1] + p[i + 1][j] - p[i][j];
        }
    }
    p
}

fn rect(p: &[Vec<i64>], r1: usize, c1: usize, r2: usize, c2: usize) -> i64 {
    p[r2][c2] - p[r1][c2] - p[r2][c1] + p[r1][c1]
}

// brute is the oracle: add up every cell of the rectangle, counting the reads.
fn brute(a: &[Vec<i64>], r1: usize, c1: usize, r2: usize, c2: usize, reads: &mut usize) -> i64 {
    let mut sum = 0;
    for row in &a[r1..r2] {
        sum += row[c1..c2].iter().sum::<i64>();
    }
    *reads += (r2 - r1) * (c2 - c1);
    sum
}

// Bug 1: forget to add the corner back.
fn rect_no_corner(p: &[Vec<i64>], r1: usize, c1: usize, r2: usize, c2: usize) -> i64 {
    p[r2][c2] - p[r1][c2] - p[r2][c1]
}

// Bug 2: a table with no zero border (same shape as the grid, t[i][j] = P[i+1][j+1])
// queried with the bordered formula. `at` returns None for reads outside the table:
// `get` is the Rust way to classify them instead of panicking.
fn at(t: &[Vec<i64>], i: usize, j: usize) -> Option<i64> {
    t.get(i)?.get(j).copied()
}

fn rect_no_border(t: &[Vec<i64>], r1: usize, c1: usize, r2: usize, c2: usize) -> Option<i64> {
    Some(at(t, r2, c2)? - at(t, r1, c2)? - at(t, r2, c1)? + at(t, r1, c1)?)
}

// span draws a non-empty half-open range [lo, hi) of [0, size], uniform over pairs lo < hi.
fn span(rng: &mut Pcg, size: usize) -> (usize, usize) {
    let a = rng.int_n(size as u64 + 1) as usize;
    let mut b = rng.int_n(size as u64) as usize;
    if b >= a {
        b += 1;
    }
    (a.min(b), a.max(b))
}

fn main() {
    let mut rng = Pcg::new(1, 2);
    let mut sink = 0;

    println!("(a) a 4x5 grid, its 5x6 table with a zero border, one worked query");
    let a = vec![
        vec![3, 1, 4, 1, 5],
        vec![9, 2, 6, 5, 3],
        vec![5, 8, 9, 7, 9],
        vec![3, 2, 3, 8, 4],
    ];
    let p = build(&a);
    for row in &p {
        for v in row {
            print!("{v:>5}");
        }
        println!();
    }
    println!(
        "build P[3][4] = a[2][3] + P[2][4] + P[3][3] - P[2][3] = {} + {} + {} - {} = {}",
        a[2][3], p[2][4], p[3][3], p[2][3], p[3][4]
    );
    let (r1, c1, r2, c2) = (1, 1, 3, 4);
    println!(
        "query rows [{r1},{r2}) x cols [{c1},{c2}): P[{r2}][{c2}] - P[{r1}][{c2}] - P[{r2}][{c1}] + P[{r1}][{c1}] = {} - {} - {} + {} = {}",
        p[r2][c2],
        p[r1][c2],
        p[r2][c1],
        p[r1][c1],
        rect(&p, r1, c1, r2, c2)
    );
    println!(
        "brute force over the {} cells: {}\n",
        (r2 - r1) * (c2 - c1),
        brute(&a, r1, c1, r2, c2, &mut sink)
    );

    println!(
        "(b) every non-empty rectangle of a 6x6 grid (entries 1-9) against brute force, three bugs"
    );
    let g: Vec<Vec<i64>> = (0..6)
        .map(|_| (0..6).map(|_| 1 + rng.int_n(9) as i64).collect())
        .collect();
    let gp = build(&g);
    // drop the zero row and column: the "no border" table
    let cut: Vec<Vec<i64>> = (0..6).map(|i| gp[i + 1][1..].to_vec()).collect();
    let (mut total, mut bad_ok, mut bad1, mut has_corner) = (0, 0, 0, 0);
    let (mut bad2, mut out_of_range, mut right2) = (0, 0, 0);
    for r1 in 0..6 {
        for r2 in r1 + 1..=6 {
            for c1 in 0..6 {
                for c2 in c1 + 1..=6 {
                    let want = brute(&g, r1, c1, r2, c2, &mut sink);
                    total += 1;
                    if rect(&gp, r1, c1, r2, c2) != want {
                        bad_ok += 1;
                    }
                    if rect_no_corner(&gp, r1, c1, r2, c2) != want {
                        bad1 += 1;
                    }
                    if r1 > 0 && c1 > 0 {
                        has_corner += 1;
                    }
                    match rect_no_border(&cut, r1, c1, r2, c2) {
                        None => out_of_range += 1,
                        Some(v) if v != want => bad2 += 1,
                        Some(_) => right2 += 1,
                    }
                }
            }
        }
    }
    // Bug 3: the caller means INCLUSIVE ends, the table wants EXCLUSIVE ones.
    let (mut incl, mut bad3, mut smaller) = (0, 0, 0);
    for r1 in 0..6 {
        for r2 in r1..6 {
            for c1 in 0..6 {
                for c2 in c1..6 {
                    let want = brute(&g, r1, c1, r2 + 1, c2 + 1, &mut sink);
                    let got = rect(&gp, r1, c1, r2, c2); // BUG: r2, c2 passed as inclusive
                    incl += 1;
                    if got != want {
                        bad3 += 1;
                    }
                    if got < want {
                        smaller += 1;
                    }
                }
            }
        }
    }
    println!("  correct formula:            {bad_ok} wrong of {total}");
    println!(
        "  bug 1, no +P[r1][c1]:       {bad1} wrong of {total}; rectangles with r1 > 0 and c1 > 0: {has_corner}"
    );
    println!(
        "  bug 2, no zero border:      {out_of_range} out of range, {bad2} wrong, {right2} right by luck, of {total}"
    );
    println!(
        "  bug 3, inclusive/exclusive: {bad3} wrong of {incl}; too small in {smaller} of them\n"
    );

    const M: usize = 1000;
    const N: usize = 1000;
    const Q: usize = 10_000;
    println!("(c) a {M}x{N} grid of digits, {Q} random rectangles: cells read");
    let big: Vec<Vec<i64>> = (0..M)
        .map(|_| (0..N).map(|_| rng.int_n(10) as i64).collect())
        .collect();
    let bp = build(&big);
    let build_reads = 4 * M * N; // per cell: a[i][j] and three table cells
    let (mut brute_reads, mut mismatch) = (0, 0);
    for _ in 0..Q {
        let (r1, r2) = span(&mut rng, M);
        let (c1, c2) = span(&mut rng, N);
        if brute(&big, r1, c1, r2, c2, &mut brute_reads) != rect(&bp, r1, c1, r2, c2) {
            mismatch += 1;
        }
    }
    let query_reads = 4 * Q;
    let avg = brute_reads as f64 / Q as f64;
    println!(
        "  brute force: {brute_reads} reads ({avg:.0} per query); prefix sums: {build_reads} to build + {query_reads} for the queries = {}",
        build_reads + query_reads
    );
    println!(
        "  ratio {:.1}x fewer reads; mismatches {mismatch}; break-even at about {:.0} queries\n",
        brute_reads as f64 / (build_reads + query_reads) as f64,
        build_reads as f64 / (avg - 4.0)
    );

    const QD: usize = 1_000_000;
    println!("(d) the left-to-right order on the same table, {QD} random rectangles");
    // The table holds sums of digits, so every entry is non-negative and the
    // conversion to u64 is lossless; try_from makes that an assertion, not a cast.
    let up = |v: i64| u64::try_from(v).expect("prefix sums of digits are non-negative");
    let (mut step2, mut step3, mut any_neg, mut wrap_bad) = (0, 0, 0, 0);
    let (mut borrowed, mut pair_neg, mut pair_bad) = (0, 0, 0);
    for _ in 0..QD {
        let (r1, r2) = span(&mut rng, M);
        let (c1, c2) = span(&mut rng, N);
        let want = rect(&bp, r1, c1, r2, c2);
        let s2 = bp[r2][c2] - bp[r1][c2]; // signed: look at every middle step
        let s3 = s2 - bp[r2][c1];
        if s2 < 0 {
            step2 += 1;
        }
        if s3 < 0 {
            step3 += 1;
        }
        if s2 < 0 || s3 < 0 {
            any_neg += 1;
        }
        let (p22, p12, p21, p11) = (
            up(bp[r2][c2]),
            up(bp[r1][c2]),
            up(bp[r2][c1]),
            up(bp[r1][c1]),
        );
        // unsigned: a plain `-` would panic in a debug build and wrap under -O,
        // so the wrap is spelled out: it dips below zero, then wraps back.
        if p22.wrapping_sub(p12).wrapping_sub(p21).wrapping_add(p11) != up(want) {
            wrap_bad += 1;
        }
        // checked_sub is Rust's borrow detector: None means the chain underflowed.
        if p22
            .checked_sub(p12)
            .and_then(|d| d.checked_sub(p21))
            .is_none()
        {
            borrowed += 1;
        }
        let (x, y) = (p22 + p11, p12 + p21); // paired order: diagonal pair minus the other pair
        if x < y {
            pair_neg += 1;
        }
        if x.wrapping_sub(y) != up(want) {
            pair_bad += 1;
        }
    }
    println!(
        "  signed left-to-right: step 2 negative in {step2}, step 3 negative in {step3}; some middle step negative in {any_neg} ({:.1}%)",
        100.0 * any_neg as f64 / QD as f64
    );
    println!(
        "  uint64 left-to-right: {wrap_bad} wrong answers (it wraps below zero and back); a borrow in the chain in {borrowed} (same count: {})",
        borrowed == any_neg
    );
    println!(
        "  paired order (P22 + P11) - (P12 + P21): goes negative in {pair_neg}, wrong answers {pair_bad}"
    );
}
```

**Rust output:**

```
(a) a 4x5 grid, its 5x6 table with a zero border, one worked query
    0    0    0    0    0    0
    0    3    4    8    9   14
    0   12   15   25   31   39
    0   17   28   47   60   77
    0   20   33   55   76   97
build P[3][4] = a[2][3] + P[2][4] + P[3][3] - P[2][3] = 7 + 31 + 47 - 25 = 60
query rows [1,3) x cols [1,4): P[3][4] - P[1][4] - P[3][1] + P[1][1] = 60 - 9 - 17 + 3 = 37
brute force over the 6 cells: 37

(b) every non-empty rectangle of a 6x6 grid (entries 1-9) against brute force, three bugs
  correct formula:            0 wrong of 441
  bug 1, no +P[r1][c1]:       225 wrong of 441; rectangles with r1 > 0 and c1 > 0: 225
  bug 2, no zero border:      216 out of range, 214 wrong, 11 right by luck, of 441
  bug 3, inclusive/exclusive: 441 wrong of 441; too small in 441 of them

(c) a 1000x1000 grid of digits, 10000 random rectangles: cells read
  brute force: 1106882007 reads (110688 per query); prefix sums: 4000000 to build + 40000 for the queries = 4040000
  ratio 274.0x fewer reads; mismatches 0; break-even at about 36 queries

(d) the left-to-right order on the same table, 1000000 random rectangles
  signed left-to-right: step 2 negative in 0, step 3 negative in 498042; some middle step negative in 498042 (49.8%)
  uint64 left-to-right: 0 wrong answers (it wraps below zero and back); a borrow in the chain in 498042 (same count: true)
  paired order (P22 + P11) - (P12 + P21): goes negative in 0, wrong answers 0
```

> **Go vs Rust:** Identical output, byte for byte (also diffed against a live run of the Go program; same PCG-DXSM port, same grids and rectangles). The table holds `i64` values and the indices are `usize` that are never decremented (half-open bounds plus the zero border mean the formula reads only `r1`, `r2`, `c1`, `c2`), so inclusion-exclusion cannot underflow an index; the no-border bug reads through `get`, which gives `None` where Go needs a hand-written `(value, ok)` helper. In (d) the unsigned chain is written with explicit `wrapping_sub` (it dips below zero and wraps back) and `checked_sub` (its `None` is Go's `bits.Sub64` borrow; the two counts agree): a plain `u64` `-` that goes below zero panics with "attempt to subtract with overflow" in a plain `rustc` build and wraps silently under `-O` (checked in a scratch program, not in this one), so the program never does one.

**Complexity:** build Θ(mn), query Θ(1) with 4 reads, space Θ(mn); brute force is Θ(area) per rectangle. On the 1,000×1,000 grid the 10,000 queries cost 4,040,000 reads against 1,106,882,007 for brute force (274.0x fewer), with break-even at about 36 queries and 0 mismatches. Dropping the corner term is wrong on exactly the 225 of 441 rectangles with r1 > 0 and c1 > 0. The left-to-right chain has a negative middle step on 498,042 of 1,000,000 queries (49.8%), always at step 3: that step equals the rectangle's sum minus the corner block P[r1][c1], so it is negative exactly when the corner block is heavier. The paired order never goes negative, because its final subtraction yields the rectangle's sum, which is not negative on a grid of digits.

---

> Next tier: [🔴 hard](3-hard.md) — counting windows, 3Sum, a brute-force audit of when a window is
> legal, the aggregates with no inverse, and the capstone.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
