# Idea 2, explicit variation: a mark on every type

**Status:** an idea being explored. Nothing here is in `README.md` or `DOCS.md` yet. This is a
variation of [idea 2](idea-2-ownership-in-the-type.md): the same idea, with nothing left
unmarked.

Lines marked **(Jack)** are things Jack said. Lines marked **(sketch)** are one way it could
be written, shown only to make the idea concrete. Lines marked **(open)** still need a
decision.

---

## 1. The idea

"I want symbols everywhere. I want it to always be explicit." (Jack)

Idea 2 puts ownership inside the type. As first written, a plain `int` has no mark, so one
kind of owner is written and the other isn't. In this variation **every type starts with one
of three marks**, and no type is written without one:

| Mark | Who owns it | Where the value is |
|---|---|---|
| `#T` | this holder | right here, in the holder's own place |
| `^T` | this holder | somewhere else, on the heap |
| `&T` | another holder | wherever that owner put it |

`^` does not mean "owner". It means "on the heap". A holder owns its value unless the mark is
`&`.

```go
var count: #int = 3;                    // mine, right here
var big: ^[1000]#int = ^[...];          // mine, on the heap
var view: &[1000]#int = &big^;          // borrowed

const Node: #type = {#int, ^Node, &Node};      // value here, next on the heap, prev borrowed

const add: #(#int, #int)#int = (const x: #int, const y: #int){ return x + y; };
const bump: #(&int)void = (var x: &int){ x^ = x^ + 1; };
```

The parts of a type are types too, so they carry a mark as well: `[3]#int` is three ints
held in place, `[3]^Node` is three nodes each on the heap, and a function's type says how
every parameter is passed.

## 2. Where each symbol goes (sketch)

One rule decides every position: **what adds a mark goes in front, and what removes one goes
behind.**

| | Written | Position | Effect on the type |
|---|---|---|---|
| a type's mark | `#T`, `^T`, `&T` | front | |
| make a heap value | `^v` | front | `T` becomes `^T` |
| make a borrower | `&x` | front | `T` becomes `&T` |
| move ownership | `<-p` | front | stays `^T`, and `p` is left `null` |
| follow a link | `p^` | behind | `^T` or `&T` becomes `T` |

What is behind a name is read first, left to right. What is in front is applied after it.
No brackets are ever needed.

## 3. Nested cases (sketch)

```go
const Node: #type = {#int, ^Node, &Node};        // value, next (heap), prev (borrowed)

var list: ^Node = ^{ value: #int = 1; next: ^Node = null; prev: &Node = null; };

list^.next^.next^.value          // follow, field, follow, field, follow, field: a path, left to right
var cur: &Node = &list^.next^;   // follow the path to the second node, then borrow it
cur^.value = 5;                  // write through a borrower

var kids: #[2]^Node = [^{ ... }, ^{ ... }];
kids[0]^.value                   // index, follow, field

var big: ^[1000]#int = ^[...];
big^[3] = 7;                     // follow, index
var picks: #[2]&int = [&big^[0], &big^[1]];     // follow, index, then borrow
picks[1]^ = 9;

var slot: &^Node = &list^.next;  // a borrower of the link itself, not of the node
slot^^.value                     // follow to the link, follow to the node, field
```

## 4. Converting between the forms (sketch)

| From | To | Written |
|---|---|---|
| a value | a borrower | `&x` |
| a heap owner | a borrower | `&p^` (borrow what `p` points at) |
| either link | the value | `p^` |
| a value | a heap owner | `^v` |
| a borrower | a heap owner | not allowed; `^b^` makes a new owner holding a copy |
| a heap owner | another heap owner | `<-p` moves it (section 6); `^p^` makes a copy |

## 5. Why each position

- **Types in front.** MyLang's own test (README 3.6) found that only all-in-front types are
  never ambiguous. A mark behind, `[3]int^`, could mean an array of links or a link to an
  array.
- **Follow behind.** It is a use, like `a[0]`, `f(1)` and `p.x`, so a path reads left to
  right in the same order as the type. In front it would need brackets: `(*(*list).next).value`.
- **Make in front.** The expression then looks like its type: `^5` is a `^int` and `&x` is a
  `&int`. You also see "heap" or "borrowed" before a long value, not after its last line.
  And the same `^` can mean "make" in front and "follow" behind without clashing.

The reasons for choosing `^` and `&` themselves are in idea 2, section 8.4.

## 6. Moving ownership (sketch)

A borrower can't outlive what it borrows. To make a value outlive the scope that made it,
the value has to be on the heap and its ownership has to move, and the move is always written
out. (Jack)

**The word `hand` is not used.** (Jack) The move is a symbol in front of the old owner. `<-`
is a placeholder for it. (sketch)

```go
const Node: #type = {#int, ^Node};       // value, next

list^.next = <-list^.next^.next;         // the third node moves up, and its old link is left null
                                         // the second node has no owner now, so it is freed
```

A function gives back a heap value the same way, and its type shows it: the type ends in a
`^`, so the caller knows it will own what comes back.

```go
const makeBig: #(#int)^[1000]#int = (const x: #int){
    var big: ^[1000]#int = ^[...];
    return <-big;                        // the array moves out to the caller instead of being freed
};

var b: ^[1000]#int = makeBig(1);         // b owns the array now, and nothing was copied
```

- **A move goes in front,** like the other two things that produce a link (`^v` and `&x`). It
  changes who frees the memory, so it should be the first thing read.
- **Only a `^` link can be moved.** A `#` value lives in its holder, and a `&` link owns
  nothing.
- **The old owner is left `null`,** so there is still exactly one owner.
- **A plain `=` between two heap owners is an error.** You say which you mean: `<-p` moves
  it, and `^p^` makes a new heap copy of what `p` points at.
- **Ownership can't be moved into an inner scope.** (Jack)

| You want | You write | The rule |
|---|---|---|
| to use a value for a while | `&x` | the borrower has to end first |
| a value that outlives this scope | `^v` to make it, `<-p` to pass it on | one owner at a time; the old one is left `null` |

Other ways the move could be spelled (open):

| Spelling | Example | Note |
|---|---|---|
| an arrow in front | `a = <-b;` | shows the value leaving the old owner; used above |
| one character in front, such as `~p` or `@p` | `a = ~b;` | shorter, but the character says nothing about moving |
| an arrow in place of the `=` | `a <- b;` | reads well, but a move inside an array's brackets or after `return` has no `=` to replace |

## 7. Compared with idea 2 as first written

| | Idea 2 as first written | This variation |
|---|---|---|
| a value held in place | `var count: int = 3;` | `var count: #int = 3;` |
| a heap value | `var big: ^[1000]int = ^[...];` | `var big: ^[1000]#int = ^[...];` |
| a struct shape | `{int, ^Node, &Node}` | `{#int, ^Node, &Node}` |
| a function's type | `(&int)void` | `#(&int)void` |
| what `^` means | an owner link | on the heap; every holder owns unless its mark is `&` |
| what it costs | one kind of owner is not written | every type is one character longer |

## 8. What is still open

- **The symbol for "right here".** `#` is a placeholder. It is the commonest mark, so a
  lighter one would make long types easier to read.
- **How far it goes.** As written, every part of a type is marked. A lighter version marks
  only the front of a declared type and the links inside it (`#[3]int`, `{int, ^Node, &Node}`),
  but then some parts are unmarked again.
- **The symbol for the move** (section 6).
- **Fields and indexing through a link.** Here every hop is written (`list^.next^.value`).
  The README (10.4) lets `p.x` and `p[0]` go straight through a pointer.
- **Everything open in idea 2, section 9, is open here too:** read-only borrowers, `null` for
  a link, removing a node that a stored borrower still points at, returning a borrower from a
  function, and the two README decisions it reverses (8.6 and 9.6).
