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

## 8. Pointers under this idea (discussed, not decided)

These came up in the discussion and are **not** part of the idea yet:

- A pointer would say its role too: `var owner p: *Node` owns the heap value it points at,
  and `var borrower q: *Node` holds an address it doesn't own.
- An owner pointer is only ever given a `hand` or `null`. A borrower pointer is given `&x`,
  another pointer or `null`.
- A pointer going into a function is always a borrower, and a pointer coming out is always
  handed, so function types need no extra mark.

## 9. What it costs, and what is still open

- **Every line is one word longer,** and parameters feel it most:
  `(const owner x: int, const owner y: int)`.
- **A call doesn't show sharing.** `bump(count)` changes `count`, and neither the call nor
  the function's type `(int)void` says so. This is problem 14.1 in the README, and it stays.
- **What `owner` means on a pointer.** On a plain type an owner owns its own value. On a
  pointer it would own the value pointed at. Section 8 is where that gets decided.
- **Pointer fields.** Fields carry no words, so a struct can't yet say whether a pointer
  field owns or borrows what it points at.
- **A `heap` parameter.** Whether a parameter can be `heap`, and what that would mean.
- **A handed value that nothing receives,** such as `makeBig(1);` on its own.
