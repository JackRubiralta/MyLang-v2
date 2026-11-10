# Idea 4: nothing is freed while something still points at it

**Status:** a sketch, written because Jack asked for a solution to the dangling cases ("we
have to think of solution"). Nothing here is in `DOCS.md`. It builds on DOCS section 8.

Lines marked **(Jack)** are things Jack said. Everything else is a sketch.

---

## 1. The problem

Both cases Jack looked at have one cause. A borrower or an ownership points at a heap value,
or inside one, and something frees that heap value while the pointer is still used.

```go
var p: ^int = ^5;
var b: &int = &p^;               // b borrows the 5
p = ^6;                          // the 5 is freed
b^                               // b still leads to where the 5 was
```

"Don't do this though." (Jack)

```go
var h: @^Node = @list^.next;     // the address of a field inside the first node
list = makeNode(9);              // the first node is freed
var @n: ^Node = h;               // h points into freed memory
```

"Makes sense though, even though that causes the problem… we have to think of solution."
(Jack)

Rule 8.5 doesn't catch either one, because it compares the scopes of names, and here a heap
value ends before the name that owned it does.

## 2. The rule

**A heap value can't be freed or moved away while a borrower or an ownership that points at
it, or inside it, is still used further down.** The compiler raises the error at the line
that would free or move it.

## 3. What frees or moves a heap value

| The line | What it does to the heap value `p` owns |
|---|---|
| `p = …;` and `p = null;` | frees it: an owner that is given something new frees what it had |
| `@p = …;` | frees it, for the same reason |
| `@q = @p;` and `var @q: T = @p;` | moves it to `q` |
| `take(@p)` | moves it into the function, which frees it unless it gives it back |
| `change(&p)` where the function can change `p` | may do any of the above |

The same line on an owner further up frees or moves everything below it: giving `list` a
new value frees every node of the list.

## 4. How the compiler checks it

A function body runs from top to bottom, with no `if` and no loops. So the check is one
pass down the body:

1. When a borrower or an ownership is made, note the path it was made from: `&p^` points at
   what `p` owns, `@list^.next` points inside the first node of `list`.
2. When it is pointed at something else, note the new path.
3. At every line in section 3, look at the pointers that are used again further down. If one
   of them points at the value being freed or moved, or inside it, the line is an error.

This is the idea behind Rust's borrow checker. Rust needs a hard analysis because of
branches and loops. Here it is a read from top to bottom.

## 5. What it rejects, and what you write instead

```go
var p: ^int = ^5;
var b: &int = &p^;
p = ^6;                          // ERROR: b still points at p's value and is used below
b^
```

To change the value where it is, so `b` sees it: `p^ = 6;`.

```go
var h: @^Node = @list^.next;
list = makeNode(9);              // ERROR: h still points inside the first node and is used below
var @n: ^Node = h;
```

To do both, take the node out first:

```go
var h: @^Node = @list^.next;
var @n: ^Node = h;               // the second node is out, and h isn't used again
list = makeNode(9);              // fine
```

## 6. What it allows

Only a pointer **at or inside the part being freed** is in the way. A cursor can change
what comes after it:

```go
var cur: &Node = &list^.next^;   // the second node
cur^.next = makeNode(9);         // fine: this frees what was behind the second node,
                                 // and cur points at the second node itself
cur^.value
```

A pointer that isn't used again doesn't block anything.

## 7. Calls

During a call the caller is waiting, so the caller's pointers can only be put in danger by
what the function does through its arguments. The function's type says what it may do:

| The argument | The function may | So it is an error when another pointer still in use points… |
|---|---|---|
| `f(x)`, a copy | nothing to the caller's value | never |
| `f(&x)`, to read | nothing | never |
| `f(&x)`, to change | free what is owned below `x` | inside what `x` owns |
| `f(@x)` | take `x`'s value | at `x`'s value, or inside it |

Two arguments of one call count too: `f(&list^.next^, @list)` hands over the list while the
first argument points into it.

Inside the function the same rule runs on its own body, with each parameter as a starting
point.

## 8. What it needs

- **Whether a `&T` parameter reads or changes has to be in the function's type.** Today
  `(&int)void` covers both (DOCS 8.16). Without it every `&` argument has to count as "may
  change", and then even a helper that only reads can't be called while a cursor is in use.
- **A rule for a borrower that a function returns:** it points into what the function's
  borrowed arguments point into.

## 9. What it doesn't cover

- **A borrower kept inside a heap value,** like the `prev` in DOCS 8.14. The compiler can't
  see when it will be used, so it can't tell when its target may be freed. It stays the
  programmer's job, or needs a check while running.
- **A top-level `var` that a called function changes.** The call doesn't show it.

## 10. The other way: a check while running

Every heap value gets a number that changes when the value is freed. A borrower or an
ownership remembers the number it saw. Following a pointer whose number no longer matches
fails while the program runs, the same way following `null` does.

| | The rule in section 2 | A check while running |
|---|---|---|
| found | when compiling | when the bad line runs |
| covers `prev` and top-level `var`s | no | yes |
| cost | some correct programs have to be reordered | every pointer and every heap value is bigger, and every follow does a check the code doesn't show |

The two can be combined: the rule for pointers held in variables, and the check only for
pointers kept inside heap values.

## 11. Open

- Whether to take the rule in section 2.
- "Still used further down", or "still in scope". The second is simpler to state and
  rejects more.
- Whether a value may be moved while it is pointed at. It isn't freed by a move, so the
  compiler could follow it to its new owner. The sketch forbids it, which is simpler.
- What to do about `prev` (section 9).
