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
- **"I would have `@` in the type, as `q@` returns the type of `@`… I want all this type stuff
  to be consistent and elegant."**

## 2. What an ownership is in memory

A heap owner `p: ^int` is a place on the stack that holds the address of a value on the
heap. That stored address is the ownership. Nothing else has to exist in memory for it.

```
before                              after  q = @p;

p: [ address ] ----> [ 5 ]          p: [ null ]
q: [ null ]                         q: [ address ] ----> [ 5 ]
```

`p` names the address as something to copy from, `p^` follows it, and `@p` names it as
something to let go of.

A borrower keeps pointing at the original. Letting go leaves nothing pointing back: the
address is copied into the new owner and `null` is written in the old one. If an ownership
were kept as a pointer to the original, to be taken from later, the line `f(@r)` would no
longer say whether `r` is empty afterwards.

## 3. `@` as a type (Jack), and the rules that make it consistent (sketch)

### 3.1 Three marks, written the same way

| Type | Made with | It is |
|---|---|---|
| `^T` | `^v` | a `T` on the heap, owned here |
| `&T` | `&x` | a `T` that `x` still owns |
| `@T` | `@x` | a `T` that nothing else holds any more: `x` has let go of it |

Each mark goes in front in the type, and in front of the value that makes it, so every
expression looks like its type: `^5` is a `^int`, `&count` is an `&int`, and `@q` is an
`@^int`.

```go
var p: ^int = ^5;       // ^5 is a ^int
var b: &int = &count;   // &count is an &int
var r: @^int = @p;      // @p is an @^int
```

(Earlier sketches wrote the mark behind, `q@`. In front follows the rule in DOCS 8.2: what
makes a marked value goes in front, like the mark in its type. (open))

### 3.2 What `@x` does

`@x` gives `x`'s value and makes `x` let go of it: every heap owner in `x` is left `null`.
Nothing is copied.

```go
var q: ^int = ^5;
var r: @^int = @q;      // r owns the 5 now, and q is null
```

### 3.3 An `@T` is a `T`

- **It is used like one.** `r^` is the int. Nothing is written to remove the `@`.
- **It goes wherever a `T` goes:** into a `^int` holder, a `^Node` field, a returned value.
- **It needs no conversion,** because nothing about the value changes: the same bytes, in
  the same place. Every other conversion (`^v`, `p^`, `convert`) changes how or where the
  value is stored. `@` only records that nothing else holds it.

This is the answer to "how do we convert an ownership type to a value": there is nothing to
convert.

### 3.4 One rule for `=`

`=` puts the right side's value in the left holder. If the right side still belongs to a
holder, the left gets a copy. If nothing holds it, the left takes it. DOCS 8.8 already says
the second half for a value made on the spot. `@x` is how a named value becomes one that
nothing holds.

| Line | The right side | What happens |
|---|---|---|
| `var c: ^int = q;` | still `q`'s | `c` gets a copy |
| `var r: ^int = @q;` | let go | `r` takes it, and `q` is left `null` |
| `var n: ^int = ^5;` | made on the spot | `n` takes it |
| `q = @r;` | let go | `q`'s old value is freed, `q` takes `r`'s, and `r` is left `null` |

So the left side needs no mark, and "a move carries `@` on both sides" (DOCS 8.9) is no
longer a rule. Every line that empties an owner still shows `@` on that owner.

### 3.5 A holder of type `@T` refuses copies

A holder whose type starts with `@` only accepts a value that nothing else holds: an `@x`,
or one made on the spot. Its type is fixed by its declaration like any other, so this holds
for its whole life.

```go
var s: @^int = @r;      // fine
s = c;                  // ERROR: c still holds its value, and s doesn't take copies: write @c
s = @c;                 // fine: s takes c's value
s = ^7;                 // fine: made on the spot
```

A function says the same thing in its type, and its parameter is declared with exactly that
type:

```go
const keep: (@^int)void = (var owned: @^int){ ... };

keep(@q);               // fine
keep(^5);               // fine
keep(q);                // ERROR: keep doesn't take copies: write @q
```

| Parameter type | `f(q)` | `f(@q)` | `f(^5)` |
|---|---|---|---|
| `^int` | a copy | taken, and `q` is left `null` | taken |
| `@^int` | error | taken, and `q` is left `null` | taken |

### 3.6 What follows

Because an `@T` is a `T`, these need no extra rule:

```go
const Node: type = {int, ^Node};

list^.next = @list^.next^.next;                        // the third node moves up

list = ^{ value: int = 0; next: ^Node = @list; };      // a new first node takes the old list

var kids: [2]^Node = [@a, @b];                         // each element is taken

const takeNext: (&Node)^Node = (var n: &Node){
    return @n^.next;                                   // the caller owns that node now
};
```

A function's result is always a value that nothing else holds, so a return type never needs
the `@`.

## 4. A handle on someone else's ownership: `&^T`

DOCS.md already has a way to hold on to another variable's owner and move from it later: a
borrower of the owner itself, made with `&p`. It is the address of the owner's own place, a
pointer to a pointer.

```go
const takeFrom: (&^int)^int = (var o: &^int){
    return @o^;                // lets go of the caller's owner, which is left null
};

var p: ^int = ^5;
var mine: ^int = takeFrom(&p);     // p is null, and mine owns the 5
```

| Written, for `o: &^int` | Gives |
|---|---|
| `o^` | the owner: compare it with `null`, copy it, or give it a new value |
| `o^^` | the plain value |
| `@o^` | the value itself, let go: the caller's owner is left `null` |

## 5. Not only on the heap (sketch)

`@` works on every variable, as the mirror of copying. A copy duplicates the heap values
inside. Letting go hands them over:

| `x` is | `x` (copy) | `@x` (let go) |
|---|---|---|
| a heap owner, `^T` | a new heap value | the same heap value; `x` is left `null` |
| a struct or array that holds heap owners | every heap value inside is copied | the plain parts are copied, and every owner inside is handed over and left `null` |
| a plain value that owns nothing on the heap | a copy | the same as a copy: there is nothing to let go of |
| a borrower | another borrower | another borrower: it owns nothing |

```go
var a: Node = { value: int = 1; next: ^Node = makeNode(2); };
var b: Node = @a;       // b.value is a copy of a.value
                        // b.next takes the node a.next owned, and a.next is null
```

It has to work on every type for functions that take a type (DOCS 4.3): such a function
can't know whether a `T` is on the heap.

**The limit.** A value on the stack can't change place: it lives in its holder's own memory
until that holder ends. So letting go of a plain value never avoids copying its bytes, and
can't make it outlive its block. Only what is on the heap really changes owner.

## 6. Considered and dropped

- **The mark on the receiving name** (`var r@: ^int = q@;`). Dropped (Jack): `@q` has a type,
  so the holder's type should say it.
- **`^` or `&` on a declared name** (`var x^: int = 3;`). The type slot would say `int` for a
  `^int`, and it couldn't tell an owner from a borrower, since both are followed with `^`.
- **Both sides always marked, with the types required to be equal.** A returned value and a
  struct field would then need rules of their own, and a holder declared `@^int` would have
  to be written differently from one declared `^int` in every later assignment.
- **An ownership kept as a pointer to the original variable,** taken from later. The call
  would no longer show whether the caller's owner is empty afterwards.

## 7. Open

- **In front or behind:** `@q` like `^5` and `&count`, or `q@`.
- **`@` on a value with no heap owners:** the same as a copy (section 5), or an error.
- **`@[2]^Node` and `[2]@^Node`:** whether a whole array that was let go fits a holder whose
  elements are `@`.
- **Whether `@` may be written in a return type.**
- **The symbol.** `@` is still a placeholder.
- **What it changes in DOCS.md:** 8.9 (no mark on the left, and the mark in front), 8.12 (a
  parameter can refuse copies), and the one-line forms in 8.13.
