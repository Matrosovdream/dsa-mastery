# Examples — the convention

One folder per lesson: `examples/NN-topic/`. Each folder holds a graded library of complete,
**run-verified** `package main` programs.

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

**Complexity:** time O(1) amortized per op · space O(cap) · allocations: 1 (the buffer)
````

The **Complexity** line is the DSA-specific addition to this repo's format — every example ends with
one, because the whole point is being able to state it.

## The verification rule

Nothing is added to a tier file until it has been:

1. Compiled — `go build`
2. Formatted — `gofmt -l` returns nothing
3. Vetted — `go vet` is clean
4. **Run** — the `Output:` block is real stdout, copy-pasted, never hand-written
5. Race-checked — `go run -race` for anything with a goroutine

If an example claims a complexity, the hard tier should include the benchmark that demonstrates it.

## Adding more

Keep the numbering going in the matching tier file, then add the entry to that lesson's `README.md`
index **and** its `PROGRESS.md`. Numbers are global across the three files (1–8, 9–17, 18–26) so an
example is always unambiguous.

---
*Plan: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md).*
