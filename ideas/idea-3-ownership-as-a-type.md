# Idea 3: ownership as a type

**Status:** an idea being explored. It builds on section 8 of `DOCS.md` and is not in it yet.

Lines marked **(Jack)** are things Jack said. Lines marked **(sketch)** are one way it could
be written, shown only to make the idea concrete. Lines marked **(open)** still need a
decision.

---

## 1. The idea (Jack)

- "I think we should have an ownership type. `p@` gives that ownership type. That's why we're
  able to set it to something else."
- "When we do `r@` and pass that into a function, that function should be explicit that it's
  receiving the owner type. Or do `var owned@: ^int`."
- "It would be something more intrinsic to the variable, because every variable would be able
  to be called on to get its ownership… and in theory, this should work not only on heap."

## 2. One way to read it (sketch)

`x@` means **"x, to be taken, not copied"**, and it has a type of its own, `@T`.

| Written | Type | Meaning |
|---|---|---|
| `x` | `T` | the value: giving it to something makes a copy |
| `x@` | `@T` | the same value, to be taken: giving it to something empties `x` |
| `name@: T` | | a holder that takes what it is given |
| `(@T)void` | | a function whose parameter takes what it is given |

```go
var r@: ^int = q@;                          // r takes q's heap int, and q is left null

const keep: (@^int)void = (var owned@: ^int){ ... };
keep(r@);                                   // the call and the type both show the take
```

The rule "a move carries `@` on both sides" then stops being a rule of its own. It is the
ordinary rule that both sides of `=` have the same type:

| Line | Types | |
|---|---|---|
| `q = p;` | `^int` and `^int` | fine: a copy |
| `q@ = p@;` | `@^int` and `@^int` | fine: a take |
| `q = p@;` | `^int` and `@^int` | error: the types differ |
| `q@ = p;` | `@^int` and `^int` | error: the types differ |

## 3. Why do it

- **One rule fewer.** Both-sides is just type matching.
- **Every call matches its function's type.** `f(n)` goes with `(int)`, `f(&n)` with `(&int)`,
  and `f(r@)` with `(@^int)`. Today `use(big)` and `use(big@)` both fit `(^[1000]int)void`,
  the one place where two different calls fit one type.
- **Functions written for any type.** A function that takes a type `T` (DOCS 4.3) can't know
  whether a `T` is on the heap. If it should be able to take a `T` without copying, `@` has
  to be allowed on every type. This is the reason "not only on heap" is right.

## 4. What taking does, for every type (sketch)

Taking mirrors copying. A copy duplicates the heap values inside. A take takes them over.

| The value is | `=` (copy) | `@` (take) |
|---|---|---|
| a heap owner, `^T` | a new heap value | the same heap value: the source is left `null` |
| a plain value that owns nothing on the heap | a copy | the same as a copy: there is nothing to take |
| a struct or array that holds heap owners | every heap value inside is copied | every heap owner inside is taken and left `null`; the plain parts are copied |
| a borrower, `&T` | another borrower | another borrower: it owns nothing |

```go
const Node: type = {int, ^Node};
var a: Node = { value: int = 1; next: ^Node = makeNode(2); };

var b@: Node = a@;      // b.value is a copy of a.value
                        // b.next takes over the node a.next owned, and a.next is null
```

The source stays an ordinary variable afterwards. Only the heap owners in it are `null`.

## 5. From an ownership to a value (sketch)

An ownership is never held on its own and converted later. `x@` goes straight into a holder
marked `@`: a variable, a field or a parameter. From then on that holder is an ordinary
variable of type `T`, and its value is read the usual way (`owned^` for a `^int`).

## 6. In memory (sketch)

Nothing new is stored.

- **For a `^T`** the ownership is the address the owner already holds. Taking copies that
  address into the new owner and writes `null` in the old one. The value doesn't move.
- **For a plain value** the bytes are copied into the new holder, and the heap addresses
  inside it are taken as above.

A borrower keeps pointing at the original. A take leaves nothing pointing back. If an
ownership were kept as a pointer to the original, to be taken from later, the line `f(r@)`
would no longer say whether `r` is empty afterwards.

## 7. The limit

A value on the stack can't change place: it lives in its holder's own memory until that
holder ends. So taking a plain value never avoids copying its bytes, and it can't make the
value outlive its block. Only what is on the heap really changes owner. "Works on every
variable" holds as a rule, and the effect is on the heap owners.

## 8. Open

- **Whether a plain `T` parameter also accepts `x@`.** If not, the call and the type always
  match, but a caller who is finished with a value still pays for a copy when the parameter
  is `(^T)`. If so, the caller chooses, as in DOCS 8.12 now.
- **A value nothing else holds** (`^5`, the result of a call) given to an `@` holder.
- **In front or behind:** `@x` like `&x`, or `x@` like `arr[1]`.
- **Whether an `@T` can be stored,** for example in an array. The sketch says no.
- **The symbol.** `@` is still a placeholder.
