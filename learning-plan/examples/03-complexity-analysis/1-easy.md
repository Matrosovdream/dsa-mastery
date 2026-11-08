# Step 03 — Complexity Analysis · 🟢 Easy

Examples **1–6**: the notation, the cases, and the ratio test.

Every example here **counts operations** instead of timing them. Counts are exact, identical on every
machine, and immune to cache effects — which is why this tier's output is fully **deterministic** and
yours should match character for character.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🟡 medium](2-medium.md)

---

## 1. Three bounds on one function

`🟢 easy` · *O vs Ω vs Θ*

Linear search's worst case costs **exactly n** comparisons — not "about n". With an exact cost in
hand, the three symbols stop being interchangeable jargon and become three checkable claims. The one
that surprises people: `O(n²)` is a **true** statement about this function.

**Steps:**

1. Confirm the worst case is exactly n by dividing the count by n.
2. Work through each claim against the definition (`O` = at most, `Ω` = at least, `Θ` = both).
3. Notice which two claims are true but useless.

```go
package main

import "fmt"

// search does at most one comparison per element. The exact worst-case cost is
// T(n) = n comparisons -- not "about n", exactly n.
func search(xs []int, target int) (idx, comparisons int) {
	for i, v := range xs {
		comparisons++
		if v == target {
			return i, comparisons
		}
	}
	return -1, comparisons
}

func main() {
	fmt.Printf("%8s %14s %14s\n", "n", "worst T(n)", "T(n)/n")
	for _, n := range []int{10, 100, 1000, 10000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i
		}
		_, worst := search(xs, -1) // absent: every element compared
		fmt.Printf("%8d %14d %14.1f\n", n, worst, float64(worst)/float64(n))
	}

	fmt.Println()
	fmt.Println("worst-case T(n) = n exactly. Which of these claims are TRUE?")
	fmt.Println()

	claims := []struct {
		claim string
		true_ bool
		why   string
	}{
		{"T(n) = O(n)", true, "n <= 1*n for all n>=1 -- an upper bound, and a tight one"},
		{"T(n) = O(n^2)", true, "n <= 1*n^2 for all n>=1 -- ALSO a valid upper bound, just useless"},
		{"T(n) = O(1)", false, "no constant c has n <= c for all n"},
		{"T(n) = Omega(n)", true, "n >= 1*n -- a lower bound"},
		{"T(n) = Omega(1)", true, "n >= 1 -- also valid, also useless"},
		{"T(n) = Theta(n)", true, "O(n) AND Omega(n) both hold -- the bound is tight"},
		{"T(n) = Theta(n^2)", false, "O(n^2) holds but Omega(n^2) does not"},
	}
	for _, c := range claims {
		mark := "false"
		if c.true_ {
			mark = "TRUE "
		}
		fmt.Printf("  %-18s %s  %s\n", c.claim, mark, c.why)
	}

	fmt.Println()
	fmt.Println("O is an upper bound, Omega a lower bound, Theta both at once.")
	fmt.Println("Saying O(n^2) is not WRONG here -- it is just weak. Say Theta when you mean tight.")
}
```

**Output:**

```
       n     worst T(n)         T(n)/n
      10             10            1.0
     100            100            1.0
    1000           1000            1.0
   10000          10000            1.0

worst-case T(n) = n exactly. Which of these claims are TRUE?

  T(n) = O(n)        TRUE   n <= 1*n for all n>=1 -- an upper bound, and a tight one
  T(n) = O(n^2)      TRUE   n <= 1*n^2 for all n>=1 -- ALSO a valid upper bound, just useless
  T(n) = O(1)        false  no constant c has n <= c for all n
  T(n) = Omega(n)    TRUE   n >= 1*n -- a lower bound
  T(n) = Omega(1)    TRUE   n >= 1 -- also valid, also useless
  T(n) = Theta(n)    TRUE   O(n) AND Omega(n) both hold -- the bound is tight
  T(n) = Theta(n^2)  false  O(n^2) holds but Omega(n^2) does not

O is an upper bound, Omega a lower bound, Theta both at once.
Saying O(n^2) is not WRONG here -- it is just weak. Say Theta when you mean tight.
```

**Rust:**

```rust
// search does at most one comparison per element. The exact worst-case cost is
// T(n) = n comparisons -- not "about n", exactly n.
// Rust returns Option<usize> for the index where Go returns -1.
fn search(xs: &[i64], target: i64) -> (Option<usize>, u64) {
    let mut comparisons = 0;
    for (i, &v) in xs.iter().enumerate() {
        comparisons += 1;
        if v == target {
            return (Some(i), comparisons);
        }
    }
    (None, comparisons)
}

struct Claim {
    claim: &'static str,
    holds: bool,
    why: &'static str,
}

fn main() {
    println!("{:>8} {:>14} {:>14}", "n", "worst T(n)", "T(n)/n");
    for n in [10, 100, 1000, 10000] {
        let xs: Vec<i64> = (0..n).collect();
        let (_, worst) = search(&xs, -1); // absent: every element compared
        println!("{n:>8} {worst:>14} {:>14.1}", worst as f64 / n as f64);
    }

    println!();
    println!("worst-case T(n) = n exactly. Which of these claims are TRUE?");
    println!();

    let claims = [
        Claim {
            claim: "T(n) = O(n)",
            holds: true,
            why: "n <= 1*n for all n>=1 -- an upper bound, and a tight one",
        },
        Claim {
            claim: "T(n) = O(n^2)",
            holds: true,
            why: "n <= 1*n^2 for all n>=1 -- ALSO a valid upper bound, just useless",
        },
        Claim {
            claim: "T(n) = O(1)",
            holds: false,
            why: "no constant c has n <= c for all n",
        },
        Claim {
            claim: "T(n) = Omega(n)",
            holds: true,
            why: "n >= 1*n -- a lower bound",
        },
        Claim {
            claim: "T(n) = Omega(1)",
            holds: true,
            why: "n >= 1 -- also valid, also useless",
        },
        Claim {
            claim: "T(n) = Theta(n)",
            holds: true,
            why: "O(n) AND Omega(n) both hold -- the bound is tight",
        },
        Claim {
            claim: "T(n) = Theta(n^2)",
            holds: false,
            why: "O(n^2) holds but Omega(n^2) does not",
        },
    ];
    for c in &claims {
        let mark = if c.holds { "TRUE " } else { "false" };
        println!("  {:<18} {}  {}", c.claim, mark, c.why);
    }

    println!();
    println!("O is an upper bound, Omega a lower bound, Theta both at once.");
    println!("Saying O(n^2) is not WRONG here -- it is just weak. Say Theta when you mean tight.");
}
```

**Rust output:**

```
       n     worst T(n)         T(n)/n
      10             10            1.0
     100            100            1.0
    1000           1000            1.0
   10000          10000            1.0

worst-case T(n) = n exactly. Which of these claims are TRUE?

  T(n) = O(n)        TRUE   n <= 1*n for all n>=1 -- an upper bound, and a tight one
  T(n) = O(n^2)      TRUE   n <= 1*n^2 for all n>=1 -- ALSO a valid upper bound, just useless
  T(n) = O(1)        false  no constant c has n <= c for all n
  T(n) = Omega(n)    TRUE   n >= 1*n -- a lower bound
  T(n) = Omega(1)    TRUE   n >= 1 -- also valid, also useless
  T(n) = Theta(n)    TRUE   O(n) AND Omega(n) both hold -- the bound is tight
  T(n) = Theta(n^2)  false  O(n^2) holds but Omega(n^2) does not

O is an upper bound, Omega a lower bound, Theta both at once.
Saying O(n^2) is not WRONG here -- it is just weak. Say Theta when you mean tight.
```

> **Go vs Rust:** Rust returns `Option<usize>` where Go returns `-1`, and the claims table is an array of a small `struct` with a `holds: bool` field. The counts match Go's exactly: worst-case comparisons are n, ratio 1.0 on every row.

**Complexity:** the function is Θ(n) worst case · the *lesson* is that O(n), O(n²), Ω(n) and Ω(1) are all simultaneously true, and only Θ(n) pins it down

---

## 2. The case and the bound are different axes

`🟢 easy` · *Best / average / worst*

The most common confusion in complexity: people treat "worst case" and "Big-O" as the same idea. They
are perpendicular. Every case has its own O, Ω and Θ. Here the exact average is derived
(`(n+1)/2`) and then confirmed empirically with 100,000 random lookups.

**Steps:**

1. Measure the best case (target first) and the worst (target absent).
2. Compute the true average by searching for *every* element once.
3. Confirm it against random targets, and check both against the closed form.

```go
package main

import (
	"fmt"
	"math/rand/v2"
)

func search(xs []int, target int) int {
	comparisons := 0
	for _, v := range xs {
		comparisons++
		if v == target {
			return comparisons
		}
	}
	return comparisons
}

func main() {
	const n = 1000
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i
	}

	best := search(xs, xs[0])
	worst := search(xs, -1)

	// Average over every possible successful target, each equally likely.
	total := 0
	for _, v := range xs {
		total += search(xs, v)
	}
	average := float64(total) / float64(n)

	// And an empirical average with uniformly random targets, to confirm.
	rng := rand.New(rand.NewPCG(1, 2))
	sum := 0
	const trials = 100000
	for i := 0; i < trials; i++ {
		sum += search(xs, xs[rng.IntN(n)])
	}
	empirical := float64(sum) / float64(trials)

	fmt.Printf("linear search over n = %d\n\n", n)
	fmt.Printf("  best case      %8d comparisons   (target is first)\n", best)
	fmt.Printf("  average case   %8.1f comparisons   (exact: (n+1)/2 = %.1f)\n", average, float64(n+1)/2)
	fmt.Printf("  empirical avg  %8.1f comparisons   (%d random targets)\n", empirical, trials)
	fmt.Printf("  worst case     %8d comparisons   (target is absent)\n", worst)

	fmt.Println()
	fmt.Println("the CASE and the BOUND are two different axes -- all four of these are true:")
	fmt.Println("  best case    is Theta(1)")
	fmt.Println("  average case is Theta(n)   -- (n+1)/2 is still linear")
	fmt.Println("  worst case   is Theta(n)")
	fmt.Println("  worst case   is O(n^2)     -- true, weak, and the reason to prefer Theta")

	fmt.Println()
	fmt.Println("\"O(n) algorithm\" almost always means \"worst case is Theta(n)\".")
	fmt.Println("When someone says a hash map is O(1), they mean AVERAGE case -- worst is O(n).")
}
```

**Output:**

```
linear search over n = 1000

  best case             1 comparisons   (target is first)
  average case      500.5 comparisons   (exact: (n+1)/2 = 500.5)
  empirical avg     500.4 comparisons   (100000 random targets)
  worst case         1000 comparisons   (target is absent)

the CASE and the BOUND are two different axes -- all four of these are true:
  best case    is Theta(1)
  average case is Theta(n)   -- (n+1)/2 is still linear
  worst case   is Theta(n)
  worst case   is O(n^2)     -- true, weak, and the reason to prefer Theta

"O(n) algorithm" almost always means "worst case is Theta(n)".
When someone says a hash map is O(1), they mean AVERAGE case -- worst is O(n).
```

Note that the average case is **Θ(n)**, not something better. Halving the work does not change the
class — which is exactly why "average case" is rarely the interesting number for a scan.

**Rust:**

```rust
// The kit's Rng (splitmix64) stands in for Go's math/rand/v2 PCG: a seeded,
// reproducible generator, but a different one, so the empirical line will not
// match Go's digits.
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
    /// like rand.IntN(n): uniform-enough in [0, n)
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
}

// search returns how many comparisons it took to find target (or to give up).
fn search(xs: &[u64], target: u64) -> u64 {
    let mut comparisons = 0;
    for &v in xs {
        comparisons += 1;
        if v == target {
            return comparisons;
        }
    }
    comparisons
}

fn main() {
    const N: u64 = 1000;
    let xs: Vec<u64> = (0..N).collect();

    let best = search(&xs, xs[0]);
    let worst = search(&xs, u64::MAX); // absent: u64 has no -1, any non-member will do

    // Average over every possible successful target, each equally likely.
    let total: u64 = xs.iter().map(|&v| search(&xs, v)).sum();
    let average = total as f64 / N as f64;

    // And an empirical average with uniformly random targets, to confirm.
    let mut rng = Rng::new(1);
    const TRIALS: u64 = 100_000;
    let sum: u64 = (0..TRIALS)
        .map(|_| search(&xs, xs[rng.below(N) as usize]))
        .sum();
    let empirical = sum as f64 / TRIALS as f64;

    println!("linear search over n = {N}\n");
    println!("  best case      {best:>8} comparisons   (target is first)");
    println!(
        "  average case   {average:>8.1} comparisons   (exact: (n+1)/2 = {:.1})",
        (N + 1) as f64 / 2.0
    );
    println!("  empirical avg  {empirical:>8.1} comparisons   ({TRIALS} random targets)");
    println!("  worst case     {worst:>8} comparisons   (target is absent)");

    println!();
    println!("the CASE and the BOUND are two different axes -- all four of these are true:");
    println!("  best case    is Theta(1)");
    println!("  average case is Theta(n)   -- (n+1)/2 is still linear");
    println!("  worst case   is Theta(n)");
    println!("  worst case   is O(n^2)     -- true, weak, and the reason to prefer Theta");

    println!();
    println!("\"O(n) algorithm\" almost always means \"worst case is Theta(n)\".");
    println!("When someone says a hash map is O(1), they mean AVERAGE case -- worst is O(n).");
}
```

**Rust sample output:**

```
linear search over n = 1000

  best case             1 comparisons   (target is first)
  average case      500.5 comparisons   (exact: (n+1)/2 = 500.5)
  empirical avg     501.0 comparisons   (100000 random targets)
  worst case         1000 comparisons   (target is absent)

the CASE and the BOUND are two different axes -- all four of these are true:
  best case    is Theta(1)
  average case is Theta(n)   -- (n+1)/2 is still linear
  worst case   is Theta(n)
  worst case   is O(n^2)     -- true, weak, and the reason to prefer Theta

"O(n) algorithm" almost always means "worst case is Theta(n)".
When someone says a hash map is O(1), they mean AVERAGE case -- worst is O(n).
```

> **Go vs Rust:** Every count matches Go except the empirical line: 501.0 here vs Go's 500.4. That is the random-number generator (the kit's seeded splitmix64 instead of Go's PCG), not the algorithm; with 100,000 uniform targets the standard error is about 0.9, so both land within one standard error of the exact 500.5. Rust's `u64` has no `-1`, so the "absent" target is `u64::MAX`.

**Complexity:** best Θ(1) · average Θ(n), exactly (n+1)/2 · worst Θ(n) · space Θ(1)

---

## 3. Constants and lower-order terms drop

`🟢 easy` · *Why the notation throws information away*

`T(n) = 3n + 7` is Θ(n) because `T(n)/(3n) → 1`. That is the justification for dropping the 3 and the
7 — and the second half of this example is the reminder that dropping them is a *choice with a cost*:
`n²/100` beats `3n+7` all the way to n ≈ 300.

**Steps:**

1. Count exactly, then divide by `3n` and watch the ratio approach 1.
2. Race the linear function against a quadratic one with a tiny constant.
3. Find where the crossover actually is.

```go
package main

import "fmt"

// setup does 7 operations regardless of n; the loop does 3 per element.
// Exact cost: T(n) = 3n + 7.
func work(n int) int {
	ops := 0
	for i := 0; i < 7; i++ { // fixed setup cost
		ops++
	}
	for i := 0; i < n; i++ {
		ops += 3 // three operations per element
	}
	return ops
}

// slowStart is n^2/100: asymptotically worse, but with a tiny constant.
func slowStart(n int) int { return n * n / 100 }

func main() {
	fmt.Println("T(n) = 3n + 7 -- watch the constant and the +7 stop mattering:")
	fmt.Printf("\n%9s %12s %12s %12s\n", "n", "T(n)", "T(n)/n", "T(n)/(3n)")
	for _, n := range []int{1, 10, 100, 10000, 1000000} {
		t := work(n)
		fmt.Printf("%9d %12d %12.2f %12.4f\n", n, t, float64(t)/float64(n), float64(t)/float64(3*n))
	}
	fmt.Println("\nT(n)/(3n) -> 1, so T(n) = Theta(n). The 3 and the 7 are gone.")

	fmt.Println("\n----------------------------------------------------------------")
	fmt.Println("\nBut constants are not gone in REALITY. 3n + 7 vs n^2/100:")
	fmt.Printf("\n%9s %14s %14s   %s\n", "n", "3n + 7", "n^2/100", "cheaper")
	for _, n := range []int{10, 100, 200, 300, 1000, 10000} {
		lin, quad := work(n), slowStart(n)
		cheaper := "linear"
		if quad < lin {
			cheaper = "QUADRATIC"
		}
		fmt.Printf("%9d %14d %14d   %s\n", n, lin, quad, cheaper)
	}

	fmt.Println("\nthe quadratic wins until n ~ 300, because 1/100 < 3.")
	fmt.Println("asymptotics say what happens EVENTUALLY. Constants say what happens today.")
}
```

**Output:**

```
T(n) = 3n + 7 -- watch the constant and the +7 stop mattering:

        n         T(n)       T(n)/n    T(n)/(3n)
        1           10        10.00       3.3333
       10           37         3.70       1.2333
      100          307         3.07       1.0233
    10000        30007         3.00       1.0002
  1000000      3000007         3.00       1.0000

T(n)/(3n) -> 1, so T(n) = Theta(n). The 3 and the 7 are gone.

----------------------------------------------------------------

But constants are not gone in REALITY. 3n + 7 vs n^2/100:

        n         3n + 7        n^2/100   cheaper
       10             37              1   QUADRATIC
      100            307            100   QUADRATIC
      200            607            400   QUADRATIC
      300            907            900   QUADRATIC
     1000           3007          10000   linear
    10000          30007        1000000   linear

the quadratic wins until n ~ 300, because 1/100 < 3.
asymptotics say what happens EVENTUALLY. Constants say what happens today.
```

**Rust:**

```rust
// work does 7 operations regardless of n; the loop does 3 per element.
// Exact cost: T(n) = 3n + 7.
fn work(n: u64) -> u64 {
    let mut ops = 0;
    for _ in 0..7 {
        ops += 1; // fixed setup cost
    }
    for _ in 0..n {
        ops += 3; // three operations per element
    }
    ops
}

// slow_start is n^2/100: asymptotically worse, but with a tiny constant.
fn slow_start(n: u64) -> u64 {
    n * n / 100
}

fn main() {
    println!("T(n) = 3n + 7 -- watch the constant and the +7 stop mattering:");
    println!(
        "\n{:>9} {:>12} {:>12} {:>12}",
        "n", "T(n)", "T(n)/n", "T(n)/(3n)"
    );
    for n in [1, 10, 100, 10000, 1000000] {
        let t = work(n);
        println!(
            "{n:>9} {t:>12} {:>12.2} {:>12.4}",
            t as f64 / n as f64,
            t as f64 / (3 * n) as f64
        );
    }
    println!("\nT(n)/(3n) -> 1, so T(n) = Theta(n). The 3 and the 7 are gone.");

    println!("\n----------------------------------------------------------------");
    println!("\nBut constants are not gone in REALITY. 3n + 7 vs n^2/100:");
    println!("\n{:>9} {:>14} {:>14}   cheaper", "n", "3n + 7", "n^2/100");
    for n in [10, 100, 200, 300, 1000, 10000] {
        let (lin, quad) = (work(n), slow_start(n));
        let cheaper = if quad < lin { "QUADRATIC" } else { "linear" };
        println!("{n:>9} {lin:>14} {quad:>14}   {cheaper}");
    }

    println!("\nthe quadratic wins until n ~ 300, because 1/100 < 3.");
    println!("asymptotics say what happens EVENTUALLY. Constants say what happens today.");
}
```

**Rust output:**

```
T(n) = 3n + 7 -- watch the constant and the +7 stop mattering:

        n         T(n)       T(n)/n    T(n)/(3n)
        1           10        10.00       3.3333
       10           37         3.70       1.2333
      100          307         3.07       1.0233
    10000        30007         3.00       1.0002
  1000000      3000007         3.00       1.0000

T(n)/(3n) -> 1, so T(n) = Theta(n). The 3 and the 7 are gone.

----------------------------------------------------------------

But constants are not gone in REALITY. 3n + 7 vs n^2/100:

        n         3n + 7        n^2/100   cheaper
       10             37              1   QUADRATIC
      100            307            100   QUADRATIC
      200            607            400   QUADRATIC
      300            907            900   QUADRATIC
     1000           3007          10000   linear
    10000          30007        1000000   linear

the quadratic wins until n ~ 300, because 1/100 < 3.
asymptotics say what happens EVENTUALLY. Constants say what happens today.
```

> **Go vs Rust:** Output is identical to Go, including the crossover rows (quadratic cheaper through n=300, linear from n=1000). Rust has no default `int`: the counts are `u64`, chosen deliberately, and the float ratios need explicit `as f64` casts just as Go needs `float64(...)`.

**Complexity:** `3n+7` is Θ(n) · `n²/100` is Θ(n²) · the crossover sits at n ≈ 300, and no amount of asymptotic reasoning will tell you that

---

## 4. The ratio test

`🟢 easy` · *Identifying a complexity class from numbers*

The single most useful tool in this lesson, and you will use it for the rest of the plan. **Double the
input, divide the costs, read the answer off the ratio.** It works because a Θ(nᵏ) function multiplies
by 2ᵏ when n doubles, while a logarithm only gains an additive constant.

**Steps:**

1. Write six functions that each *return their own operation count*.
2. Evaluate each at n and 2n.
3. Match the ratio against the bands.

```go
package main

import "fmt"

// Each function returns the number of basic operations it performs for input n.
// Nothing is timed -- these counts are identical on every machine.

func constant(n int) int {
	return 1
}

func logarithmic(n int) int {
	ops := 0
	for i := n; i > 0; i /= 2 {
		ops++
	}
	return ops
}

func linear(n int) int {
	ops := 0
	for i := 0; i < n; i++ {
		ops++
	}
	return ops
}

func linearithmic(n int) int {
	ops := 0
	for i := 1; i <= n; i *= 2 { // log n outer iterations
		for j := 0; j < n; j++ { // n inner
			ops++
		}
	}
	return ops
}

func quadratic(n int) int {
	ops := 0
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ {
			ops++
		}
	}
	return ops
}

func cubic(n int) int {
	ops := 0
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ {
			for k := 0; k < n; k++ {
				ops++
			}
		}
	}
	return ops
}

func main() {
	funcs := []struct {
		name      string
		f         func(int) int
		predicted string
	}{
		{"O(1)", constant, "1.00"},
		{"O(log n)", logarithmic, "~1 (+1 per doubling)"},
		{"O(n)", linear, "2.00"},
		{"O(n log n)", linearithmic, "2.0-2.3"},
		{"O(n^2)", quadratic, "4.00"},
		{"O(n^3)", cubic, "8.00"},
	}

	const n = 64

	fmt.Printf("doubling the input from n=%d to n=%d:\n\n", n, 2*n)
	fmt.Printf("%-12s %12s %12s %10s   %s\n", "complexity", "T(n)", "T(2n)", "ratio", "predicted")
	for _, fn := range funcs {
		a, b := fn.f(n), fn.f(2*n)
		fmt.Printf("%-12s %12d %12d %10.2f   %s\n", fn.name, a, b, float64(b)/float64(a), fn.predicted)
	}

	fmt.Println()
	fmt.Println("THE RATIO TEST: double the input, divide the costs, read the answer.")
	fmt.Println("  ratio ~1  -> logarithmic (grows by an additive constant, not a factor)")
	fmt.Println("  ratio ~2  -> linear")
	fmt.Println("  ratio 2-2.3 -> linearithmic (the log term adds a little on top of 2)")
	fmt.Println("  ratio ~4  -> quadratic     (2^2)")
	fmt.Println("  ratio ~8  -> cubic         (2^3)")
	fmt.Println()
	fmt.Println("this works on measured times too -- that is how you identify")
	fmt.Println("the complexity of code you did not write (examples 14 and 16).")
}
```

**Output:**

```
doubling the input from n=64 to n=128:

complexity           T(n)        T(2n)      ratio   predicted
O(1)                    1            1       1.00   1.00
O(log n)                7            8       1.14   ~1 (+1 per doubling)
O(n)                   64          128       2.00   2.00
O(n log n)            448         1024       2.29   2.0-2.3
O(n^2)               4096        16384       4.00   4.00
O(n^3)             262144      2097152       8.00   8.00

THE RATIO TEST: double the input, divide the costs, read the answer.
  ratio ~1  -> logarithmic (grows by an additive constant, not a factor)
  ratio ~2  -> linear
  ratio 2-2.3 -> linearithmic (the log term adds a little on top of 2)
  ratio ~4  -> quadratic     (2^2)
  ratio ~8  -> cubic         (2^3)

this works on measured times too -- that is how you identify
the complexity of code you did not write (examples 14 and 16).
```

Note **O(n)** at 2.00 versus **O(n log n)** at 2.29 — on exact counts the two are cleanly separated.
Hold onto that: example 14 shows the separation disappearing once you measure time instead.

**Rust:**

```rust
// Each function returns the number of basic operations it performs for input n.
// Nothing is timed -- these counts are identical on every machine.

fn constant(_n: u64) -> u64 {
    1
}

fn logarithmic(n: u64) -> u64 {
    let mut ops = 0;
    let mut i = n;
    while i > 0 {
        ops += 1;
        i /= 2;
    }
    ops
}

fn linear(n: u64) -> u64 {
    let mut ops = 0;
    for _ in 0..n {
        ops += 1;
    }
    ops
}

fn linearithmic(n: u64) -> u64 {
    let mut ops = 0;
    let mut i = 1;
    while i <= n {
        // log n outer iterations
        for _ in 0..n {
            // n inner
            ops += 1;
        }
        i *= 2;
    }
    ops
}

fn quadratic(n: u64) -> u64 {
    let mut ops = 0;
    for _ in 0..n {
        for _ in 0..n {
            ops += 1;
        }
    }
    ops
}

fn cubic(n: u64) -> u64 {
    let mut ops = 0;
    for _ in 0..n {
        for _ in 0..n {
            for _ in 0..n {
                ops += 1;
            }
        }
    }
    ops
}

struct Row {
    name: &'static str,
    f: fn(u64) -> u64, // a plain function pointer: no closure, no allocation
    predicted: &'static str,
}

fn main() {
    let funcs = [
        Row {
            name: "O(1)",
            f: constant,
            predicted: "1.00",
        },
        Row {
            name: "O(log n)",
            f: logarithmic,
            predicted: "~1 (+1 per doubling)",
        },
        Row {
            name: "O(n)",
            f: linear,
            predicted: "2.00",
        },
        Row {
            name: "O(n log n)",
            f: linearithmic,
            predicted: "2.0-2.3",
        },
        Row {
            name: "O(n^2)",
            f: quadratic,
            predicted: "4.00",
        },
        Row {
            name: "O(n^3)",
            f: cubic,
            predicted: "8.00",
        },
    ];

    const N: u64 = 64;

    println!("doubling the input from n={N} to n={}:\n", 2 * N);
    println!(
        "{:<12} {:>12} {:>12} {:>10}   predicted",
        "complexity", "T(n)", "T(2n)", "ratio"
    );
    for fun in &funcs {
        let (a, b) = ((fun.f)(N), (fun.f)(2 * N));
        println!(
            "{:<12} {a:>12} {b:>12} {:>10.2}   {}",
            fun.name,
            b as f64 / a as f64,
            fun.predicted
        );
    }

    println!();
    println!("THE RATIO TEST: double the input, divide the costs, read the answer.");
    println!("  ratio ~1  -> logarithmic (grows by an additive constant, not a factor)");
    println!("  ratio ~2  -> linear");
    println!("  ratio 2-2.3 -> linearithmic (the log term adds a little on top of 2)");
    println!("  ratio ~4  -> quadratic     (2^2)");
    println!("  ratio ~8  -> cubic         (2^3)");
    println!();
    println!("this works on measured times too -- that is how you identify");
    println!("the complexity of code you did not write (examples 14 and 16).");
}
```

**Rust output:**

```
doubling the input from n=64 to n=128:

complexity           T(n)        T(2n)      ratio   predicted
O(1)                    1            1       1.00   1.00
O(log n)                7            8       1.14   ~1 (+1 per doubling)
O(n)                   64          128       2.00   2.00
O(n log n)            448         1024       2.29   2.0-2.3
O(n^2)               4096        16384       4.00   4.00
O(n^3)             262144      2097152       8.00   8.00

THE RATIO TEST: double the input, divide the costs, read the answer.
  ratio ~1  -> logarithmic (grows by an additive constant, not a factor)
  ratio ~2  -> linear
  ratio 2-2.3 -> linearithmic (the log term adds a little on top of 2)
  ratio ~4  -> quadratic     (2^2)
  ratio ~8  -> cubic         (2^3)

this works on measured times too -- that is how you identify
the complexity of code you did not write (examples 14 and 16).
```

> **Go vs Rust:** Output is identical to Go: 1, 7/8, 64/128, 448/1024, 4096/16384, 262144/2097152. The table holds plain `fn(u64) -> u64` pointers (no closures, no allocation), and the unused parameter of `constant` has to be spelled `_n` — Go lets you ignore a parameter silently, Rust warns.

**Complexity:** each function is its own label · the ratio test is Θ(1) work on top of two evaluations, and it is the cheapest diagnostic in this plan

---

## 5. Nested loops that aren't quadratic

`🟢 easy` · *Counting the innermost body*

"Two nested loops" is not a complexity. What matters is how many times the **innermost body** runs in
total, and two of the four double loops below are Θ(n log n). The last one — `j += i` — is the
harmonic series, and it catches almost everyone.

**Steps:**

1. Count the innermost body for all four shapes.
2. Use the ratio test on the counts (4 = quadratic, ~2.2 = linearithmic).
3. Verify the two closed forms exactly.

```go
package main

import "fmt"

// Four double loops. Only two of them are quadratic. "Nested loop" is not a
// complexity -- you have to count how many times the inner body actually runs.

// full: the inner loop runs n times, always. n^2.
func full(n int) int {
	ops := 0
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ {
			ops++
		}
	}
	return ops
}

// triangular: the inner loop shrinks. n(n+1)/2 -- half the work, still Theta(n^2).
func triangular(n int) int {
	ops := 0
	for i := 0; i < n; i++ {
		for j := i; j < n; j++ {
			ops++
		}
	}
	return ops
}

// doubling: the OUTER loop multiplies, so it runs only log2(n) times. n log n.
func doubling(n int) int {
	ops := 0
	for i := 1; i <= n; i *= 2 {
		for j := 0; j < n; j++ {
			ops++
		}
	}
	return ops
}

// harmonic: the inner step grows with i, so it runs n/i times.
// Total = n(1 + 1/2 + 1/3 + ... + 1/n) = n * H(n) ~ n ln n.
func harmonic(n int) int {
	ops := 0
	for i := 1; i <= n; i++ {
		for j := 0; j < n; j += i {
			ops++
		}
	}
	return ops
}

func main() {
	funcs := []struct {
		name  string
		f     func(int) int
		truth string
	}{
		{"for j := 0; j < n", full, "Theta(n^2)"},
		{"for j := i; j < n", triangular, "Theta(n^2)  <- half the work, same class"},
		{"outer i *= 2", doubling, "Theta(n log n)"},
		{"inner j += i", harmonic, "Theta(n log n)  <- the harmonic series"},
	}

	fmt.Printf("%-20s %10s %10s %10s %8s   %s\n", "inner loop", "n=64", "n=128", "n=256", "ratio", "actual class")
	for _, fn := range funcs {
		a, b, c := fn.f(64), fn.f(128), fn.f(256)
		fmt.Printf("%-20s %10d %10d %10d %8.2f   %s\n", fn.name, a, b, c, float64(c)/float64(b), fn.truth)
	}

	fmt.Println()
	fmt.Println("ratio ~4 = quadratic; ratio ~2.2 = linearithmic. The shape of the code lies;")
	fmt.Println("the count does not. Always ask how many times the INNER BODY runs, in total.")
	fmt.Println()
	fmt.Println("checking the closed forms at n=256:")
	fmt.Printf("  triangular: n(n+1)/2 = %d, counted %d  -- exact\n", 256*257/2, triangular(256))
	fmt.Printf("  doubling:   n(log2(n)+1) = %d, counted %d  -- exact\n", 256*9, doubling(256))
	fmt.Println("  (the outer loop runs for i = 1,2,4,...,256: that is log2(n)+1 = 9 iterations,")
	fmt.Println("   so the cost is 9n, not 8n. The +1 is a constant -- same complexity class.)")
}
```

**Output:**

```
inner loop                 n=64      n=128      n=256    ratio   actual class
for j := 0; j < n          4096      16384      65536     4.00   Theta(n^2)
for j := i; j < n          2080       8256      32896     3.98   Theta(n^2)  <- half the work, same class
outer i *= 2                448       1024       2304     2.25   Theta(n log n)
inner j += i                337        765       1713     2.24   Theta(n log n)  <- the harmonic series

ratio ~4 = quadratic; ratio ~2.2 = linearithmic. The shape of the code lies;
the count does not. Always ask how many times the INNER BODY runs, in total.

checking the closed forms at n=256:
  triangular: n(n+1)/2 = 32896, counted 32896  -- exact
  doubling:   n(log2(n)+1) = 2304, counted 2304  -- exact
  (the outer loop runs for i = 1,2,4,...,256: that is log2(n)+1 = 9 iterations,
   so the cost is 9n, not 8n. The +1 is a constant -- same complexity class.)
```

The triangular loop does **half** the work of the full one (32,896 vs 65,536) and is still Θ(n²) —
constants don't change the class. Meanwhile the two loops that look almost identical to it are a
whole class cheaper.

**Rust:**

```rust
// Four double loops. Only two of them are quadratic. "Nested loop" is not a
// complexity -- you have to count how many times the inner body actually runs.

// full: the inner loop runs n times, always. n^2.
fn full(n: u64) -> u64 {
    let mut ops = 0;
    for _ in 0..n {
        for _ in 0..n {
            ops += 1;
        }
    }
    ops
}

// triangular: the inner loop shrinks. n(n+1)/2 -- half the work, still Theta(n^2).
fn triangular(n: u64) -> u64 {
    let mut ops = 0;
    for i in 0..n {
        for _ in i..n {
            ops += 1;
        }
    }
    ops
}

// doubling: the OUTER loop multiplies, so it runs only log2(n) times. n log n.
fn doubling(n: u64) -> u64 {
    let mut ops = 0;
    let mut i = 1;
    while i <= n {
        for _ in 0..n {
            ops += 1;
        }
        i *= 2;
    }
    ops
}

// harmonic: the inner step grows with i, so it runs n/i times.
// Total = n(1 + 1/2 + 1/3 + ... + 1/n) = n * H(n) ~ n ln n.
// (0..n).step_by(i) is Go's `for j := 0; j < n; j += i`.
fn harmonic(n: u64) -> u64 {
    let mut ops = 0;
    for i in 1..=n {
        for _ in (0..n).step_by(i as usize) {
            ops += 1;
        }
    }
    ops
}

struct Shape {
    name: &'static str,
    f: fn(u64) -> u64,
    truth: &'static str,
}

fn main() {
    let funcs = [
        Shape {
            name: "for j := 0; j < n",
            f: full,
            truth: "Theta(n^2)",
        },
        Shape {
            name: "for j := i; j < n",
            f: triangular,
            truth: "Theta(n^2)  <- half the work, same class",
        },
        Shape {
            name: "outer i *= 2",
            f: doubling,
            truth: "Theta(n log n)",
        },
        Shape {
            name: "inner j += i",
            f: harmonic,
            truth: "Theta(n log n)  <- the harmonic series",
        },
    ];

    println!(
        "{:<20} {:>10} {:>10} {:>10} {:>8}   actual class",
        "inner loop", "n=64", "n=128", "n=256", "ratio"
    );
    for fun in &funcs {
        let (a, b, c) = ((fun.f)(64), (fun.f)(128), (fun.f)(256));
        println!(
            "{:<20} {a:>10} {b:>10} {c:>10} {:>8.2}   {}",
            fun.name,
            c as f64 / b as f64,
            fun.truth
        );
    }

    println!();
    println!("ratio ~4 = quadratic; ratio ~2.2 = linearithmic. The shape of the code lies;");
    println!("the count does not. Always ask how many times the INNER BODY runs, in total.");
    println!();
    println!("checking the closed forms at n=256:");
    println!(
        "  triangular: n(n+1)/2 = {}, counted {}  -- exact",
        256 * 257 / 2,
        triangular(256)
    );
    println!(
        "  doubling:   n(log2(n)+1) = {}, counted {}  -- exact",
        256 * 9,
        doubling(256)
    );
    println!("  (the outer loop runs for i = 1,2,4,...,256: that is log2(n)+1 = 9 iterations,");
    println!("   so the cost is 9n, not 8n. The +1 is a constant -- same complexity class.)");
}
```

**Rust output:**

```
inner loop                 n=64      n=128      n=256    ratio   actual class
for j := 0; j < n          4096      16384      65536     4.00   Theta(n^2)
for j := i; j < n          2080       8256      32896     3.98   Theta(n^2)  <- half the work, same class
outer i *= 2                448       1024       2304     2.25   Theta(n log n)
inner j += i                337        765       1713     2.24   Theta(n log n)  <- the harmonic series

ratio ~4 = quadratic; ratio ~2.2 = linearithmic. The shape of the code lies;
the count does not. Always ask how many times the INNER BODY runs, in total.

checking the closed forms at n=256:
  triangular: n(n+1)/2 = 32896, counted 32896  -- exact
  doubling:   n(log2(n)+1) = 2304, counted 2304  -- exact
  (the outer loop runs for i = 1,2,4,...,256: that is log2(n)+1 = 9 iterations,
   so the cost is 9n, not 8n. The +1 is a constant -- same complexity class.)
```

> **Go vs Rust:** Output is identical to Go, including both exact closed forms (32896 and 2304). Go's `for j := 0; j < n; j += i` becomes `(0..n).step_by(i)`, which panics on a step of 0 — safe here because `i` starts at 1.

**Complexity:** full Θ(n²) · triangular Θ(n²), exactly n(n+1)/2 · doubling Θ(n log n), exactly n(log₂n+1) · harmonic Θ(n log n) via n·H(n) ≈ n ln n

---

## 6. Sequential adds, nested multiplies

`🟢 easy` · *Composition rules*

Two rules cover almost every analysis you will ever do. The interesting part is the third row: when an
O(n) pass runs before an O(n²) pass, the linear one contributes **0.25%** of the work at n=400. That
number is the whole justification for dropping lower-order terms.

**Steps:**

1. Compose two O(n) steps sequentially, then nest them.
2. Put an O(n) step next to an O(n²) step and check the ratio.
3. Compute what fraction of the total the linear term actually is.

```go
package main

import "fmt"

func loopN(n int) int { return n }

func loopNSquared(n int) int { return n * n }

// sequential: do one thing, then the other. Costs add.
func sequential(n int) int { return loopN(n) + loopN(n) }

// nested: do the second thing for every step of the first. Costs multiply.
func nested(n int) int { return loopN(n) * loopN(n) }

// mixed: an O(n) pass followed by an O(n^2) pass. The bigger term swallows the other.
func mixed(n int) int { return loopN(n) + loopNSquared(n) }

func main() {
	fmt.Println("two rules cover almost every analysis you will do:")
	fmt.Println("  SEQUENTIAL steps ADD        -> keep the biggest term")
	fmt.Println("  NESTED steps MULTIPLY")
	fmt.Println()

	rows := []struct {
		name  string
		f     func(int) int
		class string
	}{
		{"O(n) then O(n)", sequential, "Theta(n)      2n, drop the 2"},
		{"O(n) inside O(n)", nested, "Theta(n^2)"},
		{"O(n) then O(n^2)", mixed, "Theta(n^2)    the n is noise"},
	}

	fmt.Printf("%-20s %12s %12s %12s %8s   %s\n", "composition", "n=100", "n=200", "n=400", "ratio", "class")
	for _, r := range rows {
		a, b, c := r.f(100), r.f(200), r.f(400)
		fmt.Printf("%-20s %12d %12d %12d %8.2f   %s\n", r.name, a, b, c, float64(c)/float64(b), r.class)
	}

	fmt.Println()
	fmt.Println("look at 'O(n) then O(n^2)': at n=400 the linear pass contributes")
	fmt.Printf("  %d of %d operations -- %.2f%% of the total.\n",
		loopN(400), mixed(400), 100*float64(loopN(400))/float64(mixed(400)))
	fmt.Println("that is why lower-order terms are dropped: they vanish as a fraction.")
	fmt.Println()
	fmt.Println("the trap: 'sequential' only applies to steps that run ONE AFTER ANOTHER.")
	fmt.Println("a cheap-looking call INSIDE a loop is nested -- it multiplies (see example 13).")
}
```

**Output:**

```
two rules cover almost every analysis you will do:
  SEQUENTIAL steps ADD        -> keep the biggest term
  NESTED steps MULTIPLY

composition                 n=100        n=200        n=400    ratio   class
O(n) then O(n)                200          400          800     2.00   Theta(n)      2n, drop the 2
O(n) inside O(n)            10000        40000       160000     4.00   Theta(n^2)
O(n) then O(n^2)            10100        40200       160400     3.99   Theta(n^2)    the n is noise

look at 'O(n) then O(n^2)': at n=400 the linear pass contributes
  400 of 160400 operations -- 0.25% of the total.
that is why lower-order terms are dropped: they vanish as a fraction.

the trap: 'sequential' only applies to steps that run ONE AFTER ANOTHER.
a cheap-looking call INSIDE a loop is nested -- it multiplies (see example 13).
```

**Rust:**

```rust
fn loop_n(n: u64) -> u64 {
    n
}

fn loop_n_squared(n: u64) -> u64 {
    n * n
}

// sequential: do one thing, then the other. Costs add.
fn sequential(n: u64) -> u64 {
    loop_n(n) + loop_n(n)
}

// nested: do the second thing for every step of the first. Costs multiply.
fn nested(n: u64) -> u64 {
    loop_n(n) * loop_n(n)
}

// mixed: an O(n) pass followed by an O(n^2) pass. The bigger term swallows the other.
fn mixed(n: u64) -> u64 {
    loop_n(n) + loop_n_squared(n)
}

struct Row {
    name: &'static str,
    f: fn(u64) -> u64,
    class: &'static str,
}

fn main() {
    println!("two rules cover almost every analysis you will do:");
    println!("  SEQUENTIAL steps ADD        -> keep the biggest term");
    println!("  NESTED steps MULTIPLY");
    println!();

    let rows = [
        Row {
            name: "O(n) then O(n)",
            f: sequential,
            class: "Theta(n)      2n, drop the 2",
        },
        Row {
            name: "O(n) inside O(n)",
            f: nested,
            class: "Theta(n^2)",
        },
        Row {
            name: "O(n) then O(n^2)",
            f: mixed,
            class: "Theta(n^2)    the n is noise",
        },
    ];

    println!(
        "{:<20} {:>12} {:>12} {:>12} {:>8}   class",
        "composition", "n=100", "n=200", "n=400", "ratio"
    );
    for r in &rows {
        let (a, b, c) = ((r.f)(100), (r.f)(200), (r.f)(400));
        println!(
            "{:<20} {a:>12} {b:>12} {c:>12} {:>8.2}   {}",
            r.name,
            c as f64 / b as f64,
            r.class
        );
    }

    println!();
    println!("look at 'O(n) then O(n^2)': at n=400 the linear pass contributes");
    println!(
        "  {} of {} operations -- {:.2}% of the total.",
        loop_n(400),
        mixed(400),
        100.0 * loop_n(400) as f64 / mixed(400) as f64
    );
    println!("that is why lower-order terms are dropped: they vanish as a fraction.");
    println!();
    println!("the trap: 'sequential' only applies to steps that run ONE AFTER ANOTHER.");
    println!("a cheap-looking call INSIDE a loop is nested -- it multiplies (see example 13).");
}
```

**Rust output:**

```
two rules cover almost every analysis you will do:
  SEQUENTIAL steps ADD        -> keep the biggest term
  NESTED steps MULTIPLY

composition                 n=100        n=200        n=400    ratio   class
O(n) then O(n)                200          400          800     2.00   Theta(n)      2n, drop the 2
O(n) inside O(n)            10000        40000       160000     4.00   Theta(n^2)
O(n) then O(n^2)            10100        40200       160400     3.99   Theta(n^2)    the n is noise

look at 'O(n) then O(n^2)': at n=400 the linear pass contributes
  400 of 160400 operations -- 0.25% of the total.
that is why lower-order terms are dropped: they vanish as a fraction.

the trap: 'sequential' only applies to steps that run ONE AFTER ANOTHER.
a cheap-looking call INSIDE a loop is nested -- it multiplies (see example 13).
```

> **Go vs Rust:** Output is identical to Go, including the 0.25% share of the linear pass (400 of 160400). The composition rules are plain integer arithmetic on `u64`, so there is nothing language-specific to see — which is the point of counting instead of timing.

**Complexity:** sequential → Θ(max of the parts) · nested → Θ(product of the parts) · and the "cheap call inside a loop" is nested, which is example 13's entire subject

---

> Next tier: [🟡 medium](2-medium.md) — amortized analysis, space, and recurrences.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
