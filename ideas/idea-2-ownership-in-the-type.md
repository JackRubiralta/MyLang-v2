# Idea 2: ownership inside the type

**Status:** an idea being explored. Nothing here is in `README.md` or `DOCS.md` yet.

Lines marked **(Jack)** are things Jack said. Lines marked **(sketch)** are one way it could
be written, shown only to make the idea concrete. Lines marked **(open)** still need a
decision.

---

## 1. The idea

Whether something owns or borrows is written **inside the type**, and an operation converts
between the forms, the way `&` and `*` do for pointers. (Jack)

Two modifiers go in front of a type, like `*` and `[3]` do:

| Type | Meaning |
|---|---|
| `T` | the value itself, held directly. It belongs to whatever holds it. |
| `owner T` | a link to a `T` that this holder owns. The `T` is on the heap. |
| `borrower T` | a link to a `T` that something else owns. |

Declarations go back to `const` and `var` only, with no extra word:

```go
var count: int = 3;                         // the value itself
var big: owner [1000]int = [...];           // owns an array on the heap
var same: borrower int = &count;            // borrows count
```

A plain value needs no word, because it can only belong to whatever holds it.

## 2. One word for the whole type, or nested? (Jack's question)

It nests. `owner` and `borrower` are modifiers, so they stack in any combination like every
other modifier in MyLang:

| Type | Read as |
|---|---|
| `owner Node` | a link to a node that this holder owns |
| `[3]owner Node` | an array of 3 links, each owning its own node |
| `owner [3]Node` | one link that owns an array of 3 nodes |
| `{int, owner Node, borrower Node}` | a struct holding a value, a link it owns and a link it borrows |
| `(borrower Node)owner Node` | a function that borrows a node and gives back one the caller owns |
| `{int}borrower Node` | a map whose values are borrowed links |

It has to nest for the idea to be worth it. One word around a whole type couldn't say "this
field owns and that field borrows", which is the problem idea 1 ran into.

### What owning covers when types nest

1. A value held directly belongs to whatever holds it.
2. An `owner` link owns the value it points at, and everything that value owns in turn.
3. A `borrower` link owns nothing, and anything reached through it is borrowed too.

```go
const Node: type = {int, owner Node, borrower Node};      // value, next, prev
```

Owning a `Node` means owning its `value` and everything down its `next` chain. It does not
mean owning what `prev` points at.

## 3. Converting between the forms (sketch)

A value never changes type by itself, so each change of form is written out:

| From | To | Written | What happens |
|---|---|---|---|
| a value `T` | `borrower T` | `&x` | no copy: the borrower refers to `x` |
| `owner T` | `borrower T` | `&*p` | no copy: it refers to what `p` owns |
| `owner T` or `borrower T` | the value `T` | `*p` | reads the value, or writes it with `*p = …` |
| a value `T` | `owner T` | written in place, or an operator | a new value on the heap, a copy |
| `owner T` | `owner T` | a move, or a plain copy | (open, section 6) |
| `borrower T` | `owner T` | not allowed | a borrower can't give away what it doesn't own |

```go
var count: int = 3;
var b: borrower int = &count;       // value to borrower
*b = 5;                             // count is now 5
var copy: int = *b;                 // borrower back to a value: copy is 5

var big: owner [1000]int = [...];
var view: borrower [1000]int = &*big;     // owner to borrower
view[0]                                   // indexing goes straight through a link
```

The spellings are placeholders. (Jack: "we'll think of something", "kind of the way that
you convert a pointer back to its value")

## 4. What it would change

| Today | Under this idea |
|---|---|
| kinds `const`, `var`, `ref`, `heap` | kinds `const`, `var` only |
| `ref r: int = b;` | `var r: borrower int = &b;` |
| `heap big: [3]int = …;` | `var big: owner [3]int = …;` |
| the pointer type `*T` | `owner T` or `borrower T` |
| a fat pointer `*[]T` | `owner []T` or `borrower []T` |
| `bump(count)` with a `ref` parameter | `bump(&count)` with a `borrower int` parameter |
| a function's type can't show sharing or handing | `(borrower int)void`, `(int)owner Node` |

- **There is no separate pointer type.** `owner T` and `borrower T` are the two kinds of
  pointer: underneath, each is an address. The plain `*T` would only be needed for a pointer
  that is neither, which nothing checks. (to decide: whether such an unchecked pointer exists
  at all)
- **Function types show everything.** A caller holding a function value can tell from the
  type whether it shares, borrows or gives back something owned. That removes problem 14.1 in
  the README.
- **The call shows sharing.** `bump(&count)` says `count` may change.
- **A function can take ownership,** with a parameter of type `owner Node`.
- **Struct fields say it in the shape,** so every value of a type agrees.

## 5. A linked list under this idea (sketch)

```go
const Node: type = {int, owner Node};

var list: Node = { value: int = 1; next: owner Node = null; };

list.next = { value: int = 2; next: owner Node = null; };        // written in place, so list.next owns it
list.next.next = { value: int = 3; next: owner Node = null; };

var cur: borrower Node = &list;          // borrows the first node
cur = &*list.next;                       // now borrows the second
```

A block written straight into an `owner` place becomes a new value owned by that place, the
same way a number written in the code takes the type of the place it's written for. No
`heap` word is needed. (Jack: doesn't want to write `heap` to put a node on the heap)

Functions:

```go
const bump: (borrower int)void = (var x: borrower int){ *x = *x + 1; };
bump(&count);

const makeNode: (int)owner Node = (const v: int){
    return { value: int = v; next: owner Node = null; };
};
var first: owner Node = makeNode(1);
```

## 6. Rules that carry over, and what they'd need

- **One owner.** Every heap value has exactly one `owner` link.
- **A borrower can't outlive what it borrows.** (Jack) For a name, the compiler compares
  scopes. For a borrower stored inside a struct, the same open case as idea 1 section 8.4
  applies.
- **Two links to one value work** when one is an `owner` and the other a `borrower`. (Jack)
- **A borrower of a `const` has to be read-only.** (Jack) Where "read-only" is written is
  open: one sketch is that a `const` name holding a borrower can read through it and a `var`
  name can also write through it.
- **Owner to owner.** A move that leaves `null` behind is sketched in 8.9. The word `hand`
  is not used for it. (Jack)

## 7. Compared with idea 1

| | Idea 1: word on the declaration | Idea 2: word in the type |
|---|---|---|
| Where it's written | `var owner x: int`, `var borrower y: int` | `var x: int`, `var y: borrower int` |
| A plain value | says `owner` every time | says nothing |
| A struct field | needs the word in the shape anyway | already in the shape |
| An array of links | one word for the whole array | each link's type says it |
| A function's type | can't show it | shows it |
| A call that shares | `bump(count)`: doesn't show | `bump(&count)`: shows |
| Converting | nothing to write | `&` and `*` written out |
| Kinds of name | `const`, `var` plus a role word | `const`, `var` |

## 8. Symbols instead of the words (sketch)

Since `owner T` and `borrower T` replace the pointer type, the pointer and reference symbols
are free to be used instead of the words. (Jack)

### 8.1 The rule for where a symbol goes

MyLang already has this pattern: what **builds** goes in front, and what **uses** goes
behind.

| | Type (in front) | Building a value (in front) | Using it (behind) |
|---|---|---|---|
| array | `[3]int` | `[1, 2, 3]` | `a[0]` |
| function | `(int)int` | `(const x: int){ … }` | `f(1)` |
| struct | `{int, int}` | `{ x: int = 1; … }` | `p.x` |
| owner link | `^int` | `^5` | `p^` |
| borrower link | `&int` | `&count` | `b^` |

So the symbols go **before** in a type and when making a link, and **after** when following
one. That keeps rule 3.3 of the README true: a type reads in the order you use the value.

### 8.2 The three symbols

| Symbol | Where | Meaning |
|---|---|---|
| `^T` | in a type | an owner link to a `T` |
| `&T` | in a type | a borrower link to a `T` |
| `^v` | in front of a value | a new owned copy of `v`, on the heap |
| `&x` | in front of a variable | a borrower of `x` |
| `p^` | behind a link | the value the link points at (works for both kinds) |

```go
var count: int = 3;
var b: &int = &count;               // b borrows count
b^ = 5;                             // follow b: count is now 5
var copy: int = b^;                 // copy is 5

var big: ^[1000]int = ^[...];       // big owns an array on the heap
var view: &[1000]int = &big^;       // borrow what big owns
view^[0]                            // follow, then index

const Node: type = {int, ^Node, &Node};      // value, next (owns), prev (borrows)

const bump: (&int)void = (var x: &int){ x^ = x^ + 1; };
bump(&count);
```

### 8.3 The conversions with symbols

| From | To | Written |
|---|---|---|
| a value | a borrower | `&x` |
| an owner | a borrower | `&p^` (borrow what `p` points at) |
| either link | the value | `p^` |
| a value | an owner | `^v` |
| a borrower | an owner | not allowed; `^b^` makes a new owner holding a copy |
| an owner | another owner | `<-p` moves it (8.9); `^p^` makes a copy |

What is behind a name is read before what is in front of it, so none of these needs brackets:
`&p^` is "follow `p`, then borrow".

Jack's list of what has to convert: a borrower or a heap value into a plain value, and a plain
value into a heap value or a borrower. Three symbols do all four:

| Conversion | Written |
|---|---|
| a borrower into a plain value | `b^` |
| a heap value into a plain value | `p^` |
| a plain value into a heap value | `^v` |
| a plain value into a borrower | `&x` |

The other rows of the first table are two of these in a row. Only passing a heap value from
one owner to another needs something new, and that is the move (8.9).

**`^v` makes a copy.** (Jack) `v` stays as it is, and the heap gets a value of its own. A
value written on the spot, like `^5` or `^{ … }`, has no original, so it is simply built on
the heap. `p^` put into a plain variable copies as well. Only `&x` and `<-p` give a link
without copying anything.

### 8.4 Why these symbols

- **Types read in the order they're used, with no brackets.** For `ps: [2]&int` the use is
  `ps[0]^`: index, then follow. For `p: ^[3]int` it is `p^[0]`: follow, then index. With the
  C form `*p` in front, the second one needs brackets, `(*p)[0]`.
- **`^` is not an operator between two values,** so `a^ * b` can only mean "follow `a`, times
  `b`". A `*` behind a name would be confused with multiplying.
- **`&` already means "borrow"** in C, C++, Go and Rust.
- **Pascal and Odin use this form:** the pointer type is `^T`, following is `p^`, and Odin
  takes an address with `&x`.

### 8.5 Other spellings considered

| Spelling | Type | Follow | Problem |
|---|---|---|---|
| words | `owner T`, `borrower T` | `*p` | long, and still needs symbols for converting |
| C style | `*T`, `&T` | `*p` in front | breaks the order rule and needs brackets |
| Zig style | `*T`, `&T` | `p.*` behind | works, but `*T` reads like an unchecked C pointer |
| symbols behind the type | `int^`, `int&` | `p^` | MyLang puts every modifier in front (3.1): `[3]int^` could be read two ways |

### 8.6 Open with this spelling

- **Fields, indexing and calls through a link.** The README (10.4) lets `p[0]` and `p.x` go
  straight through a pointer. With `^` behind there is no bracket problem, so every hop could
  be written: `list.next^.next^.value`. More to write, but each `^` shows a jump to another
  place in memory.
- **A block written straight into an owner place** (section 5) would no longer be needed:
  `list.next = ^{ … };` keeps the same type on both sides of the `=`.

### 8.7 Three cases, two marks (Jack)

| Written | Who owns it | Where the value is |
|---|---|---|
| `T`, no mark | this holder | right here, in the holder's own place |
| `^T` | this holder | somewhere else, on the heap |
| `&T` | another holder | wherever that owner put it |

- **No mark hides nothing.** There are three cases and the other two are always marked, so a
  type with no mark can only be "mine, right here". (Jack: "we know because we're just not
  saying")
- **The first case needs no mark because its ownership never goes anywhere.** A plain value
  lives inside its holder and ends with it. It can be copied, but it can't be passed to
  another owner. Ownership only moves for a value on the heap. (Jack)
- **`^` means "on the heap", not "owner".** A holder owns its value unless the mark is `&`.
- **A plain value can still hold a heap link.** A `Node` held in place has a `^Node` field.
  The `Node` itself never moves, but the link inside it follows the heap rules in 8.9.

A variation that marks the first case too (`#T`) is in
[idea-2-explicit-variation.md](idea-2-explicit-variation.md). Jack found it too much: it adds
a character to every type to say something already known.

### 8.8 Before or after: every position, with nested cases (sketch)

One rule decides every position: **what adds a mark goes in front, and what removes one goes
behind.**

| | Written | Position | Effect on the type |
|---|---|---|---|
| a type's mark | `^T`, `&T` | front | |
| make a heap value | `^v` | front | `T` becomes `^T` |
| make a borrower | `&x` | front | `T` becomes `&T` |
| move ownership | `<-p` | front | stays `^T`, and `p` is left `null` |
| follow a link | `p^` | behind | `^T` or `&T` becomes `T` |

What is behind a name is read first, left to right. What is in front is applied after it.
No brackets are ever needed.

The marks in a type and the `^`s in its use come in the same order:

| Type | It is | Used as |
|---|---|---|
| `[3]^Node` | three links, each to a node on the heap | `a[0]^.value` |
| `^[3]Node` | one link to three nodes held together on the heap | `a^[0].value` |
| `[2]&int` | two borrowers | `a[1]^` |
| `&[2]int` | one borrower of a whole array | `a^[1]` |
| `{int, ^Node, &Node}` | a value, a heap link and a borrower | `n.next^.prev^.value` |
| `&^Node` | a borrower of a link | `s^^.value` |

```go
const Node: type = {int, ^Node, &Node};          // value, next (heap), prev (borrowed)

var list: ^Node = ^{ value: int = 1; next: ^Node = null; prev: &Node = null; };

list^.next^.next^.value          // follow, field, follow, field, follow, field: a path, left to right
var cur: &Node = &list^.next^;   // follow the path to the second node, then borrow it
cur^.value = 5;                  // write through a borrower

var kids: [2]^Node = [^{ ... }, ^{ ... }];
kids[0]^.value                   // index, follow, field

var big: ^[1000]int = ^[...];
big^[3] = 7;                     // follow, index
var picks: [2]&int = [&big^[0], &big^[1]];       // follow, index, then borrow
picks[1]^ = 9;

var slot: &^Node = &list^.next;  // a borrower of the link itself, not of the node
slot^^.value                     // follow to the link, follow to the node, field
```

Why each position:

- **Types in front.** MyLang's own test (README 3.6) found that only all-in-front types are
  never ambiguous. A mark behind, `[3]int^`, could mean an array of links or a link to an
  array.
- **Follow behind.** It is a use, like `a[0]`, `f(1)` and `p.x`, so a path reads left to
  right in the same order as the type. In front it would need brackets: `(*(*list).next).value`.
- **Make in front.** The expression then looks like its type: `^5` is a `^int` and `&x` is a
  `&int`. You also see "heap" or "borrowed" before a long value, not after its last line.
  And `^` behind a name already means "follow", so "make" has to be on the other side.

Open: a type is a value in MyLang, so `^Node` can be written where a value is expected
(`const P: type = ^Node;`). In front of a type the mark builds a type, and in front of a value
it builds a link. The two can't be confused as long as a type itself can't be put on the heap
or borrowed.

### 8.9 Moving ownership (sketch)

A borrower can't outlive what it borrows. To make a value outlive the scope that made it,
the value has to be on the heap and its ownership has to move, and the move is always written
out. (Jack)

**The word `hand` is not used.** (Jack) The move is a symbol in front of the old owner. `<-`
is a placeholder for it. (sketch)

```go
const Node: type = {int, ^Node};         // value, next

list^.next = <-list^.next^.next;         // the third node moves up, and its old link is left null
                                         // the second node has no owner now, so it is freed

list = ^{ value: int = 0; next: ^Node = <-list; };    // a new first node takes over the old list
```

A function gives back a heap value the same way, and its type shows it: the type ends in a
`^`, so the caller knows it will own what comes back.

```go
const makeBig: (int)^[1000]int = (const x: int){
    var big: ^[1000]int = ^[...];
    return <-big;                        // the array moves out to the caller instead of being freed
};

var b: ^[1000]int = makeBig(1);          // b owns the array now, and nothing was copied
```

- **A move goes in front,** like the other two things that produce a link (`^v` and `&x`). It
  changes who frees the memory, so it should be the first thing read.
- **Only a `^` link can be moved.** A plain value lives in its holder, and a `&` link owns
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

Open with moving:

- **Copying a plain value that holds a heap link.** `var b: Node = a;` would give the node
  behind `a.next` two owners. Either it is an error and you move or copy the link yourself,
  or it copies every node behind it, which is work the line doesn't show.
- **Moving out of a `const` owner.** The move changes the old owner to `null`, so it would
  have to be an error.
- **A `^T` parameter:** whether a heap value can be moved into a function at all.

## 9. What is still open

- **The spellings** of the conversions in section 3, and how a value becomes an `owner`.
- **Read-only borrowers:** where it's written that a borrower can't change what it borrows.
- **`null`:** whether an `owner` or `borrower` link can be empty, which a list's last node
  needs.
- **The symbol for the move,** and the three open points under 8.9. The word `hand` is not
  used. (Jack)
- **Whether `^` in front of a type and in front of a value can share one symbol** (8.8).
- **Removing a node that a stored borrower still points at** (idea 1, section 8.4).
- **Returning a borrower from a function:** what it is allowed to borrow from.
- **It reverses two earlier decisions.** The README says `ref` is not a type (8.6) and that
  `heap` is not part of the type (9.6). The reason given there was that a reference type
  couldn't go everywhere other types go. Here a `borrower T` can go everywhere, and the
  lifetime rule is what keeps it safe.
