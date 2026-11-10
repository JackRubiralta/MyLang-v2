# Idea 1: owner, heap or borrower on every declaration

**Status:** an idea being explored. Nothing here is in `README.md` or `DOCS.md` yet.

Lines marked **(Jack)** are things Jack said. Lines marked **(to check)** were filled in to
make the idea complete, and still need a yes.

---

## 1. The idea

Every declaration says two things before the name:

```go
kind role name: Type = value;
```

| Word | Choices | What it says |
|---|---|---|
| `kind` | `const`, `var` | whether the value can change |
| `role` | `owner`, `heap`, `borrower` | who is responsible for the value, and where it lives |

| Role | Meaning |
|---|---|
| `owner` | this name owns the value, and the value is on the stack |
| `heap` | this name owns the value, and the value is on the heap |
| `borrower` | another name owns the value, and this name only uses it |

- **Every declaration says its role, every time.** (Jack)
- **The role never changes after the declaration.** (Jack)
- **`heap` is a kind of owner,** so nothing is written twice: there is no `owner heap`.
  (Jack: "maybe we have owner, borrower, heap")

## 2. The six combinations

```go
var owner count: int = 3;                   // owns it, on the stack, can change
const owner limit: int = 10;                // owns it, on the stack, never changes

var heap grid: [1000]int = [...];           // owns it, on the heap, can change
const heap table: [1000]int = [...];        // owns it, on the heap, never changes

var borrower same: int = count;             // another name for count, and can change it
const borrower view: [1000]int = table;     // another name for table, read-only
```

## 3. What it replaces

| Today | Under this idea |
|---|---|
| `ref r: int = b;` | `var borrower r: int = b;` |
| `heap grid: [3]int = …;` (always behaves like `var`) | `var heap grid: [3]int = …;` |
| no `const` heap value | `const heap table: [1000]int = …;` |
| no read-only share | `const borrower view: []int = scores;` |

- **The word `ref` goes away.** `borrower` does its job. (Jack)
- **`heap` stops being a kind of name and becomes a role,** so `const` and `var` stay
  separate from it and a `const` heap value is possible. (Jack: "I would want const")

## 4. The rules

### 4.1 What each role can be given

| Role | Is given | What happens |
|---|---|---|
| `owner` | a value | it gets its own copy, on the stack |
| `heap` | a value | it gets its own copy, on the heap |
| `heap` | a handed heap value (section 5) | it takes the value over, with no copy |
| `borrower` | an existing variable, array element or map entry | it becomes another name for it |

A borrower can never be given a plain value: `var borrower bad: int = 3;` is an error,
because `3` has no owner. (to check: this is today's rule for `ref`)

### 4.2 `const` and `var`

- **A borrower of a `const` has to be `const` too.** (Jack)
- A `const` borrower of a `var` is allowed. It is a read-only view: the owner can still
  change the value, and the borrower sees the change. (to check)

| The owner is | `var borrower` | `const borrower` |
|---|---|---|
| `var` | allowed: can read and change | allowed: can only read |
| `const` | **error** | allowed: can only read |

```go
const owner limit: int = 10;
var borrower a: int = limit;        // ERROR: limit is const, so its borrower has to be const
const borrower b: int = limit;      // fine

var owner count: int = 3;
const borrower c: int = count;      // fine: c can read count but not change it
count = 4;                          // c now reads 4
```

### 4.3 A borrower can't outlive its owner

A borrower has to be declared in the same scope as its owner or in an inner one. (Jack, and
it is already rule 8.4 in the README for `ref`)

```go
var owner x: int = 3;
{
    var borrower y: int = x;        // fine: y ends before x
}
```

### 4.4 Same memory

A borrower and its owner are the same memory. A write through a `var borrower` changes the
owner's value, and nothing is copied or freed. (Jack)

### 4.5 When a value is removed

A value is removed when its owner's scope ends: an `owner` value comes off the stack, and a
`heap` value is freed. A borrower never removes anything.

## 5. `hand` under this idea

`hand` moves a heap value from one `heap` name to another without copying it.

```go
const owner makeBig: (int)[1000]int = (const owner x: int){
    var heap big: [1000]int = [...];
    hand big;                              // big is handed to the caller instead of being freed
};

var heap b: [1000]int = makeBig(1);        // b takes the value over: no copy
var owner c: [1000]int = makeBig(1);       // c is a copy on the stack; the handed value is freed
```

- **Only a `heap` value can be handed,** because only heap memory can outlive its scope.
- **After a hand, the old name is empty** and can't be used again. The role is fixed, so it
  can't turn into a borrower. (follows from "the role never changes")
- **Ownership can't be handed into an inner scope.** (Jack)
- **`hand` passes on the value and everything that value owns.** (Jack, already in the README)

## 6. Functions and parameters

A parameter is declared like any name, so it says its role too:

```go
const owner double: (int)int = (const owner n: int){ return n * 2; };      // n is the function's own copy
const owner bump: (int)void = (var borrower x: int){ x = x + 1; };         // x is the caller's variable
const owner total: ([]int)int = (const borrower v: []int){ ... };          // shared, read-only, no copy
```

| Parameter | What the function gets |
|---|---|
| `const owner` | its own copy, which it can't change |
| `var owner` | its own copy, which it can change |
| `var borrower` | the caller's variable itself |
| `const borrower` | the caller's value, shared and read-only |

A function and a named type are values, so their declarations say `owner` as well. (to check)

## 7. Structs

A struct's fields carry no words of their own. They follow the struct's kind and role, the
same way they already follow its `const` or `var`:

```go
const owner Point: type = {int, int};

var owner p: Point = { x: int = 1; y: int = 2; };        // every field is on the stack and can change
var heap q: Point = { x: int = 1; y: int = 2; };         // every field is on the heap
const borrower view: Point = p;                          // a read-only name for p
```

## 8. Pointers and linked data (discussed, not decided)

Lines marked **(sketch)** are one way it could be written, shown only to make the idea
concrete.

### 8.1 What a linked list needs

- **A pointer in the type.** A `Node` can't contain a `Node`, because it would have no fixed
  size, so the shape has to be `{int, *Node}`.
- **`null`** for the last node's link.
- **A pointer says whether it owns or borrows.** `var owner p: *Node` owns the value it
  points at, and `var borrower q: *Node` holds an address it doesn't own.
- **What an owning pointer points at is always on the heap,** because a node made while the
  program runs has to outlive the code that made it. So the word `heap` isn't written on the
  right-hand side. (Jack: doesn't want to write `heap` to put a node on the heap)

```go
const owner Node: type = {int, owner *Node};

var owner list: Node = { value: int = 1; next: *Node = null; };

list.next = &{ value: int = 2; next: *Node = null; };        // (sketch) a new node, owned by list.next
list.next.next = &{ value: int = 3; next: *Node = null; };

var borrower cur: *Node = list.next;                         // a name that borrows a node
```

The word on `list` only says where the first node lives: `var owner list` puts it on the
stack and `var heap list` puts it on the heap. Every later node is on the heap either way.

### 8.2 Moving ownership (open)

Building a list needs no move, because a new node never had another owner. Relinking does:
to remove the second node, the third has to move up, and it can only have one owner.

```go
list.next = take list.next.next;     // (sketch) the tail moves up, and its old place is left null
```

- Whether `hand` stays, in what form, is **not decided**. (Jack: "I don't know about doing
  hand again")
- One possible rule: a move takes what an owning pointer holds and leaves `null` behind.
  That would replace section 5's rules about the old name going empty.
- With no move at all the language still works, but setting one owning pointer from another
  has to copy everything it owns.

### 8.3 Two pointers to one value

**Two pointers can point at the same value when one owns it and the other borrows it. The
borrower just can't outlive the owner.** (Jack)

That makes a second link possible: a doubly linked list, a tree node that points back at its
parent, or a list that also keeps a pointer to its last node.

```go
const owner Node: type = {int, owner *Node, borrower *Node};     // value, next, prev

var owner a: Node = { value: int = 1; next: *Node = null; prev: *Node = null; };
a.next = &{ value: int = 2; next: *Node = null; prev: *Node = &a; };   // (sketch)
```

The second node is owned by `a.next`, and its `prev` borrows `a`. The rule holds here with
no checking: the second node is freed when `a` is, so its `prev` can never outlive `a`.

Two things this needs:

1. **The field has to say which it is, and the type has to hold it** (`owner *Node`,
   `borrower *Node` in the shape). If the word only lived in each value's block, two values of
   the same type could disagree about whether a field owns. (to check)
2. **A rule for the one case that scopes can't check** (8.4).

### 8.4 Removing a node that a stored borrower still points at (open)

For a borrower that is a name, the compiler compares scopes. A borrower stored inside a node
has no scope of its own, and one moment can go wrong:

```go
var owner list: *Node = ...;       // list owns a, a.next owns b, and b.prev borrows a

list = take list.next;             // b moves up and a is freed,
                                   // but b.prev still points at a
```

The compiler can't see that `b.prev` points at `a`, because that link was set somewhere else
while the program ran. Three ways to handle it, none chosen yet:

| Option | What happens | Cost |
|---|---|---|
| You clear it yourself | you write `b.prev = null;` before removing `a`, and the compiler trusts you | a mistake leaves a dangling pointer |
| Checked while running | each node counts its stored borrowers, and removing one that's still borrowed fails | bookkeeping the code doesn't show |
| Not allowed | a node with a borrower field can't be moved or removed, only freed with its whole structure | such structures can be built but not edited |

Borrowers never free anything, so none of the three can leak.

### 8.5 Functions

- A pointer going into a function is always a borrower, and a pointer coming out is always
  owned, so function types need no extra mark. (to check)

## 9. What it costs, and what is still open

- **Every line is one word longer,** and parameters feel it most:
  `(const owner x: int, const owner y: int)`.
- **A call doesn't show sharing.** `bump(count)` changes `count`, and neither the call nor
  the function's type `(int)void` says so. This is problem 14.1 in the README, and it stays.
- **What `owner` means on a pointer.** On a plain type an owner owns its own value. On a
  pointer it owns the value pointed at (8.1).
- **Pointer fields.** Other fields carry no words, but a pointer field has to say `owner` or
  `borrower`, and that has to be part of the type (8.3).
- **Moving ownership.** Whether `hand` stays, or a smaller move replaces it (8.2).
- **Removing a node that a stored borrower still points at** (8.4).
- **Copying a struct that owns nodes.** `var owner b: Node = list;` can't share the nodes,
  so it would have to copy all of them.
- **A `heap` parameter.** Whether a parameter can be `heap`, and what that would mean.
- **A handed value that nothing receives,** such as `makeBig(1);` on its own.
