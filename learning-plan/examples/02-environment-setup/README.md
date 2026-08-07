# Step 02 — Environment & Toolkit · Examples

A library of **16 runnable examples**, split into three files by difficulty.

> ⚠️ **These are `go test` packages, not `go run` programs.** Unlike lesson 01, each example is a
> *folder* holding two to five files. Create the folder, add the files, run the command shown.

```bash
mkdir -p /tmp/dsa-t && cd /tmp/dsa-t   # once: go mod init scratch
mkdir ex01                             # add the files, then:
go test -v ./ex01
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added — the output blocks are
real stdout. Note which is which:

- **Deterministic** (1–5, 11 seeds, 12, 13, 15): everything above the `ok ... 0.31s` timing line matches exactly.
- **Sample output** (6–9, 14, 16): timings and iteration counts vary by machine. Produced on an Apple M4, Go 1.26.3.
- **Nondeterministic** (10, 11 fuzzing): race reports carry live addresses; fuzzing finds a different counterexample and writes a different filename each run.

| Tier | File | Examples |
|------|------|----------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–6 |
| 🟡 Medium | [2-medium.md](2-medium.md) | 7–11 |
| 🔴 Hard | [3-hard.md](3-hard.md) | 12–16 |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md) — writing the test

- [1. Your first algorithm test](1-easy.md#1-your-first-algorithm-test)
- [2. The table-driven form](1-easy.md#2-the-table-driven-form)
- [3. nil vs empty, and how to compare slices](1-easy.md#3-nil-vs-empty-and-how-to-compare-slices)
- [4. The edge-case checklist](1-easy.md#4-the-edge-case-checklist)
- [5. 100% coverage, still wrong](1-easy.md#5-100-coverage-still-wrong)
- [6. Benchmarks in a test file](1-easy.md#6-benchmarks-in-a-test-file)

### 🟡 [Medium](2-medium.md) — measuring, and what tests can't see

- [7. The empty control](2-medium.md#7-the-empty-control)
- [8. The size sweep](2-medium.md#8-the-size-sweep)
- [9. Before and after, with statistics](2-medium.md#9-before-and-after-with-statistics)
- [10. The race a passing test cannot see](2-medium.md#10-the-race-a-passing-test-cannot-see)
- [11. Fuzzing writes your regression test](2-medium.md#11-fuzzing-writes-your-regression-test)

### 🔴 [Hard](3-hard.md) — the harness

- [12. A generic differential oracle](3-hard.md#12-a-generic-differential-oracle)
- [13. One property is never enough](3-hard.md#13-one-property-is-never-enough)
- [14. Where did the time go](3-hard.md#14-where-did-the-time-go)
- [15. An allocation budget that fails the build](3-hard.md#15-an-allocation-budget-that-fails-the-build)
- [16. The harness](3-hard.md#16-the-harness)

## The commands, in one place

| Goal | Command |
|------|---------|
| Run tests, verbosely | `go test -v ./...` |
| Run one subtest | `go test -v -run 'TestMax/negatives' ./...` |
| Coverage | `go test -cover ./...` |
| Benchmarks only | `go test -bench=. -benchmem -run='^$' ./...` |
| Compare before/after | `go test -bench=X -count=10 ... > old.txt` then `benchstat old.txt new.txt` |
| Data races | `go test -race ./...` |
| Fuzz for 30s | `go test -fuzz=FuzzX -fuzztime=30s ./...` |
| CPU profile | `go test -bench=. -run='^$' -cpuprofile=cpu.out -o pkg.test ./...` |
| Read a profile | `go tool pprof -top pkg.test cpu.out` |
| All gates | `./check.sh` |

## The five results worth remembering

| # | Finding | Number |
|---|---------|--------|
| 5 | Statement coverage can be perfect while the function is broken | 100.0% covered, 2 wrong answers |
| 7 | A benchmark can measure the harness instead of your code | Square 1.607 ns vs empty control 1.654 ns |
| 8 | You can read a complexity class off a benchmark sweep | linear ×8 per step, binary +3 ns per step |
| 10 | A passing concurrent test proves nothing | 5/5 passes, race found on the 1st `-race` run |
| 11 | The fuzzer finds what your table missed, and writes the regression test | bug found in 0.02s, minimized to 3 elements |

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
