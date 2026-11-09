# Step 15 — Binary Search & Its Variants · 🔴 Hard

Examples **12–16**: all 32 boundary variants run against an oracle, bisection on floats and on their
bit patterns, branchless and Eytzinger searches measured, exponential search for lopsided
intersections, and the capstone.

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

> ← Back to [🟡 medium](2-medium.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. Every boundary bug, enumerated

`🔴 hard` · *How to stop getting the boundaries wrong*

A lower bound has only five decisions in it: the loop test (`lo < hi` or `lo <= hi`), the initial `hi`
(`n` or `n-1`), the update when `p(mid)` is true (`hi = mid` or `hi = mid-1`), the update when it is
false (`lo = mid` or `lo = mid+1`), and whether `mid` rounds down or up. That makes 32 programs that all
look like binary search. Instead of reasoning about each one, this example runs all 32 against a linear
oracle on every non-decreasing array of length 0..6 over three values and every target: 420 cases per
variant, a few milliseconds in total. Each run is guarded, so a variant that would read `a[n]` is
reported as *out of range* and one whose window stops shrinking is reported as *non-terminating* after
64 steps. The correct variants are Θ(log n); everything else fails in one of three ways. (Lesson 02
example 5 showed test coverage missing one of these bugs; this is the whole space of them.)

**Steps:**

1. Write one `lowerBound(a, x, variant)` with the five decisions as flags, an index check before every read, and a step cap.
2. Run all 32 variants over every small sorted array and target, compare with the oracle, and classify each case.
3. Read the table: which variants survive, the first input that breaks each of the others, and what the closed template does with unsigned indexes.

```go
package main

import (
	"fmt"
	"strings"
)

// A lower bound ("first index i with a[i] >= x, or n if none") has exactly five
// decisions in it. Each has two plausible spellings, so there are 2^5 = 32
// programs that all LOOK like binary search:
//
//   loop test       lo < hi        | lo <= hi
//   initial hi      hi = n         | hi = n-1
//   p(mid) true     hi = mid       | hi = mid-1
//   p(mid) false    lo = mid       | lo = mid+1
//   mid rounding    floor          | ceil
//
// Instead of arguing about which ones are right, we run all 32 against an
// oracle on EVERY small input and let the results classify them.

type variant struct {
	le     bool // loop test is lo <= hi (else lo < hi)
	hiN1   bool // initial hi is n-1 (else n)
	tMinus bool // on p(mid) true: hi = mid-1 (else hi = mid)
	fPlus  bool // on p(mid) false: lo = mid+1 (else lo = mid)
	ceil   bool // mid rounds up (else down)
}

func (v variant) String() string {
	pick := func(b bool, yes, no string) string {
		if b {
			return yes
		}
		return no
	}
	return fmt.Sprintf("%-6s %-6s %-11s %-11s %-5s",
		pick(v.le, "lo<=hi", "lo<hi"), pick(v.hiN1, "hi=n-1", "hi=n"),
		pick(v.tMinus, "T:hi=mid-1", "T:hi=mid"), pick(v.fPlus, "F:lo=mid+1", "F:lo=mid"),
		pick(v.ceil, "ceil", "floor"))
}

type outcome int

const (
	ok      outcome = iota
	wrong           // terminated, returned the wrong index
	outOfRg         // would have read a[mid] with mid outside [0, n)
	hang            // still looping after stepCap iterations
)

var outcomeName = [...]string{"ok", "wrong", "out of range", "non-terminating"}

// stepCap: a correct search on n <= 6 needs at most 3 steps. 64 is far beyond
// any terminating run, so "still looping at 64" means the window stopped shrinking.
const stepCap = 64

type result struct {
	got, steps, badIdx int
	out                outcome
	hiWentNeg          bool // hi became -1 at some point (matters for unsigned indexes)
}

// lowerBound runs one variant. It never reads out of range and never loops
// forever: the index is checked BEFORE a[mid] is read, and the loop is capped.
// Every variant returns lo -- the value a lower bound "ends at".
func lowerBound(a []int, x int, v variant) result {
	n := len(a)
	lo, hi := 0, n
	if v.hiN1 {
		hi = n - 1
	}
	r := result{hiWentNeg: hi < 0}
	for (!v.le && lo < hi) || (v.le && lo <= hi) {
		if r.steps == stepCap {
			r.out = hang
			return r
		}
		r.steps++
		mid := lo + (hi-lo)/2 // floor (hi-lo >= 0 inside the loop)
		if v.ceil {
			mid = lo + (hi-lo+1)/2
		}
		if mid < 0 || mid >= n { // a real program would panic here
			r.out, r.badIdx = outOfRg, mid
			return r
		}
		if a[mid] >= x { // p(mid) true
			if v.tMinus {
				hi = mid - 1
			} else {
				hi = mid
			}
		} else if v.fPlus {
			lo = mid + 1
		} else {
			lo = mid
		}
		if hi < 0 {
			r.hiWentNeg = true
		}
	}
	r.got = lo
	return r
}

// oracle: obviously correct, obviously slow.
func oracle(a []int, x int) int {
	for i, v := range a {
		if v >= x {
			return i
		}
	}
	return len(a)
}

// allSorted returns every non-decreasing array of length 0..maxLen over 0..maxVal.
func allSorted(maxLen, maxVal int) [][]int {
	var out [][]int
	var grow func(cur []int, min int)
	grow = func(cur []int, min int) {
		out = append(out, append([]int(nil), cur...))
		if len(cur) == maxLen {
			return
		}
		for v := min; v <= maxVal; v++ {
			grow(append(cur, v), v)
		}
	}
	grow(nil, 0)
	return out
}

func show(a []int) string {
	s := make([]string, len(a))
	for i, v := range a {
		s[i] = fmt.Sprint(v)
	}
	return "[" + strings.Join(s, " ") + "]"
}

// closedUnsigned is the closed template (lo<=hi, hi=n-1, hi=mid-1, lo=mid+1)
// written with uint indexes. Go's unsigned arithmetic wraps by spec: 0-1 is
// 18446744073709551615, the loop keeps going and mid lands far outside a.
func closedUnsigned(a []int, x int) (got uint, oor bool) {
	lo, hi := uint(0), uint(len(a))-1 // wraps when len(a) == 0
	for lo <= hi {
		mid := lo + (hi-lo)/2
		if mid >= uint(len(a)) {
			return 0, true
		}
		if a[mid] >= x {
			hi = mid - 1 // wraps when mid == 0
		} else {
			lo = mid + 1
		}
	}
	return lo, false
}

func main() {
	arrays := allSorted(6, 2)
	targets := []int{-1, 0, 1, 2, 3}
	cases := len(arrays) * len(targets)
	fmt.Printf("inputs: %d non-decreasing arrays (length 0..6, values 0..2) x %d targets = %d cases per variant\n\n",
		len(arrays), len(targets), cases)

	fmt.Printf("  #   %-43s %5s %5s %5s %5s  %5s  %s\n",
		"variant", "ok", "wrong", "oor", "hang", "steps", "verdict")
	var correct []int
	verdicts := map[outcome]int{}
	shows := map[outcome]int{} // variants showing this failure at least once
	type firstFail struct {
		id   int
		text string
	}
	var fails []firstFail
	negHi := map[int]int{} // variant id -> cases where hi went to -1
	for id := 0; id < 32; id++ {
		v := variant{le: id&16 != 0, hiN1: id&8 != 0, tMinus: id&4 != 0, fPlus: id&2 != 0, ceil: id&1 != 0}
		var count [4]int
		maxSteps, first := 0, ""
		for _, a := range arrays {
			for _, x := range targets {
				r := lowerBound(a, x, v)
				want := oracle(a, x)
				if r.out == ok && r.got != want {
					r.out = wrong
				}
				count[r.out]++
				if r.hiWentNeg {
					negHi[id+1]++
				}
				if r.out == ok && r.steps > maxSteps {
					maxSteps = r.steps
				}
				if r.out != ok && first == "" {
					switch r.out {
					case wrong:
						first = fmt.Sprintf("returns %d, want %d", r.got, want)
					case outOfRg:
						first = fmt.Sprintf("reads a[%d], n = %d", r.badIdx, len(a))
					case hang:
						first = fmt.Sprintf("no exit after %d steps", stepCap)
					}
					first = fmt.Sprintf("a=%s x=%d: %s", show(a), x, first)
				}
			}
		}
		// A variant's verdict is its most severe failure: a hang is worse than
		// a crash, a crash is worse than a quiet wrong answer.
		verdict := ok
		for _, o := range []outcome{wrong, outOfRg, hang} {
			if count[o] > 0 {
				verdict = o
				shows[o]++
			}
		}
		verdicts[verdict]++
		steps := "-"
		if verdict == ok {
			correct = append(correct, id+1)
			steps = fmt.Sprint(maxSteps)
		} else {
			fails = append(fails, firstFail{id + 1, first})
		}
		fmt.Printf("  %2d  %s %5d %5d %5d %5d  %5s  %s\n",
			id+1, v, count[ok], count[wrong], count[outOfRg], count[hang], steps, outcomeName[verdict])
	}
	fmt.Println("\n  (steps = most loop iterations any case needed, for the correct variants)")

	fmt.Println("\nfirst failing input of each broken variant (the oracle found all of them):")
	for _, f := range fails {
		fmt.Printf("  %2d  %s\n", f.id, f.text)
	}

	fmt.Printf("\nsummary over %d variants (each counted once, by its worst failure): %d correct, %d wrong answer, %d out of range, %d non-terminating\n",
		32, verdicts[ok], verdicts[wrong], verdicts[outOfRg], verdicts[hang])
	fmt.Printf("variants that fail that way at least once: %d wrong answer, %d out of range, %d non-terminating\n",
		shows[wrong], shows[outOfRg], shows[hang])
	fmt.Print("correct variants:")
	for _, id := range correct {
		fmt.Printf(" #%d", id)
	}
	fmt.Println()
	for _, id := range correct {
		v := variant{le: (id-1)&16 != 0, hiN1: (id-1)&8 != 0, tMinus: (id-1)&4 != 0, fPlus: (id-1)&2 != 0, ceil: (id-1)&1 != 0}
		fmt.Printf("  #%-2d %s  hi became -1 in %d of %d cases\n", id, v, negHi[id], cases)
	}

	// The closed template's hi = -1 is harmless with Go's signed int. With an
	// unsigned index it wraps, and the same "correct" code reads out of range.
	oorU, okU := 0, 0
	for _, a := range arrays {
		for _, x := range targets {
			got, oor := closedUnsigned(a, x)
			switch {
			case oor:
				oorU++
			case int(got) == oracle(a, x):
				okU++
			}
		}
	}
	var zero uint
	fmt.Printf("\nthe closed template with uint indexes: %d ok, %d out of range (0-1 wraps to %d)\n",
		okU, oorU, zero-1)

	fmt.Println("\nthe rule: pick ONE template and never mix its parts:")
	fmt.Println("  half-open  lo, hi := 0, n;   for lo < hi;  mid rounds down;  true: hi = mid    false: lo = mid+1")
	fmt.Println("  every update must shrink the window and keep the answer inside it -- the oracle checks both.")
}
```

**Output:**

```
inputs: 84 non-decreasing arrays (length 0..6, values 0..2) x 5 targets = 420 cases per variant

  #   variant                                        ok wrong   oor  hang  steps  verdict
   1  lo<hi  hi=n   T:hi=mid    F:lo=mid    floor   204     0     0   216      -  non-terminating
   2  lo<hi  hi=n   T:hi=mid    F:lo=mid    ceil      5     0   125   290      -  non-terminating
   3  lo<hi  hi=n   T:hi=mid    F:lo=mid+1  floor   420     0     0     0      3  ok
   4  lo<hi  hi=n   T:hi=mid    F:lo=mid+1  ceil    107     0    77   236      -  non-terminating
   5  lo<hi  hi=n   T:hi=mid-1  F:lo=mid    floor   204    58     0   158      -  non-terminating
   6  lo<hi  hi=n   T:hi=mid-1  F:lo=mid    ceil    195   100   125     0      -  out of range
   7  lo<hi  hi=n   T:hi=mid-1  F:lo=mid+1  floor   367    53     0     0      -  wrong
   8  lo<hi  hi=n   T:hi=mid-1  F:lo=mid+1  ceil    290    53    77     0      -  out of range
   9  lo<hi  hi=n-1 T:hi=mid    F:lo=mid    floor   204     6     0   210      -  non-terminating
  10  lo<hi  hi=n-1 T:hi=mid    F:lo=mid    ceil     14   116     0   290      -  non-terminating
  11  lo<hi  hi=n-1 T:hi=mid    F:lo=mid+1  floor   304   116     0     0      -  wrong
  12  lo<hi  hi=n-1 T:hi=mid    F:lo=mid+1  ceil    128    42     0   250      -  non-terminating
  13  lo<hi  hi=n-1 T:hi=mid-1  F:lo=mid    floor   204    46     0   170      -  non-terminating
  14  lo<hi  hi=n-1 T:hi=mid-1  F:lo=mid    ceil    204   216     0     0      -  wrong
  15  lo<hi  hi=n-1 T:hi=mid-1  F:lo=mid+1  floor   270   150     0     0      -  wrong
  16  lo<hi  hi=n-1 T:hi=mid-1  F:lo=mid+1  ceil    325    95     0     0      -  wrong
  17  lo<=hi hi=n   T:hi=mid    F:lo=mid    floor     0     0     5   415      -  non-terminating
  18  lo<=hi hi=n   T:hi=mid    F:lo=mid    ceil      0     0   130   290      -  non-terminating
  19  lo<=hi hi=n   T:hi=mid    F:lo=mid+1  floor     0     0   121   299      -  non-terminating
  20  lo<=hi hi=n   T:hi=mid    F:lo=mid+1  ceil      0     0   143   277      -  non-terminating
  21  lo<=hi hi=n   T:hi=mid-1  F:lo=mid    floor   199     0     5   216      -  non-terminating
  22  lo<=hi hi=n   T:hi=mid-1  F:lo=mid    ceil    190     0   130   100      -  non-terminating
  23  lo<=hi hi=n   T:hi=mid-1  F:lo=mid+1  floor   299     0   121     0      -  out of range
  24  lo<=hi hi=n   T:hi=mid-1  F:lo=mid+1  ceil    277     0   143     0      -  out of range
  25  lo<=hi hi=n-1 T:hi=mid    F:lo=mid    floor     5     0     0   415      -  non-terminating
  26  lo<=hi hi=n-1 T:hi=mid    F:lo=mid    ceil      5     0     0   415      -  non-terminating
  27  lo<=hi hi=n-1 T:hi=mid    F:lo=mid+1  floor   121     0     0   299      -  non-terminating
  28  lo<=hi hi=n-1 T:hi=mid    F:lo=mid+1  ceil    121     0     0   299      -  non-terminating
  29  lo<=hi hi=n-1 T:hi=mid-1  F:lo=mid    floor   204     0     0   216      -  non-terminating
  30  lo<=hi hi=n-1 T:hi=mid-1  F:lo=mid    ceil    204     0     0   216      -  non-terminating
  31  lo<=hi hi=n-1 T:hi=mid-1  F:lo=mid+1  floor   420     0     0     0      3  ok
  32  lo<=hi hi=n-1 T:hi=mid-1  F:lo=mid+1  ceil    420     0     0     0      3  ok

  (steps = most loop iterations any case needed, for the correct variants)

first failing input of each broken variant (the oracle found all of them):
   1  a=[0] x=1: no exit after 64 steps
   2  a=[0] x=-1: reads a[1], n = 1
   4  a=[0] x=-1: reads a[1], n = 1
   5  a=[0] x=1: no exit after 64 steps
   6  a=[0] x=-1: reads a[1], n = 1
   7  a=[0 0 0 0 0 1] x=1: returns 4, want 5
   8  a=[0] x=-1: reads a[1], n = 1
   9  a=[0] x=1: returns 0, want 1
  10  a=[0] x=1: returns 0, want 1
  11  a=[0] x=1: returns 0, want 1
  12  a=[0] x=1: returns 0, want 1
  13  a=[0] x=1: returns 0, want 1
  14  a=[0] x=1: returns 0, want 1
  15  a=[0] x=1: returns 0, want 1
  16  a=[0] x=1: returns 0, want 1
  17  a=[] x=-1: reads a[0], n = 0
  18  a=[] x=-1: reads a[0], n = 0
  19  a=[] x=-1: reads a[0], n = 0
  20  a=[] x=-1: reads a[0], n = 0
  21  a=[] x=-1: reads a[0], n = 0
  22  a=[] x=-1: reads a[0], n = 0
  23  a=[] x=-1: reads a[0], n = 0
  24  a=[] x=-1: reads a[0], n = 0
  25  a=[0] x=-1: no exit after 64 steps
  26  a=[0] x=-1: no exit after 64 steps
  27  a=[0] x=-1: no exit after 64 steps
  28  a=[0] x=-1: no exit after 64 steps
  29  a=[0] x=1: no exit after 64 steps
  30  a=[0] x=1: no exit after 64 steps

summary over 32 variants (each counted once, by its worst failure): 3 correct, 5 wrong answer, 4 out of range, 20 non-terminating
variants that fail that way at least once: 12 wrong answer, 12 out of range, 20 non-terminating
correct variants: #3 #31 #32
  #3  lo<hi  hi=n   T:hi=mid    F:lo=mid+1  floor  hi became -1 in 0 of 420 cases
  #31 lo<=hi hi=n-1 T:hi=mid-1  F:lo=mid+1  floor  hi became -1 in 204 of 420 cases
  #32 lo<=hi hi=n-1 T:hi=mid-1  F:lo=mid+1  ceil   hi became -1 in 204 of 420 cases

the closed template with uint indexes: 216 ok, 204 out of range (0-1 wraps to 18446744073709551615)

the rule: pick ONE template and never mix its parts:
  half-open  lo, hi := 0, n;   for lo < hi;  mid rounds down;  true: hi = mid    false: lo = mid+1
  every update must shrink the window and keep the answer inside it -- the oracle checks both.
```

**Rust:**

```rust
// A lower bound ("first index i with a[i] >= x, or n if none") has exactly five
// decisions in it. Each has two plausible spellings, so there are 2^5 = 32
// programs that all LOOK like binary search:
//
//   loop test       lo < hi        | lo <= hi
//   initial hi      hi = n         | hi = n-1
//   p(mid) true     hi = mid       | hi = mid-1
//   p(mid) false    lo = mid       | lo = mid+1
//   mid rounding    floor          | ceil
//
// Instead of arguing about which ones are right, we run all 32 against an
// oracle on EVERY small input and let the results classify them.
//
// Rust-specific: the variants are written with `isize`, not `usize`. With
// `usize`, `hi = n - 1` at n = 0 and `hi = mid - 1` at mid = 0 would underflow
// (a panic in a debug build, a silent wrap with -O) before the variant even
// got to make its mistake. The last section shows exactly that.

#[derive(Clone, Copy)]
struct Variant {
    le: bool,      // loop test is lo <= hi (else lo < hi)
    hi_n1: bool,   // initial hi is n-1 (else n)
    t_minus: bool, // on p(mid) true: hi = mid-1 (else hi = mid)
    f_plus: bool,  // on p(mid) false: lo = mid+1 (else lo = mid)
    ceil: bool,    // mid rounds up (else down)
}

impl Variant {
    fn from_id(id: usize) -> Self {
        Variant {
            le: id & 16 != 0,
            hi_n1: id & 8 != 0,
            t_minus: id & 4 != 0,
            f_plus: id & 2 != 0,
            ceil: id & 1 != 0,
        }
    }
}

impl std::fmt::Display for Variant {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        let pick = |b: bool, yes: &'static str, no: &'static str| if b { yes } else { no };
        write!(
            f,
            "{:<6} {:<6} {:<11} {:<11} {:<5}",
            pick(self.le, "lo<=hi", "lo<hi"),
            pick(self.hi_n1, "hi=n-1", "hi=n"),
            pick(self.t_minus, "T:hi=mid-1", "T:hi=mid"),
            pick(self.f_plus, "F:lo=mid+1", "F:lo=mid"),
            pick(self.ceil, "ceil", "floor")
        )
    }
}

// Go uses an int "enum" plus a separate got/badIdx; in Rust each outcome
// carries its own data, so a wrong answer cannot forget its index.
#[derive(Clone, Copy)]
enum Outcome {
    Ok(isize),         // terminated, returned this index (checked later)
    OutOfRange(isize), // would have read a[mid] with mid outside [0, n)
    Hang,              // still looping after STEP_CAP iterations
}

const NAMES: [&str; 4] = ["ok", "wrong", "out of range", "non-terminating"];
const OK: usize = 0;
const WRONG: usize = 1;
const OOR: usize = 2;
const HANG: usize = 3;

// a correct search on n <= 6 needs at most 3 steps. 64 is far beyond any
// terminating run, so "still looping at 64" means the window stopped shrinking.
const STEP_CAP: usize = 64;

struct Run {
    out: Outcome,
    steps: usize,
    hi_went_neg: bool, // hi became -1 at some point (matters for usize indexes)
}

// lower_bound runs one variant. It never indexes out of bounds and never loops
// forever: `a.get(mid)` returns None instead of panicking, and the loop is capped.
// Every variant returns lo -- the value a lower bound "ends at".
fn lower_bound(a: &[i32], x: i32, v: Variant) -> Run {
    let n = a.len() as isize;
    let (mut lo, mut hi): (isize, isize) = (0, if v.hi_n1 { n - 1 } else { n });
    let mut r = Run {
        out: Outcome::Hang,
        steps: 0,
        hi_went_neg: hi < 0,
    };
    while (!v.le && lo < hi) || (v.le && lo <= hi) {
        if r.steps == STEP_CAP {
            return r; // out stays Hang
        }
        r.steps += 1;
        // floor; hi - lo >= 0 inside the loop, so / rounds down
        let mid = if v.ceil {
            lo + (hi - lo + 1) / 2
        } else {
            lo + (hi - lo) / 2
        };
        // usize::try_from rejects a negative mid, get() rejects mid >= n:
        // the classified version of what would otherwise be a panic.
        let Some(&am) = usize::try_from(mid).ok().and_then(|m| a.get(m)) else {
            r.out = Outcome::OutOfRange(mid);
            return r;
        };
        if am >= x {
            hi = if v.t_minus { mid - 1 } else { mid };
        } else {
            lo = if v.f_plus { mid + 1 } else { mid };
        }
        if hi < 0 {
            r.hi_went_neg = true;
        }
    }
    r.out = Outcome::Ok(lo);
    r
}

// oracle: obviously correct, obviously slow.
fn oracle(a: &[i32], x: i32) -> usize {
    a.iter().position(|&v| v >= x).unwrap_or(a.len())
}

// all_sorted returns every non-decreasing array of length 0..=max_len over 0..=max_val,
// in the same order as the Go version (pre-order: the array, then its extensions).
fn all_sorted(max_len: usize, max_val: i32) -> Vec<Vec<i32>> {
    fn grow(cur: &mut Vec<i32>, min: i32, max_len: usize, max_val: i32, out: &mut Vec<Vec<i32>>) {
        out.push(cur.clone());
        if cur.len() == max_len {
            return;
        }
        for v in min..=max_val {
            cur.push(v);
            grow(cur, v, max_len, max_val, out);
            cur.pop();
        }
    }
    let mut out = Vec::new();
    grow(&mut Vec::new(), 0, max_len, max_val, &mut out);
    out
}

fn show(a: &[i32]) -> String {
    let s: Vec<String> = a.iter().map(|v| v.to_string()).collect();
    format!("[{}]", s.join(" "))
}

// The closed template (lo<=hi, hi=n-1, hi=mid-1, lo=mid+1) written with usize.
// `sub` is the subtraction to use: wrapping_sub (what -O does to a plain `-`,
// and what Go's uint does by spec) or checked_sub (None = a debug build panics).
// Returns Ok(lo), or Err("oor") / Err("underflow").
fn closed_usize(
    a: &[i32],
    x: i32,
    sub: fn(usize, usize) -> Option<usize>,
) -> Result<usize, &'static str> {
    let mut lo = 0usize;
    let mut hi = sub(a.len(), 1).ok_or("underflow")?; // underflows when a is empty
    while lo <= hi {
        let mid = lo + (hi - lo) / 2;
        let &am = a.get(mid).ok_or("oor")?;
        if am >= x {
            hi = sub(mid, 1).ok_or("underflow")?; // underflows when mid == 0
        } else {
            lo = mid + 1;
        }
    }
    Ok(lo)
}

fn main() {
    let arrays = all_sorted(6, 2);
    let targets = [-1, 0, 1, 2, 3];
    let cases = arrays.len() * targets.len();
    println!(
        "inputs: {} non-decreasing arrays (length 0..6, values 0..2) x {} targets = {} cases per variant\n",
        arrays.len(),
        targets.len(),
        cases
    );

    println!(
        "  #   {:<43} {:>5} {:>5} {:>5} {:>5}  {:>5}  verdict",
        "variant", "ok", "wrong", "oor", "hang", "steps"
    );
    let mut correct = Vec::new();
    let mut verdicts = [0usize; 4];
    let mut shows = [0usize; 4]; // variants showing this failure at least once
    let mut fails: Vec<(usize, String)> = Vec::new();
    let mut neg_hi = [0usize; 33]; // variant id -> cases where hi went to -1
    for id in 0..32 {
        let v = Variant::from_id(id);
        let mut count = [0usize; 4];
        let mut max_steps = 0;
        let mut first: Option<String> = None;
        for a in &arrays {
            for &x in &targets {
                let r = lower_bound(a, x, v);
                let want = oracle(a, x) as isize;
                let (kind, what) = match r.out {
                    Outcome::Ok(got) if got == want => (OK, String::new()),
                    Outcome::Ok(got) => (WRONG, format!("returns {got}, want {want}")),
                    Outcome::OutOfRange(m) => (OOR, format!("reads a[{m}], n = {}", a.len())),
                    Outcome::Hang => (HANG, format!("no exit after {STEP_CAP} steps")),
                };
                count[kind] += 1;
                if r.hi_went_neg {
                    neg_hi[id + 1] += 1;
                }
                if kind == OK {
                    max_steps = max_steps.max(r.steps);
                } else if first.is_none() {
                    first = Some(format!("a={} x={x}: {what}", show(a)));
                }
            }
        }
        // A variant's verdict is its most severe failure: a hang is worse than
        // a crash, a crash is worse than a quiet wrong answer.
        let mut verdict = OK;
        for o in [WRONG, OOR, HANG] {
            if count[o] > 0 {
                verdict = o;
                shows[o] += 1;
            }
        }
        verdicts[verdict] += 1;
        let steps = match first {
            None => {
                correct.push(id + 1);
                max_steps.to_string()
            }
            Some(text) => {
                fails.push((id + 1, text));
                "-".to_string()
            }
        };
        println!(
            "  {:>2}  {v} {:>5} {:>5} {:>5} {:>5}  {:>5}  {}",
            id + 1,
            count[OK],
            count[WRONG],
            count[OOR],
            count[HANG],
            steps,
            NAMES[verdict]
        );
    }
    println!("\n  (steps = most loop iterations any case needed, for the correct variants)");

    println!("\nfirst failing input of each broken variant (the oracle found all of them):");
    for (id, text) in &fails {
        println!("  {id:>2}  {text}");
    }

    println!(
        "\nsummary over 32 variants (each counted once, by its worst failure): {} correct, {} wrong answer, {} out of range, {} non-terminating",
        verdicts[OK], verdicts[WRONG], verdicts[OOR], verdicts[HANG]
    );
    println!(
        "variants that fail that way at least once: {} wrong answer, {} out of range, {} non-terminating",
        shows[WRONG], shows[OOR], shows[HANG]
    );
    let ids: Vec<String> = correct.iter().map(|id| format!(" #{id}")).collect();
    println!("correct variants:{}", ids.concat());
    for &id in &correct {
        println!(
            "  #{id:<2} {}  hi became -1 in {} of {cases} cases",
            Variant::from_id(id - 1),
            neg_hi[id]
        );
    }

    // The closed template's hi = -1 is harmless with isize. With usize it is
    // an underflow: wrapping (Go's uint, Rust -O) turns it into an out-of-range
    // read; checked_sub reports it where it happens.
    let tally = |sub: fn(usize, usize) -> Option<usize>| {
        let (mut ok, mut oor, mut under) = (0, 0, 0);
        for a in &arrays {
            for &x in &targets {
                match closed_usize(a, x, sub) {
                    Ok(got) if got == oracle(a, x) => ok += 1,
                    Ok(_) => {}
                    Err("oor") => oor += 1,
                    Err(_) => under += 1,
                }
            }
        }
        (ok, oor, under)
    };
    let (ok_w, oor_w, _) = tally(|a, b| Some(a.wrapping_sub(b)));
    println!(
        "\nthe closed template with usize, wrapping_sub: {ok_w} ok, {oor_w} out of range (0-1 wraps to {})",
        0usize.wrapping_sub(1)
    );
    let (ok_c, _, under_c) = tally(usize::checked_sub);
    println!(
        "  the same code with checked_sub:               {ok_c} ok, {under_c} underflows caught at the subtraction (where a plain `-` panics in a debug build)"
    );

    println!("\nthe rule: pick ONE template and never mix its parts:");
    println!(
        "  half-open  (lo, hi) = (0, n);  while lo < hi;  mid rounds down;  true: hi = mid    false: lo = mid+1"
    );
    println!(
        "  every update must shrink the window and keep the answer inside it -- the oracle checks both."
    );
}
```

**Rust output:**

```
inputs: 84 non-decreasing arrays (length 0..6, values 0..2) x 5 targets = 420 cases per variant

  #   variant                                        ok wrong   oor  hang  steps  verdict
   1  lo<hi  hi=n   T:hi=mid    F:lo=mid    floor   204     0     0   216      -  non-terminating
   2  lo<hi  hi=n   T:hi=mid    F:lo=mid    ceil      5     0   125   290      -  non-terminating
   3  lo<hi  hi=n   T:hi=mid    F:lo=mid+1  floor   420     0     0     0      3  ok
   4  lo<hi  hi=n   T:hi=mid    F:lo=mid+1  ceil    107     0    77   236      -  non-terminating
   5  lo<hi  hi=n   T:hi=mid-1  F:lo=mid    floor   204    58     0   158      -  non-terminating
   6  lo<hi  hi=n   T:hi=mid-1  F:lo=mid    ceil    195   100   125     0      -  out of range
   7  lo<hi  hi=n   T:hi=mid-1  F:lo=mid+1  floor   367    53     0     0      -  wrong
   8  lo<hi  hi=n   T:hi=mid-1  F:lo=mid+1  ceil    290    53    77     0      -  out of range
   9  lo<hi  hi=n-1 T:hi=mid    F:lo=mid    floor   204     6     0   210      -  non-terminating
  10  lo<hi  hi=n-1 T:hi=mid    F:lo=mid    ceil     14   116     0   290      -  non-terminating
  11  lo<hi  hi=n-1 T:hi=mid    F:lo=mid+1  floor   304   116     0     0      -  wrong
  12  lo<hi  hi=n-1 T:hi=mid    F:lo=mid+1  ceil    128    42     0   250      -  non-terminating
  13  lo<hi  hi=n-1 T:hi=mid-1  F:lo=mid    floor   204    46     0   170      -  non-terminating
  14  lo<hi  hi=n-1 T:hi=mid-1  F:lo=mid    ceil    204   216     0     0      -  wrong
  15  lo<hi  hi=n-1 T:hi=mid-1  F:lo=mid+1  floor   270   150     0     0      -  wrong
  16  lo<hi  hi=n-1 T:hi=mid-1  F:lo=mid+1  ceil    325    95     0     0      -  wrong
  17  lo<=hi hi=n   T:hi=mid    F:lo=mid    floor     0     0     5   415      -  non-terminating
  18  lo<=hi hi=n   T:hi=mid    F:lo=mid    ceil      0     0   130   290      -  non-terminating
  19  lo<=hi hi=n   T:hi=mid    F:lo=mid+1  floor     0     0   121   299      -  non-terminating
  20  lo<=hi hi=n   T:hi=mid    F:lo=mid+1  ceil      0     0   143   277      -  non-terminating
  21  lo<=hi hi=n   T:hi=mid-1  F:lo=mid    floor   199     0     5   216      -  non-terminating
  22  lo<=hi hi=n   T:hi=mid-1  F:lo=mid    ceil    190     0   130   100      -  non-terminating
  23  lo<=hi hi=n   T:hi=mid-1  F:lo=mid+1  floor   299     0   121     0      -  out of range
  24  lo<=hi hi=n   T:hi=mid-1  F:lo=mid+1  ceil    277     0   143     0      -  out of range
  25  lo<=hi hi=n-1 T:hi=mid    F:lo=mid    floor     5     0     0   415      -  non-terminating
  26  lo<=hi hi=n-1 T:hi=mid    F:lo=mid    ceil      5     0     0   415      -  non-terminating
  27  lo<=hi hi=n-1 T:hi=mid    F:lo=mid+1  floor   121     0     0   299      -  non-terminating
  28  lo<=hi hi=n-1 T:hi=mid    F:lo=mid+1  ceil    121     0     0   299      -  non-terminating
  29  lo<=hi hi=n-1 T:hi=mid-1  F:lo=mid    floor   204     0     0   216      -  non-terminating
  30  lo<=hi hi=n-1 T:hi=mid-1  F:lo=mid    ceil    204     0     0   216      -  non-terminating
  31  lo<=hi hi=n-1 T:hi=mid-1  F:lo=mid+1  floor   420     0     0     0      3  ok
  32  lo<=hi hi=n-1 T:hi=mid-1  F:lo=mid+1  ceil    420     0     0     0      3  ok

  (steps = most loop iterations any case needed, for the correct variants)

first failing input of each broken variant (the oracle found all of them):
   1  a=[0] x=1: no exit after 64 steps
   2  a=[0] x=-1: reads a[1], n = 1
   4  a=[0] x=-1: reads a[1], n = 1
   5  a=[0] x=1: no exit after 64 steps
   6  a=[0] x=-1: reads a[1], n = 1
   7  a=[0 0 0 0 0 1] x=1: returns 4, want 5
   8  a=[0] x=-1: reads a[1], n = 1
   9  a=[0] x=1: returns 0, want 1
  10  a=[0] x=1: returns 0, want 1
  11  a=[0] x=1: returns 0, want 1
  12  a=[0] x=1: returns 0, want 1
  13  a=[0] x=1: returns 0, want 1
  14  a=[0] x=1: returns 0, want 1
  15  a=[0] x=1: returns 0, want 1
  16  a=[0] x=1: returns 0, want 1
  17  a=[] x=-1: reads a[0], n = 0
  18  a=[] x=-1: reads a[0], n = 0
  19  a=[] x=-1: reads a[0], n = 0
  20  a=[] x=-1: reads a[0], n = 0
  21  a=[] x=-1: reads a[0], n = 0
  22  a=[] x=-1: reads a[0], n = 0
  23  a=[] x=-1: reads a[0], n = 0
  24  a=[] x=-1: reads a[0], n = 0
  25  a=[0] x=-1: no exit after 64 steps
  26  a=[0] x=-1: no exit after 64 steps
  27  a=[0] x=-1: no exit after 64 steps
  28  a=[0] x=-1: no exit after 64 steps
  29  a=[0] x=1: no exit after 64 steps
  30  a=[0] x=1: no exit after 64 steps

summary over 32 variants (each counted once, by its worst failure): 3 correct, 5 wrong answer, 4 out of range, 20 non-terminating
variants that fail that way at least once: 12 wrong answer, 12 out of range, 20 non-terminating
correct variants: #3 #31 #32
  #3  lo<hi  hi=n   T:hi=mid    F:lo=mid+1  floor  hi became -1 in 0 of 420 cases
  #31 lo<=hi hi=n-1 T:hi=mid-1  F:lo=mid+1  floor  hi became -1 in 204 of 420 cases
  #32 lo<=hi hi=n-1 T:hi=mid-1  F:lo=mid+1  ceil   hi became -1 in 204 of 420 cases

the closed template with usize, wrapping_sub: 216 ok, 204 out of range (0-1 wraps to 18446744073709551615)
  the same code with checked_sub:               216 ok, 204 underflows caught at the subtraction (where a plain `-` panics in a debug build)

the rule: pick ONE template and never mix its parts:
  half-open  (lo, hi) = (0, n);  while lo < hi;  mid rounds down;  true: hi = mid    false: lo = mid+1
  every update must shrink the window and keep the answer inside it -- the oracle checks both.
```

> **Go vs Rust:** Same 32-row table, same first failing inputs, same 3 / 5 / 4 / 20 split: the counts belong to the algorithm, not the language. Rust's differences are in how the bugs surface: the variants use `isize` because with `usize` the closed template's `hi = n - 1` (n = 0) and `hi = mid - 1` (mid = 0) are arithmetic underflows (204 of 420 cases), which `checked_sub` catches at the subtraction and a plain `-` would panic on in a debug build, while Go's `uint` and Rust's `wrapping_sub` silently wrap and fail later as a read of `a[huge]`. Reads go through `a.get(mid)`, which returns `None` instead of the panic `a[mid]` would raise.

**Complexity:** the three correct variants are Θ(log n) (at most 3 steps for n ≤ 6) · the oracle covers 84 arrays × 5 targets = 420 cases per variant · measured: 3 correct (the half-open template and the closed `lo<=hi, hi=n-1, hi=mid-1, lo=mid+1` with either rounding), 5 wrong answer, 4 out of range, 20 non-terminating · the closed template's `hi` goes to −1 in 204 of 420 cases, which is harmless with `int` and reads out of range with `uint`.

---

## 13. Bisection on floats, and on their bits

`🔴 hard` · *Real-valued answers*

Search-on-the-answer works over real numbers too, but `for hi-lo > eps` hides a trap: `float64` is not
the real line. Once `lo` and `hi` are adjacent floats there is nothing between them, `(lo+hi)/2` rounds
to one of the two, and the window never shrinks again. If `eps` is below the spacing of floats near the
answer, the loop never ends, and at large magnitudes even `eps = 1e-9` is far below it. This program
measures the trap under an iteration cap, then shows three ways out: a relative `eps`, a fixed number of
halvings (which must be large enough for the range), and the exact one, binary search over the **bit
pattern**. For positive doubles `a < b` exactly when `bits(a) < bits(b)`, so the ordinary integer lower
bound finds the exact boundary float in at most 63 steps. Bisection costs Θ(log((hi−lo)/ε)) iterations
when ε is above the float spacing and is unbounded otherwise.

**Steps:**

1. Write `bisectEps` with an iteration cap, and run it on √2 with `eps = 1e-18` and on ∛(1e300) with `eps = 1e-9`.
2. Retry with a relative `eps` and with fixed halving counts, and compare how far each answer is from `math.Sqrt` / `math.Cbrt`.
3. Run the half-open template over `math.Float64bits` and check the result is exactly the boundary float: `prev(f)² < 2 ≤ f²`.

```go
package main

import (
	"fmt"
	"math"
)

// Binary search on the answer works over real numbers too: keep [lo, hi] with
// p(lo) false and p(hi) true, halve until the window is small. The textbook
// stopping rule is "while hi - lo > eps". It has a trap: float64 is not the
// real line. Between two ADJACENT floats there is nothing, so once lo and hi
// are neighbours, (lo+hi)/2 rounds to one of them and the window never shrinks
// again. If eps is smaller than the spacing of floats near the answer, the loop
// never ends. Every loop below is capped, so the trap is measured, not entered.

const iterCap = 10_000

// bisectEps runs "while hi-lo > eps" (eps absolute, or relative to hi when rel
// is true) and reports how many iterations ran and when the window last moved.
func bisectEps(lo, hi, eps float64, rel bool, p func(float64) bool) (l, h float64, iters, lastMove int, done bool) {
	for iters = 0; iters < iterCap; iters++ {
		tol := eps
		if rel {
			tol = eps * hi
		}
		if hi-lo <= tol {
			return lo, hi, iters, lastMove, true
		}
		mid := lo + (hi-lo)/2
		if p(mid) {
			if mid != hi {
				lastMove = iters + 1
			}
			hi = mid
		} else {
			if mid != lo {
				lastMove = iters + 1
			}
			lo = mid
		}
	}
	return lo, hi, iters, lastMove, false
}

// bisectN runs a fixed number of halvings: always terminates, but is only
// accurate if n >= log2((hi-lo) / spacing of floats near the answer).
func bisectN(lo, hi float64, n int, p func(float64) bool) float64 {
	for range n {
		mid := lo + (hi-lo)/2
		if p(mid) {
			hi = mid
		} else {
			lo = mid
		}
	}
	return hi
}

// bitsSearch: for POSITIVE float64s, a < b exactly when bits(a) < bits(b)
// (sign 0, then exponent, then mantissa: the bit pattern is a sorted index of
// all floats). So "the first float f in [lo, hi] with p(f)" is an ordinary
// integer lower bound over the bit patterns -- the half-open template, on uint64.
// It finds the exact boundary float, in at most 63 steps for any positive range.
func bitsSearch(lo, hi float64, p func(float64) bool) (float64, int) {
	l, h := math.Float64bits(lo), math.Float64bits(hi)+1 // [l, h): h is one past hi
	steps := 0
	for l < h {
		mid := l + (h-l)/2
		steps++
		if p(math.Float64frombits(mid)) {
			h = mid
		} else {
			l = mid + 1
		}
	}
	return math.Float64frombits(l), steps
}

// ulps: how many floats apart a and b are (both positive).
func ulps(a, b float64) int64 {
	return int64(math.Float64bits(a)) - int64(math.Float64bits(b))
}

func main() {
	sq := func(x float64) bool { return x*x >= 2 }
	// x*x*x overflows to +Inf for x above ~5.6e102; +Inf >= 1e300 is true,
	// which is the right answer there, so the predicate stays monotone.
	cube := func(x float64) bool { return x*x*x >= 1e300 }

	fmt.Println("(a) sqrt(2) on [1, 2] with  while hi-lo > 1e-18:")
	lo, hi, _, last, done := bisectEps(1, 2, 1e-18, false, sq)
	fmt.Printf("  terminated: %v (cap %d iterations); the window last moved at iteration %d\n", done, iterCap, last)
	fmt.Printf("  lo = %.17e\n  hi = %.17e\n", lo, hi)
	fmt.Printf("  hi-lo = %.3e, adjacent floats: %v -> eps 1e-18 is below the spacing, so it can never be met\n",
		hi-lo, math.Nextafter(lo, math.Inf(1)) == hi)

	fmt.Println("\n(b) cube root of 1e300 on [1, 1e300]:")
	lo, hi, _, last, done = bisectEps(1, 1e300, 1e-9, false, cube)
	fmt.Printf("  absolute eps 1e-9:  terminated: %v, window last moved at iteration %d, hi-lo = %.3e (1 ulp: %v)\n",
		done, last, hi-lo, math.Nextafter(lo, math.Inf(1)) == hi)
	lo, hi, it, _, done := bisectEps(1, 1e300, 1e-12, true, cube)
	fmt.Printf("  relative eps 1e-12: terminated: %v after %d iterations, hi = %.15e\n", done, it, hi)

	fmt.Println("\n(c) a fixed iteration count always terminates -- but is it enough?")
	for _, n := range []int{100, 1100} {
		r2 := bisectN(1, 2, n, sq)
		r3 := bisectN(1, 1e300, n, cube)
		fmt.Printf("  %4d halvings: sqrt(2) -> %.17e (%+d ulps from math.Sqrt)\n", n, r2, ulps(r2, math.Sqrt(2)))
		fmt.Printf("  %4s            cbrt(1e300) -> %.17e (relative error %.1e)\n", "", r3, math.Abs(r3-1e100)/1e100)
	}
	fmt.Printf("  [1, 1e300] needs ~log2(1e300 / spacing near 1e100) = %.0f halvings to reach one ulp\n",
		math.Log2(1e300/(math.Nextafter(1e100, math.Inf(1))-1e100)))

	fmt.Println("\n(d) search the BIT PATTERNS instead: exact boundary float, at most 63 steps")
	fmt.Printf("  bits(1.0) = %016x  bits(2.0) = %016x  bits(1e300) = %016x\n",
		math.Float64bits(1), math.Float64bits(2), math.Float64bits(1e300))
	f, steps := bitsSearch(1, 2, sq)
	fmt.Printf("  first f in [1, 2] with f*f >= 2:          %.17e  in %2d steps\n", f, steps)
	fmt.Printf("    math.Sqrt(2) = %.17e, distance %+d ulps\n", math.Sqrt(2), ulps(f, math.Sqrt(2)))
	fmt.Printf("    check: prev(f)^2 = %.17e < 2 <= f^2 = %.17e\n",
		math.Nextafter(f, 0)*math.Nextafter(f, 0), f*f)
	f, steps = bitsSearch(1, 1e300, cube)
	fmt.Printf("  first f in [1, 1e300] with f*f*f >= 1e300: %.17e  in %2d steps\n", f, steps)
	fmt.Printf("    math.Cbrt(1e300) = %.17e, distance %+d ulps\n", math.Cbrt(1e300), ulps(f, math.Cbrt(1e300)))

	fmt.Println("\nthe rule: never stop a float bisection on an absolute eps you picked by feel.")
	fmt.Println("  use a fixed count large enough for the range, a relative eps, or search the bits.")
}
```

**Output:**

```
(a) sqrt(2) on [1, 2] with  while hi-lo > 1e-18:
  terminated: false (cap 10000 iterations); the window last moved at iteration 52
  lo = 1.41421356237309492e+00
  hi = 1.41421356237309515e+00
  hi-lo = 2.220e-16, adjacent floats: true -> eps 1e-18 is below the spacing, so it can never be met

(b) cube root of 1e300 on [1, 1e300]:
  absolute eps 1e-9:  terminated: false, window last moved at iteration 717, hi-lo = 1.943e+84 (1 ulp: true)
  relative eps 1e-12: terminated: true after 705 iterations, hi = 1.000000000000309e+100

(c) a fixed iteration count always terminates -- but is it enough?
   100 halvings: sqrt(2) -> 1.41421356237309515e+00 (+0 ulps from math.Sqrt)
                  cbrt(1e300) -> 7.88860905221011847e+269 (relative error 7.9e+169)
  1100 halvings: sqrt(2) -> 1.41421356237309515e+00 (+0 ulps from math.Sqrt)
                  cbrt(1e300) -> 1.00000000000000002e+100 (relative error 0.0e+00)
  [1, 1e300] needs ~log2(1e300 / spacing near 1e100) = 717 halvings to reach one ulp

(d) search the BIT PATTERNS instead: exact boundary float, at most 63 steps
  bits(1.0) = 3ff0000000000000  bits(2.0) = 4000000000000000  bits(1e300) = 7e37e43c8800759c
  first f in [1, 2] with f*f >= 2:          1.41421356237309515e+00  in 52 steps
    math.Sqrt(2) = 1.41421356237309515e+00, distance +0 ulps
    check: prev(f)^2 = 1.99999999999999956e+00 < 2 <= f^2 = 2.00000000000000044e+00
  first f in [1, 1e300] with f*f*f >= 1e300: 1.00000000000000002e+100  in 62 steps
    math.Cbrt(1e300) = 1.00000000000000002e+100, distance +0 ulps

the rule: never stop a float bisection on an absolute eps you picked by feel.
  use a fixed count large enough for the range, a relative eps, or search the bits.
```

**Rust:**

```rust
// Binary search on the answer works over real numbers too: keep [lo, hi] with
// p(lo) false and p(hi) true, halve until the window is small. The textbook
// stopping rule is "while hi - lo > eps". It has a trap: f64 is not the real
// line. Between two ADJACENT floats there is nothing, so once lo and hi are
// neighbours, (lo+hi)/2 rounds to one of them and the window never shrinks
// again. If eps is smaller than the spacing of floats near the answer, the loop
// never ends. Every loop below is capped, so the trap is measured, not entered.

const ITER_CAP: usize = 10_000;

/// Result of a capped "while hi-lo > eps" bisection.
struct Bisect {
    lo: f64,
    hi: f64,
    iters: usize,
    last_move: usize, // iteration at which lo or hi last changed
    done: bool,       // false = hit ITER_CAP
}

// bisect_eps runs "while hi-lo > eps" (eps absolute, or relative to hi when
// `rel` is true) and reports how many iterations ran and when the window last moved.
fn bisect_eps(mut lo: f64, mut hi: f64, eps: f64, rel: bool, p: impl Fn(f64) -> bool) -> Bisect {
    let mut last_move = 0;
    for iters in 0..ITER_CAP {
        let tol = if rel { eps * hi } else { eps };
        if hi - lo <= tol {
            return Bisect {
                lo,
                hi,
                iters,
                last_move,
                done: true,
            };
        }
        let mid = lo + (hi - lo) / 2.0;
        let side = if p(mid) { &mut hi } else { &mut lo };
        if *side != mid {
            last_move = iters + 1;
        }
        *side = mid;
    }
    Bisect {
        lo,
        hi,
        iters: ITER_CAP,
        last_move,
        done: false,
    }
}

// bisect_n runs a fixed number of halvings: always terminates, but is only
// accurate if n >= log2((hi-lo) / spacing of floats near the answer).
fn bisect_n(mut lo: f64, mut hi: f64, n: usize, p: impl Fn(f64) -> bool) -> f64 {
    for _ in 0..n {
        let mid = lo + (hi - lo) / 2.0;
        if p(mid) {
            hi = mid;
        } else {
            lo = mid;
        }
    }
    hi
}

// bits_search: for POSITIVE f64s, a < b exactly when a.to_bits() < b.to_bits()
// (sign 0, then exponent, then mantissa: the bit pattern is a sorted index of
// all floats). So "the first float f in [lo, hi] with p(f)" is an ordinary
// integer lower bound over the bit patterns -- the half-open template, on u64.
// It finds the exact boundary float, in at most 63 steps for any positive range.
fn bits_search(lo: f64, hi: f64, p: impl Fn(f64) -> bool) -> (f64, u32) {
    let (mut l, mut h) = (lo.to_bits(), hi.to_bits() + 1); // [l, h): h is one past hi
    let mut steps = 0;
    while l < h {
        let mid = l + (h - l) / 2;
        steps += 1;
        if p(f64::from_bits(mid)) {
            h = mid;
        } else {
            l = mid + 1;
        }
    }
    (f64::from_bits(l), steps)
}

/// how many floats apart a and b are (both positive).
fn ulps(a: f64, b: f64) -> i64 {
    a.to_bits() as i64 - b.to_bits() as i64
}

/// Go's %.Ne: Rust's {:e} writes `1.4e0`, Go writes `1.4e+00`.
fn go_exp(v: f64, prec: usize) -> String {
    let s = format!("{v:.prec$e}");
    let (mantissa, exp) = s.split_once('e').unwrap();
    let exp: i32 = exp.parse().unwrap();
    format!(
        "{mantissa}e{}{:02}",
        if exp < 0 { '-' } else { '+' },
        exp.abs()
    )
}

fn main() {
    let sq = |x: f64| x * x >= 2.0;
    // x*x*x overflows to +inf for x above ~5.6e102; +inf >= 1e300 is true,
    // which is the right answer there, so the predicate stays monotone.
    let cube = |x: f64| x * x * x >= 1e300;

    println!("(a) sqrt(2) on [1, 2] with  while hi-lo > 1e-18:");
    let b = bisect_eps(1.0, 2.0, 1e-18, false, sq);
    println!(
        "  terminated: {} (cap {ITER_CAP} iterations); the window last moved at iteration {}",
        b.done, b.last_move
    );
    println!("  lo = {}\n  hi = {}", go_exp(b.lo, 17), go_exp(b.hi, 17));
    println!(
        "  hi-lo = {}, adjacent floats: {} -> eps 1e-18 is below the spacing, so it can never be met",
        go_exp(b.hi - b.lo, 3),
        b.lo.next_up() == b.hi
    );

    println!("\n(b) cube root of 1e300 on [1, 1e300]:");
    let b = bisect_eps(1.0, 1e300, 1e-9, false, cube);
    println!(
        "  absolute eps 1e-9:  terminated: {}, window last moved at iteration {}, hi-lo = {} (1 ulp: {})",
        b.done,
        b.last_move,
        go_exp(b.hi - b.lo, 3),
        b.lo.next_up() == b.hi
    );
    let b = bisect_eps(1.0, 1e300, 1e-12, true, cube);
    println!(
        "  relative eps 1e-12: terminated: {} after {} iterations, hi = {}",
        b.done,
        b.iters,
        go_exp(b.hi, 15)
    );

    println!("\n(c) a fixed iteration count always terminates -- but is it enough?");
    for n in [100, 1100] {
        let r2 = bisect_n(1.0, 2.0, n, sq);
        let r3 = bisect_n(1.0, 1e300, n, cube);
        println!(
            "  {n:>4} halvings: sqrt(2) -> {} ({:+} ulps from f64::sqrt)",
            go_exp(r2, 17),
            ulps(r2, 2f64.sqrt())
        );
        println!(
            "  {:>4}            cbrt(1e300) -> {} (relative error {})",
            "",
            go_exp(r3, 17),
            go_exp((r3 - 1e100).abs() / 1e100, 1)
        );
    }
    println!(
        "  [1, 1e300] needs ~log2(1e300 / spacing near 1e100) = {:.0} halvings to reach one ulp",
        (1e300 / (1e100f64.next_up() - 1e100)).log2()
    );

    println!("\n(d) search the BIT PATTERNS instead: exact boundary float, at most 63 steps");
    println!(
        "  bits(1.0) = {:016x}  bits(2.0) = {:016x}  bits(1e300) = {:016x}",
        1f64.to_bits(),
        2f64.to_bits(),
        1e300f64.to_bits()
    );
    let (f, steps) = bits_search(1.0, 2.0, sq);
    println!(
        "  first f in [1, 2] with f*f >= 2:          {}  in {steps:>2} steps",
        go_exp(f, 17)
    );
    println!(
        "    f64::sqrt(2) = {}, distance {:+} ulps",
        go_exp(2f64.sqrt(), 17),
        ulps(f, 2f64.sqrt())
    );
    let prev = f.next_down();
    println!(
        "    check: prev(f)^2 = {} < 2 <= f^2 = {}",
        go_exp(prev * prev, 17),
        go_exp(f * f, 17)
    );
    let (f, steps) = bits_search(1.0, 1e300, cube);
    println!(
        "  first f in [1, 1e300] with f*f*f >= 1e300: {}  in {steps:>2} steps",
        go_exp(f, 17)
    );
    println!(
        "    f64::cbrt(1e300) = {}, distance {:+} ulps",
        go_exp(1e300f64.cbrt(), 17),
        ulps(f, 1e300f64.cbrt())
    );

    println!("\nthe rule: never stop a float bisection on an absolute eps you picked by feel.");
    println!("  use a fixed count large enough for the range, a relative eps, or search the bits.");
}
```

**Rust output:**

```
(a) sqrt(2) on [1, 2] with  while hi-lo > 1e-18:
  terminated: false (cap 10000 iterations); the window last moved at iteration 52
  lo = 1.41421356237309492e+00
  hi = 1.41421356237309515e+00
  hi-lo = 2.220e-16, adjacent floats: true -> eps 1e-18 is below the spacing, so it can never be met

(b) cube root of 1e300 on [1, 1e300]:
  absolute eps 1e-9:  terminated: false, window last moved at iteration 717, hi-lo = 1.943e+84 (1 ulp: true)
  relative eps 1e-12: terminated: true after 705 iterations, hi = 1.000000000000309e+100

(c) a fixed iteration count always terminates -- but is it enough?
   100 halvings: sqrt(2) -> 1.41421356237309515e+00 (+0 ulps from f64::sqrt)
                  cbrt(1e300) -> 7.88860905221011847e+269 (relative error 7.9e+169)
  1100 halvings: sqrt(2) -> 1.41421356237309515e+00 (+0 ulps from f64::sqrt)
                  cbrt(1e300) -> 1.00000000000000002e+100 (relative error 0.0e+00)
  [1, 1e300] needs ~log2(1e300 / spacing near 1e100) = 717 halvings to reach one ulp

(d) search the BIT PATTERNS instead: exact boundary float, at most 63 steps
  bits(1.0) = 3ff0000000000000  bits(2.0) = 4000000000000000  bits(1e300) = 7e37e43c8800759c
  first f in [1, 2] with f*f >= 2:          1.41421356237309515e+00  in 52 steps
    f64::sqrt(2) = 1.41421356237309515e+00, distance +0 ulps
    check: prev(f)^2 = 1.99999999999999956e+00 < 2 <= f^2 = 2.00000000000000044e+00
  first f in [1, 1e300] with f*f*f >= 1e300: 1.00000000000000002e+100  in 62 steps
    f64::cbrt(1e300) = 1.00000000000000002e+100, distance +0 ulps

the rule: never stop a float bisection on an absolute eps you picked by feel.
  use a fixed count large enough for the range, a relative eps, or search the bits.
```

> **Go vs Rust:** Every number is identical: the same 52 / 717 / 705 iteration counts, the same bit patterns, 52 and 62 bit-search steps, and 0 ulps from `f64::sqrt` / `f64::cbrt`; the only lines that differ name the reference function. Both languages use IEEE-754 doubles with round-to-nearest, so the trap is the same; Rust spells `math.Float64bits` / `Float64frombits` as `to_bits` / `from_bits` and `math.Nextafter(x, ±Inf)` as `next_up` / `next_down` (stable since 1.86). `{:e}` prints `1.4e0`, so `go_exp` rewrites it to Go's `1.4e+00`.

**Complexity:** Θ(log((hi−lo)/ε)) iterations when ε is above the float spacing, unbounded otherwise · measured: √2 with `eps = 1e-18` stops moving after 52 halvings with `lo`, `hi` adjacent and never exits (cap 10,000); ∛(1e300) with `eps = 1e-9` stops after 717 with `hi-lo` ≈ 1.9e84; a relative `eps = 1e-12` exits after 705; 100 fixed halvings are exact for √2 but leave ∛(1e300) at 7.9e269 · the bit search takes 52 and 62 steps and lands 0 ulps from `math.Sqrt` and `math.Cbrt`.

---

## 14. Faster than the textbook: branchless and Eytzinger

`🔴 hard` · *The faster rewrite*

Every search here is Θ(log n) with about ⌈log₂ n⌉ probes, and yet the fastest one is several times quicker
than the textbook loop. Two rewrites change the constant. A **branchless** lower bound runs a loop
whose trip count depends only on n, and picks the next base with arithmetic instead of an `if`, so
random queries no longer cost a branch misprediction per probe. The **Eytzinger layout** (Khuong & Morin
2017) stores the sorted array in the breadth-first order of the implicit search tree (node `k` has children
`2k` and `2k+1`), so the probes that every query makes sit together at the front of the array and stay in
cache. This is arithmetic on one flat array, not a pointer tree (that was lesson 04, example 14). The
program first proves all of them agree with `slices.BinarySearch` and counts their probes, then times
them at 8 KiB, 128 KiB, 2 MiB and 32 MiB. It also contains a trap that only the assembly shows: in Go, the
branchless loop written with an `if` still compiles to a branch.

**Steps:**

1. Write `lowerBound`, the base-and-length loop (once with an `if`, once with a branch-free `less()` built on `bits.Sub64`), and the Eytzinger build and search.
2. Check all of them against `slices.BinarySearch` on 100,000 random queries per size and count probes, then time each one over the same pre-generated queries, best of 3, next to a loop-only control.
3. Compare the ratios across sizes, and read the assembly (`go build -gcflags=-S`) to see which loops still branch.

```go
package main

import (
	"fmt"
	"math/bits"
	"math/rand/v2"
	"slices"
	"time"
)

// Three ways to compute the same lower bound (first i with a[i] >= x). Same
// Theta(log n), same number of probes -- different constants.

// lowerBound: the half-open template from this lesson, written with an if/else.
func lowerBound(a []int64, x int64) int {
	lo, hi := 0, len(a)
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		if a[mid] < x {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

// branchlessIf: keep a base and a length. The loop runs ceil(log2 n) times for
// EVERY query (it depends only on n), and each step either moves base or not,
// so the only thing left to decide is that one "if". In C or Rust the compiler
// usually turns it into a conditional move. Go does NOT: its branch-elimination
// pass refuses a conditional move whose result feeds a load address (base is
// used in a[base+half]) -- cmd/compile/internal/ssa/branchelim.go, issue 26306.
// So in Go this is still a branch (checked in the assembly).
func branchlessIf(a []int64, x int64) int {
	n := len(a)
	if n == 0 {
		return 0
	}
	base := 0
	for n > 1 {
		half := n / 2
		if a[base+half] < x {
			base += half
		}
		n -= half
	}
	if a[base] < x {
		base++
	}
	return base
}

// less returns 1 if a < b, else 0, without a branch: flipping the sign bits
// makes signed order equal unsigned order, and the borrow out of a-b is
// exactly "a < b". bits.Sub64 is a compiler intrinsic (one SUBS on arm64),
// and there is no if, so there is nothing for the compiler to branch on.
func less(a, b int64) int {
	_, borrow := bits.Sub64(uint64(a)^(1<<63), uint64(b)^(1<<63), 0)
	return int(borrow)
}

// branchless: the same loop with the "if" replaced by arithmetic:
// half & -1 = half, half & 0 = 0.
func branchless(a []int64, x int64) int {
	n := len(a)
	if n == 0 {
		return 0
	}
	base := 0
	for n > 1 {
		half := n / 2
		base += half & -less(a[base+half], x)
		n -= half
	}
	return base + less(a[base], x)
}

// Eytzinger layout (Khuong & Morin 2017): store the sorted array in the BFS
// order of the implicit balanced search tree, 1-based: node k has children 2k
// and 2k+1. The first few levels -- the probes EVERY query makes -- sit
// together at the front of the array and stay in cache. Not a pointer tree:
// the "pointers" are arithmetic.
func eytzinger(a []int64) []int64 {
	b := make([]int64, len(a)+1) // b[0] unused
	i := 0
	var fill func(k int)
	fill = func(k int) { // in-order walk of the implicit tree = sorted order
		if k < len(b) {
			fill(2 * k)
			b[k] = a[i]
			i++
			fill(2*k + 1)
		}
	}
	fill(1)
	return b
}

// eytzSearch walks down from the root: right if b[k] < x, else left. When it
// falls off the tree, the answer is the last node where it went LEFT: strip
// the trailing 1-bits (the right turns) and one more bit. Returns the node
// index k in b, or 0 when every element is < x.
// The step is written with less(), so it is branch-free like branchless().
func eytzSearch(b []int64, x int64) int {
	k := 1
	for k < len(b) {
		k = 2*k + less(b[k], x)
	}
	return k >> (bits.TrailingZeros(^uint(k)) + 1)
}

func stdSearch(a []int64, x int64) int {
	i, _ := slices.BinarySearch(a, x)
	return i
}

// ---- probe-counting copies, used only by the correctness pass ----

func lowerBoundP(a []int64, x int64) (int, int) {
	lo, hi, p := 0, len(a), 0
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		p++
		if a[mid] < x {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo, p
}

func branchlessP(a []int64, x int64) (int, int) {
	base, n, p := 0, len(a), 0
	for n > 1 {
		half := n / 2
		p++
		base += half & -less(a[base+half], x)
		n -= half
	}
	p++
	return base + less(a[base], x), p
}

func eytzSearchP(b []int64, x int64) (int, int) {
	k, p := 1, 0
	for k < len(b) {
		p++
		k = 2*k + less(b[k], x)
	}
	return k >> (bits.TrailingZeros(^uint(k)) + 1), p
}

var sink int

func size(bytes int) string {
	if bytes >= 1<<20 {
		return fmt.Sprintf("%d MiB", bytes>>20)
	}
	return fmt.Sprintf("%d KiB", bytes>>10)
}

// bestNsPerQuery runs search over every query 5 times and keeps the fastest
// pass. Searching does not modify a or b, so the same input serves every pass.
func bestNsPerQuery(qs []int64, search func(x int64) int) float64 {
	best := time.Duration(1 << 62)
	for range 5 {
		s := 0
		t := time.Now()
		for _, q := range qs {
			s += search(q)
		}
		if el := time.Since(t); el < best {
			best = el
		}
		sink += s
	}
	return float64(best.Nanoseconds()) / float64(len(qs))
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))
	exps := []int{10, 14, 18, 22}

	fmt.Println("(a) correctness: 100,000 PCG queries per size, every method vs slices.BinarySearch")
	fmt.Printf("  %-6s %10s   %-28s %s\n", "n", "mismatches", "avg probes (lb / bl / eytz)", "min..max (lb / bl / eytz)")
	for _, e := range exps {
		n := 1 << e
		a := make([]int64, n)
		for i := range a {
			a[i] = 2 * int64(i) // even keys: an odd query is absent
		}
		b := eytzinger(a)
		bad := 0
		var sum, mn, mx [3]int
		mn = [3]int{1 << 30, 1 << 30, 1 << 30}
		const Q = 100_000
		for range Q {
			x := rng.Int64N(2*int64(n) + 2) // [0, 2n+1]: present, absent, past the end
			want := stdSearch(a, x)
			i1, p1 := lowerBoundP(a, x)
			i2, p2 := branchlessP(a, x)
			i3 := branchlessIf(a, x)
			k, p3 := eytzSearchP(b, x)
			// Eytzinger returns a node, not a sorted index: compare the element
			// found (or "none" when want == n).
			eOK := (k == 0 && want == n) || (k != 0 && want < n && b[k] == a[want])
			if i1 != want || i2 != want || i3 != want || !eOK {
				bad++
			}
			for j, p := range []int{p1, p2, p3} {
				sum[j] += p
				mn[j], mx[j] = min(mn[j], p), max(mx[j], p)
			}
		}
		fmt.Printf("  2^%-4d %10d   %5.2f / %5.2f / %5.2f        %2d..%-2d / %2d..%-2d / %2d..%-2d\n", e, bad,
			float64(sum[0])/Q, float64(sum[1])/Q, float64(sum[2])/Q, mn[0], mx[0], mn[1], mx[1], mn[2], mx[2])
	}

	fmt.Println("\n(b) ns per query (best of 5 passes over 2^18 pre-generated random queries)")
	names := []string{"control", "slices.Bin", "lowerBound", "bl-if", "branchless", "Eytzinger"}
	fmt.Printf("  %-6s %7s", "n", "array")
	for _, nm := range names {
		fmt.Printf(" %11s", nm)
	}
	fmt.Println()
	type row struct {
		e  int
		ns []float64
	}
	var rows []row
	for _, e := range exps {
		n := 1 << e
		a := make([]int64, n)
		for i := range a {
			a[i] = 2 * int64(i)
		}
		b := eytzinger(a)
		qs := make([]int64, 1<<18) // generated OUTSIDE the timed loops
		for i := range qs {
			qs[i] = rng.Int64N(2 * int64(n))
		}
		// Every row goes through the same func value, so each pays one
		// indirect call per query; the control row measures exactly that
		// (loop + call + sum) with no search.
		fns := []func(int64) int{
			func(x int64) int { return int(x) },
			func(x int64) int { return stdSearch(a, x) },
			func(x int64) int { return lowerBound(a, x) },
			func(x int64) int { return branchlessIf(a, x) },
			func(x int64) int { return branchless(a, x) },
			func(x int64) int { return eytzSearch(b, x) },
		}
		r := row{e: e}
		fmt.Printf("  2^%-4d %7s", e, size(8*n))
		for _, f := range fns {
			ns := bestNsPerQuery(qs, f)
			r.ns = append(r.ns, ns)
			fmt.Printf(" %11.1f", ns)
		}
		fmt.Println()
		rows = append(rows, r)
	}

	fmt.Println("\n  array = one sorted []int64 (the Eytzinger copy is the same size again)")
	fmt.Println("  bl-if = the branchless loop written with an if; branchless = the same loop with less()")
	fmt.Println("\nwhat this run shows (ratios computed from the table above; > 1 means the second is faster):")
	fmt.Printf("  %-6s %-11s %18s %17s %16s\n", "n", "fastest", "slices/branchless", "slices/Eytzinger", "branchless/Eytz")
	for _, r := range rows {
		best := 1
		for j := 2; j < len(r.ns); j++ {
			if r.ns[j] < r.ns[best] {
				best = j
			}
		}
		fmt.Printf("  2^%-4d %-11s %17.2fx %16.2fx %15.2fx\n",
			r.e, names[best], r.ns[1]/r.ns[4], r.ns[1]/r.ns[5], r.ns[4]/r.ns[5])
	}
	_ = sink
}
```

**Sample output:**

```
(a) correctness: 100,000 PCG queries per size, every method vs slices.BinarySearch
  n      mismatches   avg probes (lb / bl / eytz)  min..max (lb / bl / eytz)
  2^10            0   10.00 / 11.00 / 10.00        10..11 / 11..11 / 10..11
  2^14            0   14.00 / 15.00 / 14.00        14..15 / 15..15 / 14..15
  2^18            0   18.00 / 19.00 / 18.00        18..19 / 19..19 / 18..19
  2^22            0   22.00 / 23.00 / 22.00        22..22 / 23..23 / 22..22

(b) ns per query (best of 5 passes over 2^18 pre-generated random queries)
  n        array     control  slices.Bin  lowerBound       bl-if  branchless   Eytzinger
  2^10     8 KiB         0.9        38.5        37.0        37.2         9.3         7.0
  2^14   128 KiB         0.9        56.5        54.6        52.5        15.6        11.6
  2^18     2 MiB         0.9        99.6        97.7        99.2        42.3        20.7
  2^22    32 MiB         0.9       233.2       239.2       241.7       150.5        76.6

  array = one sorted []int64 (the Eytzinger copy is the same size again)
  bl-if = the branchless loop written with an if; branchless = the same loop with less()

what this run shows (ratios computed from the table above; > 1 means the second is faster):
  n      fastest      slices/branchless  slices/Eytzinger  branchless/Eytz
  2^10   Eytzinger                4.14x             5.48x            1.32x
  2^14   Eytzinger                3.62x             4.89x            1.35x
  2^18   Eytzinger                2.36x             4.82x            2.04x
  2^22   Eytzinger                1.55x             3.04x            1.96x
```

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// Three ways to compute the same lower bound (first i with a[i] >= x). Same
// Theta(log n), same number of probes -- different constants.
//
// Every timed search is #[inline(never)]: each one stays a real function (so
// its assembly can be read on its own) and every row pays the same call.

/// The half-open template from this lesson, written with an if/else.
#[inline(never)]
fn lower_bound(a: &[i64], x: i64) -> usize {
    let (mut lo, mut hi) = (0, a.len());
    while lo < hi {
        let mid = lo.midpoint(hi);
        if a[mid] < x {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    lo
}

/// Keep a base and a length. The loop runs ceil(log2 n) times for EVERY
/// query (it depends only on n), and each step either moves base or not, so
/// the only thing left to decide is that one `if`. (Whether rustc turns it
/// into a conditional move is checked in the assembly -- see the note.)
#[inline(never)]
fn branchless_if(a: &[i64], x: i64) -> usize {
    let mut n = a.len();
    if n == 0 {
        return 0;
    }
    let mut base = 0;
    while n > 1 {
        let half = n / 2;
        if a[base + half] < x {
            base += half;
        }
        n -= half;
    }
    base + usize::from(a[base] < x)
}

/// The same loop with arithmetic instead of an `if`. Go needs a borrow-bit
/// trick for "bool to int"; Rust has it built in: usize::from(bool) is 0 or 1.
#[inline(never)]
fn branchless(a: &[i64], x: i64) -> usize {
    let mut n = a.len();
    if n == 0 {
        return 0;
    }
    let mut base = 0;
    while n > 1 {
        let half = n / 2;
        base += half * usize::from(a[base + half] < x);
        n -= half;
    }
    base + usize::from(a[base] < x)
}

/// Eytzinger layout (Khuong & Morin 2017): the sorted array in the BFS order
/// of the implicit balanced search tree, 1-based: node k has children 2k and
/// 2k+1. The first few levels -- the probes EVERY query makes -- sit together
/// at the front of the array and stay in cache. Not a pointer tree: the
/// "pointers" are arithmetic.
fn eytzinger(a: &[i64]) -> Vec<i64> {
    // in-order walk of the implicit tree = sorted order. A nested fn instead
    // of Go's recursive closure: Rust closures cannot call themselves.
    fn fill(k: usize, a: &[i64], i: &mut usize, b: &mut [i64]) {
        if k < b.len() {
            fill(2 * k, a, i, b);
            b[k] = a[*i];
            *i += 1;
            fill(2 * k + 1, a, i, b);
        }
    }
    let mut b = vec![0; a.len() + 1]; // b[0] unused
    fill(1, a, &mut 0, &mut b);
    b
}

/// Walk down from the root: right if b[k] < x, else left. When it falls off
/// the tree, the answer is the last node where it went LEFT: strip the
/// trailing 1-bits (the right turns) and one more bit. Returns the node index
/// k in b, or 0 when every element is < x.
#[inline(never)]
fn eytz_search(b: &[i64], x: i64) -> usize {
    let mut k = 1;
    while k < b.len() {
        k = 2 * k + usize::from(b[k] < x);
    }
    k >> ((!k).trailing_zeros() + 1)
}

/// std's lower bound: partition_point returns the first index where the
/// predicate is false. (binary_search would also do here -- the keys are
/// distinct -- but only partition_point promises the FIRST match.)
#[inline(never)]
fn std_search(a: &[i64], x: i64) -> usize {
    a.partition_point(|&v| v < x)
}

// ---- probe-counting copies, used only by the correctness pass ----

fn lower_bound_p(a: &[i64], x: i64) -> (usize, u32) {
    let (mut lo, mut hi, mut p) = (0, a.len(), 0);
    while lo < hi {
        let mid = lo.midpoint(hi);
        p += 1;
        if a[mid] < x {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    (lo, p)
}

fn branchless_p(a: &[i64], x: i64) -> (usize, u32) {
    let (mut base, mut n, mut p) = (0, a.len(), 0);
    while n > 1 {
        let half = n / 2;
        p += 1;
        base += half * usize::from(a[base + half] < x);
        n -= half;
    }
    (base + usize::from(a[base] < x), p + 1)
}

fn eytz_search_p(b: &[i64], x: i64) -> (usize, u32) {
    let (mut k, mut p) = (1, 0);
    while k < b.len() {
        p += 1;
        k = 2 * k + usize::from(b[k] < x);
    }
    (k >> ((!k).trailing_zeros() + 1), p)
}

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN / Int64N, so the same seed (1, 2) gives exactly Go's queries.
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

    /// like rand.Int64N(n): exactly uniform in [0, n), Lemire's method with rejection.
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

/// Run `search` over every query 5 times and keep the fastest pass. Searching
/// does not modify a or b, so the same input serves every pass. black_box on
/// the function reference keeps the call indirect (like Go's func value), and
/// on the sum so the work cannot be skipped.
fn best_ns_per_query(qs: &[i64], search: &dyn Fn(i64) -> usize) -> f64 {
    let mut best = Duration::MAX;
    for _ in 0..5 {
        let f = black_box(search);
        let t = Instant::now();
        let mut s = 0usize;
        for &q in qs {
            s = s.wrapping_add(f(q));
        }
        best = best.min(t.elapsed());
        black_box(s);
    }
    best.as_nanos() as f64 / qs.len() as f64
}

fn size(bytes: usize) -> String {
    if bytes >= 1 << 20 {
        format!("{} MiB", bytes >> 20)
    } else {
        format!("{} KiB", bytes >> 10)
    }
}

fn main() {
    let mut rng = Pcg::new(1, 2);
    let exps = [10, 14, 18, 22];

    println!("(a) correctness: 100,000 PCG queries per size, every method vs partition_point");
    println!(
        "  {:<6} {:>10}   {:<28} min..max (lb / bl / eytz)",
        "n", "mismatches", "avg probes (lb / bl / eytz)"
    );
    for &e in &exps {
        let n = 1usize << e;
        let a: Vec<i64> = (0..n as i64).map(|i| 2 * i).collect(); // even keys: an odd query is absent
        let b = eytzinger(&a);
        let mut bad = 0;
        let (mut sum, mut mn, mut mx) = ([0u32; 3], [u32::MAX; 3], [0u32; 3]);
        const Q: u32 = 100_000;
        for _ in 0..Q {
            let x = rng.int_n(2 * n as u64 + 2) as i64; // [0, 2n+1]: present, absent, past the end
            let want = std_search(&a, x);
            let (i1, p1) = lower_bound_p(&a, x);
            let (i2, p2) = branchless_p(&a, x);
            let i3 = branchless_if(&a, x);
            let (k, p3) = eytz_search_p(&b, x);
            // Eytzinger returns a node, not a sorted index: compare the element
            // found (or "none" when want == n). a.get(want) is None exactly then.
            let e_ok = match (k, a.get(want)) {
                (0, None) => true,
                (k, Some(&v)) if k != 0 => b[k] == v,
                _ => false,
            };
            if i1 != want || i2 != want || i3 != want || !e_ok {
                bad += 1;
            }
            for (j, p) in [p1, p2, p3].into_iter().enumerate() {
                sum[j] += p;
                mn[j] = mn[j].min(p);
                mx[j] = mx[j].max(p);
            }
        }
        let avg = |j: usize| f64::from(sum[j]) / f64::from(Q);
        println!(
            "  2^{e:<4} {bad:>10}   {:5.2} / {:5.2} / {:5.2}        {:>2}..{:<2} / {:>2}..{:<2} / {:>2}..{:<2}",
            avg(0),
            avg(1),
            avg(2),
            mn[0],
            mx[0],
            mn[1],
            mx[1],
            mn[2],
            mx[2]
        );
    }

    println!("\n(b) ns per query (best of 5 passes over 2^18 pre-generated random queries)");
    let names = [
        "control",
        "p_point",
        "lowerBound",
        "bl-if",
        "branchless",
        "Eytzinger",
    ];
    print!("  {:<6} {:>7}", "n", "array");
    for nm in names {
        print!(" {nm:>11}");
    }
    println!();
    let mut rows: Vec<(u32, Vec<f64>)> = Vec::new();
    for &e in &exps {
        let n = 1usize << e;
        let a: Vec<i64> = (0..n as i64).map(|i| 2 * i).collect();
        let b = eytzinger(&a);
        // generated OUTSIDE the timed loops
        let qs: Vec<i64> = (0..1 << 18)
            .map(|_| rng.int_n(2 * n as u64) as i64)
            .collect();
        // Every row goes through the same &dyn Fn, so each pays one indirect
        // call per query; the control row measures exactly that (loop + call
        // + sum) with no search.
        let fns: [&dyn Fn(i64) -> usize; 6] = [
            &|x| x as usize,
            &|x| std_search(&a, x),
            &|x| lower_bound(&a, x),
            &|x| branchless_if(&a, x),
            &|x| branchless(&a, x),
            &|x| eytz_search(&b, x),
        ];
        print!("  2^{e:<4} {:>7}", size(8 * n));
        let ns: Vec<f64> = fns
            .iter()
            .map(|f| {
                let t = best_ns_per_query(&qs, *f);
                print!(" {t:>11.1}");
                t
            })
            .collect();
        println!();
        rows.push((e, ns));
    }

    println!("\n  array = one sorted Vec<i64> (the Eytzinger copy is the same size again)");
    println!("  p_point = std partition_point; bl-if = the branchless loop written with an if;");
    println!("  branchless = the same loop with usize::from(bool)");
    println!(
        "\nwhat this run shows (ratios computed from the table above; > 1 means the second is faster):"
    );
    println!(
        "  {:<6} {:<11} {:>18} {:>17} {:>16}",
        "n", "fastest", "p_point/branchless", "p_point/Eytzinger", "branchless/Eytz"
    );
    for (e, ns) in &rows {
        let best = (2..ns.len()).fold(1, |b, j| if ns[j] < ns[b] { j } else { b });
        println!(
            "  2^{e:<4} {:<11} {:>17.2}x {:>16.2}x {:>15.2}x",
            names[best],
            ns[1] / ns[4],
            ns[1] / ns[5],
            ns[4] / ns[5]
        );
    }
}
```

**Rust sample output:**

```
(a) correctness: 100,000 PCG queries per size, every method vs partition_point
  n      mismatches   avg probes (lb / bl / eytz)  min..max (lb / bl / eytz)
  2^10            0   10.00 / 11.00 / 10.00        10..11 / 11..11 / 10..11
  2^14            0   14.00 / 15.00 / 14.00        14..15 / 15..15 / 14..15
  2^18            0   18.00 / 19.00 / 18.00        18..19 / 19..19 / 18..19
  2^22            0   22.00 / 23.00 / 22.00        22..22 / 23..23 / 22..22

(b) ns per query (best of 5 passes over 2^18 pre-generated random queries)
  n        array     control     p_point  lowerBound       bl-if  branchless   Eytzinger
  2^10     8 KiB         0.8         4.4         7.7         5.2         6.2         4.6
  2^14   128 KiB         0.8         7.9        14.1         9.3        11.0         7.8
  2^18     2 MiB         0.8        26.2        43.9        31.0        33.6        15.1
  2^22    32 MiB         0.8       168.0       199.7       173.0       184.7        91.1

  array = one sorted Vec<i64> (the Eytzinger copy is the same size again)
  p_point = std partition_point; bl-if = the branchless loop written with an if;
  branchless = the same loop with usize::from(bool)

what this run shows (ratios computed from the table above; > 1 means the second is faster):
  n      fastest     p_point/branchless p_point/Eytzinger  branchless/Eytz
  2^10   p_point                  0.71x             0.97x            1.36x
  2^14   Eytzinger                0.72x             1.01x            1.40x
  2^18   Eytzinger                0.78x             1.73x            2.22x
  2^22   Eytzinger                0.91x             1.84x            2.03x
```

> **Go vs Rust:** Mismatches and probe counts are identical to Go's, because the PCG port generates the same queries. The timings are not. Checked with `rustc -O --emit asm` (arm64): rustc compiles every search here to `csel`/`cset`, including the if/else `lower_bound` and `bl-if`, and std's `partition_point` loop is branch-free with no bounds check. So std ran at about 4.4 ns/query at 8 KiB, against about 38 ns for Go's `slices.BinarySearch` on the same queries, both behind one indirect call. In Rust, Eytzinger only pulls ahead of std from 2 MiB (about 1.5–2.2× across runs). The hand-written branchless loop was 0.7–0.95× of std's speed up to 2 MiB. A plausible cause, not measured: its bounds check, which std's loop does not have.

**Complexity:** all Θ(log n): measured probes per query are 10–11 / 11 / 10–11 (lower bound / branchless / Eytzinger) at n = 2¹⁰ and 22 / 23 / 22 at 2²², with 0 mismatches. The layout changes cache behaviour, not the number of comparisons. In every run Eytzinger was the fastest at every size. Up to 2 MiB it was about 4.7–6× faster than `slices.BinarySearch`, and the branch-free loop about 2.2–4.3× faster. At 32 MiB the margins shrink and swing with machine load: Eytzinger was roughly 1.4–3.8× faster, and the branch-free loop anywhere from 0.6× (slower) to 1.8×. `slices.BinarySearch`, `lowerBound` and `bl-if` stay within noise of each other. The assembly (`go build -gcflags=-S`, arm64) shows why: all three compile their comparison to a conditional branch (`BLE`/`BGT`). Go's branch-elimination pass refuses a conditional move whose result feeds a load address (`cmd/compile/internal/ssa/branchelim.go`, issue 26306). `less()` becomes `SUBS` + `NGC`, and the only branches left in `branchless` and `eytzSearch` are the loop exit and the bounds check.

---

## 15. Exponential search and lopsided intersections

`🔴 hard` · *Galloping*

Exponential search ("galloping") probes positions 0, 1, 3, 7, 15, … until it passes the answer, then
runs the ordinary binary search on the last gap. If the answer is at position i, that costs about
2·log₂ i probes, Θ(log i), whatever n is. It even works when n is unknown, as on a stream or a
sequence with no end. Its main use is intersecting a small sorted list with a huge one. Merge costs
Θ(m+n), a binary search per element costs Θ(m log n), and galloping on from the last match costs
Θ(m log(n/m)), because each search only has to cover the gap to the next match. Counted comparisons
show where each method wins. Galloping is not best everywhere: plain binary search beats it for a
few lookups, and merge beats it when the lists are the same size. It is also never far from the
best, so it makes a good default when you do not know the sizes. The merge step of Timsort
(Python's `list.sort`) gallops for the same reason.

**Steps:**

1. Write `gallopSearch` over a predicate with no known end, and count its probes against a binary
   search over `[0, 2⁶²)` for answers from 0 to 2⁶² − 1.
2. Intersect A (m values) with B (1,000,000 values) four ways: merge, a binary search over the whole
   of B for each element, a binary search over the rest of B, and galloping from the last match.
   Count element comparisons, and check that all four return the same list.
3. Read the `fewest` column, then the timing table, which shows that a comparison inside a merge
   costs less than a probe inside a search.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"strings"
	"time"
)

// ---------- (a) searching a sequence with no known end ----------

// search is the half-open lower bound of example 1 over [lo, hi): the
// first i with p(i) true, or hi if none. p(hi) is never called.
func search(lo, hi int, p func(int) bool) int {
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		if p(mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

// gallopSearch finds the first true in [0, n) without paying for n:
// probe 0, 1, 3, 7, 15, ... (the gap doubles each time) until a probe is
// true or the end is passed, then run the ordinary search on the last gap.
// Invariant during the gallop: everything before lo is false.
// If the answer is i, the gallop makes about log2(i) probes and the last
// gap holds about i/2 candidates, so the total is about 2*log2(i): it
// depends on WHERE the answer is, not on n. n = math.MaxInt means "no end".
func gallopSearch(n int, p func(int) bool) int {
	lo, hi, step := 0, 0, 1
	for hi < n && !p(hi) {
		lo = hi + 1
		if step >= n-hi { // hi+step would pass the end (and could overflow)
			hi = n
		} else {
			hi += step
		}
		step *= 2
	}
	return search(lo, hi, p) // p(hi) is true, or hi == n: a valid sentinel
}

// ---------- (b) intersecting a small sorted list with a big one ----------
// Specialised to slices (no closures), so the timings in (c) compare the
// algorithms rather than the cost of calling a func value. Each function
// returns the common values and the number of element comparisons it made.

// lowerBoundIn: first k in [lo, len(b)) with b[k] >= x. One comparison per probe.
func lowerBoundIn(b []int, lo, x int) (k, cmps int) {
	hi := len(b)
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		cmps++
		if b[mid] >= x {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo, cmps
}

// gallopIn: the same answer as lowerBoundIn, galloping from lo (probes at
// lo, lo+1, lo+3, lo+7, ...). Cheap when the answer is close to lo.
func gallopIn(b []int, lo, x int) (k, cmps int) {
	hi, step, n := lo, 1, len(b)
	for hi < n {
		cmps++
		if b[hi] >= x {
			break
		}
		lo = hi + 1
		if step >= n-hi {
			hi = n
		} else {
			hi += step
		}
		step *= 2
	}
	for lo < hi { // the last gap: b[hi] >= x (or hi == n) is the sentinel
		mid := int(uint(lo+hi) >> 1)
		cmps++
		if b[mid] >= x {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo, cmps
}

// merge walks both lists once: Theta(m+n). One three-way comparison per step.
func merge(a, b, dst []int) ([]int, int) {
	i, j, cmps := 0, 0, 0
	for i < len(a) && j < len(b) {
		cmps++
		switch {
		case a[i] < b[j]:
			i++
		case a[i] > b[j]:
			j++
		default:
			dst = append(dst, a[i])
			i++
			j++
		}
	}
	return dst, cmps
}

// probeEach looks up every element of a in b: Theta(m log n).
// mode "whole" searches all of b each time; "suffix" starts where the
// previous answer was (a is sorted, so the next answer cannot be earlier);
// "gallop" gallops from there: Theta(m log(n/m)).
func probeEach(a, b, dst []int, mode string) ([]int, int) {
	start, total := 0, 0
	for _, x := range a {
		var k, c int
		switch mode {
		case "whole":
			k, c = lowerBoundIn(b, 0, x)
		case "suffix":
			k, c = lowerBoundIn(b, start, x)
		default:
			k, c = gallopIn(b, start, x)
		}
		total += c
		start = k
		if k < len(b) {
			total++ // the equality check that decides "found"
			if b[k] == x {
				dst = append(dst, x)
				start = k + 1
			}
		}
	}
	return dst, total
}

// sortedGaps returns n strictly increasing values whose gaps are uniform in
// [1, 2g-1] (mean g). Two lists built this way over the same span overlap.
func sortedGaps(rng *rand.Rand, n, g int) []int {
	s, v := make([]int, n), 0
	for i := range s {
		v += 1 + rng.IntN(2*g-1)
		s[i] = v
	}
	return s
}

// bestOf3 returns the fastest of three ~50 ms measurements, per call of f.
func bestOf3(f func()) float64 {
	best := math.Inf(1)
	for range 3 {
		reps, start := 0, time.Now()
		for time.Since(start) < 50*time.Millisecond {
			f()
			reps++
		}
		best = min(best, float64(time.Since(start).Nanoseconds())/float64(reps))
	}
	return best
}

func main() {
	// (a) probes to find position i, with and without knowing the end
	fmt.Println("(a) first true at position i: galloping (no end needed) vs binary search over [0, 2^62)")
	fmt.Printf("%22s %8s %8s\n", "i", "gallop", "binary")
	lastWin, firstLoss := 0, -1
	for _, i := range []int{0, 1, 10, 1000, 1_000_000, 1_000_000_000, 1 << 31, 1 << 40, 1_000_000_000_000_000, 1<<62 - 1} {
		g, b := 0, 0
		gi := gallopSearch(math.MaxInt, func(k int) bool { g++; return k >= i })
		bi := search(0, 1<<62, func(k int) bool { b++; return k >= i })
		if gi != i || bi != i {
			fmt.Println("  WRONG ANSWER", i, gi, bi)
		}
		fmt.Printf("%22d %8d %8d\n", i, g, b)
		if g < b {
			lastWin = i
		} else if firstLoss < 0 {
			firstLoss = i
		}
	}
	fmt.Printf("  galloping used fewer probes up to i = %d and more from i = %d on: about 2*log2(i) vs log2(2^62) = 62\n", lastWin, firstLoss)

	// (b) A (m values) intersect B (n = 1,000,000 values), comparisons counted
	rng := rand.New(rand.NewPCG(1, 2))
	const n = 1_000_000
	B := sortedGaps(rng, n, 4) // values up to about 4n
	fmt.Printf("\n(b) |A| = m, |B| = %d, both strictly increasing over the same span; element comparisons\n", n)
	fmt.Printf("%9s %8s %10s %11s %11s %10s  %s\n", "m", "common", "merge", "bin-whole", "bin-suffix", "gallop", "fewest")
	names := []string{"merge", "bin-whole", "bin-suffix", "gallop"}
	var A100, A100k []int
	for _, m := range []int{10, 100, 1_000, 10_000, 100_000, 1_000_000} {
		A := sortedGaps(rng, m, 4*n/m)
		if m == 100 {
			A100 = A
		}
		if m == 100_000 {
			A100k = A
		}
		r0, c0 := merge(A, B, nil)
		r1, c1 := probeEach(A, B, nil, "whole")
		r2, c2 := probeEach(A, B, nil, "suffix")
		r3, c3 := probeEach(A, B, nil, "gallop")
		if !slices.Equal(r0, r1) || !slices.Equal(r0, r2) || !slices.Equal(r0, r3) {
			fmt.Println("  RESULTS DIFFER at m =", m)
		}
		counts := []int{c0, c1, c2, c3}
		best := slices.Index(counts, slices.Min(counts))
		fmt.Printf("%9d %8d %10d %11d %11d %10d  %s", m, len(r0), c0, c1, c2, c3, names[best])
		if best != 3 {
			fmt.Printf(" (gallop %.2fx of it)", float64(c3)/float64(counts[best]))
		}
		fmt.Println()
	}
	fmt.Println("  all four methods returned identical intersections in every row")

	// (c) one timing table. Nothing is mutated (the inputs are only read), so
	// no copies are needed; the output buffer is reused, and its length is
	// added to a sink so the work cannot be skipped.
	fmt.Println("\n(c) ns per whole intersection (best of 3)")
	fmt.Printf("%9s %12s %12s %12s %12s\n", "m", "merge", "bin-whole", "bin-suffix", "gallop")
	buf, sink := make([]int, 0, n), 0
	for _, A := range [][]int{A100, A100k} {
		ns := []float64{
			bestOf3(func() { r, _ := merge(A, B, buf[:0]); sink += len(r) }),
			bestOf3(func() { r, _ := probeEach(A, B, buf[:0], "whole"); sink += len(r) }),
			bestOf3(func() { r, _ := probeEach(A, B, buf[:0], "suffix"); sink += len(r) }),
			bestOf3(func() { r, _ := probeEach(A, B, buf[:0], "gallop"); sink += len(r) }),
		}
		var row strings.Builder
		for _, t := range ns {
			fmt.Fprintf(&row, " %12.0f", t)
		}
		fastest := slices.Index(ns, slices.Min(ns))
		fmt.Printf("%9d%s  fastest: %s, gallop/merge time %.3f\n", len(A), row.String(), names[fastest], ns[3]/ns[0])
		if len(A) == 100_000 {
			// Comparisons are not all the same price: merge's walk is
			// sequential, a search's probes jump around an 8 MB array.
			_, cm := merge(A, B, buf[:0])
			_, cg := probeEach(A, B, buf[:0], "gallop")
			fmt.Printf("  ns per comparison at m = %d: merge %.2f, gallop %.2f\n", len(A), ns[0]/float64(cm), ns[3]/float64(cg))
		}
	}
	if sink == 0 {
		fmt.Println("(sink)")
	}
}
```

**Sample output:**

```
(a) first true at position i: galloping (no end needed) vs binary search over [0, 2^62)
                     i   gallop   binary
                     0        1       63
                     1        2       63
                    10        8       62
                  1000       20       62
               1000000       40       62
            1000000000       60       62
            2147483648       64       62
         1099511627776       82       62
      1000000000000000      100       62
   4611686018427387903      124       62
  galloping used fewer probes up to i = 1000000000 and more from i = 2147483648 on: about 2*log2(i) vs log2(2^62) = 62

(b) |A| = m, |B| = 1000000, both strictly increasing over the same span; element comparisons
        m   common      merge   bin-whole  bin-suffix     gallop  fewest
       10        4     955153         208         196        338  bin-suffix (gallop 1.72x of it)
      100       27    1000066        2081        1835       2619  bin-suffix (gallop 1.43x of it)
     1000      255     996183       20955       19585      20958  bin-suffix (gallop 1.07x of it)
    10000     2588    1002451      209486      195826     144555  gallop
   100000    24785    1075023     2094902     1957398     802044  gallop
  1000000   249704    1749778    20951315    19881854    2881633  merge (gallop 1.65x of it)
  all four methods returned identical intersections in every row

(c) ns per whole intersection (best of 3)
        m        merge    bin-whole   bin-suffix       gallop
      100       765535         1483         1251         2422  fastest: bin-suffix, gallop/merge time 0.003
   100000      1301693      4379372      3150586      1798506  fastest: merge, gallop/merge time 1.382
  ns per comparison at m = 100000: merge 1.21, gallop 2.24
```

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// ---------- (a) searching a sequence with no known end ----------

/// The half-open lower bound of example 1 over [lo, hi): the first i with
/// p(i) true, or hi if none. p(hi) is never called. `usize::midpoint`
/// rounds down and cannot overflow (Go writes `int(uint(lo+hi) >> 1)`).
fn search(mut lo: usize, mut hi: usize, mut p: impl FnMut(usize) -> bool) -> usize {
    while lo < hi {
        let mid = lo.midpoint(hi);
        if p(mid) {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    lo
}

/// First true in [0, n) without paying for n: probe 0, 1, 3, 7, 15, ...
/// (the gap doubles) until a probe is true or the end is passed, then the
/// ordinary search on the last gap. Invariant: everything before lo is
/// false. Answer at i -> about log2(i) gallop probes plus about log2(i/2)
/// in the gap: ~2*log2(i), independent of n. n = usize::MAX means "no end".
fn gallop_search(n: usize, mut p: impl FnMut(usize) -> bool) -> usize {
    let (mut lo, mut hi, mut step) = (0, 0, 1);
    while hi < n && !p(hi) {
        lo = hi + 1;
        // hi + step would pass the end (and could overflow): stop at n
        hi = if step >= n - hi { n } else { hi + step };
        step *= 2;
    }
    search(lo, hi, p) // p(hi) is true, or hi == n: a valid sentinel
}

// ---------- (b) intersecting a small sorted list with a big one ----------
// Plain functions over slices, so the timings in (c) compare algorithms.
// (Rust closures are monomorphised and usually inlined, so a predicate
// version would cost about the same; Go's func values are indirect calls.)
// Each returns how many element comparisons it made.

/// First k in [lo, b.len()) with b[k] >= x. One comparison per probe.
fn lower_bound_in(b: &[u64], mut lo: usize, x: u64) -> (usize, u64) {
    let (mut hi, mut cmps) = (b.len(), 0);
    while lo < hi {
        let mid = lo.midpoint(hi);
        cmps += 1;
        if b[mid] >= x {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    (lo, cmps)
}

/// Same answer as lower_bound_in, galloping from lo (probes at lo, lo+1,
/// lo+3, lo+7, ...). Cheap when the answer is close to lo.
fn gallop_in(b: &[u64], mut lo: usize, x: u64) -> (usize, u64) {
    let (mut hi, mut step, n, mut cmps) = (lo, 1, b.len(), 0);
    while hi < n {
        cmps += 1;
        if b[hi] >= x {
            break;
        }
        lo = hi + 1;
        hi = if step >= n - hi { n } else { hi + step };
        step *= 2;
    }
    while lo < hi {
        // the last gap: b[hi] >= x (or hi == n) is the sentinel
        let mid = lo.midpoint(hi);
        cmps += 1;
        if b[mid] >= x {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    (lo, cmps)
}

/// Walk both lists once: Theta(m+n). One three-way comparison per step.
fn merge(a: &[u64], b: &[u64], out: &mut Vec<u64>) -> u64 {
    let (mut i, mut j, mut cmps) = (0, 0, 0);
    while i < a.len() && j < b.len() {
        cmps += 1;
        match a[i].cmp(&b[j]) {
            std::cmp::Ordering::Less => i += 1,
            std::cmp::Ordering::Greater => j += 1,
            std::cmp::Ordering::Equal => {
                out.push(a[i]);
                i += 1;
                j += 1;
            }
        }
    }
    cmps
}

/// How probe_each looks up each element of a (Go passes a string).
#[derive(Clone, Copy)]
enum Mode {
    Whole,  // all of b every time: Theta(m log n)
    Suffix, // from the previous answer on (a is sorted)
    Gallop, // gallop from the previous answer: Theta(m log(n/m))
}

fn probe_each(a: &[u64], b: &[u64], out: &mut Vec<u64>, mode: Mode) -> u64 {
    let (mut start, mut total) = (0, 0);
    for &x in a {
        let (k, c) = match mode {
            Mode::Whole => lower_bound_in(b, 0, x),
            Mode::Suffix => lower_bound_in(b, start, x),
            Mode::Gallop => gallop_in(b, start, x),
        };
        total += c;
        start = k;
        if let Some(&v) = b.get(k) {
            total += 1; // the equality check that decides "found"
            if v == x {
                out.push(x);
                start = k + 1;
            }
        }
    }
    total
}

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) gives exactly the lists Go generates.
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

/// n strictly increasing values with gaps uniform in [1, 2g-1] (mean g).
fn sorted_gaps(rng: &mut Pcg, n: usize, g: u64) -> Vec<u64> {
    let mut v = 0;
    (0..n)
        .map(|_| {
            v += 1 + rng.int_n(2 * g - 1);
            v
        })
        .collect()
}

/// The fastest of three ~50 ms measurements, in ns per call of f.
fn best_of3(mut f: impl FnMut()) -> f64 {
    (0..3)
        .map(|_| {
            let (mut reps, start) = (0u32, Instant::now());
            while start.elapsed() < Duration::from_millis(50) {
                f();
                reps += 1;
            }
            start.elapsed().as_nanos() as f64 / f64::from(reps)
        })
        .fold(f64::INFINITY, f64::min)
}

/// index of the smallest value (first one on ties), like slices.Index(s, slices.Min(s))
fn argmin<T: PartialOrd + Copy>(xs: &[T]) -> usize {
    (1..xs.len()).fold(0, |b, i| if xs[i] < xs[b] { i } else { b })
}

fn main() {
    // (a) probes to find position i, with and without knowing the end
    println!(
        "(a) first true at position i: galloping (no end needed) vs binary search over [0, 2^62)"
    );
    println!("{:>22} {:>8} {:>8}", "i", "gallop", "binary");
    let (mut last_win, mut first_loss) = (0, None);
    for i in [
        0usize,
        1,
        10,
        1000,
        1_000_000,
        1_000_000_000,
        1 << 31,
        1 << 40,
        1_000_000_000_000_000,
        (1 << 62) - 1,
    ] {
        let (mut g, mut b) = (0, 0);
        let gi = gallop_search(usize::MAX, |k| {
            g += 1;
            k >= i
        });
        let bi = search(0, 1 << 62, |k| {
            b += 1;
            k >= i
        });
        if gi != i || bi != i {
            println!("  WRONG ANSWER {i} {gi} {bi}");
        }
        println!("{i:>22} {g:>8} {b:>8}");
        if g < b {
            last_win = i;
        } else if first_loss.is_none() {
            first_loss = Some(i);
        }
    }
    println!(
        "  galloping used fewer probes up to i = {last_win} and more from i = {} on: about 2*log2(i) vs log2(2^62) = 62",
        first_loss.map_or("never".to_string(), |i| i.to_string())
    );

    // (b) A (m values) intersect B (n = 1,000,000 values), comparisons counted
    let mut rng = Pcg::new(1, 2);
    const N: usize = 1_000_000;
    let b = sorted_gaps(&mut rng, N, 4); // values up to about 4n
    println!(
        "\n(b) |A| = m, |B| = {N}, both strictly increasing over the same span; element comparisons"
    );
    println!(
        "{:>9} {:>8} {:>10} {:>11} {:>11} {:>10}  fewest",
        "m", "common", "merge", "bin-whole", "bin-suffix", "gallop"
    );
    let names = ["merge", "bin-whole", "bin-suffix", "gallop"];
    let mut timed: Vec<Vec<u64>> = Vec::new(); // A for m = 100 and 100,000
    let mut std_line = String::new();
    for m in [10, 100, 1_000, 10_000, 100_000, 1_000_000] {
        let a = sorted_gaps(&mut rng, m, (4 * N / m) as u64);
        let mut r: [Vec<u64>; 4] = Default::default();
        let counts = [
            merge(&a, &b, &mut r[0]),
            probe_each(&a, &b, &mut r[1], Mode::Whole),
            probe_each(&a, &b, &mut r[2], Mode::Suffix),
            probe_each(&a, &b, &mut r[3], Mode::Gallop),
        ];
        if r[1..].iter().any(|x| *x != r[0]) {
            println!("  RESULTS DIFFER at m = {m}");
        }
        let best = argmin(&counts);
        print!(
            "{m:>9} {:>8} {:>10} {:>11} {:>11} {:>10}  {}",
            r[0].len(),
            counts[0],
            counts[1],
            counts[2],
            counts[3],
            names[best]
        );
        if best != 3 {
            print!(
                " (gallop {:.2}x of it)",
                counts[3] as f64 / counts[best] as f64
            );
        }
        println!();
        if m == 1_000 {
            // The idiomatic suffix search: std's partition_point on the
            // sub-slice &b[start..] -- no index arithmetic, same answers.
            let (mut start, mut calls, mut probes, mut found) = (0, 0u64, 0, Vec::new());
            let mut disagree = 0;
            for &x in &a {
                let (k, c) = lower_bound_in(&b, start, x); // hand-written, for the count
                probes += c;
                start += b[start..].partition_point(|&v| {
                    calls += 1;
                    v < x
                });
                disagree += usize::from(k != start);
                if b.get(start) == Some(&x) {
                    found.push(x);
                    start += 1;
                }
            }
            std_line = format!(
                "  m = 1000, bin-suffix via std partition_point(&b[start..]): {calls} predicate calls vs {probes} hand-written probes\n  (std calls the predicate ceil(log2 len) + 1 times per search); positions that disagree: {disagree}, same result: {}",
                found == r[0]
            );
        }
        if m == 100 || m == 100_000 {
            timed.push(a);
        }
    }
    println!("  all four methods returned identical intersections in every row");
    println!("{std_line}");

    // (c) one timing table. The inputs are only read, so no copies are
    // needed; the output Vec is cleared and reused, and black_box keeps the
    // optimiser from skipping the work.
    println!("\n(c) ns per whole intersection (best of 3)");
    println!(
        "{:>9} {:>12} {:>12} {:>12} {:>12}",
        "m", "merge", "bin-whole", "bin-suffix", "gallop"
    );
    let mut buf = Vec::with_capacity(N);
    for a in &timed {
        let mut run = |mode: Option<Mode>| {
            best_of3(|| {
                buf.clear();
                let (a, b) = (black_box(&a[..]), black_box(&b[..]));
                match mode {
                    None => merge(a, b, &mut buf),
                    Some(md) => probe_each(a, b, &mut buf, md),
                };
                black_box(buf.len());
            })
        };
        let ns = [
            run(None),
            run(Some(Mode::Whole)),
            run(Some(Mode::Suffix)),
            run(Some(Mode::Gallop)),
        ];
        let row: String = ns.iter().map(|t| format!(" {t:>12.0}")).collect();
        println!(
            "{:>9}{row}  fastest: {}, gallop/merge time {:.3}",
            a.len(),
            names[argmin(&ns)],
            ns[3] / ns[0]
        );
        if a.len() == 100_000 {
            // Comparisons are not all the same price: merge's walk is
            // sequential, a search's probes jump around an 8 MB array.
            let cm = merge(a, &b, &mut Vec::new());
            let cg = probe_each(a, &b, &mut Vec::new(), Mode::Gallop);
            println!(
                "  ns per comparison at m = {}: merge {:.2}, gallop {:.2}",
                a.len(),
                ns[0] / cm as f64,
                ns[3] / cg as f64
            );
        }
    }
}
```

**Rust sample output:**

```
(a) first true at position i: galloping (no end needed) vs binary search over [0, 2^62)
                     i   gallop   binary
                     0        1       63
                     1        2       63
                    10        8       62
                  1000       20       62
               1000000       40       62
            1000000000       60       62
            2147483648       64       62
         1099511627776       82       62
      1000000000000000      100       62
   4611686018427387903      124       62
  galloping used fewer probes up to i = 1000000000 and more from i = 2147483648 on: about 2*log2(i) vs log2(2^62) = 62

(b) |A| = m, |B| = 1000000, both strictly increasing over the same span; element comparisons
        m   common      merge   bin-whole  bin-suffix     gallop  fewest
       10        4     955153         208         196        338  bin-suffix (gallop 1.72x of it)
      100       27    1000066        2081        1835       2619  bin-suffix (gallop 1.43x of it)
     1000      255     996183       20955       19585      20958  bin-suffix (gallop 1.07x of it)
    10000     2588    1002451      209486      195826     144555  gallop
   100000    24785    1075023     2094902     1957398     802044  gallop
  1000000   249704    1749778    20951315    19881854    2881633  merge (gallop 1.65x of it)
  all four methods returned identical intersections in every row
  m = 1000, bin-suffix via std partition_point(&b[start..]): 19988 predicate calls vs 18585 hand-written probes
  (std calls the predicate ceil(log2 len) + 1 times per search); positions that disagree: 0, same result: true

(c) ns per whole intersection (best of 3)
        m        merge    bin-whole   bin-suffix       gallop
      100       514041         3089         6258         5627  fastest: bin-whole, gallop/merge time 0.011
   100000      1078450      3201016      4962167      1180047  fastest: merge, gallop/merge time 1.094
  ns per comparison at m = 100000: merge 1.00, gallop 1.47
```

> **Go vs Rust:** Every count in (a) and (b) is identical to Go's, because the PCG port generates the same lists. The extra Rust line shows std `partition_point` on `&b[start..]` giving the same positions with about 7% more predicate calls (19,988 against 18,585), since it always makes ⌈log₂ len⌉ + 1 calls. The timings differ. Across runs, Rust's hand-written searches at m = 100 took about 2–9× as long as Go's, and at m = 100,000 its suffix search took about 1.4–2× as long, while its whole-B search was faster than Go's (about 3.1–3.6 ms against 4.3–4.8 ms). The generated assembly, read but not timed, shows rustc compiling the probe loop to `csel`/`csinc` (branchless) and Go compiling it to a conditional branch. Why that makes Rust slower here was not established. One plausible cause, not measured: the m = 100 loop repeats the same 100 searches, which a branch predictor can learn. On fresh random queries in example 16, the branchless Rust loop was about twice as fast as Go's.

**Complexity:** exponential search is Θ(log i): 20 probes for i = 1000 and 60 for i = 10⁹, against 62–63 for a binary search over 2⁶². It needs 124 at i ≈ 2⁶², so it wins only while i is below about √n. In counted comparisons with n = 10⁶, merge, Θ(m+n), stays near 10⁶ and is fewest at m = n (1.75M, galloping 2.88M). A binary search per element, Θ(m log n), costs about 21 comparisons per element; searching only the rest of B makes it fewest for m ≤ 1,000, where galloping costs 1.07–1.72× as much. Galloping, Θ(m log(n/m)), is fewest at m = 10⁴ and 10⁵ (145K and 802K). In time, merge and galloping were close at m = 100,000 (galloping/merge between about 0.95 and 1.6 across runs). Galloping makes 25% fewer comparisons, but each one costs about 2–2.5 ns against about 1.2 ns for a merge step. The binary search over all of B took about 2.5–4.5× as long as either. At m = 100, merge took several hundred times as long as any search.

---

## 16. Capstone: a timestamp index, built, proved, measured

`🔴 hard` · *Applied mini-build*

This example puts the lesson together in one small, useful type: a read-only index over a sorted
`[]int64` of event timestamps, duplicates allowed. It answers `Count(t1, t2)` for the half-open
`[t1, t2)`, `Range` (a sub-slice, not a copy), `Prev(t)` (the last event ≤ t), `Next(t)` (the first
event ≥ t), and `MinWindow(k)`, the narrowest width w such that some window `[t, t+w)` holds k
events. The first four take one or two bounds, Θ(log n). `MinWindow` is binary search on the
answer: it searches the width range with `FirstTrue`, whose midpoint cannot overflow even on
`[1, MaxInt64]`, and checks each candidate width with the index's own lower bound. Everything runs on
the one half-open template. The proof uses brute-force oracles. The first oracle is exhaustive over
every sorted array of length ≤ 6 over {0..3}, then over arrays built from `MinInt64` and `MaxInt64`
and their neighbours. The second is random: 1,000,000 PCG timestamps over a narrow range, so each
timestamp repeats about five times. A probe-count audit runs on every bound.

**Steps:**

1. Build the `Index` with one `bound(t, strict)` loop. Derive `Count`, `Range`, `Prev` and `Next`
   from it, and write `MinWindow` as `FirstTrue` over the width with a feasibility check.
2. Prove it in three ways: every small array and every query against linear-scan oracles; 200,000
   random queries of each kind against the stdlib bounds; and `MinWindow` against an O(n²)
   pair scan. Look for `0 mismatches` and `0 over ceil(log2(n+1))`.
3. Time `Count` against the same query done with `slices.BinarySearch` and `sort.Search`, and read
   the probes per query.

```go
package main

import (
	"errors"
	"fmt"
	"math"
	"math/bits"
	"math/rand/v2"
	"slices"
	"sort"
	"time"
)

// ======================= (1) BUILD =======================

// Audit collects probe counts from every bound an Index computes, so the
// lesson's promise (at most ceil(log2(n+1)) probes) is checked on EVERY call.
type Audit struct{ calls, probes, maxProbes, violations int }

// Index is a read-only index over sorted event timestamps (duplicates allowed).
type Index struct {
	ts    []int64
	limit int    // bits.Len(n) = ceil(log2(n+1)): the most probes one bound may take
	audit *Audit // nil: no bookkeeping
}

func NewIndex(ts []int64, a *Audit) (*Index, error) {
	if !slices.IsSorted(ts) {
		return nil, errors.New("timestamps must be sorted")
	}
	return &Index{ts: ts, limit: bits.Len(uint(len(ts))), audit: a}, nil
}

// bound is the ONE template: half-open [lo, hi), p(lo-1) false, p(hi) true.
// p(i) is ts[i] >= t (lower bound) or ts[i] > t (upper bound, strict).
func (x *Index) bound(t int64, strict bool) int {
	lo, hi, probes := 0, len(x.ts), 0
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		probes++
		if v := x.ts[mid]; v > t || (v == t && !strict) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	if a := x.audit; a != nil {
		a.calls++
		a.probes += probes
		a.maxProbes = max(a.maxProbes, probes)
		if probes > x.limit {
			a.violations++
		}
	}
	return lo
}

func (x *Index) LowerBound(t int64) int { return x.bound(t, false) } // first ts[i] >= t
func (x *Index) UpperBound(t int64) int { return x.bound(t, true) }  // first ts[i] > t

// Count of events in [t1, t2). Half-open, so adjacent ranges never double
// count -- and no range can include math.MaxInt64 itself (Prev can).
func (x *Index) Count(t1, t2 int64) int {
	if t2 <= t1 {
		return 0
	}
	return x.LowerBound(t2) - x.LowerBound(t1)
}

// Range returns the events in [t1, t2) as a sub-slice: no copy.
func (x *Index) Range(t1, t2 int64) []int64 {
	if t2 <= t1 {
		return nil
	}
	return x.ts[x.LowerBound(t1):x.LowerBound(t2)]
}

// Prev is the last event <= t; Next is the first event >= t.
func (x *Index) Prev(t int64) (int64, bool) {
	if i := x.UpperBound(t); i > 0 {
		return x.ts[i-1], true
	}
	return 0, false
}

func (x *Index) Next(t int64) (int64, bool) {
	if i := x.LowerBound(t); i < len(x.ts) {
		return x.ts[i], true
	}
	return 0, false
}

// FirstTrue: smallest w in [lo, hi] with p(w), given p monotone and p(hi)
// true. The midpoint is computed in uint64: exact for ANY lo <= hi, even
// lo = 1, hi = MaxInt64, where (lo+hi)/2 wraps negative (example 2).
func FirstTrue(lo, hi int64, p func(int64) bool) int64 {
	for lo < hi {
		mid := int64(uint64(lo) + (uint64(hi)-uint64(lo))/2)
		if p(mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

// MinWindow: the smallest w such that some window [t, t+w) holds >= k
// events -- binary search on the answer. feasible(w) is monotone (a wider
// window holds at least as much) and costs one LowerBound per distinct start.
func (x *Index) MinWindow(k int) (int64, bool) {
	n := len(x.ts)
	if k < 1 || k > n {
		return 0, false
	}
	feasible := func(w int64) bool {
		// Only windows that start AT an event matter: slide any other one
		// right until its left edge meets its first event; it loses nothing.
		for i := 0; i+k <= n; i++ {
			if i > 0 && x.ts[i] == x.ts[i-1] {
				continue // the first copy of a timestamp sees at least as much
			}
			end := n // t+w would pass MaxInt64: every later event fits
			if t := x.ts[i]; t <= math.MaxInt64-w {
				end = x.LowerBound(t + w)
			}
			if end-i >= k {
				return true
			}
		}
		return false
	}
	// w = span+1 always works (one window holds everything), if it fits.
	hi := int64(math.MaxInt64)
	if span := uint64(x.ts[n-1]) - uint64(x.ts[0]); span < math.MaxInt64 {
		hi = int64(span) + 1
	} else if !feasible(hi) {
		return 0, false // even the widest int64 window is too narrow
	}
	return FirstTrue(1, hi, feasible), true
}

// ======================= brute-force oracles =======================
// Straight from the definitions, no searching, no shared code.

func bruteCount(ts []int64, t1, t2 int64) int {
	c := 0
	for _, e := range ts {
		if t1 <= e && e < t2 {
			c++
		}
	}
	return c
}

func brutePrev(ts []int64, t int64) (best int64, ok bool) {
	for _, e := range ts {
		if e <= t && (!ok || e > best) {
			best, ok = e, true
		}
	}
	return
}

func bruteNext(ts []int64, t int64) (best int64, ok bool) {
	for _, e := range ts {
		if e >= t && (!ok || e < best) {
			best, ok = e, true
		}
	}
	return
}

// bruteMinWindow tries every pair of events i <= j holding >= k events:
// the window [ts[i], ts[j]] needs width ts[j]-ts[i]+1 (computed in uint64).
func bruteMinWindow(ts []int64, k int) (int64, bool) {
	best, ok := uint64(0), false
	for i := range ts {
		for j := i + k - 1; j < len(ts); j++ {
			if d := uint64(ts[j]) - uint64(ts[i]); d < math.MaxInt64 && (!ok || d+1 < best) {
				best, ok = d+1, true
			}
		}
	}
	return int64(best), ok
}

// rangeOK: Range must return exactly the events in [t1, t2).
func rangeOK(r, ts []int64, t1, t2 int64) bool {
	for _, e := range r {
		if e < t1 || e >= t2 {
			return false
		}
	}
	return len(r) == bruteCount(ts, t1, t2)
}

// checkAll runs every query on one array against the oracles; returns
// (queries, mismatches).
func checkAll(ts, qs []int64, a *Audit) (q, bad int) {
	x, err := NewIndex(ts, a)
	if err != nil {
		return 1, 1
	}
	try := func(ok bool) {
		q++
		if !ok {
			bad++
		}
	}
	for _, t := range qs {
		p, pok := x.Prev(t)
		bp, bpok := brutePrev(ts, t)
		try(p == bp && pok == bpok)
		nx, nok := x.Next(t)
		bn, bnok := bruteNext(ts, t)
		try(nx == bn && nok == bnok)
		for _, t2 := range qs {
			try(x.Count(t, t2) == bruteCount(ts, t, t2))
			try(rangeOK(x.Range(t, t2), ts, t, t2))
		}
	}
	for k := 1; k <= len(ts)+1; k++ {
		w, ok := x.MinWindow(k)
		bw, bok := bruteMinWindow(ts, k)
		try(w == bw && ok == bok)
	}
	return
}

// everySorted calls f on every non-decreasing array over vals, length 0..maxLen.
func everySorted(vals []int64, maxLen int, f func([]int64)) {
	var rec func(cur []int64, from int)
	rec = func(cur []int64, from int) {
		f(slices.Clone(cur))
		if len(cur) == maxLen {
			return
		}
		for i := from; i < len(vals); i++ {
			rec(append(cur, vals[i]), i)
		}
	}
	rec(nil, 0)
}

// bestOf3 returns the fastest of three ~100 ms measurements, in ns per call.
func bestOf3(f func()) float64 {
	best := math.Inf(1)
	for range 3 {
		reps, start := 0, time.Now()
		for time.Since(start) < 100*time.Millisecond {
			f()
			reps++
		}
		best = min(best, float64(time.Since(start).Nanoseconds())/float64(reps))
	}
	return best
}

func show(v int64, ok bool) string {
	if !ok {
		return "none"
	}
	return fmt.Sprint(v)
}

func main() {
	const minI, maxI = math.MinInt64, math.MaxInt64
	fmt.Println("(1) BUILD")
	x, _ := NewIndex([]int64{10, 20, 20, 20, 35, 50, 50, 90}, nil)
	fmt.Println("  ts =", x.ts)
	fmt.Printf("  Count(20, 50) = %d  Range(20, 50) = %v\n", x.Count(20, 50), x.Range(20, 50))
	fmt.Printf("  Prev(5) = %s  Prev(19) = %s  Prev(20) = %s  Next(51) = %s  Next(91) = %s\n",
		show(x.Prev(5)), show(x.Prev(19)), show(x.Prev(20)), show(x.Next(51)), show(x.Next(91)))
	fmt.Print("  MinWindow(k) for k = 1..9:")
	for k := 1; k <= 9; k++ {
		fmt.Print(" ", show(x.MinWindow(k)))
	}
	fmt.Println()
	e, _ := NewIndex([]int64{minI, 0, maxI - 1}, nil)
	lo, hi := int64(1), int64(maxI)
	fmt.Printf("  ts = %v: MinWindow(2) = %s, searched over [1, MaxInt64]\n", e.ts, show(e.MinWindow(2)))
	fmt.Printf("    (there (lo+hi)/2 would wrap to %d on the first step)\n", (lo+hi)/2)
	e, _ = NewIndex([]int64{minI, maxI}, nil)
	fmt.Printf("  ts = %v: MinWindow(2) = %s (it needs width 2^64), Count(MinInt64, MaxInt64) = %d, Prev(MaxInt64) = %s\n",
		e.ts, show(e.MinWindow(2)), e.Count(minI, maxI), show(e.Prev(maxI)))

	fmt.Println("\n(2) PROVE")
	audit := &Audit{}
	for _, suite := range []struct {
		name   string
		vals   []int64
		maxLen int
		qs     []int64
	}{
		{"values {0..3}, length 0..6", []int64{0, 1, 2, 3}, 6, []int64{-1, 0, 1, 2, 3, 4}},
		{"edge values, length 0..5", []int64{minI, minI + 1, -1, 0, 1, maxI - 1, maxI}, 5,
			[]int64{minI, minI + 1, -1, 0, 1, maxI - 1, maxI}},
	} {
		arrays, q, bad := 0, 0, 0
		everySorted(suite.vals, suite.maxLen, func(ts []int64) {
			a, b := checkAll(ts, suite.qs, audit)
			arrays, q, bad = arrays+1, q+a, bad+b
		})
		fmt.Printf("  exhaustive, %s: %d arrays, %d queries, %d mismatches\n", suite.name, arrays, q, bad)
	}

	// Small random arrays with heavy duplication, MinWindow vs the O(n^2) oracle.
	rng := rand.New(rand.NewPCG(1, 2))
	cases, bad := 0, 0
	for range 2000 {
		ts := make([]int64, 1+rng.IntN(40))
		for i := range ts {
			ts[i] = 1_000 + int64(rng.IntN(50))
		}
		slices.Sort(ts)
		x, _ := NewIndex(ts, audit)
		for k := 1; k <= len(ts)+1; k++ {
			w, ok := x.MinWindow(k)
			bw, bok := bruteMinWindow(ts, k)
			cases++
			if w != bw || ok != bok {
				bad++
			}
		}
	}
	fmt.Printf("  random small arrays (1-40 events over 50 values): %d MinWindow queries, %d mismatches\n", cases, bad)

	// The big one: 1,000,000 events over a 200,000 ms range.
	const n, span, Q = 1_000_000, 200_000, 200_000
	const base = int64(1_700_000_000_000) // a Unix time in ms (Nov 2023)
	ts := make([]int64, n)
	for i := range ts {
		ts[i] = base + int64(rng.IntN(span))
	}
	slices.Sort(ts)
	big, _ := NewIndex(ts, audit)
	distinct := len(slices.Compact(slices.Clone(ts)))
	fmt.Printf("  big: %d events, %d distinct timestamps (%.1f events per timestamp)\n", n, distinct, float64(n)/float64(distinct))
	randT := func() int64 { return base - 1_000 + int64(rng.IntN(span+2_000)) } // a bit outside both ends too
	t1s, t2s, ps := make([]int64, Q), make([]int64, Q), make([]int64, Q)
	for i := range Q {
		t1s[i] = randT()
		t2s[i] = t1s[i] + int64(rng.IntN(20_000)) - 1_000 // some empty or reversed ranges
		ps[i] = randT()
	}
	// The stdlib reference: slices.BinarySearch = lower bound (earliest
	// duplicate, documented); sort.Search with > = upper bound.
	lb := func(t int64) int { i, _ := slices.BinarySearch(ts, t); return i }
	ub := func(t int64) int { return sort.Search(n, func(i int) bool { return ts[i] > t }) }
	bad = 0
	for i := range Q {
		t1, t2, t := t1s[i], t2s[i], ps[i]
		want := 0
		if t2 > t1 {
			want = lb(t2) - lb(t1)
		}
		r := big.Range(t1, t2)
		if big.Count(t1, t2) != want || len(r) != want || (want > 0 && &r[0] != &ts[lb(t1)]) {
			bad++
		}
		p, pok := big.Prev(t)
		if j := ub(t); pok != (j > 0) || (pok && p != ts[j-1]) {
			bad++
		}
		nx, nok := big.Next(t)
		if j := lb(t); nok != (j < n) || (nok && nx != ts[j]) {
			bad++
		}
	}
	fmt.Printf("  big vs stdlib bounds: %d queries of each kind (Count, Range, Prev, Next), %d mismatches\n", Q, bad)
	bad = 0
	for i := range 50 {
		t1, t2, t := t1s[i], t2s[i], ps[i]
		p, pok := big.Prev(t)
		bp, bpok := brutePrev(ts, t)
		nx, nok := big.Next(t)
		bn, bnok := bruteNext(ts, t)
		if big.Count(t1, t2) != bruteCount(ts, t1, t2) || !rangeOK(big.Range(t1, t2), ts, t1, t2) ||
			p != bp || pok != bpok || nx != bn || nok != bnok {
			bad++
		}
	}
	fmt.Printf("  big vs brute-force scans: first 50 queries of each kind, %d mismatches\n", bad)
	fmt.Print("  big MinWindow(k) vs the sliding formula min(ts[i+k-1]-ts[i])+1:")
	bad = 0
	for _, k := range []int{1, 20, 1_000, n} {
		w, _ := big.MinWindow(k)
		want := int64(maxI)
		for i := 0; i+k <= n; i++ {
			want = min(want, ts[i+k-1]-ts[i]+1)
		}
		fmt.Printf(" k=%d -> %d", k, w)
		if w != want {
			bad++
		}
	}
	fmt.Printf("; %d mismatches\n", bad)
	fmt.Printf("  invariant: %d bound calls, max %d probes, %d over ceil(log2(n+1)) (= %d for n = %d)\n",
		audit.calls, audit.maxProbes, audit.violations, bits.Len(uint(n)), n)

	fmt.Println("\n(3) MEASURE: Count(t1, t2) over the 200,000 random ranges, ns per query (best of 3)")
	// Queries are only read, never modified, so they need no copying.
	sink := 0
	timeIt := func(count func(t1, t2 int64) int) float64 {
		return bestOf3(func() {
			for i := range Q {
				sink += count(t1s[i], t2s[i])
			}
		}) / Q
	}
	quiet, _ := NewIndex(ts, nil) // same index without the audit bookkeeping
	probesIdx := 0                // probes per Count, read from the audit in an untimed pass
	for i := range Q {
		a0 := audit.probes
		big.Count(t1s[i], t2s[i])
		probesIdx += audit.probes - a0
	}
	calls := 0
	for i := range Q {
		if t1s[i] < t2s[i] {
			sort.Search(n, func(j int) bool { calls++; return ts[j] >= t2s[i] })
			sort.Search(n, func(j int) bool { calls++; return ts[j] >= t1s[i] })
		}
	}
	rows := []struct {
		name   string
		f      func(t1, t2 int64) int
		probes string
	}{
		{"Index.Count, audit on", big.Count, fmt.Sprintf("%.2f", float64(probesIdx)/Q)},
		{"Index.Count, audit off", quiet.Count, "same loop"},
		{"slices.BinarySearch x2", func(t1, t2 int64) int {
			if t2 <= t1 {
				return 0
			}
			return lb(t2) - lb(t1)
		}, "same loop (source)"},
		{"sort.Search x2", func(t1, t2 int64) int {
			if t2 <= t1 {
				return 0
			}
			return sort.Search(n, func(j int) bool { return ts[j] >= t2 }) -
				sort.Search(n, func(j int) bool { return ts[j] >= t1 })
		}, fmt.Sprintf("%.2f", float64(calls)/Q)},
	}
	fmt.Printf("  %-24s %8s  %s\n", "method", "ns/query", "probes/query")
	ns := make([]float64, len(rows))
	for i, r := range rows {
		ns[i] = timeIt(r.f)
		fmt.Printf("  %-24s %8.1f  %s\n", r.name, ns[i], r.probes)
	}
	fmt.Printf("  slowest/fastest row = %.2f; Index (audit off) spends %.1f ns per probe\n",
		slices.Max(ns)/slices.Min(ns), ns[1]/(float64(probesIdx)/Q))
	if sink == 0 {
		fmt.Println("(sink)")
	}
}
```

**Sample output:**

```
(1) BUILD
  ts = [10 20 20 20 35 50 50 90]
  Count(20, 50) = 4  Range(20, 50) = [20 20 20 35]
  Prev(5) = none  Prev(19) = 10  Prev(20) = 20  Next(51) = 90  Next(91) = none
  MinWindow(k) for k = 1..9: 1 1 1 11 26 31 41 81 none
  ts = [-9223372036854775808 0 9223372036854775806]: MinWindow(2) = 9223372036854775807, searched over [1, MaxInt64]
    (there (lo+hi)/2 would wrap to -4611686018427387904 on the first step)
  ts = [-9223372036854775808 9223372036854775807]: MinWindow(2) = none (it needs width 2^64), Count(MinInt64, MaxInt64) = 1, Prev(MaxInt64) = 9223372036854775807

(2) PROVE
  exhaustive, values {0..3}, length 0..6: 210 arrays, 18858 queries, 0 mismatches
  exhaustive, edge values, length 0..5: 792 arrays, 92961 queries, 0 mismatches
  random small arrays (1-40 events over 50 values): 43185 MinWindow queries, 0 mismatches
  big: 1000000 events, 198568 distinct timestamps (5.0 events per timestamp)
  big vs stdlib bounds: 200000 queries of each kind (Count, Range, Prev, Next), 0 mismatches
  big vs brute-force scans: first 50 queries of each kind, 0 mismatches
  big MinWindow(k) vs the sliding formula min(ts[i+k-1]-ts[i])+1: k=1 -> 1 k=20 -> 2 k=1000 -> 176 k=1000000 -> 200000; 0 mismatches
  invariant: 3707794 bound calls, max 20 probes, 0 over ceil(log2(n+1)) (= 20 for n = 1000000)

(3) MEASURE: Count(t1, t2) over the 200,000 random ranges, ns per query (best of 3)
  method                   ns/query  probes/query
  Index.Count, audit on       189.2  37.85
  Index.Count, audit off      189.0  same loop
  slices.BinarySearch x2      200.7  same loop (source)
  sort.Search x2              182.0  37.85
  slowest/fastest row = 1.10; Index (audit off) spends 5.0 ns per probe
```

**Rust:**

```rust
use std::cell::Cell;
use std::hint::black_box;
use std::time::{Duration, Instant};

// ======================= (1) BUILD =======================

/// Probe counts from every bound an Index computes, so the promise (at most
/// ceil(log2(n+1)) probes) is checked on EVERY call. `Cell` lets a `&self`
/// method update them: Go mutates through a pointer, Rust needs interior
/// mutability to do that through a shared reference.
#[derive(Default)]
struct Audit {
    calls: Cell<usize>,
    probes: Cell<usize>,
    max_probes: Cell<u32>,
    violations: Cell<usize>,
}

/// A read-only index over sorted event timestamps (duplicates allowed).
/// It BORROWS the slice ('a): the borrow checker guarantees nobody can sort
/// or modify the timestamps while the index exists.
struct Index<'a> {
    ts: &'a [i64],
    limit: u32, // bits of n = ceil(log2(n+1)): the most probes one bound may take
    audit: Option<&'a Audit>,
}

impl<'a> Index<'a> {
    fn new(ts: &'a [i64], audit: Option<&'a Audit>) -> Result<Self, &'static str> {
        if !ts.is_sorted() {
            return Err("timestamps must be sorted");
        }
        let limit = usize::BITS - ts.len().leading_zeros(); // Go's bits.Len
        Ok(Index { ts, limit, audit })
    }

    /// The ONE template: half-open [lo, hi), p(lo-1) false, p(hi) true.
    /// p(i) is ts[i] >= t (lower bound) or ts[i] > t (upper bound, strict).
    fn bound(&self, t: i64, strict: bool) -> usize {
        let (mut lo, mut hi, mut probes) = (0, self.ts.len(), 0);
        while lo < hi {
            let mid = lo.midpoint(hi);
            probes += 1;
            let v = self.ts[mid];
            if v > t || (v == t && !strict) {
                hi = mid;
            } else {
                lo = mid + 1;
            }
        }
        if let Some(a) = self.audit {
            a.calls.set(a.calls.get() + 1);
            a.probes.set(a.probes.get() + probes as usize);
            a.max_probes.set(a.max_probes.get().max(probes));
            if probes > self.limit {
                a.violations.set(a.violations.get() + 1);
            }
        }
        lo
    }

    fn lower_bound(&self, t: i64) -> usize {
        self.bound(t, false) // first ts[i] >= t
    }
    fn upper_bound(&self, t: i64) -> usize {
        self.bound(t, true) // first ts[i] > t
    }

    /// Events in [t1, t2). Half-open: adjacent ranges never double count --
    /// and no range can include i64::MAX itself (prev can).
    fn count(&self, t1: i64, t2: i64) -> usize {
        if t2 <= t1 {
            return 0;
        }
        self.lower_bound(t2) - self.lower_bound(t1)
    }

    /// The events in [t1, t2) as a sub-slice of the borrowed data: no copy.
    fn range(&self, t1: i64, t2: i64) -> &'a [i64] {
        if t2 <= t1 {
            return &[];
        }
        &self.ts[self.lower_bound(t1)..self.lower_bound(t2)]
    }

    /// Last event <= t, first event >= t: Option instead of Go's (v, ok).
    fn prev(&self, t: i64) -> Option<i64> {
        self.upper_bound(t).checked_sub(1).map(|i| self.ts[i])
    }
    fn next(&self, t: i64) -> Option<i64> {
        self.ts.get(self.lower_bound(t)).copied()
    }

    /// Smallest w such that some window [t, t+w) holds >= k events: binary
    /// search on the answer. feasible(w) is monotone and costs one
    /// lower_bound per distinct start.
    fn min_window(&self, k: usize) -> Option<i64> {
        let (ts, n) = (self.ts, self.ts.len());
        if k < 1 || k > n {
            return None;
        }
        let feasible = |w: i64| {
            // Only windows that start AT an event matter (slide any other one
            // right until it meets its first event: it loses nothing).
            (0..=n - k).any(|i| {
                if i > 0 && ts[i] == ts[i - 1] {
                    return false; // the first copy of a timestamp sees at least as much
                }
                // checked_add: None when t+w passes i64::MAX -- then every
                // later event fits (Go writes the test t <= MaxInt64-w)
                let end = ts[i].checked_add(w).map_or(n, |e| self.lower_bound(e));
                end - i >= k
            })
        };
        // w = span+1 always works (one window holds everything), if it fits.
        let span = ts[n - 1].abs_diff(ts[0]); // u64: exact even for MIN..MAX
        let hi = match i64::try_from(span) {
            Ok(s) if s < i64::MAX => s + 1,
            _ if feasible(i64::MAX) => i64::MAX,
            _ => return None, // even the widest i64 window is too narrow
        };
        Some(first_true(1, hi, feasible))
    }
}

/// Smallest w in [lo, hi] with p(w), given p monotone and p(hi) true.
/// The midpoint is lo + (hi-lo)/2 computed through u64 (`abs_diff`), the
/// FLOOR for any lo <= hi, even lo = 1, hi = i64::MAX. `i64::midpoint` is
/// not used: it rounds toward ZERO (read in the source, printed below), so
/// on a negative range mid could equal hi and the loop would never end.
fn first_true(mut lo: i64, mut hi: i64, p: impl Fn(i64) -> bool) -> i64 {
    while lo < hi {
        let mid = lo.wrapping_add_unsigned(hi.abs_diff(lo) / 2); // never wraps: lo <= mid < hi
        if p(mid) {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    lo
}

// ======================= brute-force oracles =======================
// Straight from the definitions, no searching, no shared code.

fn brute_count(ts: &[i64], t1: i64, t2: i64) -> usize {
    ts.iter().filter(|&&e| t1 <= e && e < t2).count()
}
fn brute_prev(ts: &[i64], t: i64) -> Option<i64> {
    ts.iter().copied().filter(|&e| e <= t).max()
}
fn brute_next(ts: &[i64], t: i64) -> Option<i64> {
    ts.iter().copied().filter(|&e| e >= t).min()
}

/// Every pair of events i <= j holding >= k events: the window [ts[i], ts[j]]
/// needs width ts[j]-ts[i]+1 (computed in u64; None if it exceeds i64::MAX).
fn brute_min_window(ts: &[i64], k: usize) -> Option<i64> {
    let mut best: Option<u64> = None;
    for i in 0..ts.len() {
        for j in (i + k).saturating_sub(1)..ts.len() {
            let d = ts[j].abs_diff(ts[i]);
            if d < i64::MAX as u64 && best.is_none_or(|b| d + 1 < b) {
                best = Some(d + 1);
            }
        }
    }
    best.map(|b| b as i64)
}

/// range must return exactly the events in [t1, t2).
fn range_ok(r: &[i64], ts: &[i64], t1: i64, t2: i64) -> bool {
    r.iter().all(|&e| t1 <= e && e < t2) && r.len() == brute_count(ts, t1, t2)
}

/// Every query on one array against the oracles: (queries, mismatches).
fn check_all(ts: &[i64], qs: &[i64], a: &Audit) -> (usize, usize) {
    let Ok(x) = Index::new(ts, Some(a)) else {
        return (1, 1);
    };
    let (mut q, mut bad) = (0, 0);
    let mut try_ = |ok: bool| {
        q += 1;
        bad += usize::from(!ok);
    };
    for &t in qs {
        try_(x.prev(t) == brute_prev(ts, t));
        try_(x.next(t) == brute_next(ts, t));
        for &t2 in qs {
            try_(x.count(t, t2) == brute_count(ts, t, t2));
            try_(range_ok(x.range(t, t2), ts, t, t2));
        }
    }
    for k in 1..=ts.len() + 1 {
        try_(x.min_window(k) == brute_min_window(ts, k));
    }
    (q, bad)
}

/// Calls f on every non-decreasing array over vals, length 0..=max_len.
fn every_sorted(vals: &[i64], max_len: usize, f: &mut impl FnMut(&[i64])) {
    fn rec(
        vals: &[i64],
        max_len: usize,
        cur: &mut Vec<i64>,
        from: usize,
        f: &mut impl FnMut(&[i64]),
    ) {
        f(cur);
        if cur.len() == max_len {
            return;
        }
        for i in from..vals.len() {
            cur.push(vals[i]);
            rec(vals, max_len, cur, i, f);
            cur.pop();
        }
    }
    rec(vals, max_len, &mut Vec::new(), 0, f);
}

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) gives exactly the timestamps Go generates.
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

/// The fastest of three ~100 ms measurements, in ns per call of f.
fn best_of3(mut f: impl FnMut()) -> f64 {
    (0..3)
        .map(|_| {
            let (mut reps, start) = (0u32, Instant::now());
            while start.elapsed() < Duration::from_millis(100) {
                f();
                reps += 1;
            }
            start.elapsed().as_nanos() as f64 / f64::from(reps)
        })
        .fold(f64::INFINITY, f64::min)
}

/// A Count implementation, as a trait object so the rows can share one table.
type CountFn<'a> = dyn Fn(i64, i64) -> usize + 'a;

fn show(v: Option<i64>) -> String {
    v.map_or("none".to_string(), |v| v.to_string())
}
/// Go's %v of a slice: [1 2 3]
fn go_slice(s: &[i64]) -> String {
    let parts: Vec<String> = s.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    let (min_i, max_i) = (i64::MIN, i64::MAX);
    println!("(1) BUILD");
    let data = [10, 20, 20, 20, 35, 50, 50, 90];
    let x = Index::new(&data, None).expect("sorted");
    println!("  ts = {}", go_slice(x.ts));
    println!(
        "  Count(20, 50) = {}  Range(20, 50) = {}",
        x.count(20, 50),
        go_slice(x.range(20, 50))
    );
    println!(
        "  Prev(5) = {}  Prev(19) = {}  Prev(20) = {}  Next(51) = {}  Next(91) = {}",
        show(x.prev(5)),
        show(x.prev(19)),
        show(x.prev(20)),
        show(x.next(51)),
        show(x.next(91))
    );
    let mw: Vec<String> = (1..=9).map(|k| show(x.min_window(k))).collect();
    println!("  MinWindow(k) for k = 1..9: {}", mw.join(" "));
    let edge = [min_i, 0, max_i - 1];
    let e = Index::new(&edge, None).expect("sorted");
    println!(
        "  ts = {}: MinWindow(2) = {}, searched over [1, MaxInt64]",
        go_slice(e.ts),
        show(e.min_window(2))
    );
    let (lo, hi) = (1i64, max_i);
    println!(
        "    (there (lo+hi)/2 would wrap to {} on the first step: what `+` does under -O)",
        lo.wrapping_add(hi) / 2
    );
    println!(
        "    i64::midpoint(-3, -2) = {}: rounds toward zero, mid == hi, so first_true uses the floor",
        (-3i64).midpoint(-2)
    );
    let edge2 = [min_i, max_i];
    let e = Index::new(&edge2, None).expect("sorted");
    println!(
        "  ts = {}: MinWindow(2) = {} (it needs width 2^64), Count(MinInt64, MaxInt64) = {}, Prev(MaxInt64) = {}",
        go_slice(e.ts),
        show(e.min_window(2)),
        e.count(min_i, max_i),
        show(e.prev(max_i))
    );

    println!("\n(2) PROVE");
    let audit = Audit::default();
    let edges = [min_i, min_i + 1, -1, 0, 1, max_i - 1, max_i];
    let suites: [(&str, &[i64], usize, &[i64]); 2] = [
        (
            "values {0..3}, length 0..6",
            &[0, 1, 2, 3],
            6,
            &[-1, 0, 1, 2, 3, 4],
        ),
        ("edge values, length 0..5", &edges, 5, &edges),
    ];
    for (name, vals, max_len, qs) in suites {
        let (mut arrays, mut q, mut bad) = (0, 0, 0);
        every_sorted(vals, max_len, &mut |ts| {
            let (a, b) = check_all(ts, qs, &audit);
            (arrays, q, bad) = (arrays + 1, q + a, bad + b);
        });
        println!("  exhaustive, {name}: {arrays} arrays, {q} queries, {bad} mismatches");
    }

    // Small random arrays with heavy duplication, min_window vs the O(n^2) oracle.
    let mut rng = Pcg::new(1, 2);
    let (mut cases, mut bad) = (0, 0);
    for _ in 0..2000 {
        let len = 1 + rng.int_n(40) as usize;
        let mut ts: Vec<i64> = (0..len).map(|_| 1_000 + rng.int_n(50) as i64).collect();
        ts.sort_unstable();
        let x = Index::new(&ts, Some(&audit)).expect("sorted");
        for k in 1..=ts.len() + 1 {
            cases += 1;
            bad += usize::from(x.min_window(k) != brute_min_window(&ts, k));
        }
    }
    println!(
        "  random small arrays (1-40 events over 50 values): {cases} MinWindow queries, {bad} mismatches"
    );

    // The big one: 1,000,000 events over a 200,000 ms range.
    const N: usize = 1_000_000;
    const SPAN: u64 = 200_000;
    const Q: usize = 200_000;
    const BASE: i64 = 1_700_000_000_000; // a Unix time in ms (Nov 2023)
    let mut ts: Vec<i64> = (0..N).map(|_| BASE + rng.int_n(SPAN) as i64).collect();
    ts.sort_unstable();
    let ts = ts; // frozen: the index borrows it from here on
    let big = Index::new(&ts, Some(&audit)).expect("sorted");
    let mut uniq = ts.clone();
    uniq.dedup();
    println!(
        "  big: {N} events, {} distinct timestamps ({:.1} events per timestamp)",
        uniq.len(),
        N as f64 / uniq.len() as f64
    );
    drop(uniq);
    // a bit outside both ends too (a plain fn of the rng: a closure that
    // captured `rng` mutably would lock it away from the t2 line below)
    let rand_t = |rng: &mut Pcg| BASE - 1_000 + rng.int_n(SPAN + 2_000) as i64;
    let (mut t1s, mut t2s, mut ps) = (vec![0; Q], vec![0; Q], vec![0; Q]);
    for i in 0..Q {
        t1s[i] = rand_t(&mut rng);
        t2s[i] = t1s[i] + rng.int_n(20_000) as i64 - 1_000; // some empty or reversed ranges
        ps[i] = rand_t(&mut rng);
    }
    // The std reference: partition_point is the guaranteed bound (< for the
    // lower, <= for the upper). binary_search is NOT: it returns any match.
    let lb = |t: i64| ts.partition_point(|&v| v < t);
    let ub = |t: i64| ts.partition_point(|&v| v <= t);
    let (mut bad, mut bs_wrong) = (0, 0);
    for i in 0..Q {
        let (t1, t2, t) = (t1s[i], t2s[i], ps[i]);
        let want = if t2 > t1 { lb(t2) - lb(t1) } else { 0 };
        let r = big.range(t1, t2);
        // same window of the same array: compare the start pointers
        if big.count(t1, t2) != want
            || r.len() != want
            || (want > 0 && !std::ptr::eq(r.as_ptr(), &ts[lb(t1)]))
        {
            bad += 1;
        }
        bad += usize::from(big.prev(t) != ub(t).checked_sub(1).map(|j| ts[j]));
        bad += usize::from(big.next(t) != ts.get(lb(t)).copied());
        // the tempting shortcut: binary_search's Ok(i) | Err(i) as a lower bound
        if t2 > t1 {
            let bs = |t: i64| match ts.binary_search(&t) {
                Ok(i) | Err(i) => i,
            };
            bs_wrong += usize::from(bs(t2) - bs(t1) != want);
        }
    }
    println!(
        "  big vs std partition_point bounds: {Q} queries of each kind (Count, Range, Prev, Next), {bad} mismatches"
    );
    println!(
        "  (Count via binary_search's Ok(i) | Err(i) instead: {bs_wrong} of {} non-empty Counts wrong -- it returns some duplicate, not the first)",
        (0..Q).filter(|&i| t2s[i] > t1s[i]).count()
    );
    let bad = (0..50)
        .filter(|&i| {
            let (t1, t2, t) = (t1s[i], t2s[i], ps[i]);
            big.count(t1, t2) != brute_count(&ts, t1, t2)
                || !range_ok(big.range(t1, t2), &ts, t1, t2)
                || big.prev(t) != brute_prev(&ts, t)
                || big.next(t) != brute_next(&ts, t)
        })
        .count();
    println!("  big vs brute-force scans: first 50 queries of each kind, {bad} mismatches");
    print!("  big MinWindow(k) vs the sliding formula min(ts[i+k-1]-ts[i])+1:");
    let mut bad = 0;
    for k in [1, 20, 1_000, N] {
        let w = big.min_window(k);
        // windows(k): every run of k consecutive events, as a slice
        let want = ts.windows(k).map(|win| win[k - 1] - win[0] + 1).min();
        print!(" k={k} -> {}", show(w));
        bad += usize::from(w != want);
    }
    println!("; {bad} mismatches");
    println!(
        "  invariant: {} bound calls, max {} probes, {} over ceil(log2(n+1)) (= {} for n = {N})",
        audit.calls.get(),
        audit.max_probes.get(),
        audit.violations.get(),
        big.limit
    );

    println!(
        "\n(3) MEASURE: Count(t1, t2) over the 200,000 random ranges, ns per query (best of 3)"
    );
    // Queries are only read, never modified, so they need no copying.
    let time_it = |count: &CountFn| {
        best_of3(|| {
            for i in 0..Q {
                black_box(count(black_box(t1s[i]), black_box(t2s[i])));
            }
        }) / Q as f64
    };
    let quiet = Index::new(&ts, None).expect("sorted"); // no audit bookkeeping
    // probes per Count, read from the audit in an untimed pass
    let p0 = audit.probes.get();
    for i in 0..Q {
        big.count(t1s[i], t2s[i]);
    }
    let probes_idx = (audit.probes.get() - p0) as f64 / Q as f64;
    let calls = Cell::new(0usize);
    let pp_count = |t1: i64, t2: i64| {
        if t2 <= t1 {
            return 0;
        }
        let pp = |t: i64| {
            ts.partition_point(|&v| {
                calls.set(calls.get() + 1);
                v < t
            })
        };
        pp(t2) - pp(t1)
    };
    for i in 0..Q {
        pp_count(t1s[i], t2s[i]);
    }
    let probes_pp = calls.get() as f64 / Q as f64;
    let plain_pp = |t1: i64, t2: i64| {
        if t2 <= t1 {
            return 0;
        }
        ts.partition_point(|&v| v < t2) - ts.partition_point(|&v| v < t1)
    };
    let rows: [(&str, &CountFn, String); 3] = [
        (
            "Index.count, audit on",
            &|a, b| big.count(a, b),
            format!("{probes_idx:.2}"),
        ),
        (
            "Index.count, audit off",
            &|a, b| quiet.count(a, b),
            "same loop".to_string(),
        ),
        ("partition_point x2", &plain_pp, format!("{probes_pp:.2}")),
    ];
    println!("  {:<24} {:>8}  probes/query", "method", "ns/query");
    let mut ns = Vec::new();
    for (name, f, probes) in rows {
        let t = time_it(f);
        println!("  {name:<24} {t:>8.1}  {probes}");
        ns.push(t);
    }
    let (lo, hi) = ns
        .iter()
        .fold((f64::INFINITY, 0f64), |(l, h), &t| (l.min(t), h.max(t)));
    println!(
        "  slowest/fastest row = {:.2}; Index (audit off) spends {:.1} ns per probe",
        hi / lo,
        ns[1] / probes_idx
    );
}
```

**Rust sample output:**

```
(1) BUILD
  ts = [10 20 20 20 35 50 50 90]
  Count(20, 50) = 4  Range(20, 50) = [20 20 20 35]
  Prev(5) = none  Prev(19) = 10  Prev(20) = 20  Next(51) = 90  Next(91) = none
  MinWindow(k) for k = 1..9: 1 1 1 11 26 31 41 81 none
  ts = [-9223372036854775808 0 9223372036854775806]: MinWindow(2) = 9223372036854775807, searched over [1, MaxInt64]
    (there (lo+hi)/2 would wrap to -4611686018427387904 on the first step: what `+` does under -O)
    i64::midpoint(-3, -2) = -2: rounds toward zero, mid == hi, so first_true uses the floor
  ts = [-9223372036854775808 9223372036854775807]: MinWindow(2) = none (it needs width 2^64), Count(MinInt64, MaxInt64) = 1, Prev(MaxInt64) = 9223372036854775807

(2) PROVE
  exhaustive, values {0..3}, length 0..6: 210 arrays, 18858 queries, 0 mismatches
  exhaustive, edge values, length 0..5: 792 arrays, 92961 queries, 0 mismatches
  random small arrays (1-40 events over 50 values): 43185 MinWindow queries, 0 mismatches
  big: 1000000 events, 198568 distinct timestamps (5.0 events per timestamp)
  big vs std partition_point bounds: 200000 queries of each kind (Count, Range, Prev, Next), 0 mismatches
  (Count via binary_search's Ok(i) | Err(i) instead: 165974 of 189953 non-empty Counts wrong -- it returns some duplicate, not the first)
  big vs brute-force scans: first 50 queries of each kind, 0 mismatches
  big MinWindow(k) vs the sliding formula min(ts[i+k-1]-ts[i])+1: k=1 -> 1 k=20 -> 2 k=1000 -> 176 k=1000000 -> 200000; 0 mismatches
  invariant: 3707794 bound calls, max 20 probes, 0 over ceil(log2(n+1)) (= 20 for n = 1000000)

(3) MEASURE: Count(t1, t2) over the 200,000 random ranges, ns per query (best of 3)
  method                   ns/query  probes/query
  Index.count, audit on        79.4  37.85
  Index.count, audit off       81.0  same loop
  partition_point x2           44.2  39.89
  slowest/fastest row = 1.83; Index (audit off) spends 2.1 ns per probe
```

> **Go vs Rust:** Every count and mismatch line is identical to Go's, including the PCG data and all 3,707,794 audited bound calls. Two lines are Rust-only. `i64::midpoint` rounds toward zero, so `first_true` takes the floor through `abs_diff`. And Count built on `binary_search`'s `Ok(i) | Err(i)` is wrong for 165,974 of 189,953 ranges, because it returns some duplicate rather than the first; `partition_point` is the correct reference. The Rust `Index` borrows the slice (`Index<'a>`), so the timestamps cannot change while it exists, and `Prev`/`Next` return `Option`. In time, across runs, Rust's hand-written `count` took about 80–100 ns against Go's 180–230 ns. std `partition_point` was fastest at about 43–53 ns, about 1.8–2× faster than the hand-written loop, even though it makes more predicate calls (39.89 against 37.85). The hand-written loop compiles to `csel` (assembly read, not timed). std's loop runs a fixed number of iterations and uses `select_unpredictable` (source read). The cause of the gap was not measured.

**Complexity:** `Count`, `Range`, `Prev` and `Next` cost Θ(log n), one or two bounds each. `MinWindow` is Θ(d · log n · log R) for d distinct timestamps and value range R: about log₂ R feasibility checks, each one bound per distinct start. A two-pointer check (lesson 16) would make it Θ(n log R). The proof found 0 mismatches over 18,858 exhaustive queries on {0..3}, 92,961 on the edge values (including widths that do not fit in `int64`), 43,185 random `MinWindow` queries against the O(n²) oracle, and 200,000 random queries of each kind against the stdlib bounds. Across 3,707,794 bound calls the audit saw at most 20 probes, which is ⌈log₂(n+1)⌉ for n = 10⁶, and never more. On timing, all four ways of answering `Count` took about 180–230 ns per query (37.85 probes, about 5 ns per probe), and the slowest row was at most about 1.2× the fastest across runs. `slices.BinarySearch` was no faster than the hand-written index, and the audit bookkeeping cost nothing measurable.

---

> Next lesson: [16 — Two Pointers, Sliding Window & Prefix Sums](../../16-two-pointers-windows.md) —
> the three techniques that turn O(n²) into O(n) without sorting first.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
