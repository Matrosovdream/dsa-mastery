# Step 03 — Complexity Analysis · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. Three bounds on one function**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. Three bounds on one function
- [ ] 2. The case and the bound are different axes
- [ ] 3. Constants and lower-order terms drop
- [ ] 4. The ratio test
- [ ] 5. Nested loops that aren't quadratic
- [ ] 6. Sequential adds, nested multiplies

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Why append is amortized O(1)
- [ ] 8. Amortized is not average, and is not worst case
- [ ] 9. The call stack is space
- [ ] 10. Auxiliary space vs total space
- [ ] 11. Recursion trees

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. The master theorem
- [ ] 13. Quadratic hiding in a single loop
- [ ] 14. Measuring a complexity class
- [ ] 15. What amortized O(1) costs the unlucky caller
- [ ] 16. A complexity regression gate

## Recall drill

The point of this lesson is recall. Cover the right-hand column and answer from memory; come back a
week later and do it again.

| Question | Answer |
|---|---|
| `T(n) = n` exactly. Is `T(n) = O(n²)` true? | Yes — O is an upper bound. It's just not tight. |
| Ratio 2.2 on operation counts means? | Θ(n log n) |
| Ratio 4 means? | Θ(n²) |
| `for i…n { for j := i; j<n }` | Θ(n²) — n(n+1)/2 |
| `for i:=1; i<=n; i*=2 { for j…n }` | Θ(n log n) |
| `for i…n { for j:=0; j<n; j+=i }` | Θ(n log n) — harmonic |
| Why is amortized append ~4 copies in Go, not 1? | growth factor tapers to 1.25; 1/(g−1) = 4 |
| Amortized O(1) — what does it bound? | the whole SEQUENCE, never one operation |
| Space of a recursion n deep? | Θ(n) — the call stack counts |
| `T(n) = 2T(n/2) + n` | Θ(n log n) |
| `T(n) = 2T(n/2) + 1` | Θ(n) |
| `T(n) = T(n/2) + 1` | Θ(log n) |
| `T(n) = T(n−1) + n` | Θ(n²) |
| Master theorem: `d < log_b(a)` | Case 1, leaves win, Θ(n^log_b(a)) |
| `s += x` inside a loop | Θ(n²) |
| `slices.Contains` inside a loop | Θ(n²) |

## Numbers to find on your own machine

| What | Mine (M4, Go 1.26.3) | Yours |
|------|----------------------|-------|
| append copies/append at n=1,000,000 | 4.15 | |
| worst single append in 100,000 | 88,064 copies | |
| bytes per stack frame | ~33–52 | |
| `s += x` vs `strings.Builder` at n=200,000 | 964 ms vs 425 µs | |
| prepend vs append+reverse at n=200,000 | 6.971 s vs 116 µs | |
| append tail: max/p50 | ~8,900× | |
| `Dedupe` vs `DedupeSlow` growth ratio | 2.07 vs 3.91 | |

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
