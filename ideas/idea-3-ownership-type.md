# Idea 3: an ownership type

**Status:** an idea being explored. It builds on section 8 of `DOCS.md`, and nothing here is
in `DOCS.md`.

Lines marked **(Jack)** are things Jack said. Lines marked **(sketch)** are one way it could
be written. Lines marked **(open)** still need a decision.

---

## 1. The idea (Jack)

- "I think we should have an ownership type. `p@` gives that ownership type. That's why we're
  able to set it to something else."
- "When we do `r@` and pass that into a function, that function should be explicit that it's
  receiving the owner type, or do `var owned@: ^int`."
- "It would be something more intrinsic to the variable, because every variable would be
  able to be called on to get its ownership and then change that, or get its ownership to
  set it to another variable."
- In memory: "similar to the way borrowed works, but instead we can transition ownership, so
  more like a real pointer."
- "In theory, this should work not only on heap."

## 2. What an ownership is in memory

A heap owner `p: ^int` is a place on the stack that holds the address of a value on the
heap. That stored address is the ownership. Nothing else has to exist in memory for it.

```
before                              after  q@ = p@;

p: [ address ] ----> [ 5 ]          p: [ null ]
q: [ null ]                         q: [ address ] ----> [ 5 ]
```

`p` names the address as something to copy from, `p^` follows it, and `p@` names it as
something to give away.

A borrower keeps pointing at the original. A take leaves nothing pointing back: the address
is copied into the new owner and `null` is written in the old one. If an ownership were kept
as a pointer to the original, to be taken from later, the line `f(r@)` would no longer say
whether `r` is empty afterwards.

## 3. Two things "ownership type" can mean, and what DOCS.md already has

| Meaning | Already written as | In memory |
|---|---|---|
| a thing that owns a heap value | `^T` | the address of the value |
| a handle on someone else's ownership, to change or move it later | `&^T`, made with `&p` | the address of the owner's own place: a pointer to a pointer |

A parameter of type `^int` already says the function owns what it is given. A parameter of
type `&^int` already says the function is handed the caller's ownership itself:

```go
const takeFrom: (&^int)^int = (var o: &^int){
    return o^@;                // moves the value out of the caller's owner, which is left null
};

var p: ^int = ^5;
var mine: ^int = takeFrom(&p);     // p is null, and mine owns the 5
```

Through a handle `o: &^int`:

| Written | Gives |
|---|---|
| `o^` | the owner: compare it with `null`, copy it, or give it a new value |
| `o^^` | the plain value |
| `o^@` | the value itself, taken over, with the caller's owner left `null` |

## 4. Jack's version: `x@` has a type of its own, and the receiver is marked (sketch)

`x@` means "x, to be taken, not copied", and its type is `@T`. A holder that takes is marked
too, so ownership always goes from an `@` to an `@`:

| Written | Type | Meaning |
|---|---|---|
| `x` | `T` | the value: giving it to something makes a copy |
| `x@` | `@T` | the same value, to be taken: giving it to something empties `x` |
| `name@: T` | | a holder that takes what it is given |
| `(@T)void` | | a function whose parameter takes what it is given |

```go
var r@: ^int = q@;                                     // a declaration that receives ownership
const keep: (@^int)void = (var owned@: ^int){ ... };   // a parameter that receives ownership

keep(r@);                  // fine
keep(r);                   // ERROR: keep takes an ownership: write r@
```

The rule "a move carries `@` on both sides" then stops being a rule of its own. It is the
ordinary rule that both sides of `=` have the same type:

| Line | Types | |
|---|---|---|
| `q = p;` | `^int` and `^int` | fine: a copy |
| `q@ = p@;` | `@^int` and `@^int` | fine: a take |
| `q = p@;` | `^int` and `@^int` | error: the types differ |
| `q@ = p;` | `@^int` and `^int` | error: the types differ |

For the compiler to check it through a function held in a variable, the mark has to be in
the function's **type** (`(@^int)void`), the same lesson as with `ref`.

| | It gives | It costs |
|---|---|---|
| marking the receiver | one rule with no exception: `@` on both sides everywhere; "both sides" becomes ordinary type checking; every call matches its function's type (`f(n)` with `(int)`, `f(&n)` with `(&int)`, `f(r@)` with `(@^int)`); a function can refuse a silent deep copy | two kinds of owning parameter (`^T` takes a copy, `@^T` takes yours), so a caller who is finished with a value still pays for a copy when the parameter is `(^T)`; a rule for values nothing else holds (`keep(^5)`, `keep(makeNode(1))`); a mark on a declared name is a new position |
| as DOCS.md is now | one owning parameter type, and the caller chooses copy or move at the call | the receiving side of a declaration or call is unmarked, and `use(big)` and `use(big@)` both fit one type |

## 5. Turning an ownership into a value

An ownership is never held on its own and converted later. `x@` goes straight into a holder,
and from then on that holder is an ordinary variable:

| Written | Gives | Afterwards |
|---|---|---|
| `var r: ^int = p@;` | an owner | `p` is `null`; `r^` reads the value |
| `var x: int = p@^;` | the plain value | `p` is `null`, and the heap value is freed at the end of the line |
| `var x: int = p^;` | a plain copy | `p` still owns its value |

## 6. Not only on the heap (open)

Ownership itself only moves for a value on the heap. A plain value lives inside its
variable until that variable ends, so giving it to another variable can only copy its bytes,
it can't outlive its block, and an `int` has no "empty" state to be left in.

`@` can still be given a meaning on every variable, as the mirror of copying. A copy
duplicates the heap values inside. A take takes them over:

| `x` is | `=` (copy) | `x@` (take) |
|---|---|---|
| a heap owner, `^T` | a new heap value | the same heap value; `x` is left `null` |
| a struct or array that holds heap owners | every heap value inside is copied | the plain parts are copied, and every owner inside is taken and left `null` |
| a plain value that owns nothing on the heap | a copy | nothing to take: **an error**, or **the same as a copy** |
| a borrower | another borrower | owns nothing: **an error**, or **another borrower** |

```go
const Node: type = {int, ^Node};
var a: Node = { value: int = 1; next: ^Node = makeNode(2); };

var b@: Node = a@;      // b.value is a copy of a.value
                        // b.next takes over the node a.next owned, and a.next is null
```

The last two rows are the open choice. "An error" is the stricter one. "The same as a copy"
is what a function written for any type needs (DOCS 4.3): it takes a type `T` and can't know
whether a `T` is on the heap, so if it should be able to take a `T` without copying, `@` has
to be allowed on every type.

DOCS.md (8.9) describes `@` on heap owners only, and lists the rest as not settled.

## 7. Open

- **Whether the receiver is marked** (section 4), and whether a plain `T` parameter also
  accepts `x@`.
- **`@` on values that aren't heap owners** (section 6): an error, or the same as a copy.
- **A value nothing else holds** (`^5`, the result of a call) given to an `@` holder.
- **In front or behind:** `@x` like `&x`, or `x@` like `arr[1]`.
- **Whether an `@T` can be stored,** for example in an array. The sketch says no.
- **The symbol.** `@` is still a placeholder.
