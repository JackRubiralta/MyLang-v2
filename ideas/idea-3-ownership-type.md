# Idea 3: an ownership type

**Status:** an idea being explored. Nothing here is in `DOCS.md`.

Lines marked **(Jack)** are things Jack said. Lines marked **(sketch)** are one way it could
be written. Lines marked **(open)** still need a decision.

---

## 1. The idea (Jack)

- "`p@` gives that ownership type. That's why we're able to set it to something else."
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

## 4. Turning an ownership into a value

| Written | Gives | Afterwards |
|---|---|---|
| `var r: ^int = p@;` | an owner | `p` is `null` |
| `var x: int = p@^;` | the plain value | `p` is `null`, and the heap value is freed at the end of the line |
| `var x: int = p^;` | a plain copy | `p` still owns its value |

## 5. Marking the receiver too (sketch of Jack's version)

Today only the giving side of a declaration or a call carries `@`. Jack's version marks the
receiving side as well, so ownership always goes from an `@` to an `@`:

```go
var r@: ^int = q@;                                     // a declaration that receives ownership
const keep: (@^int)void = (var owned@: ^int){ ... };   // a parameter that receives ownership

keep(r@);                  // fine
keep(r);                   // ERROR: keep takes an ownership: write r@
```

For the compiler to check it through a function held in a variable, the mark has to be in
the function's **type** (`(@^int)void`), the same lesson as with `ref`.

| | It gives | It costs |
|---|---|---|
| marking the receiver | one rule with no exception: `@` on both sides everywhere; "both sides" becomes ordinary type checking; a function can refuse a silent deep copy | two kinds of owning parameter (`^T` takes a copy, `@^T` takes yours); a rule for values nothing else holds (`keep(^5)`, `keep(makeNode(1))`); a mark on a declared name is a new position |
| as DOCS.md is now | one owning parameter type, and the caller chooses copy or move at the call | the receiving side of a declaration or call is unmarked |

## 6. Not only on the heap (open)

Ownership itself only moves for a value on the heap. A plain value lives inside its
variable, so giving it to another variable can only copy it, and an `int` has no "empty"
state to be left in.

`@` could still be written on any variable that **holds** heap owners. It would copy what
lives in the variable and move every heap owner inside it:

| `x@` where `x` is | What it would do |
|---|---|
| a heap owner, `^T` | moves the value; `x` is left `null` |
| a struct or array that holds heap owners | copies the plain parts, and moves every owner inside, each left `null` |
| a plain value that owns nothing on the heap | nothing to move: an error |
| a borrower | owns nothing: an error |

DOCS.md (8.9) currently allows only the first row, and moves a struct one owner at a time.
