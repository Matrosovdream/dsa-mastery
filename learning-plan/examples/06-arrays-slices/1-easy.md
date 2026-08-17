# Step 06 — Dynamic Arrays & Slices · 🟢 Easy

Examples **1–6**: the four operations, the idiom that solves half of them, and what the stdlib already has.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

This whole tier is **deterministic** — it rearranges data and prints it. Your output should match
character for character.

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🟡 medium](2-medium.md)

---

## 1. The four operations

`🟢 easy` · *What a dynamic array owes you*

Two of them are free and two of them shift. That asymmetry is the entire personality of the structure,
and everything else in Part 2 is a different way of dodging it.

**Steps:**

1. Implement `Insert` and `Delete` with a single `copy` call each.
2. Insert at the front, middle and end, and watch the cost change.
3. Tabulate how many elements shift by position.

```go
package main

import "fmt"

// A dynamic array is the structure you already use every day. As an ADT it owes
// you four operations, and their costs are not equal:
//
//   Get(i)        Theta(1)   -- one address calculation
//   Append(v)     Theta(1)   -- amortized (lesson 03)
//   Insert(i, v)  Theta(n)   -- everything from i onward shifts right
//   Delete(i)     Theta(n)   -- everything after i shifts left
//
// Go gives you Get and Append as language features. The other two you write --
// and both are one `copy` call once you see the shape.

// Insert puts v at index i, shifting the tail right.
func Insert(xs []int, i, v int) []int {
	xs = append(xs, 0)     // grow by one; the value does not matter
	copy(xs[i+1:], xs[i:]) // shift the tail right by one
	xs[i] = v              // drop v into the hole
	return xs
}

// Delete removes index i, shifting the tail left. Order is preserved.
func Delete(xs []int, i int) []int {
	copy(xs[i:], xs[i+1:]) // shift the tail left over the hole
	return xs[:len(xs)-1]  // and shorten
}

// countShifts reports how many elements copy() has to move.
func countShifts(n, i int, op string) int {
	if op == "insert" {
		return n - i
	}
	return n - i - 1
}

func main() {
	xs := []int{10, 20, 30, 40, 50}
	fmt.Println("start:            ", xs)

	fmt.Println()
	fmt.Println("Get is one address calculation -- Theta(1), no matter where:")
	fmt.Printf("  xs[0] = %d, xs[4] = %d\n", xs[0], xs[4])

	fmt.Println()
	fmt.Println("Insert shifts everything from i onward:")
	xs = Insert(xs, 2, 99)
	fmt.Println("  Insert(xs, 2, 99):", xs)
	xs = Insert(xs, 0, 5)
	fmt.Println("  Insert(xs, 0, 5): ", xs, " <- worst case: shifted all of them")
	xs = Insert(xs, len(xs), 60)
	fmt.Println("  Insert at the end:", xs, " <- best case: shifted none (this is append)")

	fmt.Println()
	fmt.Println("Delete shifts everything after i:")
	xs = Delete(xs, 0)
	fmt.Println("  Delete(xs, 0):    ", xs)
	xs = Delete(xs, 2)
	fmt.Println("  Delete(xs, 2):    ", xs)

	fmt.Println()
	fmt.Println("the cost is entirely about WHERE, not how many:")
	fmt.Println()
	const n = 1000
	fmt.Printf("  %-22s %14s %14s\n", "position", "insert shifts", "delete shifts")
	for _, i := range []int{0, n / 4, n / 2, n - 1, n} {
		label := fmt.Sprintf("i = %d", i)
		switch i {
		case 0:
			label += " (front)"
		case n:
			label += " (end)"
		}
		ins := countShifts(n, i, "insert")
		del := ins - 1
		if i == n {
			del = 0
		}
		fmt.Printf("  %-22s %14d %14d\n", label, ins, del)
	}

	fmt.Println()
	fmt.Println("this is THE trade-off of a dynamic array, and it never goes away:")
	fmt.Println("  random access is free, and structural change in the middle is not.")
	fmt.Println()
	fmt.Println("everything else in Part 2 is a different answer to that one sentence:")
	fmt.Println("  a linked list  (07) makes insert O(1) and access O(n) -- the mirror image")
	fmt.Println("  a stack        (08) only ever touches the cheap END")
	fmt.Println("  a ring buffer  (09) makes BOTH ends cheap by giving up contiguity")
	fmt.Println()
	fmt.Println("note copy() is not a loop you should write yourself: it compiles to a")
	fmt.Println("memmove, which moves whole cache lines at a time (lesson 04). Shifting")
	fmt.Println("1000 ints is Theta(n) but with a very small constant.")
}
```

**Output:**

```
start:             [10 20 30 40 50]

Get is one address calculation -- Theta(1), no matter where:
  xs[0] = 10, xs[4] = 50

Insert shifts everything from i onward:
  Insert(xs, 2, 99): [10 20 99 30 40 50]
  Insert(xs, 0, 5):  [5 10 20 99 30 40 50]  <- worst case: shifted all of them
  Insert at the end: [5 10 20 99 30 40 50 60]  <- best case: shifted none (this is append)

Delete shifts everything after i:
  Delete(xs, 0):     [10 20 99 30 40 50 60]
  Delete(xs, 2):     [10 20 30 40 50 60]

the cost is entirely about WHERE, not how many:

  position                insert shifts  delete shifts
  i = 0 (front)                    1000            999
  i = 250                           750            749
  i = 500                           500            499
  i = 999                             1              0
  i = 1000 (end)                      0              0

this is THE trade-off of a dynamic array, and it never goes away:
  random access is free, and structural change in the middle is not.

everything else in Part 2 is a different answer to that one sentence:
  a linked list  (07) makes insert O(1) and access O(n) -- the mirror image
  a stack        (08) only ever touches the cheap END
  a ring buffer  (09) makes BOTH ends cheap by giving up contiguity

note copy() is not a loop you should write yourself: it compiles to a
memmove, which moves whole cache lines at a time (lesson 04). Shifting
1000 ints is Theta(n) but with a very small constant.
```

**Complexity:** `Get` Θ(1) · `Append` Θ(1) amortized · `Insert`/`Delete` **Θ(n)** — and `copy` compiles to a `memmove`, so that Θ(n) has a very small constant

---

## 2. Two ways to delete

`🟢 easy` · *Order preservation is a choice*

Order-preserving delete shifts everything after the hole. Swap-delete moves the last element into it.
One is Θ(n) and one is Θ(1) — and the difference between them is Θ(n²) versus Θ(n) when you're
emptying a collection.

**Steps:**

1. Write both, and compare what they do to `[10 20 30 40 50]`.
2. Count total shifts to empty a slice from the front, each way.
3. Look past `len`, into the capacity, at what a `[]*T` delete leaves behind.

```go
package main

import "fmt"

// There are two ways to delete from a slice, and choosing between them is a
// real design decision rather than a style preference.

type item struct {
	id   int
	name string
}

// DeleteOrdered preserves the order of the remaining elements. Theta(n).
func DeleteOrdered(xs []int, i int) []int {
	copy(xs[i:], xs[i+1:])
	return xs[:len(xs)-1]
}

// DeleteSwap moves the LAST element into the hole. Theta(1) -- and it scrambles
// the order, which is fine for a set, a pool, or an unordered bag.
func DeleteSwap(xs []int, i int) []int {
	xs[i] = xs[len(xs)-1]
	return xs[:len(xs)-1]
}

// --- the pointer hygiene both versions need ---

// DeletePointersLeaky drops the length but leaves the removed pointer sitting in
// the array's spare capacity, where the GC can still see it (lesson 04).
func DeletePointersLeaky(xs []*item, i int) []*item {
	copy(xs[i:], xs[i+1:])
	return xs[:len(xs)-1]
}

// DeletePointersClean nils the vacated slot first, so the removed item can be
// collected.
func DeletePointersClean(xs []*item, i int) []*item {
	copy(xs[i:], xs[i+1:])
	xs[len(xs)-1] = nil // the slot beyond the new length
	return xs[:len(xs)-1]
}

func main() {
	fmt.Println("removing the element at index 1 from [10 20 30 40 50]:")
	fmt.Println()

	a := []int{10, 20, 30, 40, 50}
	fmt.Printf("  ordered:  %v   -- shifts %d elements, order kept\n",
		DeleteOrdered(a, 1), len(a)-2)

	b := []int{10, 20, 30, 40, 50}
	fmt.Printf("  swap:     %v   -- moves 1 element, order scrambled\n",
		DeleteSwap(b, 1))

	fmt.Println()
	fmt.Println("deleting k elements one at a time:")
	fmt.Println()
	fmt.Printf("  %10s %18s %18s\n", "n", "ordered shifts", "swap moves")
	for _, n := range []int{100, 1_000, 10_000} {
		// Worst case for ordered deletion: always remove the front.
		ordered := 0
		for k := n; k > 0; k-- {
			ordered += k - 1
		}
		fmt.Printf("  %10d %18d %18d\n", n, ordered, n)
	}
	fmt.Println()
	fmt.Println("  emptying a slice from the front is Theta(n^2) with ordered deletes")
	fmt.Println("  and Theta(n) with swap-deletes. If you do not need the order, say so.")

	fmt.Println()
	fmt.Println("the pointer trap -- both deletes leave a copy behind:")
	fmt.Println()

	items := []*item{{1, "a"}, {2, "b"}, {3, "c"}}
	leaky := DeletePointersLeaky(items, 0)
	// Look past the length, into the capacity, at what the array still holds.
	fmt.Printf("  after leaky delete:  len=%d cap=%d, slot[%d] still points at %v\n",
		len(leaky), cap(leaky), len(leaky), leaky[:cap(leaky)][len(leaky)])

	items2 := []*item{{1, "a"}, {2, "b"}, {3, "c"}}
	clean := DeletePointersClean(items2, 0)
	fmt.Printf("  after clean delete:  len=%d cap=%d, slot[%d] is %v\n",
		len(clean), cap(clean), len(clean), clean[:cap(clean)][len(clean)])

	fmt.Println()
	fmt.Println("that stale pointer keeps the removed item alive for as long as the")
	fmt.Println("slice lives. For a []int nobody cares; for a []*Session in a")
	fmt.Println("long-lived server it is a leak (lesson 04, example 3).")
	fmt.Println()
	fmt.Println("the rule: if the element type contains a pointer, zero the vacated")
	fmt.Println("slot. `clear(xs[len(xs):len(xs)+1])` works, or just assign the zero value.")

	fmt.Println()
	fmt.Println("choosing:")
	fmt.Println("  need order preserved?      -> ordered delete, Theta(n)")
	fmt.Println("  order is meaningless?      -> swap delete, Theta(1)")
	fmt.Println("  removing MANY elements?    -> neither: one filtering pass (example 3)")
}
```

**Output:**

```
removing the element at index 1 from [10 20 30 40 50]:

  ordered:  [10 30 40 50]   -- shifts 3 elements, order kept
  swap:     [10 50 30 40]   -- moves 1 element, order scrambled

deleting k elements one at a time:

           n     ordered shifts         swap moves
         100               4950                100
        1000             499500               1000
       10000           49995000              10000

  emptying a slice from the front is Theta(n^2) with ordered deletes
  and Theta(n) with swap-deletes. If you do not need the order, say so.

the pointer trap -- both deletes leave a copy behind:

  after leaky delete:  len=2 cap=3, slot[2] still points at &{3 c}
  after clean delete:  len=2 cap=3, slot[2] is <nil>

that stale pointer keeps the removed item alive for as long as the
slice lives. For a []int nobody cares; for a []*Session in a
long-lived server it is a leak (lesson 04, example 3).

the rule: if the element type contains a pointer, zero the vacated
slot. `clear(xs[len(xs):len(xs)+1])` works, or just assign the zero value.

choosing:
  need order preserved?      -> ordered delete, Theta(n)
  order is meaningless?      -> swap delete, Theta(1)
  removing MANY elements?    -> neither: one filtering pass (example 3)
```

That stale pointer keeps the removed item alive as long as the slice lives. For `[]int` nobody cares;
for `[]*Session` in a long-lived server it's the leak from [lesson 04](../04-go-memory-model/1-easy.md#3-ten-bytes-that-pin-eight-megabytes).

**Complexity:** ordered delete Θ(n) · swap delete **Θ(1)** · emptying from the front: Θ(n²) vs Θ(n)

---

## 3. The write-index idiom

`🟢 easy` · *The single most useful slice pattern*

Two cursors over one array: `read` visits everything, `write` marks where the next kept element goes.
Because `write` never overtakes `read`, you can overwrite in place — no second slice, no allocation.

Learn this one and you'll use it weekly.

**Steps:**

1. Write the filter with two cursors.
2. Write the delete-inside-the-loop version and find an input where it's *wrong*.
3. Apply the same five lines to dedupe, move-zeroes and remove-all.

```go
package main

import "fmt"

// THE slice idiom. Learn this one and you will reach for it weekly.
//
// Two cursors walk the same array: `read` visits every element, `write` marks
// where the next KEPT element goes. Because write never overtakes read, you can
// overwrite in place -- no second slice, no allocation.
//
//   for read := range xs {
//       if keep(xs[read]) {
//           xs[write] = xs[read]
//           write++
//       }
//   }
//   return xs[:write]

// FilterInPlace keeps the elements satisfying keep, in order, using no extra
// memory. Theta(n) time, Theta(1) space.
func FilterInPlace(xs []int, keep func(int) bool) []int {
	write := 0
	for read := 0; read < len(xs); read++ {
		if keep(xs[read]) {
			xs[write] = xs[read]
			write++
		}
	}
	return xs[:write]
}

// FilterAllocating is the version most people write first. Same answer,
// a whole new array.
func FilterAllocating(xs []int, keep func(int) bool) []int {
	var out []int
	for _, v := range xs {
		if keep(v) {
			out = append(out, v)
		}
	}
	return out
}

// FilterRepeatedDelete is the version to never write: deleting inside the loop
// is Theta(n) per removal, so the whole thing is Theta(n^2) -- plus it skips
// elements, because everything shifts under the cursor.
func FilterRepeatedDelete(xs []int, keep func(int) bool) []int {
	for i := 0; i < len(xs); i++ {
		if !keep(xs[i]) {
			copy(xs[i:], xs[i+1:])
			xs = xs[:len(xs)-1]
			// BUG: i now points at an element we never examined.
		}
	}
	return xs
}

func even(v int) bool { return v%2 == 0 }

func main() {
	src := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

	a := append([]int(nil), src...)
	fmt.Println("keep the even numbers from", src)
	fmt.Println()
	fmt.Printf("  in place:          %v\n", FilterInPlace(a, even))
	fmt.Printf("  allocating:        %v\n", FilterAllocating(src, even))

	// The buggy one needs consecutive removals to show its teeth.
	bad := []int{1, 3, 5, 2, 4, 7}
	fmt.Println()
	fmt.Println("now with CONSECUTIVE elements to remove, [1 3 5 2 4 7]:")
	fmt.Printf("  in place:          %v   <- correct\n",
		FilterInPlace(append([]int(nil), bad...), even))
	fmt.Printf("  delete-in-loop:    %v   <- WRONG: skipped elements\n",
		FilterRepeatedDelete(append([]int(nil), bad...), even))
	fmt.Println()
	fmt.Println("  after removing index i, the next element slides INTO index i --")
	fmt.Println("  and then i++ steps right over it. (Iterating backwards fixes the")
	fmt.Println("  correctness bug but leaves it Theta(n^2); see example 14.)")

	fmt.Println()
	fmt.Println("the same two-cursor shape solves a whole family of problems:")
	fmt.Println()

	// dedupe a sorted slice
	sorted := []int{1, 1, 2, 3, 3, 3, 4, 5, 5}
	fmt.Printf("  dedupe sorted      %v -> %v\n", sorted, dedupe(append([]int(nil), sorted...)))

	// move zeroes to the end, keeping the order of the rest
	zeros := []int{0, 1, 0, 3, 12}
	fmt.Printf("  move zeroes        %v -> %v\n", zeros, moveZeroes(append([]int(nil), zeros...)))

	// remove ALL occurrences of a value
	withDup := []int{3, 2, 2, 3, 1, 3}
	fmt.Printf("  remove all 3s      %v -> %v\n", withDup, removeAll(append([]int(nil), withDup...), 3))

	fmt.Println()
	fmt.Println("all four are the same five lines with a different `keep` condition.")
	fmt.Println("Theta(n) time, Theta(1) space, and no allocation at all -- which is why")
	fmt.Println("this beats the allocating version even though both are Theta(n).")
}

// dedupe removes adjacent duplicates. Requires sorted input.
func dedupe(xs []int) []int {
	if len(xs) == 0 {
		return xs
	}
	write := 1
	for read := 1; read < len(xs); read++ {
		if xs[read] != xs[write-1] {
			xs[write] = xs[read]
			write++
		}
	}
	return xs[:write]
}

// moveZeroes shifts every non-zero left, then fills the tail with zeroes.
func moveZeroes(xs []int) []int {
	write := 0
	for read := 0; read < len(xs); read++ {
		if xs[read] != 0 {
			xs[write] = xs[read]
			write++
		}
	}
	for i := write; i < len(xs); i++ {
		xs[i] = 0
	}
	return xs
}

// removeAll drops every element equal to v.
func removeAll(xs []int, v int) []int {
	write := 0
	for read := 0; read < len(xs); read++ {
		if xs[read] != v {
			xs[write] = xs[read]
			write++
		}
	}
	return xs[:write]
}
```

**Output:**

```
keep the even numbers from [1 2 3 4 5 6 7 8 9 10]

  in place:          [2 4 6 8 10]
  allocating:        [2 4 6 8 10]

now with CONSECUTIVE elements to remove, [1 3 5 2 4 7]:
  in place:          [2 4]   <- correct
  delete-in-loop:    [3 2 4]   <- WRONG: skipped elements

  after removing index i, the next element slides INTO index i --
  and then i++ steps right over it. (Iterating backwards fixes the
  correctness bug but leaves it Theta(n^2); see example 14.)

the same two-cursor shape solves a whole family of problems:

  dedupe sorted      [1 1 2 3 3 3 4 5 5] -> [1 2 3 4 5]
  move zeroes        [0 1 0 3 12] -> [1 3 12 0 0]
  remove all 3s      [3 2 2 3 1 3] -> [2 2 1]

all four are the same five lines with a different `keep` condition.
Theta(n) time, Theta(1) space, and no allocation at all -- which is why
this beats the allocating version even though both are Theta(n).
```

The buggy version isn't just slow, it's **incorrect**: after removing index `i` the next element
slides into `i`, and then `i++` steps straight over it. Iterating backwards fixes the correctness bug
and leaves it Θ(n²) — see example 14.

**Complexity:** Θ(n) time, **Θ(1) space, zero allocations** · the delete-in-loop version is Θ(n²) *and* wrong

---

## 4. Reverse and rotate in place

`🟢 easy` · *Θ(1) space rearrangement*

Reverse is two cursors meeting in the middle. Rotate is three reversals — a trick worth memorizing,
because there is no `slices.Rotate`.

**Steps:**

1. Reverse with two cursors; note that odd lengths need no special case.
2. Rotate by reversing the first k, then the rest, then everything.
3. Test every awkward k: 0, n, n+2, and negative.

```go
package main

import "fmt"

// Two in-place rearrangements worth knowing cold. Both are Theta(n) time and
// Theta(1) space, and the second is built entirely out of the first.

// Reverse swaps from both ends inward. The loop ends when the cursors meet,
// which is why a middle element (odd length) needs no special case.
func Reverse(xs []int) {
	for i, j := 0, len(xs)-1; i < j; i, j = i+1, j-1 {
		xs[i], xs[j] = xs[j], xs[i]
	}
}

// RotateLeft shifts every element k places toward the front, wrapping around.
//
// The trick: reverse the first k, reverse the rest, then reverse the whole
// thing. Three linear passes, no extra memory.
//
//	[1 2 3 4 5] k=2
//	reverse [1 2]      -> [2 1 | 3 4 5]
//	reverse [3 4 5]    -> [2 1 | 5 4 3]
//	reverse everything -> [3 4 5 1 2]
func RotateLeft(xs []int, k int) {
	n := len(xs)
	if n == 0 {
		return
	}
	k = ((k % n) + n) % n // normalize, including negative k
	Reverse(xs[:k])
	Reverse(xs[k:])
	Reverse(xs)
}

// RotateCopy is the obvious alternative: allocate and place each element
// directly. Also Theta(n) time -- but Theta(n) SPACE.
func RotateCopy(xs []int, k int) []int {
	n := len(xs)
	if n == 0 {
		return xs
	}
	k = ((k % n) + n) % n
	out := make([]int, n)
	for i, v := range xs {
		out[(i-k+n)%n] = v
	}
	return out
}

func show(label string, xs []int) { fmt.Printf("  %-28s %v\n", label, xs) }

func main() {
	fmt.Println("reverse, in place:")
	for _, xs := range [][]int{
		{},
		{1},
		{1, 2},
		{1, 2, 3},
		{1, 2, 3, 4, 5, 6},
	} {
		before := fmt.Sprint(xs)
		Reverse(xs)
		fmt.Printf("  %-16s -> %v\n", before, xs)
	}
	fmt.Println("  (no special case for odd length: the cursors meet on the middle")
	fmt.Println("   element and the loop condition i < j stops before swapping it)")

	fmt.Println()
	fmt.Println("rotate left by 2, via triple reversal:")
	fmt.Println()
	xs := []int{1, 2, 3, 4, 5}
	show("start", xs)
	Reverse(xs[:2])
	show("reverse the first k", xs)
	Reverse(xs[2:])
	show("reverse the rest", xs)
	Reverse(xs)
	show("reverse everything", xs)

	fmt.Println()
	fmt.Println("every k, including the awkward ones:")
	fmt.Println()
	for _, k := range []int{0, 1, 4, 5, 7, -1} {
		a := []int{1, 2, 3, 4, 5}
		RotateLeft(a, k)
		b := RotateCopy([]int{1, 2, 3, 4, 5}, k)
		agree := fmt.Sprint(a) == fmt.Sprint(b)
		fmt.Printf("  k=%-3d -> %v   copy version agrees: %v\n", k, a, agree)
	}

	fmt.Println()
	fmt.Println("k=5 and k=0 give the identity; k=7 is the same as k=2; k=-1 rotates")
	fmt.Println("right. That is what `k = ((k % n) + n) % n` buys -- Go's % keeps the")
	fmt.Println("sign of the dividend, so the extra +n and % are not optional.")

	fmt.Println()
	fmt.Println("why the triple reversal instead of the copy:")
	fmt.Println("  both are Theta(n) TIME. The reversal is Theta(1) SPACE, and it")
	fmt.Println("  touches memory sequentially three times, which is exactly the")
	fmt.Println("  access pattern lesson 04 showed is nearly free.")
	fmt.Println()
	fmt.Println("  the copy version needs a whole second array, and its writes jump")
	fmt.Println("  around by (i-k+n)%n -- a scattered pattern.")
	fmt.Println()
	fmt.Println("stdlib: slices.Reverse does the first one. There is no slices.Rotate,")
	fmt.Println("so the triple reversal is worth remembering.")
}
```

**Output:**

```
reverse, in place:
  []               -> []
  [1]              -> [1]
  [1 2]            -> [2 1]
  [1 2 3]          -> [3 2 1]
  [1 2 3 4 5 6]    -> [6 5 4 3 2 1]
  (no special case for odd length: the cursors meet on the middle
   element and the loop condition i < j stops before swapping it)

rotate left by 2, via triple reversal:

  start                        [1 2 3 4 5]
  reverse the first k          [2 1 3 4 5]
  reverse the rest             [2 1 5 4 3]
  reverse everything           [3 4 5 1 2]

every k, including the awkward ones:

  k=0   -> [1 2 3 4 5]   copy version agrees: true
  k=1   -> [2 3 4 5 1]   copy version agrees: true
  k=4   -> [5 1 2 3 4]   copy version agrees: true
  k=5   -> [1 2 3 4 5]   copy version agrees: true
  k=7   -> [3 4 5 1 2]   copy version agrees: true
  k=-1  -> [5 1 2 3 4]   copy version agrees: true

k=5 and k=0 give the identity; k=7 is the same as k=2; k=-1 rotates
right. That is what `k = ((k % n) + n) % n` buys -- Go's % keeps the
sign of the dividend, so the extra +n and % are not optional.

why the triple reversal instead of the copy:
  both are Theta(n) TIME. The reversal is Theta(1) SPACE, and it
  touches memory sequentially three times, which is exactly the
  access pattern lesson 04 showed is nearly free.

  the copy version needs a whole second array, and its writes jump
  around by (i-k+n)%n -- a scattered pattern.

stdlib: slices.Reverse does the first one. There is no slices.Rotate,
so the triple reversal is worth remembering.
```

**Complexity:** both Θ(n) time · reversal is **Θ(1) space** and three sequential passes; the copy version is Θ(n) space with scattered writes

---

## 5. A sorted slice is a data structure

`🟢 easy` · *Θ(log n) to find, Θ(n) to make room*

Kept in order, a slice answers questions a map cannot: predecessor, successor, rank, and range. The
cost of maintaining it splits cleanly in two — and only one half is expensive.

**Steps:**

1. Insert by binary-searching for the position, then shifting.
2. Answer the queries a map can't.
3. Count comparisons and shifts separately as n grows.

```go
package main

import (
	"fmt"
	"slices"
	"sort"
)

// A SORTED SLICE is a real data structure, not just a slice you sorted once.
// Kept in order, it answers questions a map cannot: predecessor, successor,
// rank, and "everything between a and b" (lesson 04, example 15).
//
// The cost of maintaining it is the interesting part, and it splits in two:
//
//   FIND the position   Theta(log n)   binary search
//   MAKE room for it    Theta(n)       shift the tail
//
// So insertion is Theta(n) -- but with two very different constants. The search
// is a handful of cache misses; the shift is one memmove.

var comparisons, shifted int

// searchPos returns the index where v belongs, keeping xs sorted.
// This is sort.Search's contract: the first index where the predicate is true.
func searchPos(xs []int, v int) int {
	lo, hi := 0, len(xs)
	for lo < hi {
		comparisons++
		mid := lo + (hi-lo)/2
		if xs[mid] < v {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

// SortedInsert puts v in its place and keeps the slice ordered.
func SortedInsert(xs []int, v int) []int {
	i := searchPos(xs, v)
	shifted += len(xs) - i

	xs = append(xs, 0)
	copy(xs[i+1:], xs[i:])
	xs[i] = v
	return xs
}

func main() {
	fmt.Println("building a sorted slice by inserting one value at a time:")
	fmt.Println()

	var xs []int
	for _, v := range []int{5, 2, 8, 1, 9, 3} {
		xs = SortedInsert(xs, v)
		fmt.Printf("  insert %d -> %v\n", v, xs)
	}

	fmt.Println()
	fmt.Println("what a sorted slice can answer that a map cannot:")
	fmt.Println()
	fmt.Printf("  contents:                %v\n", xs)

	i, found := slices.BinarySearch(xs, 8)
	fmt.Printf("  is 8 present?            %v (at index %d)\n", found, i)

	i, found = slices.BinarySearch(xs, 6)
	fmt.Printf("  is 6 present?            %v\n", found)
	fmt.Printf("  where would 6 go?        index %d\n", i)
	fmt.Printf("  predecessor of 6:        %d\n", xs[i-1])
	fmt.Printf("  successor of 6:          %d\n", xs[i])

	lo := sort.SearchInts(xs, 3)
	hi := sort.SearchInts(xs, 9)
	fmt.Printf("  everything in [3, 9):    %v\n", xs[lo:hi])
	fmt.Printf("  the 3rd smallest:        %d\n", xs[2])
	fmt.Printf("  how many are below 8?    %d\n", sort.SearchInts(xs, 8))

	fmt.Println()
	fmt.Println("none of those are possible with a map at any price -- a map has no order.")

	fmt.Println()
	fmt.Println("the cost of maintaining the order, counted:")
	fmt.Println()
	fmt.Printf("  %10s %16s %18s %16s\n", "n", "comparisons", "elements shifted", "shifts/insert")
	for _, n := range []int{100, 1_000, 10_000} {
		comparisons, shifted = 0, 0
		var s []int
		// Worst case for shifting: every new value belongs at the front.
		for v := n; v > 0; v-- {
			s = SortedInsert(s, v)
		}
		fmt.Printf("  %10d %16d %18d %16.1f\n",
			n, comparisons, shifted, float64(shifted)/float64(n))
	}

	fmt.Println()
	fmt.Println("comparisons grow like n log n -- that is the cheap half.")
	fmt.Println("elements shifted grow like n^2/2 -- that is the expensive half.")
	fmt.Println()
	fmt.Println("so building a sorted slice by repeated insertion is Theta(n^2).")
	fmt.Println("If you have all the data up front, do NOT do this: append everything")
	fmt.Println("and call slices.Sort once, which is Theta(n log n).")

	fmt.Println()
	fmt.Println("sorted insertion earns its keep only when the collection is")
	fmt.Println("read-mostly and grows slowly -- the exact case where the")
	fmt.Println("Theta(n) shift is rare and the Theta(log n) queries are constant.")
	fmt.Println("Example 12 measures where that stops being true.")
}
```

**Output:**

```
building a sorted slice by inserting one value at a time:

  insert 5 -> [5]
  insert 2 -> [2 5]
  insert 8 -> [2 5 8]
  insert 1 -> [1 2 5 8]
  insert 9 -> [1 2 5 8 9]
  insert 3 -> [1 2 3 5 8 9]

what a sorted slice can answer that a map cannot:

  contents:                [1 2 3 5 8 9]
  is 8 present?            true (at index 4)
  is 6 present?            false
  where would 6 go?        index 4
  predecessor of 6:        5
  successor of 6:          8
  everything in [3, 9):    [3 5 8]
  the 3rd smallest:        3
  how many are below 8?    4

none of those are possible with a map at any price -- a map has no order.

the cost of maintaining the order, counted:

           n      comparisons   elements shifted    shifts/insert
         100              573               4950             49.5
        1000             8977             499500            499.5
       10000           123617           49995000           4999.5

comparisons grow like n log n -- that is the cheap half.
elements shifted grow like n^2/2 -- that is the expensive half.

so building a sorted slice by repeated insertion is Theta(n^2).
If you have all the data up front, do NOT do this: append everything
and call slices.Sort once, which is Theta(n log n).

sorted insertion earns its keep only when the collection is
read-mostly and grows slowly -- the exact case where the
Theta(n) shift is rare and the Theta(log n) queries are constant.
Example 12 measures where that stops being true.
```

Comparisons grow like n log n — the cheap half. Elements shifted grow like n²/2 — the expensive half.
So building a sorted slice by repeated insertion is **Θ(n²)**: if you have the data up front, append
everything and sort once.

**Complexity:** find Θ(log n) · insert **Θ(n)** · build-by-insertion Θ(n²) · build-then-sort Θ(n log n)

---

## 6. What the stdlib already has

`🟢 easy` · *`slices`, and the cost column*

Everything above exists in the `slices` package, generic and tested. Build it once to know the cost;
call these afterwards. The point of this example is the **cost column** — `slices.Insert` is
convenient and it is still Θ(n).

**Steps:**

1. Map each hand-written function to its stdlib equivalent, with its cost.
2. Run them.
3. Find the two places where the convenient call is still a trap.

```go
package main

import (
	"cmp"
	"fmt"
	"slices"
)

// Everything examples 1-5 built by hand already exists in the `slices` package,
// generic and tested. Build them once to understand the cost; call these after.
//
// The point of this example is the COST COLUMN. `slices.Insert` is convenient
// and it is still Theta(n) -- the standard library cannot repeal the shift.

type person struct {
	name string
	age  int
}

func main() {
	fmt.Println("what you built, and what to call instead:")
	fmt.Println()
	fmt.Printf("  %-24s %-28s %s\n", "example", "stdlib", "cost")
	rows := [][3]string{
		{"1. Insert(xs, i, v)", "slices.Insert(xs, i, v...)", "Theta(n)"},
		{"1. Delete(xs, i)", "slices.Delete(xs, i, j)", "Theta(n)"},
		{"3. dedupe sorted", "slices.Compact(xs)", "Theta(n)"},
		{"4. Reverse(xs)", "slices.Reverse(xs)", "Theta(n)"},
		{"5. searchPos(xs, v)", "slices.BinarySearch(xs, v)", "Theta(log n)"},
		{"-- (clone)", "slices.Clone(xs)", "Theta(n)"},
		{"-- (compare)", "slices.Equal(a, b)", "Theta(n)"},
		{"-- (find)", "slices.Index / Contains", "Theta(n)"},
		{"-- (sort)", "slices.Sort / SortFunc", "Theta(n log n)"},
		{"-- (min/max)", "slices.Min / Max", "Theta(n)"},
	}
	for _, r := range rows {
		fmt.Printf("  %-24s %-28s %s\n", r[0], r[1], r[2])
	}

	fmt.Println()
	fmt.Println("the same operations, run:")
	fmt.Println()

	xs := []int{3, 1, 4, 1, 5, 9, 2, 6}
	fmt.Printf("  start                       %v\n", xs)
	fmt.Printf("  slices.Index(xs, 4)         %d\n", slices.Index(xs, 4))
	fmt.Printf("  slices.Contains(xs, 7)      %v\n", slices.Contains(xs, 7))
	fmt.Printf("  slices.Max / Min            %d / %d\n", slices.Max(xs), slices.Min(xs))

	ins := slices.Insert(slices.Clone(xs), 2, 99, 98)
	fmt.Printf("  slices.Insert(_, 2, 99, 98) %v   <- inserts several at once\n", ins)

	del := slices.Delete(slices.Clone(xs), 1, 4)
	fmt.Printf("  slices.Delete(_, 1, 4)      %v   <- a RANGE, not one index\n", del)

	sorted := slices.Clone(xs)
	slices.Sort(sorted)
	fmt.Printf("  slices.Sort                 %v\n", sorted)
	fmt.Printf("  slices.Compact (dedupe)     %v   <- adjacent only, so sort first\n",
		slices.Compact(slices.Clone(sorted)))

	i, ok := slices.BinarySearch(sorted, 5)
	fmt.Printf("  slices.BinarySearch(_, 5)   index %d, found %v\n", i, ok)

	rev := slices.Clone(xs)
	slices.Reverse(rev)
	fmt.Printf("  slices.Reverse              %v\n", rev)

	fmt.Println()
	fmt.Println("sorting by a field, and by several:")
	fmt.Println()
	people := []person{{"ana", 30}, {"bo", 25}, {"cy", 30}, {"di", 25}}
	fmt.Printf("  start           %v\n", people)

	byAge := slices.Clone(people)
	slices.SortFunc(byAge, func(a, b person) int { return cmp.Compare(a.age, b.age) })
	fmt.Printf("  by age          %v\n", byAge)

	byAgeName := slices.Clone(people)
	slices.SortFunc(byAgeName, func(a, b person) int {
		return cmp.Or(
			cmp.Compare(a.age, b.age),
			cmp.Compare(a.name, b.name),
		)
	})
	fmt.Printf("  by age, name    %v   <- cmp.Or chains tie-breakers\n", byAgeName)

	stable := slices.Clone(people)
	slices.SortStableFunc(stable, func(a, b person) int { return cmp.Compare(a.age, b.age) })
	fmt.Printf("  stable by age   %v\n", stable)
	fmt.Println("                  ^ identical here -- but SortFunc gives NO stability")
	fmt.Println("                    guarantee, so use SortStableFunc when ties must")
	fmt.Println("                    keep their input order. It is not free: it is a")
	fmt.Println("                    different algorithm with more bookkeeping.")

	fmt.Println()
	fmt.Println("two things the table above is really telling you:")
	fmt.Println()
	fmt.Println("  1. slices.Insert and slices.Delete are STILL Theta(n). Calling a")
	fmt.Println("     stdlib function does not remove the shift -- so a loop that")
	fmt.Println("     calls slices.Delete n times is still Theta(n^2) (example 14).")
	fmt.Println()
	fmt.Println("  2. slices.Compact only removes ADJACENT duplicates, exactly like")
	fmt.Println("     the hand-written version in example 3. On unsorted input it")
	fmt.Println("     quietly does almost nothing. Sort first, or use a map.")

	unsorted := []int{1, 2, 1, 2, 1}
	fmt.Printf("\n     slices.Compact(%v) = %v  <- not a dedupe!\n",
		unsorted, slices.Compact(slices.Clone(unsorted)))

	fmt.Println()
	fmt.Println("use the stdlib. Build it once first, so you know what it costs.")
}
```

**Output:**

```
what you built, and what to call instead:

  example                  stdlib                       cost
  1. Insert(xs, i, v)      slices.Insert(xs, i, v...)   Theta(n)
  1. Delete(xs, i)         slices.Delete(xs, i, j)      Theta(n)
  3. dedupe sorted         slices.Compact(xs)           Theta(n)
  4. Reverse(xs)           slices.Reverse(xs)           Theta(n)
  5. searchPos(xs, v)      slices.BinarySearch(xs, v)   Theta(log n)
  -- (clone)               slices.Clone(xs)             Theta(n)
  -- (compare)             slices.Equal(a, b)           Theta(n)
  -- (find)                slices.Index / Contains      Theta(n)
  -- (sort)                slices.Sort / SortFunc       Theta(n log n)
  -- (min/max)             slices.Min / Max             Theta(n)

the same operations, run:

  start                       [3 1 4 1 5 9 2 6]
  slices.Index(xs, 4)         2
  slices.Contains(xs, 7)      false
  slices.Max / Min            9 / 1
  slices.Insert(_, 2, 99, 98) [3 1 99 98 4 1 5 9 2 6]   <- inserts several at once
  slices.Delete(_, 1, 4)      [3 5 9 2 6]   <- a RANGE, not one index
  slices.Sort                 [1 1 2 3 4 5 6 9]
  slices.Compact (dedupe)     [1 2 3 4 5 6 9]   <- adjacent only, so sort first
  slices.BinarySearch(_, 5)   index 5, found true
  slices.Reverse              [6 2 9 5 1 4 1 3]

sorting by a field, and by several:

  start           [{ana 30} {bo 25} {cy 30} {di 25}]
  by age          [{bo 25} {di 25} {ana 30} {cy 30}]
  by age, name    [{bo 25} {di 25} {ana 30} {cy 30}]   <- cmp.Or chains tie-breakers
  stable by age   [{bo 25} {di 25} {ana 30} {cy 30}]
                  ^ identical here -- but SortFunc gives NO stability
                    guarantee, so use SortStableFunc when ties must
                    keep their input order. It is not free: it is a
                    different algorithm with more bookkeeping.

two things the table above is really telling you:

  1. slices.Insert and slices.Delete are STILL Theta(n). Calling a
     stdlib function does not remove the shift -- so a loop that
     calls slices.Delete n times is still Theta(n^2) (example 14).

  2. slices.Compact only removes ADJACENT duplicates, exactly like
     the hand-written version in example 3. On unsorted input it
     quietly does almost nothing. Sort first, or use a map.

     slices.Compact([1 2 1 2 1]) = [1 2 1 2 1]  <- not a dedupe!

use the stdlib. Build it once first, so you know what it costs.
```

Two traps worth carrying forward: **`slices.Delete` in a loop is still Θ(n²)** (example 14), and
**`slices.Compact` only removes adjacent duplicates** — on unsorted input it quietly does almost
nothing.

**Complexity:** the stdlib does not repeal the shift · use it for correctness, and know what each call costs

---

> Next tier: [🟡 medium](2-medium.md) — building the structure Go hands you for free.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
