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
- **"My idea is that `@x` is a type, or gives one: if `int x = 3`, `@x` is the type `@int`. Then
  I have to use `^` to own it and give [it to a] variable, like I do with the others."**

## 2. What an ownership is in memory

A heap owner `p: ^int` is a place on the stack that holds the address of a value on the
heap. That stored address is the ownership. Nothing else has to exist in memory for it.

```
before                              after  @q = @p;

p: [ address ] ----> [ 5 ]          p: [ null ]
q: [ null ]                         q: [ address ] ----> [ 5 ]
```

`p` names the address as something to copy from, `p^` follows it, and `@p` names it as
something to let go of.

A borrower keeps pointing at the original. Letting go leaves nothing pointing back: the
address is copied into the new owner and `null` is written in the old one. If an ownership
were kept as a pointer to the original, to be taken from later, the line `f(@r)` would no
longer say whether `r` is empty afterwards.

## 3. `@` as a type that works like the other two (Jack)

### 3.1 Three marks, written and used the same way (Jack)

| Type | Made with | Followed with | Following gives |
|---|---|---|---|
| `^T` | `^v` | `p^` | the value on the heap |
| `&T` | `&x` | `b^` | the value it borrows |
| `@T` | `@x` | `o^` | the value, as yours: the variable it is put in owns it |

Each mark goes in front in the type and in front of the value that makes it, so every
expression looks like its type. Each one is followed with `^` behind.

```go
var x: int = 3;
var o: @int = @x;        // @x is an @int
var y: int = o^;         // follow it: y has the 3

var p: ^int = ^5;
var h: @^int = @p;       // @p is an @^int
var q: ^int = h^;        // follow it: q owns the heap int now
```

### 3.2 Making one: `@x` (sketch)

- `@x` has type `@T` when `x` is a `T`. It works on every variable.
- **`x` lets go at that moment.** Every heap owner in `x` is left `null` on the line that
  shows `@x`. Nothing is copied.
- A plain value has nothing on the heap to let go of, so `@x` on an `int` holds a copy and
  `x` keeps its value.
- `@` in front of a value made on the spot wraps it, and nothing is emptied: `@^5` is an
  `@^int`.

### 3.3 Following one: `o^` (sketch)

- `o^` gives the `T`. Put in a variable, that variable owns it. Nothing is copied.
- **Following hands the value over, so an ownership is followed once.** After `h^`, `h` is
  empty, and following it again fails while running.
- So there is no looking through an ownership. Land it in a variable first, then use that
  variable. `var n: int = h^^;` reads the int once, and the heap value is then freed at the
  end of the line, because nothing received it.

### 3.4 Setting one ownership from another (Jack)

"`p@` gives that ownership type. That's why we're able to set it to something else." Both
sides have the same type, so this is an ordinary assignment:

```go
@q = @p;                 // both are @^int: q owns p's heap int, and p is left null
                         // q's old value is freed

@list^.next = @list^.next^.next;     // the third node moves up
```

An ownership can't be copied, so a named ownership given to another holder is an error:
`var h2: @^int = h;`. Passing one on is written `@h^` (follow it, and hand that over).

### 3.5 In a function's type (Jack)

"That function should be explicit that it's receiving the owner type." The parameter is
declared with exactly the type the function's type lists:

```go
const keep: (@^int)void = (var owned: @^int){
    var mine: ^int = owned^;     // follow it to own it
    ...
};

keep(@q);                // fine: q is left null
keep(@^5);               // fine: a value made on the spot, handed over
keep(q);                 // ERROR: q is a ^int, not an @^int: write @q
```

A function that hands back something it took returns an ownership, and the caller follows
it:

```go
const takeNext: (&Node)@^Node = (var n: &Node){
    return @n^.next;                 // n^.next is left null
};

var second: ^Node = takeNext(&list^)^;
```

### 3.6 In memory (sketch)

- **An `@^int` is the address the heap owner was holding:** the real pointer. `@p` moves
  that address out of `p` and leaves `null` there. `h^` moves it into the variable that
  follows it. The value on the heap never moves.
- **An `@int` is the int itself.**
- **It doesn't point back at the variable it came from.** If it did, like a borrower, it
  could outlive that variable, and after `keep(@q)` nothing would say whether `q` is empty.
  (Jack described it as "similar to the way borrowed works… more like a real pointer": it is
  a pointer when the value is on the heap, but to the value, not to the variable.)

### 3.7 What it costs

- **One `^` that isn't a jump in memory.** An `@^int` and a `^int` both hold the same
  address, so the first `^` in `h^` only changes the type.
- **A move into a new plain variable takes two lines** (`var h: @^int = @p;` then
  `var q: ^int = h^;`), because `(@p)^` would need brackets.
- **An ownership is used once,** so a parameter of type `@^int` has to be landed in a
  variable before its value is used.

## 4. A handle on someone else's ownership: `&^T`

DOCS.md already has a way to hold on to another variable's owner and move from it later: a
borrower of the owner itself, made with `&p`. It is the address of the owner's own place, a
pointer to a pointer.

```go
const takeFrom: (&^int)@^int = (var o: &^int){
    return @o^;                // the ownership of the caller's heap int; the caller's owner is left null
};

var p: ^int = ^5;
var mine: ^int = takeFrom(&p)^;    // p is null, and mine owns the 5
```

| Written, for `o: &^int` | Gives |
|---|---|
| `o^` | the owner: compare it with `null`, copy it, or give it a new value |
| `o^^` | the plain value |
| `@o^` | the ownership of that value: the caller's owner is left `null` |

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
var h: @Node = @a;      // a.next is left null
var b: Node = h^;       // b.value is a copy of a.value, and b.next owns the node a.next owned
```

It has to work on every type for functions that take a type (DOCS 4.3): such a function
can't know whether a `T` is on the heap.

**The limit.** A value on the stack can't change place: it lives in its holder's own memory
until that holder ends. So letting go of a plain value never avoids copying its bytes, and
can't make it outlive its block. Only what is on the heap really changes owner.

## 6. Considered and dropped

- **An `@T` used directly as a `T`, with nothing to follow.** Dropped (Jack): the three marks
  should work the same way. An ownership is made in front and followed with `^`.
- **The mark on the receiving name** (`var r@: ^int = q@;`). Dropped (Jack): `@q` has a type,
  so the holder's type should say it.
- **`^` or `&` on a declared name** (`var x^: int = 3;`). The type slot would say `int` for a
  `^int`, and it couldn't tell an owner from a borrower, since both are followed with `^`.
- **An ownership that points back at the original variable,** taken from when it is followed
  (3.6).

## 7. Open

- **When `x` lets go:** at `@x` (the sketch), or when the ownership is followed.
- **Whether a second follow is caught when compiling.** There are no loops or `if`, so a
  function body runs straight through and the compiler could see it.
- **In front or behind:** `@x` reads like its type. `x@` would let a move into a plain
  variable be one line, `var q: ^int = p@^;`, with no brackets.
- **`@` on a value with no heap owners:** a copy (3.2), or an error.
- **The symbol.** `@` is still a placeholder.
- **What it changes in DOCS.md:** 8.9 (the mark in front, both sides typed `@T`, and the
  follow), 8.12 (a parameter of type `@T`), and the list in 8.13.
