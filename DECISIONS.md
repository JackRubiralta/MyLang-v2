# MyLang: the choices, and why

`DOCS.md` says what the language is. This file says why it is that way: every choice made
while designing it (6 to 11 November 2026), and the reason behind each one.

Lines in quotes are Jack's own words from the design chat. A few rules were filled in by
Claude to make a choice complete and then accepted. They are marked **(filled in)**.

On 12 November 2026 Claude filled in every rule that was still missing, so that a compiler
can be written from `DOCS.md` alone. Those rows are marked **(filled in)** too, and each one
names the principle it follows (*Principle: …*). Jack has not reviewed them yet. A rule of
Jack's that had to change is marked **(adjusted)**, and section 13 gives its old wording.
`FILLED_IN.md` lists all of them.

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
| A name can't be declared where the same name can already be seen. That rules out a second declaration and all shadowing, for names, parameters, struct fields and the built-in names | A name then means one thing wherever it is read: "one name means one thing" (4.4). The examples in `DOCS.md` already work this way: the inner functions use `o`, `k` and `back` beside `owner`, `n` and `before`. One rule settles three gaps: declaring twice, shadowing, and why a built-in name can't be declared again. Weighed: shadowing in inner blocks, as most languages allow; no shadowing. The philosophy decides here, so the convention doesn't. *Principles: explicit; one rule, no special cases.* **(filled in)** |
| A name is used only below its declaration. A function's body can use any top-level name of its file and its own name, and a type can use any top-level type of its file | Nothing may run before its names are set up. A body runs only when it is called and a type never runs, so only those two are free of the order. *Principle: the compiler checks what it can.* **(filled in)** |
| Five kinds of statement: a declaration, an assignment, a value on its own, a `return`, a block | They are the forms the examples already use. Any value can stand on its own, so there is no list of values that may and may not. *Principle: one rule, no special cases.* **(filled in)** |
| An assignment is a statement, not a value | Every change then starts its own line, where it is easy to see. *Principle: explicit.* **(filled in)** |
| The left side of `=` starts at a `var` name, or at a value with no name that is followed with `^` | It is the `const` rule said once: what a `const` name reaches can't be changed through that name. A struct with no name can't be assigned, because nothing could ever read the change. *Principles: one rule; the compiler checks what it can.* **(filled in)** |
| An assignment works out its right side first, then finds its place | A call already works out every argument before it sets a parameter (5.7). "Values first, places after" is one order for both, and the right side can't make the place stale. Weighed: the left side first, as in Java and Go; the right side first, as in Rust and Python. *Principle: reuse before adding.* **(filled in)** |
| Everything else is worked out from left to right | It is the order the code is read in, and what most languages do. *Principles: explicit; do what most languages do.* **(filled in)** |
| A name is looked for in its block, then the blocks around it, then the file's top level, then the built-in names, and never in another file | "Every name comes from this file or from a package it imported, and you can see which" (section 9). *Principle: explicit.* **(filled in)** |

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
| Two types are the same when they have the same shape. Names of parameters and fields, and `const` and `var`, are not part of a type | "A name for a type is the same type under another name" (DOCS 4.1), and a type lists only types (4.4). *Principle: one rule.* **(filled in)** |
| Only a value written on the spot takes its type from its place: a number, `null`, an array or text, a map value, a function value. With no place a number is an `int` or a `float` | That is how `5` and `[3, 10]` already work (DOCS 3.2, 5.8). Every other value has its type already and never changes it. The everyday names are the defaults. *Principle: strict types.* **(filled in)** |
| The type of every form of value is gathered in one table (DOCS 3.8) | The rules were spread over the sections; a compiler needs them side by side. *Principle: strict types.* **(filled in)** |
| A basic type is a name, or a name reached through a package: `geom.Box` can stand wherever a type is written | A name from another file is always reached through its package (section 9), and a type is a name like any other. *Principle: one rule, no special cases.* **(filled in)** |
| The built-ins that take a type have types of their own: `convert` is a `(type, type)(A)B`, `wrap` a `(type)(T)any`, `unwrap` a `(type)(any)T`, `join` a `(type)([]T, []T)[]T` | A built-in is an ordinary function value, so it has a type that can be written. *Principle: everything is a first-class value.* **(filled in)** |
| A function type names the types it is given with names that aren't declared: the first such name is its first `type` parameter, and so on. The name itself is not part of the type | Jack's own example writes `(type)([]T)T`, so the type has to stand without the value beside it: it can then be a parameter's type or a named type. Weighed: taking the names from the function value, which leaves a bare type with nothing to name; naming the parameter inside the type, which the rule "a type lists only types" (4.4) excludes; this rule. *Principles: reuse before adding; every value follows the same rules.* **(filled in)** |
| A generic function is checked once, and a `T` can be used only in the ways every type allows | The function's type then says everything about it, so a generic function can be held in a variable and still be checked before the program runs. To add two `T`s, the caller passes the function that adds them. Weighed: checking again for each type handed in, which only works when the compiler can see which function is called. *Principles: every value follows the same rules; the compiler checks what it can.* **(filled in)** |
| A type that decides the type of something is written out. A type worked out another way (`x.T`, `wholeTypes[1]`) can be compared, stored and passed | "Every value's type is known before the program runs" (this section). A type read out of an `any` is known only while running, so it can't declare anything. *Principle: strict types.* **(filled in)** |
| A type lasts as long as the program | A type never changes (this section), so it is never an argument that ends too soon for a function given back. *Principle: one rule.* **(filled in)** |
| Underneath, a `type` value is the address of a description: the size, and how to copy, free and compare | An `any` and a generic function both need exactly that, so one thing serves both. *Principle: say what happens underneath.* **(filled in)** |
| `convert` also covers `bool` and whole numbers, `char` and every whole-number type, the two float types, and numbers or `bool`s to text and back | A `bool` and a `char` are numbers underneath (DOCS 3.1), and the docs say a `bool` is used as a number only through a conversion. Text conversion reuses `convert` instead of adding functions: both types are written, and a text that doesn't fit fails like any conversion that doesn't fit. *Principles: reuse before adding; explicit.* **(filled in)** |
| An `any` keeps the type of its first value | Its size comes from the value inside, and a value on the stack keeps its size, like an unsized array's length (4.3). Weighed: letting a `var` `any` change type, which needs room nobody can know before the program runs. *Principle: one rule, no special cases.* **(filled in)** |
| A top-level type can use any top-level type of its file, itself included, but it holds itself only through `^`, `&`, `@` or a function type | `Node` already names itself (DOCS 8.14). A type never runs, so the order can't matter, and a pointer has one size whatever it leads to. *Principle: the compiler checks what it can.* **(filled in)** |

### 4.3 Arrays and maps

| Choice | Why |
|---|---|
| A fixed array `[3]int` or an unsized `[]int` whose length comes from its first value and never changes | A value on the stack has a fixed size |
| A map has a fixed set of keys, set by its first value | The same reason: it can't grow |
| A pair `?T`: one value for `false` and one for `true`, looked up with a `bool`, written like an array of two: `const sizes: ?int = [3, 10];` | Jack: "we could have a type called pair". A pair always has both values, so a lookup never fails. `?` is used nowhere else in MyLang, and the values in `[ ]` can't be mistaken for comparisons |
| Slices: `&a[i:j]` borrows part of an array as an `&[]T`; `a[i:j]` without `&` copies those elements | Jack: "add slices … I think they make sense". It reuses what exists: the borrow mark `&`, the type `&[]T` that already borrows a whole unsized array, and every borrowing rule in section 8. The `i:j` form, start included and end left out, is what most languages use. Underneath a slice is two numbers, so nothing is copied **(filled in: the form and the rules)** |
| Underneath, a pair is an array of two, the way a `bool` is a number (`false` is 0, `true` is 1) | Jack: "this is just like an array under the hood, the same way a bool is basically just an int under the hood". An array's size is always a number, so no type goes between its brackets |
| An index and a slice bound are `int`s, and an index out of range fails while running unless the compiler can already see it | The docs check a slice this way (DOCS 5.9), and an index is the same question. A length is an `int` too, so `i == values^.length` needs no conversion. *Principles: one rule; the compiler checks what it can.* **(filled in)** |
| An array's size is a written whole number, 0 or more, and `[]` is the empty array value | "An array's size is always a number" (the row above). An empty text has to exist, so an empty array does. *Principle: one rule.* **(filled in)** |
| The entries of a map are held in the order written, and a lookup compares its key with `==` | It says what the picture "a map is like a struct" (DOCS 6.4) means for a lookup. *Principle: say what happens underneath.* **(filled in)** |
| A size that isn't in the type comes from the first value and stays: for every holder of a `[]T`, a map or an `any`. When the compiler can't see the sizes, a new value of another size fails while running | It is the docs' rule for `var a: []int` (DOCS 5.3), said once for every type without a size and every holder: a name, a field, an element. *Principle: one rule, no special cases.* **(filled in)** |
| A fixed array becomes an unsized one through a slice: `primes[:]`, `&primes[:]` | The two are different types, and a value never changes type by itself. The slice form already exists. *Principles: strict types; reuse before adding.* **(filled in)** |
| A pair value has exactly two values, and a pair has no `.length` | A pair always has both values (this section), so there is nothing to ask. *Principle: strict types.* **(filled in)** |
| Text is written `"Jack"`, and it is the array value `['J', 'a', 'c', 'k']`. There is no text type | The docs say text is an array of `char`s (DOCS 5.1). A shorter way to write the same value adds no type and no rule: length, indexing, slices, `==` and copying all come from arrays. *Principles: reuse before adding; every value follows the same rules.* **(filled in)** |
| `join(T)` joins two unsized arrays into a new one | An array never grows, so making a longer one needs one built-in. It takes a type first, like `wrap`, and works for every element type, text included. A list that grows is a heap owner given a new, longer array: the `^` shows the heap use. *Principles: a type is never called, things that act are functions; explicit.* **(filled in)** |
| A map key can have any type. The keys of a map value all differ, a lookup is checked, and a whole new value needs the same keys | A lookup needs only `==`, and every type has it (section 8). The checks are the array rules again: the compiler makes them when it can see the keys, and otherwise they are made while running. *Principle: one rule, no special cases.* **(filled in)** |

### 4.4 Structs

| Choice | Why |
|---|---|
| A struct type is `const Name: type = {T1, T2};` | A struct type is just a type, named like any other |
| A struct value is a plain block, `{ x: int = 1; y: int = 2; }`, and a function such as `makeNode` builds one | It stays declarative: the block says what is in it |
| `Point` is a type, never a value of type `Point` | One name means one thing |
| Fields carry no `const` or `var` of their own; they follow the struct's | One word on the declaration says it for the whole block |
| A struct variable holds its whole block | Copying it copies the block, so nothing can be left pointing at a removed block |
| Functions inside a struct use its fields directly, with no `this` | The fields are in the struct's scope |
| A function inside a struct or a file uses the copy it is called through | After `var other = counter;`, `other.add(1)` changes `other`, not `counter`: a copy is complete, functions included |
| A struct value is checked against its type: one field for each type, in order, with that type written. A field's value can use the fields above it | The block is a block of declarations, so the rule for names in any block applies. *Principle: one rule.* **(filled in)** |
| A field name is a label for a position in a struct type. The compiler gathers the labels from every struct value of the type in the program | The type lists only types and the value gives the names (this section), so the names can't be part of the type: a function that takes a `Point` takes any `{int, int}`. A struct on the heap or in a parameter has no block in view, so its names can only come from the type's values. The docs already picture an array as a struct "reached by position", with names as labels (DOCS 5.7). Weighed: names written in the type, which turns Jack's rule round; following each struct from its block, as for a package, which can't see through the heap; every value of a type using the same names, which stops two unrelated singletons having the same shape; this rule. *Principles: one rule, no special cases; say what happens underneath.* **(filled in)** |
| A function can't change the fields of a `const` struct. Where the compiler can't see which function a field holds, calling it through a `const` struct is an error | A `const` never changes (DOCS 13). The worst case for a function the compiler can't see is the docs' own rule for calls (5.6). *Principle: the compiler checks what it can.* **(filled in)** |
| A function read out of a struct uses the struct it was read from. Underneath it keeps the struct's address, and a copy of the struct re-aims it at the copy | It is the borrower rule for a function that uses names around it (6.1), and it is how "a copy is complete, functions included" comes out in memory. *Principle: say what happens underneath.* **(filled in)** |

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
| A struct, array, pair, map or `any` that holds a borrower, an ownership or a function can't outlive what that part points at | Field by field (5.4): the lifetime rules for a whole value are the rules for each part. *Principle: one rule, no special cases.* **(filled in)** |
| A result whose type can hold a pointer or a function counts like one at the call | The caller sees only the type, as with a borrower that comes back (5.6). *Principle: one rule.* **(filled in)** |
| A `const` is lent only where the compiler sees it is only read. Lending one to a function called through a variable is an error | It closes a gap the docs listed as not caught, with the rule 5.6 already has: through a variable every `&` argument counts as one that may change. A `const` then never changes. *Principle: the compiler checks what it can.* **(adjusted)**, see section 13 |
| A `const` ownership can only read | One rule for every `const`: what a `const` name reaches can't be changed through that name. It is the borrower table (DOCS 8.6) applied to the third mark. *Principle: one rule, no special cases.* **(filled in)** |
| `null` fits a `^T`, an `&T` and an `@T`, and following any of them when `null` fails | `null` is "no address" and all three hold an address (DOCS 8.18). *Principle: every value follows the same rules.* **(filled in)** |
| `@` works on a map entry | A map entry is a holder like an array element, and it can already be borrowed (DOCS 8.4). *Principle: no special cases.* **(filled in)** |
| `=` copies a function's two addresses, a `type` as itself, an `any` with its inside, and a package with its variables | It completes the table of what `=` copies (5.3), which `==` builds on (section 8). *Principle: every value follows the same rules.* **(filled in)** |
| A kept heap value is freed with the function that owns it, or at the end of the block its call was written in | It follows from where kept parameters live (6.1). *Principle: say what happens underneath.* **(filled in)** |
| Following a borrower kept in a heap value, after its target is gone, is not checked while running. What happens isn't defined | The docs leave this to the programmer (DOCS 8.17). A check would need something to run that the code doesn't show. It is the only mistake neither the compiler nor a check catches. *Principle: explicit.* **(filled in)** |

---

## 6. Functions

| Choice | Why |
|---|---|
| A function is written `(params){ body }`, its type is `(int, int)int`, and parameters are declared like names | Everything is declared in the same shape |
| A function that returns a value has to reach a `return` on every path | Otherwise it could end with nothing to give back |
| `return;` on its own stops a `void` function early | Jack: "that's allowed", as in most languages |
| A function's body is a block, so it sees the names around it and can change a `var` from outside | Jack: "just variables in a broader scope" |
| "Every path reaches a `return`" means the body always returns: its last statement is a `return`, or a block that always returns. Nothing is written after a `return` | No statement branches, so a body is one straight line. Code after a `return` could never run, so the compiler says so. *Principle: the compiler checks what it can.* **(filled in)** |
| A function value is checked against its type, and takes its return type from its place | The value lists parameters and the type lists the return type (DOCS 7.1), so the place supplies the one thing the value doesn't write. *Principle: strict types.* **(filled in)** |
| A call gives one argument of the right type for each parameter. A call to a function that returns nothing has no value | The usual rule, and `void` means there is nothing to use. *Principles: strict types; do what most languages do.* **(filled in)** |

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
| One call has one set of parameters: every function that a call gives back uses the same ones | This settles the point the docs left open (DOCS 8.17). A function borrows the variables it uses (this section), and a kept parameter is such a variable, so two functions that use it share it. Weighed: each function keeping its own copy, which would make a counter and its reader disagree. *Principles: one rule, no special cases; reuse before adding.* **(filled in)** |
| Copying a function copies its two addresses, so the copy uses the same variables and kept parameters. A heap value the function owns is copied | "Copying a function value copies the address" (DOCS 7.7). What it keeps on the stack is a borrow, and a copy of a borrower borrows the same thing. An owner's value is copied, as always. *Principle: every value follows the same rules.* **(filled in)** |
| The second address of a function is the place of what it keeps: a block, a struct value, a package value, a call's parameters or one heap value. Each place leads on to the place around it | It shows why one address is always enough (the row above). *Principle: say what happens underneath.* **(filled in)** |
| A function that a built-in gives back keeps nothing | `convert(float, int)` holds on to nothing of its call, so the lifetime rules never hold it back. *Principle: say what happens underneath.* **(filled in)** |
| Kept parameters stay on the stack, in the block the call was written in. So a function given back can't outlive that block. The exception is a function that keeps only a heap value it took with `@`: it owns that value and can go anywhere | A function value is two addresses (this section), and nothing goes on the heap unless the code says `^` (section 1). Together they leave only the stack of the caller for kept parameters, and only one heap address that a function can carry. Jack's `newCounter` works because of the exception. Weighed: a hidden heap block for kept parameters, which is heap use the code doesn't show; function values of different sizes, which undoes the row above; this rule. *Principles: explicit; say what happens underneath.* **(adjusted)**, see section 13 |

---

## 7. Choosing and repeating

| Choice | Why |
|---|---|
| Repeating is a function that calls itself | The language is declarative |
| Choosing is a lookup: pairs `?T` and maps of functions | A `bool` picks a slot, and a map of functions does the job of a `switch`. Recursion uses a lookup to stop |
| A program that calls deeper than the stack allows fails like any other check | The stack is fixed by the system, so it is one more check at something the code shows: a call. *Principle: one rule.* **(filled in)** |
| The last call takes over the stack of the function that makes it, when nothing in the call needs that function's names | Repeating is a function calling itself (this section), so repeating without end has to be possible: a program that reads lines until the input ends can't have a limit. The condition is the borrower rule: names that something still points at can't end. Weighed: no promise, which leaves every repeat with a limit nobody can see in the code; this rule, which Scheme also has for the same reason. *Principles: declarative; say what happens underneath.* **(filled in)** |

---

## 8. Operators

| Choice | Why |
|---|---|
| `&&`, `\|\|` and `!`, on `bool`s only | Jack: "use symbols" |
| The right side of `&&` and `\|\|` runs only when it's needed | `p != null && p^ > 10` is then safe **(filled in)** |
| The usual order of operations, and brackets that group | "Do whatever the conventions are for most programming languages." What is behind a value goes first, then what is in front, then `*`, then `+ -`, then comparisons, then `&&`, then `\|\|` |
| A type is never grouped: in a type, `(` always starts a function type | It keeps the rule from section 4.1. `(a)` is a function type when `a` is a type, and a group when it isn't |
| `&&` in front of a value or inside a type is two borrow marks | Where it stands tells the two apart: `&&int` is a borrower of a borrower |
| `/` divides and `%` gives the remainder, in the same step as `*` | "Use symbols." The same step as `*`, from left to right, is the convention. *Principles: symbols for operators; do what most languages do.* **(filled in)** |
| Whole-number division drops the part after the decimal point, and `%` has the sign of the left side | `convert(float, int)` already drops that part (DOCS 4.4), so dividing and converting agree, and `a` is always `(a / b) * b + a % b`. Most languages do the same. *Principle: reuse before adding.* **(filled in)** |
| `%` works on whole numbers only | The smaller choice: a remainder of floats is rarely needed. *Principle: keep the language small.* **(filled in)** |
| `-` in front of a signed whole number or a float | The docs already write `-7`. A minus that worked only on written numbers would be a special case. Weighed: only written numbers, with `0 - x` for the rest; a minus for every number. *Principle: one rule, no special cases.* **(filled in)** |
| A result that doesn't fit its type fails while running. Nothing wraps around | A conversion already fails when its value doesn't fit (DOCS 4.4). The same rule for `+ - * / %` means a value always fits its type, and it covers overflow and division by zero together. Weighed: wrapping, as in C, Go and Java; failing, as in Swift and Zig; leaving it open. A wrapped number is a wrong value the code doesn't show. *Principles: explicit; one rule.* **(filled in)** |
| No infinity and no "not a number": a float answer that isn't an ordinary number fails | It is the same rule again. It also keeps `==` simple: a float always equals itself, so a copy equals its original for every type. Weighed: the special IEEE values, as in most languages; failing. Here the philosophy decides. *Principles: one rule; explicit.* **(filled in)** |
| The compiler makes these checks itself when every number is written in the code | The docs do the same for a slice whose bounds are written numbers (DOCS 5.9). *Principle: the compiler checks what it can.* **(filled in)** |
| `<` `>` `<=` `>=` work on numbers and `char`s only | A `char` is a number underneath. Ordering text or structs can be written as a function, so nothing more is needed. *Principle: keep the language small.* **(filled in)** |
| `==` compares exactly what `=` copies, for every type | One rule for every type, and a copy always equals its original. A `^T` compares the values, because `=` copies the value. An `&T` compares the addresses, because `=` copies the address. Structs and arrays go part by part (5.4). Weighed: `==` only for basic types; heap owners compared by address; this rule. *Principles: one rule, no special cases; every value follows the same rules.* **(filled in)** |

---

## 9. Files and packages

| Choice | Why |
|---|---|
| A package is a file | Jack: "I think a package should be a file." One file is one scope, so every name comes from this file or from a package it imported, and you can see which |
| `const geom: package = import("geom");` | The import is an ordinary declaration in the one shape every name uses. `import` is a built-in like `convert`, and `package` is a built-in type like `type` and `any`. Zig works the same way |
| Everything at a file's top level can be used from outside, for now | The simplest start |
| Statements that run go only in the file you run; an imported file holds only declarations | Nothing runs because a file was imported, so the run order never depends on imports |
| Good practice, not required: a `const main: ()int` in the file you run, and code can still go outside it | Jack: "it's like good practice to have a const main function, which returns an int. But you can have code outside of that." When it is there, the program runs the file's statements from the top, then calls `main`, and its `int` is the program's result **(filled in)** |
| A package can be a `const` or a `var` | Jack: underneath it is like a function pointer. A package value is the address of the file's names, so it is held like a function, and a `var` one can be given another file |
| Ownership and borrowing apply to a package like any other variable: copy, borrow, heap, move | Jack: "ownership rules and stuff like that, borrowing should all apply to a package, just like any other variable". A package is used like a struct whose fields are the file's variables, and borrowing is how two parts of a program share one copy |
| A function that takes a package uses only names every package given to it has | The compiler checks each call, so no name is looked up while the program runs **(filled in)** |
| A `var` package can only be given a file with the same names and types the code uses | The same as a function in a `var` keeping its type, so the compiler still checks every name **(filled in)** |
| Each import makes a new copy of the file's variables | Jack: importing a file twice should make two copies. Every declaration makes a new value (2.4), and sharing happens only where `&` says so: to share one copy, borrow it |
| Two files can't import each other | Each import would make a new copy, forever **(filled in)** |
| A function's body can use any top-level name of its own file, wherever it is declared | Two functions in one file can call each other **(filled in)** |
| `import` takes one text written out in the code, and is always called by its own name. The text is the file's path without `.my` | The compiler has to know every file before the program runs, or it couldn't check a single name. This is the one built-in that can't be passed around. *Principle: the compiler checks what it can.* **(filled in)** |
| An import runs where it is written, and all it runs is the file's top-level declarations, from the top down | An import is a call that makes a new value (this section). The run order is still what the code shows. *Principle: explicit.* **(filled in)** |
| A file's top-level names keep their own `const` and `var`. The word on the package says only whether the whole package can be replaced or moved | Jack's examples change `geom.made` and `a.count` through a `const` package, so the word on the package can't freeze them. Fields follow their struct because they have no word of their own; a file's names each have one. *Principle: explicit.* **(filled in)** |
| The compiler follows every package from its `import`, so it knows which files a place can hold. A name has to exist with one type in all of them. Through a function it can't see, the worst case counts | This is how the two filled-in rows above about a `var` package and a function that takes a package are checked. The worst case is the rule from 5.6. *Principle: the compiler checks what it can.* **(filled in)** |
| A file's names are set up from the top down. Function values written out and types are ready from the start. A top-level call can't reach any other name that is declared below it | A body can use any top-level name (the row above), so without this a call could read a name before it has a value. Functions and types need nothing to run, so two functions can still call each other. *Principle: the compiler checks what it can.* **(filled in)** |
| A top-level `main` in the file you run has to be a `const` of type `()int`. The status is from 0 to 255, converted like `convert(int, uint8)`. With no `main` the status is 0 | It completes the `main` row above. The status reuses the rule for a conversion that has to fit. *Principle: reuse before adding.* **(filled in)** |
| Underneath, a package value is the address of its file's table of names, then its variables. A `var` package has room for the biggest file it is given | It is what makes `shapes.area` work for any file with nothing looked up while running. *Principle: say what happens underneath.* **(filled in)** |
| The `const` rules hold for a package: a `var` borrower and an ownership need a `var` package | "Just like any other variable" (the row above on ownership). *Principle: one rule, no special cases.* **(filled in)** |

---

## 10. Writing the source

All of this section is **(filled in)**.

| Choice | Why |
|---|---|
| A source file is UTF-8, and only ASCII is used outside comments and text | A `char` is 8 bits, one of the sized types Jack asked for (4.2), so names and symbols stay at one byte for one character. Comments and text can hold anything. *Principle: say what happens underneath.* **(filled in)** |
| A name is letters, digits and `_`, and case matters | What most languages do. *Principle: do what most languages do.* **(filled in)** |
| Seven reserved words: `const var return void true false null`. The built-in types and functions are ordinary names, declared before every file starts | `int` is already "an ordinary name" for `int64` (DOCS 3.1), and a built-in is an ordinary function value (DOCS 4.4). Because a name is declared only once (section 3), they can't be declared again, and no second list of reserved words is needed. *Principles: everything is a first-class value; reuse before adding.* **(filled in)** |
| Only `//` comments | One form does the job. *Principle: reuse before adding.* **(filled in)** |
| A line break never ends a statement | Every statement already ends with `;` (DOCS 2.5), so nothing else has to. *Principle: explicit.* **(filled in)** |
| Whole numbers in base ten, base sixteen (`0x`) and base two (`0b`) | Bit patterns need the other two bases: the docs name a type `bitboard` (DOCS 4.1). They are written the way most languages write them. Base eight and separators inside a number were left out. *Principles: do what most languages do; keep the language small.* **(filled in)** |
| A float is `digits.digits`, with an optional `e` and a power of ten | The docs already ask for `3.0` instead of `3` (DOCS 3.2), so a digit stands on both sides of the point. The power of ten writes very big and very small numbers. *Principle: do what most languages do.* **(filled in)** |
| The escapes `\n \t \r \0 \\ \' \" \xHH` | The usual ones. `\xHH` reaches every value a `char` can hold. *Principle: do what most languages do.* **(filled in)** |
| A character outside ASCII doesn't fit in a `char`; in a text it is several `char`s | A `char` is one byte and UTF-8 needs more. Nothing changes size out of sight. *Principles: strict types; say what happens underneath.* **(filled in)** |
| A source file's name ends in `.my` | `import("geom")` has to lead to exactly one file. *Principle: explicit.* **(filled in)** |
| These are all the symbols, and the longest symbol is the one that is read | The convention, and how `&&` is already read (section 8). *Principle: do what most languages do.* **(filled in)** |
| Each example in `DOCS.md` is its own piece of code, and can declare a name again | The examples reuse names such as `count` and `p`. In one file that would break the rule that a name is declared once (section 3). *Principle: the compiler checks what it can.* **(filled in)** |

---

## 11. While the program runs

All of this section is **(filled in)**.

| Choice | Why |
|---|---|
| A failed check stops the program at once, writes `error: FILE:LINE: REASON` to the error output, and ends with status 1 | One behaviour for every check. `0` is "everything went well" (section 9), so a failure is not 0, and 1 is the usual status for it. The line names the place, because every check sits at something written in the code. *Principles: one rule, no special cases; do what most languages do.* **(filled in)** |
| A failure can't be caught. A program that isn't sure tests first | Catching would be a new kind of statement. The tests already exist (`p != null`, `i < a.length`, `x.T == int`), and they are in the code where they happen. *Principles: reuse before adding; explicit.* **(filled in)** |
| One list of checks, in DOCS Appendix B: a value that doesn't fit its type, a size that doesn't match, a lookup that misses, a `null` that is followed, and no room left | Each comes from one of two rules: a value always fits its type, and a size that isn't in the type stays what the first value made it. *Principle: one rule.* **(filled in)** |

---

## 12. Built-in functions for every program

All of this section is **(filled in)**.

| Choice | Why |
|---|---|
| `print: ([]char)void` writes exactly the `char`s it is given | Nothing is added out of sight, not even a line break. A number is turned into text first, with `convert`, so the code shows it. *Principle: explicit.* **(filled in)** |
| `read: ()[]char` gives the next line with its `\n`, and an empty text at the end of the input | Keeping the `\n` means an empty result can only be the end, so no second function and no special value is needed. *Principle: keep the language small.* **(filled in)** |
| `exit: (int)void` ends the program with a status | The same status rule as for `main` (section 9). *Principle: reuse before adding.* **(filled in)** |
| Each of them is an ordinary function value | Like `convert` (4.2): things that act are functions. *Principle: a type is never called; built-ins are first-class.* **(filled in)** |
| Nothing else was added: no files, no program arguments, no bit operations | The language is complete without them, and each can be added later as one more built-in function. *Principle: keep the language small.* **(filled in)** |

---

## 13. Rules that were adjusted

Rules of Jack's that changed while the gaps were filled. Each is marked **(adjusted)** in
`DOCS.md`.

| Where | Old wording | New wording | Why |
|---|---|---|---|
| DOCS 7.8, function given back | "It can't outlive what was passed in… An argument handed over with `@` doesn't count" | The same, and also: it can't outlive the block its call was written in, unless it keeps only a heap value it took with `@` | Kept parameters need a place to live, and the stack of the caller is the only one that isn't hidden heap use. This is the bigger change: see `FILLED_IN.md` section 4 |
| DOCS 8.17 and 8.6, a `const` lent to a function held in a variable | Listed under "What the compiler doesn't catch": "only the type is known" | The compiler rejects the line: a `const` is lent only where the compiler sees it is only read | Otherwise a `const` could change. The rule is the one 8.12 already uses for calls through a variable |
| DOCS 8.17, "Not settled yet" | "Two functions given back by one call that use the same `var` parameter… Whether they share the kept parameter or each keep their own" | Settled: they share it (7.8) | It was the docs' own open point |
| DOCS 5.9, the row for `&a[:]` | "the whole array, the same as `&a`" | The same for an unsized array. For a fixed array, `&a` is an `&[3]T` and `&a[:]` is the `&[]T` | Strict types: an `&[3]T` and an `&[]T` are different types |
| DOCS 13, the checks while running | "Four checks happen while the program runs" | "These checks happen while the program runs", with the new checks added | The table already had five rows, and the filled-in rules add more |
| DOCS 3.2 and 13, the order of operations | step 2: "`!` and the marks"; step 3: "`*`" | step 2: "`!`, `-` and the marks"; step 3: "`*` `/` `%`" | The new operators had to be placed in the order |

---

## 14. For the compiler writer

All of this section is **(filled in)**.

| Choice | Why |
|---|---|
| The full grammar is in one place, DOCS Appendix A, in the notation of the type grammar, with the order of operations built in | A parser is written from one grammar. The rules that tell `[3]int` from `[3]`, and a block from a struct value, are listed with it. *Principle: say what happens underneath.* **(filled in)** |
| Every check is in one place, DOCS Appendix B: what the compiler rejects, with an example and the rule, and what is checked while running, with the reason it prints | Nothing has to be collected from the sections to write the compiler. *Principle: the compiler checks what it can.* **(filled in)** |

