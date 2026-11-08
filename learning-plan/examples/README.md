# Examples — the convention

One folder per lesson: `examples/NN-topic/`. Each folder holds a graded library of complete,
**run-verified** `package main` programs — and every one of them has a **Rust twin** right beside it,
so you can read the same idea in both languages.

```
examples/NN-topic/
├── README.md      # index of every example, linked by anchor
├── PROGRESS.md    # tick-box tracker, one line per example
├── 1-easy.md      # examples 1–8       — the mechanics, in isolation
├── 2-medium.md    # examples 9–17      — the structure assembled, the classic problems
└── 3-hard.md      # examples 18–26     — the hard variants, the applied build
```

Target ~26 examples per lesson (8 / 9 / 9), but the split is a guide, not a quota.

## Tiers

| Tier | Means |
|------|-------|
| 🟢 **Easy** | One idea, in isolation. "Here is the index math." "Here is the node struct." Under 30 lines. |
| 🟡 **Medium** | The full structure or the classic problem it solves. The version you'd write in an interview. |
| 🔴 **Hard** | The variant with a twist, the O(n log n) rewrite of an O(n²), the concurrent version, or the applied mini-build. |

## The shape of one example

Every example follows the same five parts:

````markdown
## 12. A ring-buffer queue

`🟡 medium` · *Queue / O(1) space*

One paragraph: what this shows and **why it's the right tool** — the idea, not a restatement of
the code. Name the complexity here.

**Steps:**

1. What to set up.
2. The key move.
3. What to observe in the output.

```go
package main

import "fmt"

func main() {
	// ...
}
```

**Output:**

```
real stdout, pasted from an actual run
```

**Rust:**

```rust
fn main() {
    // the same program, in Rust
}
```

**Rust output:**

```
real stdout, pasted from an actual run of the Rust program
```

> **Go vs Rust:** one to three sentences — only what is a language fact or what the two outputs show.

**Complexity:** time O(1) amortized per op · space O(cap) · allocations: 1 (the buffer)
````

The **Complexity** line is the DSA-specific addition to this repo's format — every example ends with
one, because the whole point is being able to state it. (The `go test` examples in lesson 02 close
with a **Takeaway** line instead; the Rust twin goes in front of it all the same.)

## Go and Rust, side by side

The Go program and its output come first and are the canonical version. The Rust twin follows
immediately, then a short **Go vs Rust** note, then the closing line. Reading order is therefore always
*Go → Go output → Rust → Rust output → what differs*.

What "the same example" means:

- **Same idea, same shape.** Same sections printed in the same order, built up in the same steps.
- **Same numbers wherever the number belongs to the algorithm** — comparison counts, operation counts,
  structure sizes. For the deterministic examples the Rust output is character-for-character the Go
  output, and where it is not, the note says why.
- **Idiomatic Rust, not transliterated Go.** `Option` instead of `-1`, ownership instead of aliasing
  surprises, `Vec<T>` / `&[T]` instead of slice headers.
- **Printed commentary is true of the Rust run.** The Go programs print prose that interprets their
  measurements. The Rust programs do not copy those claims; they print what *their* numbers show. Where
  Rust disagrees with Go, that disagreement is the finding.
- **Go-only machinery is ported as a question, not a mechanism.** The garbage collector, escape
  analysis, size classes and slice headers have no Rust counterpart, so those examples ask the nearest
  question Rust *does* have (ownership, `size_of`, allocation counts, `Box` vs inline) and the note says
  what does not exist.

Rust programs are **std-only, single-file, no crates, no Cargo**: nothing to install beyond `rustc`.
Where Go's standard library supplies something Rust's does not, the example builds a small stand-in
inline — a splitmix64 generator for `math/rand`, a counting `#[global_allocator]` for `runtime.MemStats`
and `testing.AllocsPerRun`, an `ns_per_op` loop for `testing.Benchmark`. Counting allocations this way
is deterministic, which is better than Go's GC-dependent numbers.

**Run any Rust example:**

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the example into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Always build with `-O`: the examples measure time, and a debug build is not a meaningful measurement
(it also panics on integer overflow where a release build wraps). The `go test` examples in lesson 02
become `#[cfg(test)]` modules, built with `rustc --edition 2024 --test`.

## The verification rule

Nothing is added to a tier file until it has been:

1. Compiled — `go build`
2. Formatted — `gofmt -l` returns nothing
3. Vetted — `go vet` is clean
4. **Run** — the `Output:` block is real stdout, copy-pasted, never hand-written
5. Race-checked — `go run -race` for anything with a goroutine

If an example claims a complexity, the hard tier should include the benchmark that demonstrates it.

**The Rust twin has its own gate**, and it is stricter on lints because Rust offers one for free:

1. Formatted — `rustfmt --edition 2024 --check` returns nothing
2. Linted — `clippy-driver -D warnings` is clean (no blanket `#![allow]`; a targeted `allow` needs a reason)
3. Compiled — `rustc --edition 2024 -O` (release; `--test` for the `#[cfg(test)]` examples)
4. **Run** — the **Rust output** block is the binary's real stdout, captured by the tool that inserts it,
   never typed by hand
5. Compared — for deterministic examples, diffed against the Go output; any difference is explained in the note

Races need no separate step: a data race in safe Rust does not compile, so the Rust twins of the
`-race` examples show the safe version and the note says so.

## Adding more

Keep the numbering going in the matching tier file, then add the entry to that lesson's `README.md`
index **and** its `PROGRESS.md`. Numbers are global across the three files (1–8, 9–17, 18–26) so an
example is always unambiguous. A new example is not finished until it has its Rust twin, inserted
after the Go output and before the closing **Complexity** / **Takeaway** line.

---
*Plan: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md).*
