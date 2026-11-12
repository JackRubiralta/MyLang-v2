# MyLang: the choices, and why

`DOCS.md` says what the language is. This file says why it is that way: every choice made
while designing it (6 to 11 November 2026), and the reason behind each one.

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

Explicit still leaves the default unmarked. A plain `T` is owned right where it sits, and you
know that because nothing says otherwise.

**Declarative.** Every name is declared in one shape that says what kind of name it is, what it
is called, its type and its first value:

```go
kind name: Type = value;
```

Repeating is a function calling itself.

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

---

## 3. Names and declarations

| Choice | Why |
|---|---|
| One shape for every declaration: `kind name: Type = value;` | Functions, arrays and types are values, so they are declared the same way as an `int` |
| Two kinds of name: `const` and `var` | Where a value lives and who owns it belongs to its type (`^T`, `&T`), not to the name |
| No `fn` keyword | It isn't needed, so the language stays short and clean |
| A declared name can carry `@` (`var @q: ^int = @p;`) | The `@` marks the name that takes a value over, so a move is visible where it happens |

---

## 4. Types

### 4.1 Every modifier goes in front

`[3]int`, `[]int`, `{int}bool`, `(int, int)int`, `^int`, `&int`, `@int`.

**Why.** With every modifier in front, each type has exactly one meaning and never needs
brackets. A parser was fed 3,000 random types in this form, and none of them needed a bracket
to be read correctly. Go, Zig and Odin, which also treat functions as values, write their
modifiers in front too.

It also reads in the order you use the value. `f: (int)(bool)[3]int` is used as `f(1)(true)[0]`:
call, call, index.

### 4.2 Types are values

| Choice | Why |
|---|---|
| `type` is a type, so a type can be named, stored and passed | Everything is first-class |
| A type is named with `const` (`const Point: type = {int, int};`) | A type never changes, so every value's type is known before the program runs |
| A type is never called; converting is a function of its own: `convert(float, int)(3.0)` | "A type is a value, not a function" |
| `wrap` and `unwrap` put a value in an `any` and take it out | The same reason: a type is never called |
| Generics are functions that take a type, `first(int)` | No new feature is needed, because a type is already a value |
| Real basic types: `int8` to `int64`, `uint8` to `uint64`, `float32`, `float64`, `bool`, `char`; `int`, `uint`, `float` as everyday names | Jack asked for real sized types |
| A function that returns nothing ends in `void` | Every function type ends with its return type, so in a type a `(` always starts a function type |

### 4.3 Arrays and maps

| Choice | Why |
|---|---|
| A fixed array `[3]int` or an unsized `[]int` whose length comes from its first value and never changes | A value on the stack has a fixed size |
| A map has a fixed set of keys, set by its first value | The same reason: it can't grow |
| `[bool]T`, a choice array | One slot for `false` and one for `true` lets a value be chosen by a `bool` |

### 4.4 Structs

| Choice | Why |
|---|---|
| A struct type is `const Name: type = {T1, T2};` | A struct type is just a type, named like any other |
| A struct value is a plain block, `{ x: int = 1; y: int = 2; }`, and a function such as `makeNode` builds one | It stays declarative: the block says what is in it |
| `Point` is a type, never a value of type `Point` | One name means one thing |
| Fields carry no `const` or `var` of their own; they follow the struct's | One word on the declaration says it for the whole block |
| A struct variable holds its whole block | Copying it copies the block, so nothing can be left pointing at a removed block |
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

**Why.** With ownership in the type, a function's type shows what it takes and gives back
(`(@^Node)^Node`), and a struct field shows whether it owns or borrows.

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
simple. To avoid a copy, you write it: `&` to borrow, `@` to move.

### 5.4 Field by field

What is done to a struct is done to each of its fields: copy, move, heap, borrow and ownership.
The same goes for an array and its elements.

**Why.** "A struct works the same as a variable." A type with back-links, such as a list with
`prev`, gets a copy function the programmer writes (`copyLinks`).

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

**Why.** One rule, checked by the compiler, covers three cases together: a borrower left
pointing at freed memory, returning a borrower, and moving a value into an owner that ends
sooner. Nothing extra runs while the program runs.

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

| Choice | Why |
|---|---|
| A function borrows the variables it uses from around it | It works on the variable itself, so its changes reach it, and nothing is copied out of sight |
| A variable that a function uses can't die before the function | It is the borrower rule. Returning a function that uses a local is an error |
| A function given back keeps the parameters it uses | `makeAdder(step)` "truly should work": the parameters are what the caller handed in |
| It can't outlive the arguments of the call | "If the parameter that's put into makeAdder dies before what makeAdder returns, the compiler raises an error." It is the same rule as for a borrower that comes back |
| An argument handed over with `@` doesn't count | The call took the value and the caller's variable is `null`, so nothing is left to die **(filled in)** |
| A local is kept by putting it on the heap and handing it with `@` to a parameter of a function that gives back the function using it | It needs nothing new: `^` puts the value on the heap (8.3), and a parameter that asks for an ownership takes it (8.13). A function given back keeps its parameters |
| Every function value is two addresses: its code, and what it keeps | Every function of a type has the same size, so functions go in arrays and maps |

---

## 7. Choosing and repeating

| Choice | Why |
|---|---|
| Repeating is a function that calls itself | The language is declarative |
| `if (cond) { } else if (cond) { } else { }` | "We can have if statements." Recursion needs a way to stop, and the form comes from Jack's own 2024 notes. The condition must be a `bool` |
| Choice arrays `[bool]T` and maps of functions | A lookup chooses a value in the middle of a declaration, where a statement can't go, and a map of functions does the job of a `switch` |

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

## 9. Files and packages

| Choice | Why |
|---|---|
| A package is a file | Jack: "I think a package should be a file." One file is one scope, so every name comes from this file or from a package it imported, and you can see which |
| `const geom: package = import("geom");` | The import is an ordinary declaration in the one shape every name uses. `import` is a built-in like `convert`, and `package` is a built-in type like `type` and `any`. Zig works the same way |
| Everything at a file's top level can be used from outside, for now | The simplest start |
| Statements that run go only in the file you run; an imported file holds only declarations | Nothing runs because a file was imported, so the run order never depends on imports |
| A package can be a `const` or a `var` | Jack: underneath it is like a function pointer. A package value is the address of the file's names, so it is held like a function, and a `var` one can be given another file |
| A `var` package can only be given a file with the same names and types the code uses | The same as a function in a `var` keeping its type, so the compiler still checks every name **(filled in)** |
| Importing the same file twice gives the same package; two files can't import each other | A file's variables exist once, and every file's names are ready before a file that imports it **(filled in)** |
| A function's body can use any top-level name of its own file, wherever it is declared | Two functions in one file can call each other **(filled in)** |
