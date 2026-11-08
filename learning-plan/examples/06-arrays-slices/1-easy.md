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

**Rust:**

```rust
use std::fmt::Display;

// A dynamic array is the structure you already use every day. As an ADT it owes
// you four operations, and their costs are not equal:
//
//   Get(i)        Theta(1)   -- one address calculation (plus a bounds check)
//   Append(v)     Theta(1)   -- amortized (lesson 03)
//   Insert(i, v)  Theta(n)   -- everything from i onward shifts right
//   Delete(i)     Theta(n)   -- everything after i shifts left
//
// Rust gives you Get and push as language/std features, and Vec::insert and
// Vec::remove for the other two. We write them by hand anyway, so the shape is
// visible -- and both are one `copy_within` call, Rust's memmove on a slice.
// (copy_within needs T: Copy, which is fine for integers; Vec::insert/remove
// do the same shift with ptr::copy and work for any T.)

/// Go's %v of a slice: [1 2 3]
fn list<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

/// Insert puts v at index i, shifting the tail right.
fn insert(xs: &mut Vec<i64>, i: usize, v: i64) {
    let n = xs.len();
    xs.push(0); // grow by one; the value does not matter
    xs.copy_within(i..n, i + 1); // shift the tail right by one
    xs[i] = v; // drop v into the hole
}

/// Delete removes index i, shifting the tail left. Order is preserved.
fn delete(xs: &mut Vec<i64>, i: usize) {
    let n = xs.len();
    xs.copy_within(i + 1..n, i); // shift the tail left over the hole
    xs.truncate(n - 1); // and shorten
}

#[derive(Clone, Copy)]
enum Op {
    Insert,
    Delete,
}

/// count_shifts reports how many elements copy_within has to move.
fn count_shifts(n: usize, i: usize, op: Op) -> usize {
    match op {
        Op::Insert => n - i,
        Op::Delete => n - i - 1,
    }
}

fn main() {
    let mut xs: Vec<i64> = vec![10, 20, 30, 40, 50];
    println!("start:             {}", list(&xs));

    println!();
    println!("Get is one address calculation -- Theta(1), no matter where:");
    println!("  xs[0] = {}, xs[4] = {}", xs[0], xs[4]);

    println!();
    println!("Insert shifts everything from i onward:");
    insert(&mut xs, 2, 99);
    println!("  Insert(xs, 2, 99): {}", list(&xs));
    insert(&mut xs, 0, 5);
    println!(
        "  Insert(xs, 0, 5):  {}  <- worst case: shifted all of them",
        list(&xs)
    );
    let end = xs.len(); // `insert(&mut xs, xs.len(), 60)` is a borrow error: &mut xs is taken first
    insert(&mut xs, end, 60);
    println!(
        "  Insert at the end: {}  <- best case: shifted none (this is push)",
        list(&xs)
    );

    println!();
    println!("Delete shifts everything after i:");
    delete(&mut xs, 0);
    println!("  Delete(xs, 0):     {}", list(&xs));
    delete(&mut xs, 2);
    println!("  Delete(xs, 2):     {}", list(&xs));

    println!();
    println!("the cost is entirely about WHERE, not how many:");
    println!();
    const N: usize = 1000;
    println!(
        "  {:<22} {:>14} {:>14}",
        "position", "insert shifts", "delete shifts"
    );
    for i in [0, N / 4, N / 2, N - 1, N] {
        let mut label = format!("i = {i}");
        match i {
            0 => label += " (front)",
            N => label += " (end)",
            _ => {}
        }
        let ins = count_shifts(N, i, Op::Insert);
        // deleting at i == N is not a delete at all (there is no element N)
        let del = if i == N {
            0
        } else {
            count_shifts(N, i, Op::Delete)
        };
        println!("  {label:<22} {ins:>14} {del:>14}");
    }

    println!();
    println!("this is THE trade-off of a dynamic array, and it never goes away:");
    println!("  random access is free, and structural change in the middle is not.");
    println!();
    println!("everything else in Part 2 is a different answer to that one sentence:");
    println!("  a linked list  (07) makes insert O(1) and access O(n) -- the mirror image");
    println!("  a stack        (08) only ever touches the cheap END");
    println!("  a ring buffer  (09) makes BOTH ends cheap by giving up contiguity");
    println!();
    println!("note copy_within() is not a loop you should write yourself: it is");
    println!("ptr::copy, and a release build turns it into one memmove call (lesson 04).");
    println!("Shifting 1000 ints is still Theta(n), but it is one call, not your loop.");
}
```

**Rust output:**

```
start:             [10 20 30 40 50]

Get is one address calculation -- Theta(1), no matter where:
  xs[0] = 10, xs[4] = 50

Insert shifts everything from i onward:
  Insert(xs, 2, 99): [10 20 99 30 40 50]
  Insert(xs, 0, 5):  [5 10 20 99 30 40 50]  <- worst case: shifted all of them
  Insert at the end: [5 10 20 99 30 40 50 60]  <- best case: shifted none (this is push)

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

note copy_within() is not a loop you should write yourself: it is
ptr::copy, and a release build turns it into one memmove call (lesson 04).
Shifting 1000 ints is still Theta(n), but it is one call, not your loop.
```

> **Go vs Rust:** The output is the Go output line for line, except `append` becomes `push` and the closing note names `ptr::copy`: `copy_within` is a slice method (integers only, `T: Copy`), and the std `Vec::insert`/`Vec::remove` do the same shift for any `T`. Go's `Insert` returns a slice the caller must remember to keep, because `append` may have reallocated; Rust's takes `&mut Vec`, so the caller's `Vec` is the one that grew. In the release assembly, `delete` contains one `memmove` call.

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

**Rust:**

```rust
use std::fmt::Display;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// There are two ways to delete from a Vec, and choosing between them is a
// real design decision rather than a style preference. (std has both:
// Vec::remove is the ordered one, Vec::swap_remove the swap one. We write them
// by hand, as Go does, so the cost is visible.)

/// Go's %v of a slice: [1 2 3]
fn list<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

/// delete_ordered preserves the order of the remaining elements. Theta(n).
/// Returns how many elements it had to shift.
fn delete_ordered(xs: &mut Vec<i64>, i: usize) -> usize {
    let n = xs.len();
    xs.copy_within(i + 1..n, i);
    xs.truncate(n - 1);
    n - i - 1
}

/// delete_swap moves the LAST element into the hole. Theta(1) -- and it
/// scrambles the order, which is fine for a set, a pool, or an unordered bag.
fn delete_swap(xs: &mut Vec<i64>, i: usize) {
    let last = xs.len() - 1;
    xs[i] = xs[last];
    xs.truncate(last);
}

// --- the pointer hygiene ---
//
// Go's trap: copy() DUPLICATES a pointer, truncating the length does not touch
// the slot past it, and the GC keeps whatever that slot points at alive.
//
// Rust has no such slot to forget. Items live in a Vec<Box<Item>>; Box is
// owned, not copyable, so a "shift" can only MOVE it, and Vec::remove,
// truncate and pop DROP what they remove on the spot. The memory past len is
// uninitialised (spare_capacity_mut() hands it out as MaybeUninit), so safe
// code cannot even look at it.
//
// The trap comes back only when YOU keep the length: a stack, a ring buffer,
// or any structure built on a buffer plus a `len` field (lessons 08-09). Then
// "delete" is your code, and so is the vacated slot. `Buf` below is that shape.

/// How many Items exist right now (stand-in for "has the GC freed it").
static ALIVE: AtomicUsize = AtomicUsize::new(0);

struct Item {
    id: u32,
    name: &'static str,
}

impl Item {
    fn boxed(id: u32, name: &'static str) -> Box<Item> {
        ALIVE.fetch_add(1, Relaxed);
        Box::new(Item { id, name })
    }
}

impl Drop for Item {
    fn drop(&mut self) {
        ALIVE.fetch_sub(1, Relaxed);
    }
}

fn alive() -> usize {
    ALIVE.load(Relaxed)
}

fn describe(slot: &Option<Box<Item>>) -> String {
    match slot {
        Some(it) => format!("Some(Item {{ id: {}, name: {:?} }})", it.id, it.name),
        None => "None".to_string(),
    }
}

/// A buffer that tracks its own length, like a Go slice does.
struct Buf {
    slots: Vec<Option<Box<Item>>>,
    len: usize,
}

impl Buf {
    fn new() -> Buf {
        let slots = vec![
            Some(Item::boxed(1, "a")),
            Some(Item::boxed(2, "b")),
            Some(Item::boxed(3, "c")),
        ];
        Buf {
            len: slots.len(),
            slots,
        }
    }

    /// delete_leaky drops the length but leaves the removed Box in the slot
    /// past it: the same bug as Go's DeletePointersLeaky.
    fn delete_leaky(&mut self, i: usize) {
        self.slots[i..self.len].rotate_left(1); // the removed Box rotates to the end
        self.len -= 1;
    }

    /// delete_clean also empties the vacated slot, so the removed item is dropped.
    fn delete_clean(&mut self, i: usize) {
        self.delete_leaky(i);
        self.slots[self.len] = None; // the slot beyond the new length
    }

    fn report(&self, label: &str, verdict: &str) {
        println!(
            "  {label:<19}  len={} cap={}, slot[{}] {verdict} {}, items alive: {}",
            self.len,
            self.slots.len(),
            self.len,
            describe(&self.slots[self.len]),
            alive()
        );
    }
}

fn main() {
    println!("removing the element at index 1 from [10 20 30 40 50]:");
    println!();

    let mut a: Vec<i64> = vec![10, 20, 30, 40, 50];
    let shifted = delete_ordered(&mut a, 1);
    println!(
        "  ordered:  {}   -- shifts {shifted} elements, order kept",
        list(&a)
    );

    let mut b: Vec<i64> = vec![10, 20, 30, 40, 50];
    delete_swap(&mut b, 1);
    println!(
        "  swap:     {}   -- moves 1 element, order scrambled",
        list(&b)
    );

    println!();
    println!("deleting k elements one at a time:");
    println!();
    println!(
        "  {:>10} {:>18} {:>18}",
        "n", "ordered shifts", "swap moves"
    );
    for n in [100, 1_000, 10_000] {
        // Worst case for ordered deletion: always remove the front. Run the
        // deletes for real and add up what delete_ordered reports.
        let mut xs: Vec<i64> = (0..n).collect();
        let mut ordered = 0;
        while !xs.is_empty() {
            ordered += delete_ordered(&mut xs, 0);
        }
        // Swap-delete moves one element per call, n calls.
        let mut ys: Vec<i64> = (0..n).collect();
        let mut swaps = 0;
        while !ys.is_empty() {
            delete_swap(&mut ys, 0);
            swaps += 1;
        }
        println!("  {n:>10} {ordered:>18} {swaps:>18}");
    }
    println!();
    println!("  emptying a Vec from the front is Theta(n^2) with ordered deletes");
    println!("  and Theta(n) with swap-deletes. If you do not need the order, say so.");

    println!();
    println!("the pointer trap -- in Go both deletes leave a stale pointer behind. In Rust:");
    println!();

    {
        // The Vec you get from std: remove() hands the Box back and we drop it.
        let mut items = vec![
            Item::boxed(1, "a"),
            Item::boxed(2, "b"),
            Item::boxed(3, "c"),
        ];
        items.remove(0);
        println!(
            "  {:<19}  len={} cap={}, slot[{}] is uninitialised memory, items alive: {}",
            "Vec::remove:",
            items.len(),
            items.capacity(),
            items.len(),
            alive()
        );
    }
    println!("  (Vec dropped: items alive: {})", alive());

    println!();
    println!("and the Go-shaped version, a buffer with its own len:");
    println!();
    {
        let mut leaky = Buf::new();
        leaky.delete_leaky(0);
        leaky.report("after leaky delete:", "still holds");
    }
    println!("  (Buf dropped: items alive: {})", alive());
    {
        let mut clean = Buf::new();
        clean.delete_clean(0);
        clean.report("after clean delete:", "is");
    }
    println!("  (Buf dropped: items alive: {})", alive());

    println!();
    println!("the leaky Buf keeps the removed item alive for as long as the Buf");
    println!("lives. For a Vec<i64> nobody cares; for a Vec<Box<Session>> held by a");
    println!("long-lived server it is a leak (lesson 04, example 3).");
    println!();
    println!("the rule: if YOU keep the length, empty the vacated slot with take()");
    println!("or an assignment. If std keeps it (remove, pop, truncate), you cannot");
    println!("get this wrong. A shift of Boxes also cannot duplicate a pointer the");
    println!("way Go's copy does, so the stale slot holds the removed item itself.");

    println!();
    println!("choosing:");
    println!("  need order preserved?      -> ordered delete, Theta(n)");
    println!("  order is meaningless?      -> swap delete, Theta(1)");
    println!("  removing MANY elements?    -> neither: one filtering pass (example 3)");
}
```

**Rust output:**

```
removing the element at index 1 from [10 20 30 40 50]:

  ordered:  [10 30 40 50]   -- shifts 3 elements, order kept
  swap:     [10 50 30 40]   -- moves 1 element, order scrambled

deleting k elements one at a time:

           n     ordered shifts         swap moves
         100               4950                100
        1000             499500               1000
       10000           49995000              10000

  emptying a Vec from the front is Theta(n^2) with ordered deletes
  and Theta(n) with swap-deletes. If you do not need the order, say so.

the pointer trap -- in Go both deletes leave a stale pointer behind. In Rust:

  Vec::remove:         len=2 cap=3, slot[2] is uninitialised memory, items alive: 2
  (Vec dropped: items alive: 0)

and the Go-shaped version, a buffer with its own len:

  after leaky delete:  len=2 cap=3, slot[2] still holds Some(Item { id: 1, name: "a" }), items alive: 3
  (Buf dropped: items alive: 0)
  after clean delete:  len=2 cap=3, slot[2] is None, items alive: 2
  (Buf dropped: items alive: 0)

the leaky Buf keeps the removed item alive for as long as the Buf
lives. For a Vec<i64> nobody cares; for a Vec<Box<Session>> held by a
long-lived server it is a leak (lesson 04, example 3).

the rule: if YOU keep the length, empty the vacated slot with take()
or an assignment. If std keeps it (remove, pop, truncate), you cannot
get this wrong. A shift of Boxes also cannot duplicate a pointer the
way Go's copy does, so the stale slot holds the removed item itself.

choosing:
  need order preserved?      -> ordered delete, Theta(n)
  order is meaningless?      -> swap delete, Theta(1)
  removing MANY elements?    -> neither: one filtering pass (example 3)
```

> **Go vs Rust:** The delete table matches Go digit for digit, but here the deletes really run and the shifts are what `delete_ordered` reports (Go sums `k-1` in a loop without deleting). The stale-pointer trap cannot happen on a `Vec`: `remove` drops the item at once (items alive: 2) and the spare slot is uninitialised memory safe code cannot read. It comes back only in a hand-rolled buffer with its own `len`, where the leaky delete keeps 3 items alive and the clean one 2. Moving a `Box` cannot duplicate it, so the stale slot holds the removed item (`id: 1`), while Go's `copy` leaves a duplicate of the last live element (`&{3 c}`).

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

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::fmt::Display;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Go can say "no allocation" and leave it at that. Rust has no runtime to ask
// "did that allocate?", so we count. A global allocator is an `unsafe` trait by
// definition; this one only counts events and forwards every call to the
// system allocator.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// How many allocation events (alloc + realloc) happened while running `f`.
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, R) {
    let before = ALLOCS.load(Relaxed);
    let r = f();
    (ALLOCS.load(Relaxed) - before, r)
}

/// Go's %v of a slice: [1 2 3]
fn list<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

// THE slice idiom. Learn this one and you will reach for it weekly.
//
// Two cursors walk the same array: `read` visits every element, `write` marks
// where the next KEPT element goes. Because write never overtakes read, you can
// overwrite in place -- no second Vec, no allocation.
//
//   for read in 0..xs.len() {
//       if keep(xs[read]) {
//           xs[write] = xs[read];
//           write += 1;
//       }
//   }
//   xs.truncate(write);
//
// std has this exact loop: Vec::retain (and Vec::dedup for the adjacent-
// duplicate case). We write it out so the cursors are visible.
//
// Go returns xs[:write] and the caller must remember to use it: the old
// variable still has its old length, with garbage past `write`. Here the
// function takes `&mut Vec` and truncates the caller's Vec itself.

/// filter_in_place keeps the elements satisfying keep, in order, using no extra
/// memory. Theta(n) time, Theta(1) space.
fn filter_in_place(xs: &mut Vec<i64>, keep: impl Fn(i64) -> bool) {
    let mut write = 0;
    for read in 0..xs.len() {
        if keep(xs[read]) {
            xs[write] = xs[read];
            write += 1;
        }
    }
    xs.truncate(write);
}

/// filter_allocating is the version most people write first. Same answer,
/// a whole new Vec.
fn filter_allocating(xs: &[i64], keep: impl Fn(i64) -> bool) -> Vec<i64> {
    xs.iter().copied().filter(|&v| keep(v)).collect()
}

/// filter_repeated_delete is the version to never write: deleting inside the
/// loop is Theta(n) per removal, so the whole thing is Theta(n^2) -- plus it
/// skips elements, because everything shifts under the cursor.
///
/// It needs a `while` loop with a hand-moved index. The obvious
/// `for i in 0..xs.len()` does not have Go's bug, it has a louder one: the
/// range is fixed before the loop starts, so once Vec::remove has shortened
/// the Vec, indexing runs off the end and panics (index out of bounds).
fn filter_repeated_delete(xs: &mut Vec<i64>, keep: impl Fn(i64) -> bool) {
    let mut i = 0;
    while i < xs.len() {
        if !keep(xs[i]) {
            xs.remove(i);
            // BUG: i now points at an element we never examined.
        }
        i += 1;
    }
}

fn even(v: i64) -> bool {
    v % 2 == 0
}

/// Run an in-place function on a copy of the input and hand back the result.
fn applied(input: &[i64], f: impl FnOnce(&mut Vec<i64>)) -> Vec<i64> {
    let mut v = input.to_vec();
    f(&mut v);
    v
}

fn main() {
    let src: Vec<i64> = (1..=10).collect();

    println!("keep the even numbers from {}", list(&src));
    println!();
    println!(
        "  in place:          {}",
        list(&applied(&src, |v| filter_in_place(v, even)))
    );
    println!(
        "  allocating:        {}",
        list(&filter_allocating(&src, even))
    );

    // The buggy one needs consecutive removals to show its teeth.
    let bad = [1, 3, 5, 2, 4, 7];
    println!();
    println!("now with CONSECUTIVE elements to remove, [1 3 5 2 4 7]:");
    println!(
        "  in place:          {}   <- correct",
        list(&applied(&bad, |v| filter_in_place(v, even)))
    );
    println!(
        "  delete-in-loop:    {}   <- WRONG: skipped elements",
        list(&applied(&bad, |v| filter_repeated_delete(v, even)))
    );
    println!();
    println!("  after removing index i, the next element slides INTO index i --");
    println!("  and then i += 1 steps right over it. (Iterating backwards fixes the");
    println!("  correctness bug but leaves it Theta(n^2); see example 14.)");

    println!();
    println!("the same two-cursor shape solves a whole family of problems:");
    println!();

    // dedupe a sorted Vec
    let sorted = [1, 1, 2, 3, 3, 3, 4, 5, 5];
    println!(
        "  dedupe sorted      {} -> {}",
        list(&sorted),
        list(&applied(&sorted, dedupe))
    );

    // move zeroes to the end, keeping the order of the rest
    let zeros = [0, 1, 0, 3, 12];
    println!(
        "  move zeroes        {} -> {}",
        list(&zeros),
        list(&applied(&zeros, |v| move_zeroes(v)))
    );

    // remove ALL occurrences of a value
    let with_dup = [3, 2, 2, 3, 1, 3];
    println!(
        "  remove all 3s      {} -> {}",
        list(&with_dup),
        list(&applied(&with_dup, |v| remove_all(v, 3)))
    );

    println!();
    println!("all four are the same five lines with a different `keep` condition.");
    println!("Theta(n) time and Theta(1) space. \"No allocation\" is a claim, so count it:");
    println!();
    let mut a = src.clone();
    let (in_place, ()) = allocs_of(|| filter_in_place(&mut a, even));
    let (allocating, out) = allocs_of(|| filter_allocating(&src, even));
    println!("  allocation events, in place:    {in_place}");
    println!(
        "  allocation events, allocating:  {allocating}   (building a {}-element Vec)",
        out.len()
    );
}

/// dedupe removes adjacent duplicates. Requires sorted input.
fn dedupe(xs: &mut Vec<i64>) {
    if xs.is_empty() {
        return;
    }
    let mut write = 1;
    for read in 1..xs.len() {
        if xs[read] != xs[write - 1] {
            xs[write] = xs[read];
            write += 1;
        }
    }
    xs.truncate(write);
}

/// move_zeroes shifts every non-zero left, then fills the tail with zeroes.
fn move_zeroes(xs: &mut [i64]) {
    let mut write = 0;
    for read in 0..xs.len() {
        if xs[read] != 0 {
            xs[write] = xs[read];
            write += 1;
        }
    }
    xs[write..].fill(0);
}

/// remove_all drops every element equal to v.
fn remove_all(xs: &mut Vec<i64>, v: i64) {
    let mut write = 0;
    for read in 0..xs.len() {
        if xs[read] != v {
            xs[write] = xs[read];
            write += 1;
        }
    }
    xs.truncate(write);
}
```

**Rust output:**

```
keep the even numbers from [1 2 3 4 5 6 7 8 9 10]

  in place:          [2 4 6 8 10]
  allocating:        [2 4 6 8 10]

now with CONSECUTIVE elements to remove, [1 3 5 2 4 7]:
  in place:          [2 4]   <- correct
  delete-in-loop:    [3 2 4]   <- WRONG: skipped elements

  after removing index i, the next element slides INTO index i --
  and then i += 1 steps right over it. (Iterating backwards fixes the
  correctness bug but leaves it Theta(n^2); see example 14.)

the same two-cursor shape solves a whole family of problems:

  dedupe sorted      [1 1 2 3 3 3 4 5 5] -> [1 2 3 4 5]
  move zeroes        [0 1 0 3 12] -> [1 3 12 0 0]
  remove all 3s      [3 2 2 3 1 3] -> [2 2 1]

all four are the same five lines with a different `keep` condition.
Theta(n) time and Theta(1) space. "No allocation" is a claim, so count it:

  allocation events, in place:    0
  allocation events, allocating:  2   (building a 5-element Vec)
```

> **Go vs Rust:** Results are identical to Go; the last paragraph differs because the Go claim "no allocation at all" is counted here with a counting allocator: 0 allocation events in place against 2 for the allocating version (`Vec` growing from capacity 4 to 8, a `std` detail). `Vec::retain` and `Vec::dedup` are the std versions of these loops. Go's `FilterInPlace` returns a shortened slice the caller has to remember to use; these take `&mut Vec` and truncate the caller's own `Vec`. The natural Rust spelling of the buggy loop, `for i in 0..xs.len()` with `remove`, does not skip elements, it panics with index out of bounds (the range is fixed up front), so reproducing Go's `[3 2 4]` needs a hand-moved `while` index.

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

**Rust:**

```rust
use std::fmt::Display;

// Two in-place rearrangements worth knowing cold. Both are Theta(n) time and
// Theta(1) space, and the second is built entirely out of the first.

/// Go's %v of a slice: [1 2 3]
fn list<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

/// reverse swaps from both ends inward. The loop ends when the cursors meet,
/// which is why a middle element (odd length) needs no special case.
///
/// `j` is one PAST the right cursor. With usize indexes the Go spelling
/// `j = len(xs) - 1` would underflow on an empty slice, so we keep j exclusive.
fn reverse(xs: &mut [i64]) {
    let (mut i, mut j) = (0, xs.len());
    while i + 1 < j {
        j -= 1;
        xs.swap(i, j);
        i += 1;
    }
}

/// rotate_left shifts every element k places toward the front, wrapping around.
///
/// The trick: reverse the first k, reverse the rest, then reverse the whole
/// thing. Three linear passes, no extra memory.
///
///   [1 2 3 4 5] k=2
///   reverse [1 2]      -> [2 1 | 3 4 5]
///   reverse [3 4 5]    -> [2 1 | 5 4 3]
///   reverse everything -> [3 4 5 1 2]
fn rotate_left(xs: &mut [i64], k: i64) {
    let n = xs.len();
    if n == 0 {
        return;
    }
    // Rust's % keeps the sign of the dividend, like Go's, so -1 % 5 is -1.
    // rem_euclid is the one-call normalisation: always in 0..n.
    let k = k.rem_euclid(n as i64) as usize;
    reverse(&mut xs[..k]);
    reverse(&mut xs[k..]);
    reverse(xs);
}

/// rotate_copy is the obvious alternative: allocate and place each element
/// directly. Also Theta(n) time -- but Theta(n) SPACE.
fn rotate_copy(xs: &[i64], k: i64) -> Vec<i64> {
    let n = xs.len();
    if n == 0 {
        return xs.to_vec();
    }
    let k = k.rem_euclid(n as i64) as usize;
    let mut out = vec![0; n];
    for (i, &v) in xs.iter().enumerate() {
        out[(i + n - k) % n] = v;
    }
    out
}

fn show(label: &str, xs: &[i64]) {
    println!("  {label:<28} {}", list(xs));
}

fn main() {
    println!("reverse, in place:");
    let cases: [&[i64]; 5] = [&[], &[1], &[1, 2], &[1, 2, 3], &[1, 2, 3, 4, 5, 6]];
    for case in cases {
        let mut xs = case.to_vec();
        let before = list(&xs);
        reverse(&mut xs);
        println!("  {before:<16} -> {}", list(&xs));
    }
    println!("  (no special case for odd length: the cursors meet on the middle");
    println!("   element and the loop condition i + 1 < j stops before swapping it)");

    println!();
    println!("rotate left by 2, via triple reversal:");
    println!();
    let mut xs: Vec<i64> = vec![1, 2, 3, 4, 5];
    show("start", &xs);
    reverse(&mut xs[..2]);
    show("reverse the first k", &xs);
    reverse(&mut xs[2..]);
    show("reverse the rest", &xs);
    reverse(&mut xs);
    show("reverse everything", &xs);

    println!();
    println!("every k, including the awkward ones:");
    println!();
    let mut std_agrees = true;
    for k in [0, 1, 4, 5, 7, -1] {
        let mut a = vec![1, 2, 3, 4, 5];
        rotate_left(&mut a, k);
        let b = rotate_copy(&[1, 2, 3, 4, 5], k);
        let agree = a == b;
        // std has this too: slice::rotate_left, which takes an unsigned mid.
        let mut c = vec![1, 2, 3, 4, 5];
        c.rotate_left(k.rem_euclid(5) as usize);
        std_agrees &= a == c;
        println!("  k={k:<3} -> {}   copy version agrees: {agree}", list(&a));
    }

    println!();
    println!("k=5 and k=0 give the identity; k=7 is the same as k=2; k=-1 rotates");
    let (minus_one, five) = (-1_i64, 5_i64);
    println!(
        "right. Rust's % keeps the sign of the dividend, like Go's: {minus_one} % {five} = {},",
        minus_one % five
    );
    println!(
        "and ({minus_one}).rem_euclid({five}) = {}. That one call is the whole normalisation.",
        minus_one.rem_euclid(five)
    );

    println!();
    println!("why the triple reversal instead of the copy:");
    println!("  both are Theta(n) TIME. The reversal is Theta(1) SPACE, and it");
    println!("  touches memory sequentially three times, which is exactly the");
    println!("  access pattern lesson 04 showed is nearly free.");
    println!();
    println!("  the copy version needs a whole second array, and its writes jump");
    println!("  around by (i+n-k)%n -- a scattered pattern.");
    println!();
    println!("std: <[T]>::reverse does the first one, and std has the second as well:");
    println!("<[T]>::rotate_left / rotate_right. The triple reversal is still worth");
    println!("knowing, but in Rust you do not have to write it.");
    println!("  slice::rotate_left agrees with ours on every k above: {std_agrees}");
}
```

**Rust output:**

```
reverse, in place:
  []               -> []
  [1]              -> [1]
  [1 2]            -> [2 1]
  [1 2 3]          -> [3 2 1]
  [1 2 3 4 5 6]    -> [6 5 4 3 2 1]
  (no special case for odd length: the cursors meet on the middle
   element and the loop condition i + 1 < j stops before swapping it)

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
right. Rust's % keeps the sign of the dividend, like Go's: -1 % 5 = -1,
and (-1).rem_euclid(5) = 4. That one call is the whole normalisation.

why the triple reversal instead of the copy:
  both are Theta(n) TIME. The reversal is Theta(1) SPACE, and it
  touches memory sequentially three times, which is exactly the
  access pattern lesson 04 showed is nearly free.

  the copy version needs a whole second array, and its writes jump
  around by (i+n-k)%n -- a scattered pattern.

std: <[T]>::reverse does the first one, and std has the second as well:
<[T]>::rotate_left / rotate_right. The triple reversal is still worth
knowing, but in Rust you do not have to write it.
  slice::rotate_left agrees with ours on every k above: true
```

> **Go vs Rust:** Every result line matches Go. The differences are in the prose: `slice::rotate_left` exists in std (the Go text says there is no `slices.Rotate`) and agrees with the triple reversal on every k, and `rem_euclid` replaces `((k % n) + n) % n` (Rust's `%` keeps the dividend's sign, as in Go). Indexes are `usize`, so Go's `j = len(xs)-1` would underflow on the empty slice; the cursor `j` is kept one past the end instead. `reverse(&mut xs[..2])` is Go's `Reverse(xs[:2])`: a mutable sub-view into the same memory.

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

**Rust:**

```rust
use std::fmt::Display;

// A SORTED Vec is a real data structure, not just a Vec you sorted once.
// Kept in order, it answers questions a HashMap cannot: predecessor, successor,
// rank, and "everything between a and b" (lesson 04, example 15).
//
// The cost of maintaining it is the interesting part, and it splits in two:
//
//   FIND the position   Theta(log n)   binary search
//   MAKE room for it    Theta(n)       shift the tail
//
// So insertion is Theta(n) -- but with two very different constants. The search
// is log n probes; the shift is one memmove.

/// Go's %v of a slice: [1 2 3]
fn list<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

/// What Go keeps in package-level counters. Rust has no mutable globals, so
/// the counters travel as an argument.
#[derive(Default)]
struct Stats {
    comparisons: u64,
    shifted: u64,
}

/// search_pos returns the index where v belongs, keeping xs sorted: the first
/// index whose element is >= v. std spells this `partition_point(|&x| x < v)`;
/// we write it out so we can count the probes.
fn search_pos(xs: &[i64], v: i64, stats: &mut Stats) -> usize {
    let (mut lo, mut hi) = (0, xs.len());
    while lo < hi {
        stats.comparisons += 1;
        let mid = lo + (hi - lo) / 2;
        if xs[mid] < v {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    lo
}

/// sorted_insert puts v in its place and keeps the Vec ordered.
fn sorted_insert(xs: &mut Vec<i64>, v: i64, stats: &mut Stats) {
    let i = search_pos(xs, v, stats);
    stats.shifted += (xs.len() - i) as u64;
    xs.insert(i, v); // Vec::insert: the same shift as example 1
}

/// Go's `xs[i-1]` panics at i == 0; here the missing neighbour is a value.
fn show_opt(x: Option<&i64>) -> String {
    x.map_or("none".to_string(), |v| v.to_string())
}

fn main() {
    println!("building a sorted Vec by inserting one value at a time:");
    println!();

    let mut stats = Stats::default();
    let mut xs: Vec<i64> = Vec::new();
    for v in [5, 2, 8, 1, 9, 3] {
        sorted_insert(&mut xs, v, &mut stats);
        println!("  insert {v} -> {}", list(&xs));
    }

    println!();
    println!("what a sorted Vec can answer that a HashMap cannot:");
    println!();
    println!("  contents:                {}", list(&xs));

    // binary_search is Result<usize, usize>: Ok(index) when found,
    // Err(index it would be inserted at) when not -- Go's (i, found) pair as a type.
    let found = xs.binary_search(&8);
    println!(
        "  is 8 present?            {} (at index {})",
        found.is_ok(),
        found.unwrap_or_else(|i| i)
    );

    let i = match xs.binary_search(&6) {
        Ok(i) | Err(i) => i,
    };
    println!(
        "  is 6 present?            {}",
        xs.binary_search(&6).is_ok()
    );
    println!("  where would 6 go?        index {i}");
    println!("  predecessor of 6:        {}", show_opt(xs[..i].last()));
    println!("  successor of 6:          {}", show_opt(xs.get(i)));

    // partition_point(|&x| x < v) is sort.SearchInts: the first index with x >= v.
    let lo = xs.partition_point(|&x| x < 3);
    let hi = xs.partition_point(|&x| x < 9);
    println!("  everything in [3, 9):    {}", list(&xs[lo..hi]));
    println!("  the 3rd smallest:        {}", xs[2]);
    println!(
        "  how many are below 8?    {}",
        xs.partition_point(|&x| x < 8)
    );

    println!();
    println!("none of those are possible with a HashMap at any price -- it has no order.");

    println!();
    println!("the cost of maintaining the order, counted:");
    println!();
    println!(
        "  {:>10} {:>16} {:>18} {:>16}",
        "n", "comparisons", "elements shifted", "shifts/insert"
    );
    for n in [100, 1_000, 10_000] {
        let mut stats = Stats::default();
        let mut s: Vec<i64> = Vec::new();
        // Worst case for shifting: every new value belongs at the front.
        for v in (1..=n).rev() {
            sorted_insert(&mut s, v, &mut stats);
        }
        println!(
            "  {n:>10} {:>16} {:>18} {:>16.1}",
            stats.comparisons,
            stats.shifted,
            stats.shifted as f64 / n as f64
        );
    }

    println!();
    println!("comparisons grow like n log n -- that is the cheap half.");
    println!("elements shifted grow like n^2/2 -- that is the expensive half.");
    println!();
    println!("so building a sorted Vec by repeated insertion is Theta(n^2).");
    println!("If you have all the data up front, do NOT do this: push everything");
    println!("and call sort() once, which is Theta(n log n).");

    println!();
    println!("sorted insertion earns its keep only when the collection is");
    println!("read-mostly and grows slowly -- the exact case where the");
    println!("Theta(n) shift is rare and the Theta(log n) queries are constant.");
    println!("Example 12 measures where that stops being true.");
}
```

**Rust output:**

```
building a sorted Vec by inserting one value at a time:

  insert 5 -> [5]
  insert 2 -> [2 5]
  insert 8 -> [2 5 8]
  insert 1 -> [1 2 5 8]
  insert 9 -> [1 2 5 8 9]
  insert 3 -> [1 2 3 5 8 9]

what a sorted Vec can answer that a HashMap cannot:

  contents:                [1 2 3 5 8 9]
  is 8 present?            true (at index 4)
  is 6 present?            false
  where would 6 go?        index 4
  predecessor of 6:        5
  successor of 6:          8
  everything in [3, 9):    [3 5 8]
  the 3rd smallest:        3
  how many are below 8?    4

none of those are possible with a HashMap at any price -- it has no order.

the cost of maintaining the order, counted:

           n      comparisons   elements shifted    shifts/insert
         100              573               4950             49.5
        1000             8977             499500            499.5
       10000           123617           49995000           4999.5

comparisons grow like n log n -- that is the cheap half.
elements shifted grow like n^2/2 -- that is the expensive half.

so building a sorted Vec by repeated insertion is Theta(n^2).
If you have all the data up front, do NOT do this: push everything
and call sort() once, which is Theta(n log n).

sorted insertion earns its keep only when the collection is
read-mostly and grows slowly -- the exact case where the
Theta(n) shift is rare and the Theta(log n) queries are constant.
Example 12 measures where that stops being true.
```

> **Go vs Rust:** Every number matches Go, including the comparison and shift counts (573 / 4950, 8977 / 499500, 123617 / 49995000), because the search loop and the worst-case input are the same. Differences are in the types: `binary_search` returns `Result<usize, usize>` (`Ok(index)` or `Err(insertion point)`, Go's `(i, found)` pair), `partition_point` is `sort.SearchInts`, and the missing predecessor of an index 0 is `None` where Go's `xs[i-1]` would panic. Go's package-level counters become a `Stats` struct passed by `&mut`, and "a map has no order" is true of `HashMap` only: `BTreeMap` is ordered and has `range`.

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

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::fmt::{self, Display};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Everything examples 1-5 built by hand already exists in std, on Vec and on
// slices, generic and tested. Build them once to understand the cost; call
// these after.
//
// The point of this example is the COST COLUMN. `Vec::insert` is convenient
// and it is still Theta(n) -- the standard library cannot repeal the shift.

// Rust has no runtime to ask "did that allocate?", so we count. A global
// allocator is an `unsafe` trait by definition; this one only counts events
// and forwards every call to the system allocator.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// How many allocation events (alloc + realloc) happened while running `f`.
fn allocs_of(f: impl FnOnce()) -> usize {
    let before = ALLOCS.load(Relaxed);
    f();
    ALLOCS.load(Relaxed) - before
}

/// Go's %v of a slice: [1 2 3]
fn list<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

struct Person {
    name: &'static str,
    age: u32,
}

/// Go's %v of a struct: {ana 30}
impl Display for Person {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{{{} {}}}", self.name, self.age)
    }
}

/// One line of the "run" table: the call as you would write it, then its result.
fn row(call: &str, result: impl Display) {
    println!("  {call:<34} {result}");
}

fn main() {
    println!("what you built, and what to call instead:");
    println!();
    println!("  {:<24} {:<40} cost", "example", "std");
    let rows = [
        (
            "1. Insert(xs, i, v)",
            "Vec::insert / Vec::splice",
            "Theta(n)",
        ),
        (
            "1. Delete(xs, i)",
            "Vec::remove / Vec::drain(a..b)",
            "Theta(n)",
        ),
        ("3. filter in place", "Vec::retain", "Theta(n)"),
        ("3. dedupe sorted", "Vec::dedup", "Theta(n)"),
        ("4. Reverse(xs)", "<[T]>::reverse", "Theta(n)"),
        ("4. RotateLeft(xs, k)", "<[T]>::rotate_left", "Theta(n)"),
        (
            "5. search_pos(xs, v)",
            "<[T]>::binary_search / partition_point",
            "Theta(log n)",
        ),
        ("-- (clone)", "to_vec / clone", "Theta(n)"),
        ("-- (compare)", "a == b", "Theta(n)"),
        ("-- (find)", "iter().position / contains", "Theta(n)"),
        (
            "-- (sort)",
            "sort / sort_by (stable), sort_unstable",
            "Theta(n log n)",
        ),
        ("-- (min/max)", "iter().min / max", "Theta(n)"),
    ];
    for (example, call, cost) in rows {
        println!("  {example:<24} {call:<40} {cost}");
    }

    println!();
    println!("the same operations, run:");
    println!();

    let xs: Vec<i64> = vec![3, 1, 4, 1, 5, 9, 2, 6];
    row("start", list(&xs));
    // position() returns Option<usize>: Go's Index returns -1 for "absent".
    row(
        "xs.iter().position(|&x| x == 4)",
        format!("{:?}", xs.iter().position(|&x| x == 4)),
    );
    row("xs.contains(&7)", xs.contains(&7));
    // max()/min() are Option too: an empty slice has neither.
    row(
        "xs.iter().max() / .min()",
        format!("{:?} / {:?}", xs.iter().max(), xs.iter().min()),
    );

    let mut ins = xs.clone();
    ins.splice(2..2, [99, 98]);
    row(
        "ins.splice(2..2, [99, 98])",
        format!("{}   <- inserts several at once", list(&ins)),
    );

    let mut del = xs.clone();
    del.drain(1..4);
    row(
        "del.drain(1..4)",
        format!("{}   <- a RANGE, not one index", list(&del)),
    );

    let mut sorted = xs.clone();
    sorted.sort();
    row("sorted.sort()", list(&sorted));
    let mut compact = sorted.clone();
    compact.dedup();
    row(
        "compact.dedup()  (copy of sorted)",
        format!("{}   <- adjacent only, so sort first", list(&compact)),
    );

    row(
        "sorted.binary_search(&5)",
        format!("{:?}", sorted.binary_search(&5)),
    );

    let mut rev = xs.clone();
    rev.reverse();
    row("rev.reverse()", list(&rev));

    let mut rot = xs.clone();
    rot.rotate_left(2);
    row("rot.rotate_left(2)", list(&rot));

    let mut odd = xs.clone();
    odd.retain(|&x| x % 2 == 1);
    row(
        "odd.retain(|&x| x % 2 == 1)",
        format!("{}   <- many removals, one pass", list(&odd)),
    );

    println!();
    println!("sorting by a field, and by several:");
    println!();
    let people = vec![
        Person {
            name: "ana",
            age: 30,
        },
        Person {
            name: "bo",
            age: 25,
        },
        Person {
            name: "cy",
            age: 30,
        },
        Person {
            name: "di",
            age: 25,
        },
    ];
    println!("  start           {}", list(&people));

    let mut by_age: Vec<&Person> = people.iter().collect();
    by_age.sort_by_key(|p| p.age);
    println!("  by age          {}", list(&by_age));

    let mut by_age_name: Vec<&Person> = people.iter().collect();
    by_age_name.sort_by(|a, b| a.age.cmp(&b.age).then_with(|| a.name.cmp(b.name)));
    println!(
        "  by age, name    {}   <- Ordering::then_with chains tie-breakers",
        list(&by_age_name)
    );

    let mut unstable: Vec<&Person> = people.iter().collect();
    unstable.sort_unstable_by_key(|p| p.age);
    println!("  sort_unstable   {}", list(&unstable));
    println!("                  ^ identical to `by age`, which is sort_by. Go's defaults are the");
    println!("                    other way round: there SortFunc is unstable and SortStableFunc");
    println!("                    is the opt-in. In Rust sort / sort_by are documented stable and");
    println!("                    sort_unstable / sort_unstable_by are the opt-out. Four elements");
    println!("                    happen to come out the same either way, so look at 100:");

    // 100 items, 3 distinct keys: ties are in input order iff the second field
    // (the input position) increases within each run of equal keys.
    let items: Vec<(u32, usize)> = (0..100).map(|i| ((i * 7 % 3) as u32, i)).collect();
    let ties_in_order =
        |s: &[(u32, usize)]| s.windows(2).all(|w| w[0].0 != w[1].0 || w[0].1 < w[1].1);
    let mut a = items.clone();
    a.sort_by_key(|p| p.0);
    let mut b = items.clone();
    b.sort_unstable_by_key(|p| p.0);
    println!();
    println!("  100 items, 3 distinct keys, do equal keys keep their input order?");
    println!("    sort_by_key           {}", ties_in_order(&a));
    println!("    sort_unstable_by_key  {}", ties_in_order(&b));
    println!("  (the unstable result is what this std's algorithm happens to do; the docs");
    println!("   say equal elements MAY be reordered, which is the point.)");

    // The price of stability in std: count what each sort asks the allocator for.
    let big: Vec<(u32, usize)> = (0..1000).map(|i| ((i * 7 % 3) as u32, i)).collect();
    let mut c = big.clone();
    let stable_allocs = allocs_of(|| c.sort_by_key(|p| p.0));
    let mut d = big.clone();
    let unstable_allocs = allocs_of(|| d.sort_unstable_by_key(|p| p.0));
    println!();
    println!("  allocation events while sorting 1000 items:");
    println!("    sort_by_key           {stable_allocs}");
    println!("    sort_unstable_by_key  {unstable_allocs}");
    println!("  (what this std does, not a guarantee: stability is not free in std either)");

    println!();
    println!("two things the table above is really telling you:");
    println!();
    println!("  1. Vec::insert and Vec::remove are STILL Theta(n). Calling a std");
    println!("     function does not remove the shift -- so a loop that calls");
    println!("     remove n times is still Theta(n^2) (example 14). retain is the");
    println!("     one-pass answer when you are removing many.");
    println!();
    println!("  2. Vec::dedup only removes ADJACENT duplicates, exactly like the");
    println!("     hand-written version in example 3. On unsorted input it quietly");
    println!("     does almost nothing. Sort first, or use a HashSet.");

    let unsorted = vec![1, 2, 1, 2, 1];
    let mut deduped = unsorted.clone();
    deduped.dedup();
    println!(
        "\n     {}.dedup() = {}  <- not a dedupe!",
        list(&unsorted),
        list(&deduped)
    );

    println!();
    println!("and one thing std does differently from Go: binary_search may return ANY");
    println!("matching index when there are duplicates (Go's BinarySearch returns the");
    println!("first). partition_point is the one that is always the first:");
    let dups = [1, 1, 2, 3, 3, 3, 4, 5, 5];
    println!(
        "  {}.binary_search(&3)           = {:?}",
        list(&dups),
        dups.binary_search(&3)
    );
    println!(
        "  {}.partition_point(|&x| x < 3) = {}",
        list(&dups),
        dups.partition_point(|&x| x < 3)
    );

    println!();
    println!("use std. Build it once first, so you know what it costs.");
}
```

**Rust output:**

```
what you built, and what to call instead:

  example                  std                                      cost
  1. Insert(xs, i, v)      Vec::insert / Vec::splice                Theta(n)
  1. Delete(xs, i)         Vec::remove / Vec::drain(a..b)           Theta(n)
  3. filter in place       Vec::retain                              Theta(n)
  3. dedupe sorted         Vec::dedup                               Theta(n)
  4. Reverse(xs)           <[T]>::reverse                           Theta(n)
  4. RotateLeft(xs, k)     <[T]>::rotate_left                       Theta(n)
  5. search_pos(xs, v)     <[T]>::binary_search / partition_point   Theta(log n)
  -- (clone)               to_vec / clone                           Theta(n)
  -- (compare)             a == b                                   Theta(n)
  -- (find)                iter().position / contains               Theta(n)
  -- (sort)                sort / sort_by (stable), sort_unstable   Theta(n log n)
  -- (min/max)             iter().min / max                         Theta(n)

the same operations, run:

  start                              [3 1 4 1 5 9 2 6]
  xs.iter().position(|&x| x == 4)    Some(2)
  xs.contains(&7)                    false
  xs.iter().max() / .min()           Some(9) / Some(1)
  ins.splice(2..2, [99, 98])         [3 1 99 98 4 1 5 9 2 6]   <- inserts several at once
  del.drain(1..4)                    [3 5 9 2 6]   <- a RANGE, not one index
  sorted.sort()                      [1 1 2 3 4 5 6 9]
  compact.dedup()  (copy of sorted)  [1 2 3 4 5 6 9]   <- adjacent only, so sort first
  sorted.binary_search(&5)           Ok(5)
  rev.reverse()                      [6 2 9 5 1 4 1 3]
  rot.rotate_left(2)                 [4 1 5 9 2 6 3 1]
  odd.retain(|&x| x % 2 == 1)        [3 1 1 5 9]   <- many removals, one pass

sorting by a field, and by several:

  start           [{ana 30} {bo 25} {cy 30} {di 25}]
  by age          [{bo 25} {di 25} {ana 30} {cy 30}]
  by age, name    [{bo 25} {di 25} {ana 30} {cy 30}]   <- Ordering::then_with chains tie-breakers
  sort_unstable   [{bo 25} {di 25} {ana 30} {cy 30}]
                  ^ identical to `by age`, which is sort_by. Go's defaults are the
                    other way round: there SortFunc is unstable and SortStableFunc
                    is the opt-in. In Rust sort / sort_by are documented stable and
                    sort_unstable / sort_unstable_by are the opt-out. Four elements
                    happen to come out the same either way, so look at 100:

  100 items, 3 distinct keys, do equal keys keep their input order?
    sort_by_key           true
    sort_unstable_by_key  false
  (the unstable result is what this std's algorithm happens to do; the docs
   say equal elements MAY be reordered, which is the point.)

  allocation events while sorting 1000 items:
    sort_by_key           1
    sort_unstable_by_key  0
  (what this std does, not a guarantee: stability is not free in std either)

two things the table above is really telling you:

  1. Vec::insert and Vec::remove are STILL Theta(n). Calling a std
     function does not remove the shift -- so a loop that calls
     remove n times is still Theta(n^2) (example 14). retain is the
     one-pass answer when you are removing many.

  2. Vec::dedup only removes ADJACENT duplicates, exactly like the
     hand-written version in example 3. On unsorted input it quietly
     does almost nothing. Sort first, or use a HashSet.

     [1 2 1 2 1].dedup() = [1 2 1 2 1]  <- not a dedupe!

and one thing std does differently from Go: binary_search may return ANY
matching index when there are duplicates (Go's BinarySearch returns the
first). partition_point is the one that is always the first:
  [1 1 2 3 3 3 4 5 5].binary_search(&3)           = Ok(5)
  [1 1 2 3 3 3 4 5 5].partition_point(|&x| x < 3) = 3

use std. Build it once first, so you know what it costs.
```

> **Go vs Rust:** Rust's API names differ, so this output is not line-identical, but every result matches Go's (`[3 1 99 98 4 1 5 9 2 6]`, `[3 5 9 2 6]`, `[1 2 3 4 5 6 9]`, and `dedup` leaving `[1 2 1 2 1]` alone). Absence is an `Option` (`position`, `max`) and found-or-insertion-point a `Result` (`binary_search`) instead of `-1` and `(i, ok)`. Std's defaults are the reverse of Go's: `sort`/`sort_by` are stable and `sort_unstable*` is the opt-out; at 100 items the unstable one reordered ties and the stable one did not, and the stable one made 1 allocation event against 0 (implementation details of this std). `binary_search` may return any matching index among duplicates (`Ok(5)` for the 3s at indexes 3 to 5), where Go's returns the first.

**Complexity:** the stdlib does not repeal the shift · use it for correctness, and know what each call costs

---

> Next tier: [🟡 medium](2-medium.md) — building the structure Go hands you for free.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
