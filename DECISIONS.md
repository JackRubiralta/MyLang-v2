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
| The left side of `=` is a place: it starts at a `var` name, or goes through a pointer that may be changed through | It is the `const` rule said once: what a `const` name reaches can't be changed through that name. A struct with no name isn't a place, and neither is a slice, which is a copy. *Principles: one rule; the compiler checks what it can.* **(filled in)** |
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
| Only a value written on the spot takes its type from its place: a number, `null`, an array or text, a map value, a function value. The place can be reached through brackets, `-`, the arithmetic operators and the marks `^`, `&` and `@`. With no place a number is an `int` or a `float` | That is how `5` and `[3, 10]` already work (DOCS 3.2, 5.8), and `^5` is already a `^int` where one is asked for (DOCS 8.3). Every other value has its type already and never changes it. The everyday names are the defaults. *Principle: strict types.* **(filled in)** |
| The type of every form of value is gathered in one table (DOCS 3.8) | The rules were spread over the sections; a compiler needs them side by side. *Principle: strict types.* **(filled in)** |
| A basic type is a name, or a name reached through a package: `geom.Box` can stand wherever a type is written | A name from another file is always reached through its package (section 9), and a type is a name like any other. *Principle: one rule, no special cases.* **(filled in)** |
| The built-ins that take a type have types of their own: `convert` is a `(type, type)(A)B`, `wrap` a `(type)(T)any`, `unwrap` a `(type)(any)T`, `join` a `(type)([]T, []T)[]T` | A built-in is an ordinary function value, so it has a type that can be written. *Principle: everything is a first-class value.* **(filled in)** |
| A function type names the types it is given with names that aren't declared. Each `type` parameter opens a slot, and each new name takes the first free slot to its left. A name with no slot is an error, and the name itself is not part of the type | Jack's own example writes `(type)([]T)T`, so the type has to stand without the value beside it: it can then be a parameter's type or a named type. Weighed: taking the names from the function value, which leaves a bare type with nothing to name; naming the parameter inside the type, which the rule "a type lists only types" (4.4) excludes; this rule. *Principles: reuse before adding; every value follows the same rules.* **(filled in)** |
| A generic function is checked once, and a `T` can be used only in the ways every type allows | The function's type then says everything about it, so a generic function can be held in a variable and still be checked before the program runs. To add two `T`s, the caller passes the function that adds them. Weighed: checking again for each type handed in, which only works when the compiler can see which function is called. *Principles: every value follows the same rules; the compiler checks what it can.* **(filled in)** |
| A type name is a built-in type, a `const` declared with a type written out, or a `type` parameter. A type that decides the type of something is written out with type names. A type worked out another way (`x.T`, `wholeTypes[1]`, a `const` set from one) can be compared, stored and passed | "Every value's type is known before the program runs" (this section). A type read out of an `any` is known only while running, so it can't declare anything. *Principle: strict types.* **(filled in)** |
| A type never ends too soon: it points at nothing, and it is copied into a parameter like every value | A type never changes (this section), so it is never an argument that a function given back could outlive. *Principle: one rule.* **(filled in)** |
| A top-level type name can be used anywhere in its file, and the compiler collects those names first | A type never runs, so the order can't matter, and the grammar needs to know which names are types (section 8). *Principle: the compiler checks what it can.* **(filled in)** |
| "A type is always held as a `const`" is checked where a type shows it: a name, a parameter or a field whose written type has `type` in it anywhere outside a function type. A type inside an `any`, or under a `T`, is not covered | The compiler can't see inside an `any` or a `T`. Such a type is known only while running and can never declare anything, so every value's type is still known before the program runs. *Principle: the compiler checks what it can.* **(adjusted)**, see section 13 |
| Two types that name themselves are the same when no difference shows up however far the names are followed | "A name for a type is the same type under another name" (DOCS 4.1), applied to a type whose shape never ends. *Principle: one rule, no special cases.* **(filled in)** |
| `convert` reached through a variable, with a pair of types it has no conversion for, fails while running | `convert` is a first-class value, and its type promises every pair. Where the compiler sees the call it rejects the pair; where it can't, the check moves to the running program, like every check the compiler can't make. *Principles: everything is a first-class value; the compiler checks what it can.* **(filled in)** |
| Underneath, a `type` value is a description of the type's shape. Size, copying, moving, freeing, comparing and finding the pointers inside all come from it. The compiler writes the descriptions of written-out types, and a type built from a type parameter is put together when the line runs. Two type values are equal when they describe the same type | An `any` and a generic function both need exactly that, so one thing serves both. A description that is a value needs no place that could end too soon. *Principle: say what happens underneath.* **(filled in)** |
| `convert` has these pairs, in both directions: two number types; a `char` or a `bool` and a whole-number type; a number or a `bool` and text. Each of these types converts to itself. Every other pair is an error, and so is a pair with a type parameter | A `bool` and a `char` are numbers underneath (DOCS 3.1), and the docs say a `bool` is used as a number only through a conversion. Text conversion reuses `convert` instead of adding functions: both types are written, and a text that doesn't fit fails like any conversion that doesn't fit. *Principles: reuse before adding; explicit.* **(filled in)** |
| An `any` keeps the type of its first value, and the value inside keeps its own size | Its size comes from the value inside, and a value on the stack keeps its size, like an unsized array's length (4.3). Weighed: letting a `var` `any` change type, which needs room nobody can know before the program runs. *Principle: one rule, no special cases.* **(filled in)** |
| A type can name itself, at the top level or inside a function, but it holds itself only through `^`, `&`, `@` or a function type | `Node` already names itself (DOCS 8.14), and a pointer has one size whatever it leads to. *Principle: the compiler checks what it can.* **(filled in)** |

### 4.3 Arrays and maps

| Choice | Why |
|---|---|
| A fixed array `[3]int` or an unsized `[]int` whose length comes from its first value and never changes | A value on the stack has a fixed size |
| A map has a fixed set of keys, set by its first value | The same reason: it can't grow |
| A pair `?T`: one value for `false` and one for `true`, looked up with a `bool`, written like an array of two: `const sizes: ?int = [3, 10];` | Jack: "we could have a type called pair". A pair always has both values, so a lookup never fails. `?` is used nowhere else in MyLang, and the values in `[ ]` can't be mistaken for comparisons |
| Slices: `&a[i:j]` borrows part of an array as an `&[]T`; `a[i:j]` without `&` copies those elements | Jack: "add slices … I think they make sense". It reuses what exists: the borrow mark `&`, the type `&[]T` that already borrows a whole unsized array, and every borrowing rule in section 8. The `i:j` form, start included and end left out, is what most languages use. Underneath a slice is two numbers, so nothing is copied **(filled in: the form and the rules)** |
| Underneath, a pair is an array of two, the way a `bool` is a number (`false` is 0, `true` is 1) | Jack: "this is just like an array under the hood, the same way a bool is basically just an int under the hood". An array's size is always a number, so no type goes between its brackets |
| An index and a slice bound are `int`s. The compiler knows a length when it is in the type, or when the array is a name declared with an array value written out, and then it checks a written index or bound itself. Otherwise an index out of range fails while running | The docs check a slice this way (DOCS 5.9), and an index is the same question. The docs' own `a = [1, 2, 4, 5];` is rejected for a name declared that way, so the same knowledge now serves an index, a slice and an assignment. A length is an `int` too, so `i == values^.length` needs no conversion. *Principles: one rule; the compiler checks what it can.* **(filled in)**, and **(adjusted)** for the slice: see section 13 |
| An array's size is a written whole number, 0 or more, and `[]` is the empty array value | "An array's size is always a number" (the row above). An empty text has to exist, so an empty array does. *Principle: one rule.* **(filled in)** |
| The entries of a map are held in the order written, and a lookup compares its key with `==` | It says what the picture "a map is like a struct" (DOCS 6.4) means for a lookup. *Principle: say what happens underneath.* **(filled in)** |
| A size that isn't in the type comes from the first value and stays: for every holder of a `[]T`, a map or an `any`. The compiler makes the check when the values are written out in the code, and otherwise a new value of another size fails while running | It is the docs' rule for `var a: []int` (DOCS 5.3), said once for every type without a size and every holder: a name, a field, an element. "Written out" has one meaning everywhere (DOCS 3.2), so two compilers reject the same lines. *Principle: one rule, no special cases.* **(filled in)** |
| A fixed array becomes an unsized one through a slice: `primes[:]`, `&primes[:]` | The two are different types, and a value never changes type by itself. The slice form already exists. *Principles: strict types; reuse before adding.* **(filled in)** |
| A pair value has exactly two values, a pair has no `.length`, and its two values can be assigned and borrowed like elements | A pair always has both values (this section), so there is nothing to ask. Underneath it is an array of two, so its values are holders like elements. *Principle: strict types.* **(filled in)** |
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
| A call in a field's value can run only when every field its function uses is written above the call | A function in a struct value can use a field below it (the row on functions in structs), so a call could otherwise read a field with no value yet. It is the rule for a call at the top level of a file (section 9). *Principle: one rule, no special cases.* **(filled in)** |
| A field name is a label for a position, and it belongs to the type as it is written: to a type name, or to a struct type written out. Every struct value written for one written type gives the same names, in the file that declares the type. A type name reached through a package brings its file's names | The type lists only types and the value gives the names (this section), so the names can't be part of the type: a `Point` and a `Size` of two `int`s are one type. But a reader expects `p.x` on a `Point` and `s.w` on a `Size`, and expects to find the names where the type is declared. The compiler already knows how every type was written, so the written form can carry the names with nothing added to the type. One set of names for each written type means a mistyped field name is caught. Weighed: names written in the type, which turns Jack's rule round; one set of labels for each shape, gathered from the file and all it imports, which lets a far file change what `p.x` means; several names for one position, which a typo could add; this rule. *Principles: explicit; one rule, no special cases.* **(filled in)** |
| A function can't change the fields of a `const` struct that is declared with its block. A struct value that comes into a `const` from elsewhere brings functions the compiler can't see, and that case is listed as not caught | It is the docs' own way with a `const` that a function it can't see might change (DOCS 8.17): Jack left that case unchecked, because counting the worst case would turn away every call through a variable, also the ones that only read. The same reason holds here, so the two cases get the same answer. Weighed: rejecting every call of a function field through such a `const`. *Principles: one rule, no special cases; the compiler checks what it can.* **(filled in)** |
| A function written in a struct value that uses the fields keeps the address of its struct, the way a function in a file keeps its package. A copy, a move or a `return` of the struct re-aims it. Such a function can use nothing else that needs a place, and reading it out gives a function that borrows the struct | "Files work the same way" (DOCS 9.6): one mechanism for a struct and a package. A function value has one address for what it keeps (6.1) and a struct holds only its fields (DOCS 9.8), so a function can't keep its struct and a second place as well. What it needs from outside goes into a field, which shows it. Weighed: handing the struct over at each call, which would stop a function field being read out as a value, against first-class. *Principles: one rule, no special cases; everything is a first-class value.* **(filled in)**, and **(adjusted)** for what such a function can see: section 13 |

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
| A value holds a borrower, an ownership or a function when it is one, or has one in a field, an element or an entry. An `any`, a `T` and a package count as holding all three. What lies behind a pointer doesn't count. A value can't outlive what a part it holds points at | Field by field (5.4): the lifetime rules for a whole value are the rules for each part. Behind a pointer the compiler can't follow, which is the docs' own limit (DOCS 8.17). One meaning of "holds" serves every rule. *Principle: one rule, no special cases.* **(filled in)** |
| A result that holds a pointer counts like a borrower that came back, and an argument that holds borrowers counts as lending what they borrow. A pointer behind a `^` in the result's type doesn't count. A borrower that is given back always comes from a parameter | The caller sees only the type, as with a borrower that comes back (5.6). A borrower of an outside variable would reach the caller with no origin. Behind a `^` the compiler doesn't follow, so the copy that `copyLinks` gives back stays a value of its own, as the docs say. *Principle: one rule.* **(filled in)** |
| A `var` borrower is made only from a place that can be assigned, a value made on the spot, or another `var` borrower. A copy of a `const` borrower can only read | Otherwise `var y: &int = x;` would turn a borrower that can only read into one that can change. The compiler checks it wherever it sees both sides. Through a variable only the type is known, and that case stays on the docs' list of what isn't caught (DOCS 8.17). *Principle: the compiler checks what it can.* **(filled in)** |
| A `const` that is or holds a function is never lent to a call through a variable | The compiler relies on knowing which function such a `const` holds (the row on calls in section 6). If a call it can't see could replace it, a call by name would run something the compiler never looked at, and that could free memory that is still pointed at. So this one kind of `const` is taken out of the case the docs leave unchecked. *Principle: the compiler checks what it can.* **(filled in)** |
| An ownership is given only by a `var`, and a `const` ownership can only read | One rule for every `const`: what a `const` name reaches can't be changed through that name. It is the borrower table (DOCS 8.6) applied to the third mark. *Principle: one rule, no special cases.* **(filled in)** |
| `null` fits a `^T`, an `&T` and an `@T`. Following any of them when `null` fails, and so does taking a value through a `null` ownership | `null` is "no address" and all three hold an address (DOCS 8.18). *Principle: every value follows the same rules.* **(filled in)** |
| `@` works on a map entry | A map entry is a holder like an array element, and it can already be borrowed (DOCS 8.4). *Principle: no special cases.* **(filled in)** |
| A link to a map or an `any` holds one address, and the value there holds its own size | It completes DOCS 8.18, which says it only for an unsized array. *Principle: say what happens underneath.* **(filled in)** |
| What comes in through a parameter that holds a borrower, an ownership or a function is only known to last as long as the call. It can't be put in a variable that outlasts the call | A borrower can't outlive what it borrows (DOCS 8.5), and a function can't outlive what it keeps (6.1). Inside the function nothing more is known about the caller's value. Stored through a pointer it is the docs' `linkBack` case, which stays on the list of what isn't caught. *Principle: the compiler checks what it can.* **(filled in)** |
| A call counts as a use of the pointers its function uses. "Every outside variable" means every top-level `var`, and each name of a function around the call that a function body inside that function uses | "Still used below" (5.6) has to include a function that reads the pointer when it is called further down. A name that no function body uses can't be reached by a function the compiler can't see. *Principle: one rule.* **(filled in)** |
| `=` copies a function's two addresses, a `type` as itself, an `any` with its inside, and a package with its variables | It completes the table of what `=` copies (5.3), which `==` builds on (section 8). *Principle: every value follows the same rules.* **(filled in)** |
| A heap value owned by a kept parameter is freed when the call that holds the kept parameters ends | "When the function given back ends" is not one moment once the function has been copied. The call that holds the kept parameters ends once. *Principle: say what happens underneath.* **(adjusted)**, part of A1 in section 13 |
| What the listed cases of DOCS 8.17 do while running: following a borrower whose target is gone isn't checked and isn't defined; a `const` changed through a function the compiler can't see is simply changed | The docs leave these to the programmer. A check would need something to run that the code doesn't show. Saying what happens lets a compiler writer know what may and may not be assumed. *Principle: explicit.* **(filled in)** |

---

## 6. Functions

| Choice | Why |
|---|---|
| A function is written `(params){ body }`, its type is `(int, int)int`, and parameters are declared like names | Everything is declared in the same shape |
| A function that returns a value has to reach a `return` on every path | Otherwise it could end with nothing to give back |
| `return;` on its own stops a `void` function early | Jack: "that's allowed", as in most languages |
| A function's body is a block, so it sees the names around it and can change a `var` from outside | Jack: "just variables in a broader scope" |
| "Every path reaches a `return`" means the body runs into one: a `return` among its statements, or in a block inside it. Code after a `return` is accepted and never runs | No statement branches, so a body is one straight line. Jack: `return;` stops a function early, "as in most languages", so code can stand after it. Weighed: rejecting code after a `return`, which would leave `return;` only at the very end. *Principle: do what most languages do.* **(filled in)** |
| A function value is checked against its type, and takes its return type from its place | The value lists parameters and the type lists the return type (DOCS 7.1), so the place supplies the one thing the value doesn't write. *Principle: strict types.* **(filled in)** |
| A call gives one argument of the right type for each parameter. A call to a function that returns nothing has no value | The usual rule, and `void` means there is nothing to use. *Principles: strict types; do what most languages do.* **(filled in)** |
| "Called by its own name" has an exact meaning (DOCS 7.2): a `const` declared with a function value written out; a function value called where it is written; a built-in or what it gives back; a function value that a seen function gives back; a `const` declared with any of these; the same through a one-file package or a `const` struct written out; and a lookup in a `const` array, pair or map whose functions are all written out. Everything else is a call through a variable, and counts the worst case, except for whether a `const` is changed | The docs use the two phrases in many rules without saying where the line is. The lookup case is there because every choice is a lookup: without it, each `cases[…](…)` would count as an unknown call, and choosing would cost what the compiler knows. A user function such as `first(int)` is seen like a built-in, with no special case. The exception for `const` is Jack's own (DOCS 8.17). *Principles: one rule, no special cases; the compiler checks what it can.* **(filled in)**, and **(adjusted)** for 5.6: see section 13 |

### 6.1 Functions that use variables from around them

| Choice | Why |
|---|---|
| A function borrows the variables it uses from around it | It works on the variable itself, so its changes reach it, and nothing is copied out of sight |
| A variable that a function uses can't die before the function | It is the borrower rule. Returning a function that uses a local is an error |
| A function given back keeps the parameters it uses | `makeAdder(step)` "truly should work": the parameters are what the caller handed in |
| It can't outlive the arguments of the call | "If the parameter that's put into makeAdder dies before what makeAdder returns, the compiler raises an error." It is the same rule as for a borrower that comes back |
| An argument handed over with `@` doesn't count | The call took the value and the caller's variable is `null`, so nothing is left to die **(filled in)** |
| What a value handed over with `@` holds still counts: a borrower or a function inside it keeps its own limit | "Nothing is left to die" is true of the value itself, but not of what a borrower inside it points at. Field by field (5.4). *Principle: one rule, no special cases.* **(adjusted)**, see section 13 |
| A local is kept by putting it on the heap and handing it with `@` to a parameter of a function that gives back the function using it | It needs nothing new: `^` puts the value on the heap (8.3), and a parameter that asks for an ownership takes it (8.13). A function given back keeps its parameters |
| Every function value is two addresses: its code, and what it keeps | Every function of a type has the same size, so functions go in arrays and maps |
| One call has one set of parameters: every function that a call gives back uses the same ones | This settles the point the docs left open (DOCS 8.17). A function borrows the variables it uses (this section), and a kept parameter is such a variable, so two functions that use it share it. Weighed: each function keeping its own copy, which would make a counter and its reader disagree. *Principles: one rule, no special cases; reuse before adding.* **(filled in)** |
| Copying a function copies its two addresses, so the copy uses the same variables and kept parameters | "Copying a function value copies the address" (DOCS 7.7). What a function keeps is a borrow, and a copy of a borrower borrows the same thing. *Principle: every value follows the same rules.* **(filled in)** |
| The second address of a function is the place of what it keeps: nothing, its package, its struct, the stack place of a call around it, or a call's kept parameters. A function needs its package only when it uses a `var` or a worked-out `const` of its file, or a function that does | It shows why one address is always enough (the row above), and it makes `==` on two functions exact. *Principle: say what happens underneath.* **(filled in)** |
| Which results keep parameters: where the compiler sees the function that is called, it looks at what comes back. Through a variable, a result whose type can hold a function counts as keeping every parameter | `pick(double)` and `wrap(int)(3)` give back nothing that keeps a parameter, so no rule should hold them. Looking at a function called by its own name is the docs' own method (5.6). *Principle: the compiler checks what it can.* **(filled in)** |
| A result that is or holds a function comes from the arguments it keeps, from the function that was called, and from what an argument came from when the result is that argument. It can be given back when all of that lasts beyond the call or goes along: a parameter, an outside name, kept parameters this call holds. Not a name declared in the function, and not an argument made on the spot | "A function given back keeps the parameters it uses", and "a local is never kept" (this section). The function that was called counts because what it gives back can reach its kept parameters through it. Without the first row of the list, a generic function could never give back what a function it was handed returns. *Principle: one rule, no special cases.* **(filled in)** |
| Kept parameters stay on the stack, with the function call that made the call. They go along when the function is given back again. In no other way can the function leave that call: it can't be put in a variable that outlasts it. They end when the call that holds them ends | A function value is two addresses (this section), and nothing goes on the heap unless the code says `^` (section 1). That leaves the stack. A value that is given back is taken by the caller, so kept parameters can travel upward the same way, which is how Jack's `newCounter` hands on its counter. Stored through a pointer, the function is beyond what the compiler follows (DOCS 8.17). Weighed: a hidden heap block, which is heap use the code doesn't show; function values of different sizes, which undoes the row above; no move on `return`, which breaks `newCounter`; this rule. *Principles: explicit; say what happens underneath.* **(adjusted)**, see section 13 |

---

## 7. Choosing and repeating

| Choice | Why |
|---|---|
| Repeating is a function that calls itself | The language is declarative |
| Choosing is a lookup: pairs `?T` and maps of functions | A `bool` picks a slot, and a map of functions does the job of a `switch`. Recursion uses a lookup to stop |
| A program that calls deeper than the stack allows fails like any other check | The stack is fixed by the system, so it is one more check at something the code shows: a call. *Principle: one rule.* **(filled in)** |
| The last call takes over the stack of the function that makes it, when nothing the call uses lives in that call: its names, the heap values they own, its kept parameters, and the values made on the spot in the statement | Repeating is a function calling itself (this section), so repeating without end has to be possible: a program that reads lines until the input ends can't have a limit. The condition is the borrower rule: what something still points at can't be given up. An ownership that is handed over has moved its value, so it holds nothing back. Weighed: no promise, which leaves every repeat with a limit nobody can see in the code; this rule, which Scheme has for the same reason. *Principles: declarative; say what happens underneath.* **(filled in)** |

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
| `%` works on whole numbers only, and fails only when the right side is 0 | A remainder of floats can be written with `/`, `convert` and `*`, so it needs no operator. *Principle: reuse before adding.* **(filled in)** |
| `-` in front of a signed whole number or a float. Directly in front of a written number it is part of the number | The docs already write `-7`. A minus that worked only on written numbers would be a special case. Weighed: only written numbers, with `0 - x` for the rest; a minus for every number. *Principle: one rule, no special cases.* **(filled in)** |
| A result that doesn't fit its type fails while running. Nothing wraps around | A conversion already fails when its value doesn't fit (DOCS 4.4). The same rule for `+ - * / %` means a value always fits its type, and it covers overflow and division by zero together. Weighed: wrapping, as in C, Go and Java; failing, as in Swift and Zig; leaving it open. A wrapped number is a wrong value the code doesn't show. *Principles: explicit; one rule.* **(filled in)** |
| No infinity and no "not a number": a float answer that isn't an ordinary number fails | It is the same rule again. It also keeps `==` simple: a float always equals itself, so a copy equals its original for every type. Weighed: the special IEEE values, as in most languages; failing. Here the philosophy decides. *Principles: one rule; explicit.* **(filled in)** |
| The compiler makes these checks itself when every number is written in the code | The docs do the same for a slice whose bounds are written numbers (DOCS 5.9). *Principle: the compiler checks what it can.* **(filled in)** |
| `<` `>` `<=` `>=` work on numbers and `char`s only | A `char` is a number underneath. Ordering text or structs can be written as a function, so nothing more is needed. *Principle: reuse before adding.* **(filled in)** |
| `==` compares exactly what `=` copies, for every type | One rule for every type, and a copy always equals its original. A `^T` compares the values, because `=` copies the value. An `&T` compares the addresses, because `=` copies the address. Structs and arrays go part by part (5.4). Weighed: `==` only for basic types; heap owners compared by address; this rule. *Principles: one rule, no special cases; every value follows the same rules.* **(filled in)** |

---

## 9. Files and packages

| Choice | Why |
|---|---|
| A package is a file | Jack: "I think a package should be a file." One file is one scope, so every name comes from this file or from a package it imported, and you can see which |
| `const geom: package = import("geom");` | The import is an ordinary declaration in the one shape every name uses. `import` is a built-in like `convert`, and `package` is a built-in type like `type` and `any`. Zig works the same way |
| Everything at a file's top level can be used from outside, for now | The simplest start |
| Statements that run go only in the file you run; an imported file holds only declarations | Nothing runs because a file was imported, so the run order never depends on imports. **(adjusted)**: the values of the declarations do run, where the `import` is written. See section 13 |
| Good practice, not required: a `const main: ()int` in the file you run, and code can still go outside it | Jack: "it's like good practice to have a const main function, which returns an int. But you can have code outside of that." When it is there, the program runs the file's statements from the top, then calls `main`, and its `int` is the program's result **(filled in)** |
| A package can be a `const` or a `var` | Jack: underneath it is like a function pointer. A package value is the address of the file's names, so it is held like a function, and a `var` one can be given another file |
| Ownership and borrowing apply to a package like any other variable: copy, borrow, heap, move | Jack: "ownership rules and stuff like that, borrowing should all apply to a package, just like any other variable". A package is used like a struct whose fields are the file's variables, and borrowing is how two parts of a program share one copy |
| A function that takes a package uses only names every package given to it has | The compiler checks each call, so no name is looked up while the program runs **(filled in)** |
| A `var` package can only be given a file with the same names and types the code uses | The same as a function in a `var` keeping its type, so the compiler still checks every name **(filled in)** |
| Each import makes a new copy of the file's variables | Jack: importing a file twice should make two copies. Every declaration makes a new value (2.4), and sharing happens only where `&` says so: to share one copy, borrow it |
| Two files can't import each other | Each import would make a new copy, forever **(filled in)** |
| A function's body can use any top-level name of its own file, wherever it is declared | Two functions in one file can call each other **(filled in)** |
| `import` takes one text written out in the code, and is always called by its own name. The text is the file's path without `.my`. A text that leads to no file is an error, two texts that lead to one file name the same file, and a file can't import itself | The compiler has to know every file before the program runs, or it couldn't check a single name. This is the one built-in that can't be passed around. *Principle: the compiler checks what it can.* **(filled in)**, and **(adjusted)** for the sentence in DOCS 13: section 13 |
| An import runs where it is written, and all it runs is the working out of the file's top-level declarations, from the top down | An import is a call that makes a new value (this section), and a declaration's value can be a call. So "nothing runs because a file was imported" is now "only its declarations run, where the `import` is written": the run order is still what the code shows. *Principle: explicit.* **(filled in)**, and **(adjusted)**: see section 13 |
| A file's top-level names keep their own `const` and `var`. The word on the package says only whether the whole package can be replaced or moved | Jack's examples change `geom.made` and `a.count` through a `const` package, so the word on the package can't freeze them. Fields follow their struct because they have no word of their own; a file's names each have one. *Principle: explicit.* **(filled in)** |
| The compiler works out, over the whole program, which files each place can hold. An `import` is its file. A name is a place of its own, also an array or a struct of packages. A pointer to a name shares its place. The result of a call the compiler sees is looked at. A heap value, an `any` and the result of a call through a variable are one shared place. A name has to exist with one type in every file of the place | This is how the two filled-in rows above about a `var` package and a function that takes a package are checked. Keeping the files with each name means a line in a far file can't break `plugins[0].draw()`. The shared place is only for what the compiler can't follow. *Principles: the compiler checks what it can; explicit.* **(filled in)** |
| A file's names are set up from the top down. A `const` whose value is a function value written out or a written value, and a type name, are ready from the start. A top-level call can't reach any other name that is declared below it | A body can use any top-level name (the row above), so without this a call could read a name before it has a value. What is ready from the start needs nothing to run, so two functions can still call each other. *Principle: the compiler checks what it can.* **(filled in)** |
| A top-level `main` in the file you run has to be a `const` of type `()int`. The status is from 0 to 255, converted like `convert(int, uint8)`. With no `main` the status is 0 | It completes the `main` row above. The status reuses the rule for a conversion that has to fit. *Principle: reuse before adding.* **(filled in)** |
| Underneath, a package value is the address of its file's table of names, then everything in the file that has to be worked out: every `var`, every `const` that isn't ready from the start, and the kept parameters of top-level calls. A copy or a move re-aims the functions held in it. A `var` package has room for the biggest file its place can hold, and a package that doesn't fit fails while running | It is what makes `shapes.area` work for any file with nothing looked up while running. A `const` worked out at set-up can differ between two imports, so it can't be shared the way a function written out is. A top-level value with no size in its type makes the room something only the running program knows, so that case is checked there. *Principle: say what happens underneath.* **(filled in)**, and **(adjusted)**: see section 13 |
| The `const` rules hold for a package: a `var` borrower and an ownership need a `var` package | "Just like any other variable" (the row above on ownership). *Principle: one rule, no special cases.* **(filled in)** |
| The file you run is a package too, with a package value that lasts as long as the program | "A package is a file" (this section), with no special case for the first one. *Principle: one rule, no special cases.* **(filled in)** |
| A function read out of a package, or given back by a call through it, borrows the package. While it is used below, the package can't be given another file | It keeps the package's address, so it is the borrower rule (5.6). *Principle: one rule.* **(filled in)** |

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
| Whole numbers in base ten, base sixteen (`0x`) and base two (`0b`) | They are written the way most languages write them, and they are useful for the numbers of `char`s and for sizes even before there are bit operations. The docs name a type `bitboard` (DOCS 4.1). Base eight and separators inside a number were left out: nothing needs them. *Principles: do what most languages do; reuse before adding.* **(filled in)** |
| A float is `digits.digits`, with an optional `e` and a power of ten | The docs already ask for `3.0` instead of `3` (DOCS 3.2), so a digit stands on both sides of the point. The power of ten writes very big and very small numbers. *Principle: do what most languages do.* **(filled in)** |
| A float answer that falls between two values goes to the nearest, and to the one whose last binary digit is 0 when both are equally near. A written float too big for its type is an error | It is what IEEE 754 does, and a written float is checked like a written whole number. *Principles: do what most languages do; one rule.* **(filled in)** |
| The escapes `\n \t \r \0 \\ \' \" \xHH` | The usual ones. `\xHH` reaches every value a `char` can hold. *Principle: do what most languages do.* **(filled in)** |
| A character outside ASCII doesn't fit in a `char`; in a text it is several `char`s | A `char` is one byte and UTF-8 needs more. Nothing changes size out of sight. *Principles: strict types; say what happens underneath.* **(filled in)** |
| A text is written on one line, and a line break inside it is written `\\n` | An escape already writes it, so a text needs no second form. *Principle: reuse before adding.* **(filled in)** |
| A line break in a source file is `\\n`, alone or after a `\\r` | Files from every system then compile the same. *Principle: do what most languages do.* **(filled in)** |
| A source file's name ends in `.my` | `import("geom")` has to lead to exactly one file. *Principle: explicit.* **(filled in)** |
| These are all the symbols, and the longest symbol is the one that is read | The convention, and how `&&` is already read (section 8). *Principle: do what most languages do.* **(filled in)** |
| A "written value" is a number, a `char`, a text, `true` or `false` in the code, also with a `-` directly in front of a number | Several rules let the compiler check a line when its values are written and leave it to the running program otherwise. One meaning makes every compiler reject the same lines. *Principle: the compiler checks what it can.* **(filled in)** |
| A `-` between two values subtracts, and a `-` where a value starts is the minus in front | The convention. *Principle: do what most languages do.* **(filled in)** |
| In the rules for `(`, "a type" is a type written out. After `()`, a `{` that holds types makes a function type, and anything else is a function body | `(){int, int}` and `(){ return 1; }` start alike, and the grammar has to tell them apart. *Principle: the compiler checks what it can.* **(filled in)** |
| Each example in `DOCS.md` is its own piece of code, and can declare a name again | The examples reuse names such as `count` and `p`. In one file that would break the rule that a name is declared once (section 3). *Principle: the compiler checks what it can.* **(filled in)** |

---

## 11. While the program runs

All of this section is **(filled in)**.

| Choice | Why |
|---|---|
| A failed check stops the program at once, writes `error: FILE:LINE: REASON` to the error output, and ends with status 1. `FILE` is the source file's path from the folder of the file you run, and `LINE` is the line of the operator, bracket or call that failed | One behaviour for every check. `0` is "everything went well" (section 9), so a failure is not 0, and 1 is the usual status for it. The line names the place, because every check sits at something written in the code. *Principles: one rule, no special cases; do what most languages do.* **(filled in)** |
| A failure can't be caught. A program that isn't sure tests first | Catching would be a new kind of statement. The tests already exist (`p != null`, `i < a.length`, `x.T == int`), and they are in the code where they happen. *Principles: reuse before adding; explicit.* **(filled in)** |
| One list of checks, in DOCS Appendix B: a value that doesn't fit its type, a size that doesn't match, a lookup that misses, a `null` that is followed, and no room left | Each comes from one of two rules: a value always fits its type, and a size that isn't in the type stays what the first value made it. *Principle: one rule.* **(filled in)** |
| A declaration that needs more stack than is left fails like a call that does, and a `var` package given a package that needs more room than it has fails too | Each is a size that only the running program knows. *Principle: one rule.* **(filled in)** |

---

## 12. Built-in functions for every program

All of this section is **(filled in)**.

| Choice | Why |
|---|---|
| `print: ([]char)void` writes exactly the `char`s it is given, at once | Nothing is added out of sight, not even a line break. A number is turned into text first, with `convert`, so the code shows it. *Principle: explicit.* **(filled in)** |
| `read: ()[]char` gives the next line with its `\n`, and an empty text at the end of the input | Keeping the `\n` means an empty result can only be the end, so no second function and no special value is needed. *Principle: reuse before adding.* **(filled in)** |
| `exit: (int)void` ends the program with a status | The same status rule as for `main` (section 9). *Principle: reuse before adding.* **(filled in)** |
| Each of them is an ordinary function value | Like `convert` (4.2): things that act are functions. *Principle: a type is never called; built-ins are first-class.* **(filled in)** |
| `print` and `read` fail like any other check when the output can't be written or the input can't be read | One behaviour for every failure (section 11). *Principle: one rule.* **(filled in)** |
| Nothing else was added: no files, no program arguments, no bit operations | The language is complete without them, and each can be added later as one more built-in function. This is a choice not to add, so it is not a rule in `DOCS.md`. *Principle: reuse before adding.* |

---

## 13. Rules that were adjusted

Rules of Jack's that changed while the gaps were filled. Each is marked **(adjusted)** in
`DOCS.md`. `FILLED_IN.md` section 3 has the same list, and section 4 there weighs the bigger
one.

| # | Where | Old wording | New wording | Why |
|---|---|---|---|---|
| A1 | DOCS 7.8, 8.11 and 2.6, a function given back | "It can't outlive what was passed in… An argument handed over with `@` doesn't count: … nothing is left to die." "So they live as long as it does." "The value is freed when the function given back ends." In 2.6: a parameter lives "as long as a function the call gives back uses it". Nothing said where kept parameters are | Kept parameters stay on the stack with the function call that made the call, and go along when the function is given back again. In no other way can the function leave that call: it can't be put in a variable that outlasts it. They end, and a heap value they own is freed, when the call that holds them ends. What a value handed over with `@` holds still counts. A result also comes from the function that was called, and whether a result keeps parameters at all is read off the function when the compiler sees it | Kept parameters need a place that isn't hidden heap use. "When the function given back ends" is not one moment once the function has been copied. A borrower inside a value given with `@` can still outlive its target. This is a bigger change |
| A2 | DOCS 8.12, 8.17 and 7.2, calls the compiler can't see | "For one called through a variable, an array or a map, every `&` argument counts as one that may change." "A call through a variable, an array or a map counts as changing every outside variable." In 8.17: "through a variable, an array or a map, as in `fs[i](&limit)`" | "Through a variable (7.2)" in all three places. A lookup in a `const` array, pair or map whose functions are all written out is a call the compiler sees: it counts the worst of those functions | Every choice is such a lookup. Under the old wording each `cases[…](…)` counted as changing every outside variable, so no function that chooses could be called while a pointer was in use. This is a bigger change |
| A3 | DOCS 8.17 | Heading "…and what isn't settled", with the list "Not settled yet: Two functions given back by one call that use the same `var` parameter… Whether they share the kept parameter or each keep their own" | "Nothing here is left unsettled. Two functions given back by one call share the parameters they keep (7.8)." | It was the docs' own open point |
| A4 | DOCS 5.9, the row for `&a[:]` | "the whole array, the same as `&a`" | The same for an unsized array. For a fixed array, `&a` is an `&[3]T` and `&a[:]` is the `&[]T` | Strict types: an `&[3]T` and an `&[]T` are different types |
| A5 | DOCS 5.9, an example comment | "checked while running: fails if n is over 4" | "fails if n is below 1 or over 4" | The start of that slice is 1, and the rule above the example says a start after the end fails |
| A6 | DOCS 13, the checks while running | "Four checks happen while the program runs" | "These checks happen while the program runs", with the new checks added | The table already had five rows, and the filled-in rules add more |
| A7 | DOCS 3.2 and 13, the order of operations | step 2: "`!` and the marks"; step 3: "`*`" | step 2: "`!`, `-` and the marks"; step 3: "`*` `/` `%`" | The new operators had to be placed in the order |
| A8 | DOCS 11.4 | "Nothing runs because a file was imported, so the order the program runs in never depends on imports." | "All that runs when a file is imported is the working out of those declarations' values, at the line where the `import` is written. So the order the program runs in is always the order the code shows." | A declaration's value can be a call, so importing does run something. What Jack wanted still holds: no statement runs, and the order is the one the code shows |
| A9 | DOCS 11.5 | "Functions and `const`s never change, so they are never really copied." | "A name that is ready from the start never changes and needs nothing worked out, so it is never copied: a function value written out, a written value, a type. A `const` whose value is worked out when the file is set up is held with the variables." | A worked-out `const` can differ between two imports of one file, and it can hold a function that keeps its own package |
| A10 | DOCS 4.2 | "A type can still be held as a value: … inside an `any` (4.5). It is always held as a `const`." | The rule is checked where a type shows it: a name, a parameter or a field whose type is `type`, or an array, pair, map or struct with `type` inside. A type inside an `any` or under a `T` is not covered | The compiler can't see inside an `any` or a `T`, and such a type can never declare anything |
| A11 | DOCS 5.9 | "When both are written as numbers and the length is in the type, the compiler checks them." | "When both are written values and the compiler knows the length (5.2), the compiler checks them, and a bound that is left out counts as written." | The docs already reject `a = [1, 2, 4, 5];` for an unsized `a` that was declared with three elements. The compiler knows that length, so an index and a slice use the same knowledge |
| A12 | DOCS 9.6 with 7.8 | 7.8: "a block sees the names of the blocks around it". 9.6: "A function declared inside a struct is in the struct's scope, so it uses the struct's fields directly." | The same, but a function that uses its struct's fields can use nothing else that needs a place: no name of a function around the struct value, and nothing that needs the package | A function value has one address for what it keeps, and a struct holds only its fields (DOCS 9.8). A function that keeps its struct has no second address for the place around it |
| A13 | DOCS 13 | "Built-ins are first-class values like everything else: they can be named, stored and passed." | The same, with one exception: `import` is always called by its own name with its text written out | The compiler has to know every file of a program before it runs |

---

## 14. For the compiler writer

All of this section is **(filled in)**.

| Choice | Why |
|---|---|
| The full grammar is in one place, DOCS Appendix A, in the notation of the type grammar, with the order of operations built in | A parser is written from one grammar. The rules that tell `[3]int` from `[3]`, and a block from a struct value, are listed with it. *Principle: say what happens underneath.* **(filled in)** |
| Every check is in one place, DOCS Appendix B: what the compiler rejects, with an example and the rule, and what is checked while running, with the reason it prints | Nothing has to be collected from the sections to write the compiler. *Principle: the compiler checks what it can.* **(filled in)** |

