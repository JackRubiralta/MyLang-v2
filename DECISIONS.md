# MyLang: the choices, and why

`DOCS.md` says what the language is. This file says why it is that way: every choice made
while designing it (6 to 11 November 2026), the reason behind each one, and the roads that
were tried and dropped.

Lines in quotes are Jack's own words from the design chat. A few rules were filled in by
Claude to make a choice complete and then accepted. They are marked **(filled in)**.

---

## 1. The philosophy

MyLang has three ideas behind it, and every choice below comes back to one of them.

**First-class.** Everything is a value: numbers, arrays, maps, structs, functions and even
types. Each one is declared, stored, passed and returned the same way, so there are no special
cases to learn. An array of functions or a function that returns a type uses the same rules as
an `int`.

**Explicit.** Reading the code tells you what happens. In Jack's words: "we understand what is
happening and it's very declarative, we know if something goes to heap". A value goes on the
heap only where `^` says so, it is shared only where `&` says so, and it changes owner only
where `@` says so. Nothing is copied, moved, shared or put on the heap behind your back.

Explicit doesn't mean marking everything, though. When every type had to carry a mark, even a
plain one (`#T`), Jack called it "too much": a case can go unmarked when it is the only unmarked
one. A plain `T` is owned right where it sits, and you know that because nothing says
otherwise.

**Declarative.** Every name is declared in one shape that says what kind of name it is, what it
is called, its type and its first value:

```go
kind name: Type = value;
```

There are no loops. Repeating is a function calling itself.

**Strict types** hold all three together. A value never changes type by itself: `5` is an `int`
and `^5` is a `^int`, so turning one into the other is always written out.

---

## 2. Where the work lives

| What | Where |
|---|---|
| The repository, on the laptop | `~/MyLang` |
| The repository, on GitHub | `JackRubiralta/MyLang-v2`, branch `main` |
| The rules of the language | `DOCS.md` |
| Why the rules are what they are | `DECISIONS.md` (this file) |
| Ideas explored along the way | `ideas/idea-1` to `ideas/idea-5` |

The 2024 version, with its Python lexer and parser, stays in the old `JackRubiralta/MyLang`
repository. `README.md` was the first write-up of the redesign. It was removed on 10 November
because it described the old way of doing things ("README was the old implementation"), and it
is still in the git history.

---

## 3. Names and declarations

| Choice | Why |
|---|---|
| One shape for every declaration: `kind name: Type = value;` | Functions, arrays and types are values, so there is no reason for them to be declared differently from an `int` |
| Only two kinds of name: `const` and `var` | Where a value lives and who owns it belongs to its type (`^T`, `&T`), not to the name. The old kinds `ref` and `heap` went once the type could say it |
| No `fn` keyword | It isn't needed, so the language stays short and clean |
| A declared name can carry `@` (`var @q: ^int = @p;`) | The `@` marks the name that takes a value over, so a move is visible where it happens |

---

## 4. Types

### 4.1 Every modifier goes in front

`[3]int`, `[]int`, `{int}bool`, `(int, int)int`, `^int`, `&int`, `@int`.

**Why.** Four layouts were compared: arrays after or before the type (`int[]` or `[]int`), and
functions return-first or parameters-first (`int(int)` or `(int)int`). Three independent checks
were run: a readability review, a real parser fed 3,000 random types, and a survey of other
languages. Only "everything in front" gave every type exactly one meaning with no brackets at
all. Of the 3,000 random types it needed brackets for 0, against 711, 1,216 and 648 for the
others. Go, Zig and Odin, the languages built since C that treat functions as values, also put
modifiers in front.

It also reads in the order you use the value. `f: (int)(bool)[3]int` is used as `f(1)(true)[0]`:
call, call, index.

### 4.2 Types are values

| Choice | Why |
|---|---|
| `type` is a type, so a type can be named, stored and passed | Everything is first-class |
| A type is named with `const` (`const Point: type = {int, int};`) and is never a `var` | A type never changes, so every value's type is known before the program runs |
| A type is never called: `int(3.0)` is an error | "A type is a value, not a function." Conversion is a function of its own: `convert(float, int)(3.0)` |
| `wrap` and `unwrap` for `any` | The same reason: `any(3)` would be calling a type |
| Generics are functions that take a type, `first(int)` | No new feature is needed, because a type is already a value |
| Real basic types: `int8` to `int64`, `uint8` to `uint64`, `float32`, `float64`, `bool`, `char`; `int`, `uint`, `float` as everyday names | Jack asked for real sized types instead of stand-ins |
| A function that returns nothing ends in `void` | Every function type ends with its return type, so `(` always starts a function type and never groups one |

### 4.3 Arrays and maps

| Choice | Why |
|---|---|
| A fixed array `[3]int` or an unsized `[]int` whose length comes from its first value and never changes | A value on the stack has a fixed size |
| A map has a fixed set of keys, set by its first value | The same reason: it can't grow |
| `[bool]T`, a choice array | One slot for `false` and one for `true` lets a value be chosen by a `bool` |

### 4.4 Structs

| Choice | Why |
|---|---|
| A struct type is `const Name: type = {T1, T2};` | No `struct` keyword: a struct type is just a type |
| A struct value is a plain block, `{ x: int = 1; y: int = 2; }` | It stays declarative. Making one by calling `Node(5)` was tried and dropped. A function such as `makeNode` builds the block instead |
| `Point` is a type, never a value of type `Point` | `var p: Point = Point;` would make one name mean two things |
| Fields carry no `const` or `var` of their own; they follow the struct's | One word on the declaration says it for the whole block |
| A struct variable holds its whole block, not a pointer to it | Copying it copies the block, so nothing can be left pointing at a removed block. Hidden fat pointers were rejected because they went against "explicit" |
| Functions inside a struct use its fields directly, with no `this` | The fields are in the struct's scope |

---

## 5. Ownership

### 5.1 Three cases, told apart by the type

| Type | Means |
|---|---|
| `T` | owned right here, in place. No mark, because it is the default |
| `^T` | on the heap, and owned |
| `&T` | borrowed: another holder owns it |
| `@T` | the ownership of a variable: a fourth mark, not a fourth case |

**Why.** Earlier versions had `ref` and `heap` as kinds of name, `*T` pointers, `new` and
`delete`, and the word `hand` to move a value. Each one fixed a single problem and caused
another: ownership could pass without showing, a pointer could be left dangling, or a function's
type couldn't say what it gave back. Putting ownership in the type fixed all of them at once. A
function's type now shows what it takes and gives (`(@^Node)^Node`), and a struct field shows
whether it owns or borrows.

`^` means "on the heap", not "owner". Everything owns its value unless it is marked `&`.

### 5.2 How the marks are written

What **makes** a marked value goes in front, like the mark in its type: `^5`, `&count`, `@p`.
What **uses** one goes behind, like `a[0]` and `f(1)`: `p^`.

**Why.** Every expression then looks like its type, and nothing needs brackets:
`list^.next^.value` reads left to right. Jack set this for `@` too ("keep the @"): `@x` gives an
`@T`, and you follow it with `^`, the same as the other two marks. An `@T` is never used as a `T`.

### 5.3 `=` always makes a complete copy

`=` means the same for every type: the left side gets its own copy, and the right side never
changes. A heap value is copied into a new place on the heap.

**Why.** "This is just enforced deep copies, which is fine." One meaning for `=` everywhere is
simpler than an assignment that sometimes copies and sometimes moves. To avoid a copy, you write
it: `&` to borrow, `@` to move.

### 5.4 Field by field

What is done to a struct is done to each of its fields: copy, move, heap, borrow and ownership.
The same goes for an array and its elements.

**Why.** "A struct works the same as a variable." There are no special whole-structure rules. A
rule that would have re-pointed the borrowers inside a copy was rejected. A type with back-links,
such as a list with `prev`, gets a copy function the programmer writes (`copyLinks`).

### 5.5 Moving is setting an ownership

`@q = @p;` or `var @q: ^int = @p;` moves the value, and `p` is left `null`. Writing `@p` on its
own moves nothing.

**Why.** A move changes who owns the value, so both sides are ownerships, written the same way.
The old owner is left `null` so a value never has two owners.

A parameter that asks for an ownership takes it at the call: "the function parameter in and of
itself, through the type, asks for the ownership". After `take(@big)`, `big` is always `null`,
whatever the function does with it.

### 5.6 Nothing is freed while something still points at it

While a pointer that was made by following an owner is still used further down, that owner
can't be changed: no new value, no `null`, no move, and no call that changes or takes it.

**Why.** This one compile-time rule closes three holes together: dangling borrowers, returning
a borrower, and moving a value into an owner that ends sooner. A check while the program runs
would also work, but every pointer and every heap value would carry extra numbers, and every
follow would do work the code doesn't show.

What follows from it:

- a value can be moved anywhere;
- a function can return a borrower made from a borrowed parameter;
- a function can change a variable from outside itself, and the compiler reads which.

### 5.7 Smaller ownership choices

| Choice | Why |
|---|---|
| Whether a `&T` parameter only reads or also changes stays out of the function's type | "Leave it." Function types stay short; the compiler reads the function when it is called by name |
| `@` on a value made on the spot hands it straight over: `take(@makeBig(1))` | There is no variable to leave `null` |
| A call works out every argument first, then sets the parameters, left to right | `both(@x, x^)` works: reading `x` and handing `x` over don't get in each other's way |
| A value made on the spot can be borrowed as an argument, `look(&makeBig(1)^)`, but not kept in a variable | It lives until the end of its statement, and the call is over by then |

---

## 6. Functions

| Choice | Why |
|---|---|
| A function is written `(params){ body }`, its type is `(int, int)int`, and parameters are declared like names | Everything is declared in the same shape |
| A function that returns a value has to reach a `return` on every path | Otherwise it could end with nothing to give back. The compiler counts both sides of every `if` as possible **(filled in)** |
| `return;` on its own stops a `void` function early | Jack: "that's allowed", as in most languages |
| A function's body is a block, so it sees the names around it and can change a `var` from outside | Jack: "just variables in a broader scope" |

### 6.1 Functions that use variables from around them

This took the longest to settle. Copying the variables in, the way many languages do closures,
was tried first and dropped, because a hidden copy would make `sum = sum + x` inside a helper
change a copy nobody can see.

| Choice | Why |
|---|---|
| A function borrows the variables it uses from around it | It works on the variable itself, so its changes reach it, and nothing is copied out of sight |
| A variable that a function uses can't die before the function | It is the borrower rule. Returning a function that uses a local is an error |
| A function given back keeps the parameters it uses | `makeAdder(step)` "truly should work": the parameters are what the caller handed in |
| It can't outlive the arguments of the call | "If the parameter that's put into makeAdder dies before what makeAdder returns, the compiler raises an error." It is the same rule as for a borrower that comes back |
| An argument handed over with `@` doesn't count | The call took the value and the caller's variable is `null`, so nothing is left to die **(filled in)** |
| A local is kept by declaring it after a `;` in the function's brackets: `(const i: int; var @mine: ^[3]int = @table){ … }` | It is an ordinary declaration that runs once, when the function is made. You can see exactly what the function keeps and how: a copy, a borrow, or a move. C++ (`[mine = std::move(table)]`) and Swift capture lists work the same way |
| Every function value is two addresses: its code, and what it keeps | Every function of a type has the same size, so functions go in arrays and maps |

A second way of keeping a value, writing `@name` inside the body, was tried and rejected
(`ideas/idea-5`). It broke two earlier rules: an `@T` isn't a `T`, and `@x^` is read as
`@(x^)`.

---

## 7. Choosing and repeating

| Choice | Why |
|---|---|
| No loops; repeating is a function that calls itself | The language is declarative |
| `if (cond) { } else if (cond) { } else { }` | "We can have if statements." Recursion needs a way to stop, and the form comes from Jack's own 2024 notes. The condition must be a `bool` |
| Choice arrays `[bool]T` and maps of functions stay too | A lookup chooses a value in the middle of a declaration, where a statement can't go, and a map of functions does the job of a `switch` |

These replaced an earlier decision on the same day to have no `if` at all.

---

## 8. Operators

| Choice | Why |
|---|---|
| `&&`, `\|\|` and `!`, on `bool`s only | Jack: "use symbols" |
| The right side of `&&` and `\|\|` runs only when it's needed | `if (p != null && p^ > 10)` is then safe **(filled in)** |
| The usual order of operations, and brackets that group | "Do whatever the conventions are for most programming languages." What is behind a value goes first, then what is in front, then `*`, then `+ -`, then comparisons, then `&&`, then `\|\|` |
| A type is never grouped: in a type, `(` always starts a function type | It keeps the rule from section 4.1. `(a)` is a function type when `a` is a type, and a group when it isn't |
| `&&` in front of a value or inside a type is two borrow marks | Where it stands tells the two apart: `&&int` is a borrower of a borrower |

---

## 9. Roads not taken

| Tried | Dropped because |
|---|---|
| `ref` and `heap` as kinds of name | Ownership belongs in the type, where function types can show it |
| `*T` pointers with `new` and `delete` | Jack never agreed to `new(value)`. A pointer that neither owns nor borrows can dangle |
| `hand` to move a value | Not wanted as a word ("I don't know about doing hand again"). `@` replaced it |
| `owner` and `borrower` written on every declaration (idea 1) | One word longer on every line, and the role couldn't be shown inside a type |
| A mark on every type, including plain ones (idea 2, explicit variation) | "Too much." The unmarked case is the default |
| Hidden fat pointers under arrays and structs | They went against "explicit" |
| Making a struct by calling it, `Node(5)` | A struct value is a plain block. `makeNode` builds it |
| Calling a type, `int(3.0)` and `type(3)` | A type is a value, not a function |
| No `if` at all | Recursion needs a way to stop, and an `if` reads better than a lookup |
| Copying outside variables into functions (closures) | Hidden copies: a change inside a helper would vanish |
| `@name` inside a function body to keep a value (idea 5) | It broke rules already set |

---

## 10. Still open

From `DOCS.md` 8.17:

- **A borrower kept inside a heap value,** like `prev`, whose target is removed. The compiler
  can't follow it.
- **A copy of a list with back-links.** It still points into the original until its links are
  set again.
- **A borrower of a `const` given to a function held in a variable.**
- **Two functions given back by one call that use the same `var` parameter.** Do they share it,
  or does each keep its own?
- **A function inside a struct, after the struct is copied.**

Later: scopes across files, and the older questions (what `T` refers to in `(type)([]T)T`,
telling a one-field struct `{int}` from a map, map keys, conversions that don't fit).
