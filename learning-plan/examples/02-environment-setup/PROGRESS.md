# Step 02 — Environment & Toolkit · Progress

Build & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. Your first algorithm test**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. Your first algorithm test
- [ ] 2. The table-driven form
- [ ] 3. nil vs empty, and how to compare slices
- [ ] 4. The edge-case checklist
- [ ] 5. 100% coverage, still wrong
- [ ] 6. Benchmarks in a test file

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. The empty control
- [ ] 8. The size sweep
- [ ] 9. Before and after, with statistics
- [ ] 10. The race a passing test cannot see
- [ ] 11. Fuzzing writes your regression test

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. A generic differential oracle
- [ ] 13. One property is never enough
- [ ] 14. Where did the time go
- [ ] 15. An allocation budget that fails the build
- [ ] 16. The harness

## Setup, once

- [ ] `practice/` module exists and `go test ./...` runs in it
- [ ] `benchstat` installed — `go install golang.org/x/perf/cmd/benchstat@latest`
- [ ] `benchstat` is on `PATH` (it lands in `$(go env GOPATH)/bin`)

## The deliverable

Lesson 02 is only done when these four files exist in your practice folder and the gates are green:

- [ ] `harness_test.go` saved — `Differential`, `RandomSlice`, `IsPermutation`, `EdgeCases`, `CheckInPlace`
- [ ] `bench_test.go` saved — empty control + size sweep
- [ ] `check.sh` saved and executable (`chmod +x`)
- [ ] `./check.sh` prints **all gates passed**

## Numbers to find on your own machine

| What | Mine (M4, Go 1.26.3) | Yours |
|------|----------------------|-------|
| empty `b.Loop` control | 1.654 ns/op | |
| linear vs binary search at n=262144 | 60921 ns vs 14.66 ns (4155×) | |
| prealloc win (benchstat, `Grow`) | −77.8% time, −94.7% allocs | |
| `Dedupe(1000)` allocations | 6 | |
| time for the fuzzer to find the search bug | 0.02 s | |

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
