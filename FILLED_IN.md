# Filled in: what was added to MyLang, and why

`DOCS.md` now holds every rule needed to write the compiler and the runtime. This file lists
what Claude added to get there, so Jack can review it. Every added rule is marked
**(filled in)** in `DOCS.md` and in `DECISIONS.md`, and every changed rule is marked
**(adjusted)**.

Nothing in `UNDECIDED.md` was decided, used or built on.

---

## 1. Summary

*(The counts and the short summary are written at the end of this section, once every gap is
filled. The philosophy and the gap list below were written first, before any decision.)*

### 1.1 The philosophy, as the two files show it

These are the principles `DOCS.md` and `DECISIONS.md` show, in my own words. Every gap below
was decided against them.

| # | Principle | Where the files show it |
|---|---|---|
| 1 | **Everything is a value, under one set of rules.** A function, a type, a struct and a package are declared, stored, passed and returned like an `int` | DOCS 1, 4.1, 11.6; DECISIONS 1 |
| 2 | **Explicit.** Reading the code tells you what happens: the heap only at `^`, sharing only at `&`, a change of owner only at `@`, a change of type only at `convert`, `wrap`, `unwrap` | DOCS top, 3.2, 8.7; DECISIONS 1 |
| 3 | **The default stays unmarked.** A plain `T` is owned in place and nothing says so | DOCS 8.1; DECISIONS 1, 5.1 |
| 4 | **Strict types.** A value never changes type by itself | DOCS 3.2, 8.3; DECISIONS 1 |
| 5 | **Declarative.** One shape for every declaration, two kinds of name, no loops, a choice is a lookup | DOCS 2.1, 10; DECISIONS 3, 7 |
| 6 | **Reuse before adding.** Generics are functions that take a type. A slice reuses `&` and `&[]T`. An import is an ordinary declaration | DECISIONS 4.2, 4.3, 9 |
| 7 | **One rule, no special cases.** "The same rule as for a borrower", "files work the same way", "field by field" | DOCS 7.8, 8.8, 9.6, 11.3 |
| 8 | **The compiler checks what it can.** A check while running is kept only where the answer isn't known before, and it always sits at something written in the code | DOCS 4.4, 4.5, 8.12, 8.16 |
| 9 | **Otherwise, do what most languages do.** "Do whatever the conventions are for most programming languages" | DECISIONS 8 |
| 10 | **Symbols for operators, modifiers in front.** What makes a value goes in front, what uses one goes behind | DOCS 3.3, 8.2; DECISIONS 4.1, 5.2, 8 |
| 11 | **A type is never called.** What acts is a function, and built-ins are ordinary function values | DOCS 4.1, 4.4; DECISIONS 4.2 |
| 12 | **Say what happens underneath.** A `bool` is a number, a pair is an array of two, a function is an address | DOCS 3.1, 5.7, 5.8, 7.7, 8.18 |

### 1.2 How the files decided real cases

Reading the decided cases side by side shows the patterns behind them. I used these patterns
as the test for every new rule.

| Case | How it was decided | The pattern |
|---|---|---|
| Packages | A file is a value of type `package`, made by an ordinary declaration. Each import is a new copy, and `&` shares one | A new thing is an old thing again: a value, a declaration, a borrow |
| Pairs | `?T` is an array of two underneath, the way a `bool` is a number. The types keep them apart | One representation underneath, and strict types on top |
| Slices | `&a[i:j]` reuses the mark `&`, the type `&[]T` and every borrowing rule | No new type and no new rule where an old one fits |
| Files | A function in a file uses the copy it is called through, "the same rule as for a function inside a struct" | Say the rule once, then apply it again |
| Ownership | The type tells the three cases apart, and a struct is handled field by field | The type carries the meaning, and a compound value follows its parts |
| `=` | It means the same for every type: a complete copy | One meaning everywhere, even when it costs work |
| Arrays and maps | The length and the keys come from the first value and never change, because a value on the stack has a fixed size | A size that isn't in the type comes from the first value, and stays |
| Conversions | Written out with both types, and checked while running that the value fits | A value always fits its type, and the check sits where the code shows it |
| Calls through a variable | The compiler looks at a function called by its own name, and assumes the worst for one it can't see | Sound without a new notation: when in doubt, count it as the worst case |
| `&&` / `||`, brackets, the order of operations | The usual ones | Convention, where the philosophy has nothing to say |

Seven working rules follow from these, and most of the filled-in rules are one of them
applied again:

1. **A value always fits its type.** A written number is checked by the compiler, and a
   conversion or a sum is checked while running.
2. **A size that isn't in the type comes from the first value, and stays.** This holds for the
   length of a `[]T`, the keys of a map and the type inside an `any`.
3. **`==` compares exactly what `=` copies.** So a copy always equals its original.
4. **Field by field.** What is done to a struct or an array is done to each part: copying,
   moving, comparing, and the lifetime rules.
5. **Values first, then places.** A call works out every argument and then sets the
   parameters. An assignment works out its right side and then finds the place on the left.
6. **One name means one thing.** A name is declared once where it can be seen.
7. **By its own name, the compiler looks. Through a variable, it counts the worst case.**

When two principles pulled different ways, I took the option that kept the language smallest
and most uniform, and the row for that rule says so.

---

### 1.3 The gaps, sorted into groups that touch each other

This list was written before anything was decided. Each group was then decided as a whole.

#### Group A: writing the source
- which characters a source file may hold, and how it is encoded
- names: which characters, and whether case matters
- the reserved words, and whether the built-in names are reserved
- comments, whitespace and line breaks
- whole numbers in other bases
- floats: the exact form, and exponents
- `char` literals and escapes; how big a `char` is against the source encoding
- text: how it is written and what type it has
- `true`, `false`, `null`: what they are
- the type of a literal that has no place to take a type from
- the file name behind `import("geom")`

#### Group B: numbers (touches A, C and F)
- division and remainder, and what whole-number division does
- division by zero
- overflow in `+`, `-`, `*`
- a minus in front of a value
- the special float values
- which types `<` works on
- the type of an index and of `.length`
- conversions that 4.4 doesn't list: `bool`, float to float, float out of range

#### Group C: equality and map keys (touches B and H)
- what `==` means for arrays, pairs, structs, maps, heap owners, borrowers, ownerships,
  functions, types, `any` and packages
- which types a map key can have
- the same key twice in a map value
- comparing with `null`

#### Group D: names and scope (touches A, H, I and J)
- declaring a name twice in one block
- shadowing a name from an outer block, a parameter, a field or a built-in
- the order of declarations in a block, and using a name above its declaration
- a type that names itself (`Node`), and two types that name each other
- calling a function at the top level before a name it uses is set up
- how a name is found: block, file, built-ins, other files

#### Group E: statements and assignment (touches F and L)
- the full list of statement forms
- which expressions can stand alone
- every form of assignment: a name, an element, an entry, a field, through `^`, and `@`
- whether an assignment is a value
- what can stand on the left of `=`
- code after a `return`; `return f();` in a function that returns nothing

#### Group F: while the program runs (touches B, E, G and K)
- what a failed check does: what stops, what is printed, the exit status
- the full list of checks made while running (an index out of range is missing from the docs)
- the order in which operands, arguments and the sides of `=` are worked out
- when the top-level names of each file are set up
- the exact rule for `main`, and the exit status with and without it
- running out of stack, and repeating without end in a language with no loops
- running out of heap

#### Group G: what every program needs (touches A, B and F)
- printing
- reading input
- text: length, joining, comparing
- text to number and back
- ending the program with a status

#### Group H: structs (touches C and D)
- where a field's name comes from when the type lists only types (`pt.x` on a parameter)
- two struct values of one type with different field names
- whether a field's value can use the fields above it
- a function inside a `const` struct that changes a field
- reading a function out of a struct: which copy it uses
- what "uses the fields of the copy it is called through" is underneath

#### Group I: packages (touches D, F and H)
- whether `const` on a package freezes its variables (the examples change them)
- how the compiler checks names through a package in a `var`, a parameter or an array
- a function that takes a package, called through a variable
- what `import` takes, where it may be written, and what it runs
- how much room a `var` package needs when it is given a bigger file

#### Group J: types and generics (touches C, D and K)
- how a function type such as `(type)([]T)T` refers to the type it is given
- what a generic body may do with a `T`
- which types can be handed to a function that takes a `type`
- how long a type lives, for a function given back that keeps one
- what a `type` value is underneath
- where a type may be written; a type reached through a package
- a `var` holder of a type with no size in it: `any`, a `[]T`, a map
- arrays whose elements have no size in their type

#### Group K: functions (touches F, J and L)
- the type of a function value that has no place to take its type from
- two functions given back by one call that use the same `var` parameter (8.17)
- what copying a function value does to what it keeps
- where kept parameters live, with no heap use the code doesn't show
- how far a function given back can travel

#### Group L: ownership (touches E, H and K)
- whether an ownership `@T` can be `null`
- what a `const` ownership may do
- `@` on a map entry
- a `const` lent with `&` to a function held in a variable (8.17)
- the lifetime rules for a struct, an array or an `any` that holds a borrower or a function
- what happens when a borrower kept inside a heap value is followed after its target is gone

#### Group M: for the compiler writer
- the type of every expression, in one place
- a complete grammar, in one place
- every check the compiler makes, and every check made while running, in one place

---

*Sections 2 to 5 (every filled-in rule, the adjusted rules, the bigger changes and what waits
on `UNDECIDED.md`) follow below as the groups are decided.*
