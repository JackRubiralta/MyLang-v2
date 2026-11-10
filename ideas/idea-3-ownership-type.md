# Idea 3: an ownership type

**Status:** taken into `DOCS.md` on 10 Nov 2026 (Jack: "I think this is good, update
DOCS.md"). Section 8.9 there is the rule now. This file keeps how the idea was reached and
what was dropped on the way.

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
- "I would have `@` in the type, as `q@` returns the type of `@`… I want all this type stuff
  to be consistent and elegant."
- "My idea is that `@x` is a type, or gives one: if `int x = 3`, `@x` is the type `@int`. Then
  I have to use `^` … like I do with the others."
- On `var q: ^int = h^;` as the move: "No, this wouldn't work. `@` is ownership, like as a
  variable, so `var @q =`. That's how we would have to move the ownership to `q`."

## 2. The model in one example

```go
var p: ^int = ^5;

var h: @^int = @p;       // h is p's ownership. Nothing has moved, and p still owns the 5. (sketch)
h^                       // follow it: p itself, a ^int
h^^                      // the 5

var c: ^int = h^;        // a copy, like every =. This is not a move. (Jack)
var @q: ^int = h;        // the move: q owns the 5 now, and p is left null (Jack)
```

| Written | What it is |
|---|---|
| `@p` | `p`'s ownership. Its type is `@^int` |
| `h^` | follow an ownership: the variable it is the ownership of |
| `@q = h;` and `var @q: ^int = h;` | set `q`'s ownership: `q` takes the value over |
| `@q = @p;` | the same, straight from `p` |

## 3. The three marks side by side

| Type | Made with | Followed with | Following gives |
|---|---|---|---|
| `^T` | `^v` | `p^` | the value on the heap |
| `&T` | `&x` | `b^` | the variable it borrows |
| `@T` | `@x` | `h^` | the variable it is the ownership of |

Each mark goes in front in the type and in front of the value that makes it, so every
expression looks like its type: `^5` is a `^int`, `&count` is an `&int`, `@p` is an `@^int`.
Each one is followed with `^` behind.

It works on every variable, not only heap owners (Jack):

```go
var x: int = 3;
var o: @int = @x;        // @x is an @int
o^                       // x itself: 3
```

## 4. Moving: setting a variable's ownership (Jack)

"Every variable would be able to be called on to get its ownership and then change that, or
get its ownership to set it to another variable."

`@q` on the left is `q`'s ownership, and what is put there has to be an ownership too. Both
sides have the type `@T`, so "the mark on both sides" is the ordinary rule that the types
match:

```go
@q = @p;                 // both are @^int: q owns p's heap int, and p is left null
                         // q's old value is freed

var @r: ^int = @q;       // a new variable that takes q's value over

q = @p;                  // ERROR: q is a ^int and @p is an @^int
@q = p;                  // ERROR: @q is an @^int and p is a ^int
```

What the move does depends on what the variable holds (sketch). It mirrors a copy:

| The variable is | `=` (copy) | setting its ownership (move) |
|---|---|---|
| a heap owner, `^T` | a new heap value | the same heap value, and the source is left `null` |
| a struct or array that holds heap owners | every heap value inside is copied | the plain parts are copied, and every owner inside is handed over and left `null` |
| a plain value that owns nothing on the heap | a copy | the same as a copy: there is nothing to hand over |

## 5. In a function's type (Jack)

"That function should be explicit that it's receiving the owner type." The parameter is
declared with exactly the type the function's type lists:

```go
const keep: (@^int)void = (var owned: @^int){
    var @mine: ^int = owned;     // mine takes it over, and the caller's variable is left null
    ...
};

keep(@q);                // fine
keep(q);                 // ERROR: q is a ^int, not an @^int: write @q
```

## 6. Three levels: look, change, take (sketch)

An ownership sits beside a borrower as one more level of what a holder may do to someone
else's variable:

| | read it | change it | take it away |
|---|---|---|---|
| `const b: &T` | yes | no | no |
| `var b: &T` | yes | yes | no |
| `h: @T` | yes | yes | yes |

The mark at a call then shows what the function is allowed to do to your variable: nothing
(`f(x)`, a copy), change it (`f(&x)`), or take it (`f(@x)`).

DOCS.md lets a `var` borrower move a value out today (`takeNext` in 8.12). With an ownership
type that right would belong to `@T`. Whether a borrower keeps it is open.

## 7. In memory (sketch)

"Similar to the way borrowed works, but instead we can transition ownership." (Jack)

An ownership holds the **address of the variable**, the same as a borrower. For `h: @^int`
that is a pointer to a pointer, and each `^` is one jump:

```
h: [ address of p ] ----> p: [ address ] ----> [ 5 ]

after  @q = h;

h: [ address of p ] ----> p: [ null ]
                          q: [ address ] ----> [ 5 ]
```

- The ownership always sits in a real variable: `p` until the move, `q` after it. An `@T`
  value is only a pointer to a variable, with the right to pull its value out.
- Copying an `@T` copies the address, like copying a borrower. Nothing has two owners.

## 8. What follows from that

- **An ownership can't outlive its variable,** because it points at it. This is the rule
  borrowers already have (DOCS 8.5). So `@local` can't be returned from a function. A
  function still gives a heap value back by returning it (DOCS 8.12).
- **`keep(@q)` gives `keep` the right to take `q`, not a promise that it will.** After the
  call `q` is `null` only if `keep` moved it.
- **After a move, an ownership still points at the old variable,** which is now `null`.
  Following it twice (`h^^`) then fails while running.
- **A `const` has no ownership to give:** `@limit` would allow changing or emptying it.

## 9. Considered and dropped

- **Following an ownership as the move** (`var q: ^int = h^;`). Dropped (Jack): following
  gives the variable, and `=` copies it. The move is setting a variable's ownership.
- **The variable lets go as soon as `@x` is written,** with the ownership holding the value
  on its way. Dropped with the line above: the ownership points at the variable, and nothing
  moves until it is set into another variable.
- **An `@T` used directly as a `T`, with nothing to follow.** Dropped (Jack): the three marks
  work the same way.
- **`^` or `&` on a declared name** (`var x^: int = 3;`). The type slot would say `int` for a
  `^int`, and it couldn't tell an owner from a borrower, since both are followed with `^`.

## 10. Open

- **Whether `@p` on its own moves nothing** (sections 2 and 7). That is the reading here.
- **Whether a function that is given an ownership has to use it.** If it did, `q` would
  always be `null` after `keep(@q)`. A function body runs straight through, with no loops or
  `if`, so the compiler could check it.
- **Whether a `var` borrower can still move a value out** (section 6).
- **The type slot in `var @q: ^int = h;`.** It says what `q` is. The `@` on the name says
  that `q`'s ownership is what is being set.
- **The symbol.** `@` is still a placeholder.
- **What it changes in DOCS.md:** 8.9 (the mark in front, an ownership held as a value),
  8.12 (a parameter of type `@T`), and the moves in 8.13.
