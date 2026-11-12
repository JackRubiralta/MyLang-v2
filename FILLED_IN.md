# Filled in: what was added to MyLang, and why

`DOCS.md` now holds every rule needed to write the compiler and the runtime. This file lists
what Claude added to get there, so Jack can review it. Every added rule is marked
**(filled in)** in `DOCS.md` and in `DECISIONS.md`, and every changed rule is marked
**(adjusted)**.

Nothing in `UNDECIDED.md` was decided, used or built on.

---

## 1. Summary

`DOCS.md` had the heart of the language: declarations, types, ownership, structs, choosing,
files. It did not yet say how the source is written, what most operators do at their edges,
what several rules mean for the compiler, or what happens when something goes wrong while the
program runs. Those gaps are filled now:

- **103 rules were filled in.** They are listed in section 2, each with its reason and the
  principle that decided it.
- **6 rules of Jack's were adjusted.** They are listed in section 3.
- **1 of those is a bigger change,** and it is the first thing to review. It is in section 4:
  how far a function given back can travel.
- **Nothing waits on `UNDECIDED.md`** (section 5).

What was added, in short: the source text and every literal; a full grammar (Appendix A);
`/`, `%` and a minus in front; one rule for results that don't fit; equality for every type;
the type of every value; the rules for names; every form of assignment; generics checked
once; text as an array of `char`s with `join` and `convert`; map keys; field names; the
rules a package's names follow; the set-up order and `main`; what a failed check does;
`print`, `read` and `exit`; the last-call rule for the stack; and one list of every check
(Appendix B).

The philosophy and the gap list below were written first, before any decision.

### 1.0 Where to look first

After the bigger change in section 4, these are the rules where I had to make a real choice.
Each one has the options I weighed in its row of `DECISIONS.md`:

| Rule | The choice |
|---|---|
| 10 (DOCS 2.8) | no shadowing at all, where most languages allow it |
| 27, 28 (3.2) | a result that doesn't fit fails, and there are no special float values |
| 34 (3.7) | `==` works on every type and compares what `=` copies |
| 42 (4.3) | a generic function is checked once, so `+` on a `T` is an error |
| 83 (9.3) | a function field can't be called through a `const` struct the compiler can't see into |
| 86 (9.9) | field names are labels for positions, gathered from every struct value of the type |
| 88 (10.4) | the last call takes over the stack |
| 89 (11.2) | `import` is the one built-in that can't be passed around |
| 91 (11.3) | a file's names keep their own `const` and `var` under a `const` package |

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

---

## 2. Every filled-in rule

The last column names the principle that decided the rule, by its number in the table in 1.1:
1 first-class, 2 explicit, 3 the default stays unmarked, 4 strict types, 5 declarative,
6 reuse before adding, 7 one rule and no special cases, 8 the compiler checks what it can,
9 do what most languages do, 10 symbols and marks, 11 a type is never called, 12 say what
happens underneath.

### Writing the source, names and statements

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| top | 1 | Each example is its own piece of code, and can declare a name again that an earlier example declared | The existing examples reuse names such as `count` and `p`, and rule 10 would otherwise make them errors | 8 |
| 2.7 | 2 | A source file is UTF-8, and its name ends in `.my` | `import("geom")` has to lead to exactly one file | 2 |
| 2.7 | 3 | Only ASCII outside comments and text | A `char` is 8 bits, so one character of code is one byte | 12 |
| 2.7 | 4 | Spaces, tabs and line breaks only separate things. A statement ends at its `;` | Every statement already ends with `;` (2.5) | 2 |
| 2.7 | 5 | `//` is the only comment | One form does the job | 6 |
| 2.7 | 6 | A name is letters, digits and `_`, not starting with a digit. Case matters | The convention | 9 |
| 2.7 | 7 | Seven reserved words: `const var return void true false null` | Only the words the grammar itself needs | 6 |
| 2.7 | 8 | The built-in types and functions are ordinary names, already declared when a file starts | `int` is "an ordinary name" (3.1) and built-ins are ordinary values (4.4). With rule 10 they can't be declared again, so no second reserved list is needed | 1, 6 |
| 2.7 | 9 | The list of symbols, and the longest symbol is the one that is read | The convention. `&&` is already read this way (3.2) | 9 |
| 2.8 | 10 | A name can't be declared where the same name can already be seen: no second declaration, no shadowing, for names, parameters, fields and built-in names | "One name means one thing." The docs' examples already avoid it (`o`, `k`, `back`). One rule settles declaring twice, shadowing and the built-in names | 2, 7 |
| 2.8 | 11 | A name is used only below its declaration. A function body may use any top-level name of its file and its own name, and a type may use any top-level type | Nothing may run before its names exist. A body runs later and a type never runs | 8 |
| 2.8 | 12 | A name is looked for in its block, the blocks around it, the file's top level, then the built-in names. Never in another file | "You can always tell where a name comes from" (11.1) | 2 |
| 2.9 | 13 | Five kinds of statement: declaration, assignment, a value on its own, `return`, block. A `return` only inside a function | They are the forms the examples use | 5 |
| 2.9 | 14 | Any value can stand on its own as a statement | The docs already write `4;` and `makeBig(1);`. No list of allowed forms | 7 |
| 2.9 | 15 | What can be assigned: a `var` name, an element, an entry, a field, a `var` name of a package, what a `var` pointer leads to, and an ownership. What a `const` name reaches can't be changed through it | The `const` rule of 5.2, 6.2, 8.3, 8.6 and 9.3 said once | 7 |
| 2.9 | 16 | A value with no name can't be assigned, unless the place is behind a `^` | Nothing could ever read the change | 8 |
| 2.9 | 17 | An assignment is a statement, not a value | Every change starts its own line | 2 |
| 2.9 | 18 | An assignment works out its right side first, then finds its place, frees what was there, and puts the value in | A call already works out every argument before it sets a parameter (7.2). Values first, places after | 6 |
| 2.9 | 19 | Everything else is worked out from left to right | The order the code is read in | 2, 9 |

### Numbers, operators and equality

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 3.2 | 20 | Whole numbers in base ten, `0x` and `0b`. A float is `digits.digits` with an optional `e` power | The usual forms. The docs name a type `bitboard`, which needs the other bases | 9 |
| 3.2 | 21 | The escapes `\n \t \r \0 \\ \' \" \xHH`, and any other `\` is an error | The usual ones. `\xHH` reaches every value of a `char` | 9 |
| 3.2 | 22 | A `char` is one byte. A character outside ASCII doesn't fit in one | A `char` is 8 bits (3.1), and nothing changes size out of sight | 4, 12 |
| 3.2 | 23 | A written number takes its type from the nearest thing around it that has one, and the type passes through brackets, `-` and the arithmetic operators | It makes "the place it's written for" exact | 4 |
| 3.2 | 24 | A written number with no place is an `int`, or a `float` with a decimal point | The everyday names are the defaults | 3 |
| 3.2 | 25 | `/` divides and `%` gives the remainder. Whole-number `/` drops the part after the point, `%` has the sign of the left side and works on whole numbers only | Symbols, in the usual step. `convert(float, int)` already drops that part | 10, 9, 6 |
| 3.2 | 26 | `-` in front of a signed whole number or a float. In front of a written number it is part of the number, so `-128` is an `int8` | The docs already write `-7`. A minus for written numbers only would be a special case | 7 |
| 3.2 | 27 | A result always fits its type: an answer out of range, or a division by zero, fails while running. Nothing wraps | It is the rule for conversions (4.4) applied to arithmetic | 7, 2 |
| 3.2 | 28 | Floats follow IEEE 754, but an answer that is too big or isn't a number fails. There is no infinity and no "not a number" | The same rule again. A float then always equals itself | 7 |
| 3.2 | 29 | The compiler makes these checks itself when every number is written in the code | As it does for written slice bounds (5.9) | 8 |
| 3.2 | 30 | `==` and `!=` work on every type. `<` `>` `<=` `>=` work on numbers and `char`s. Comparisons don't chain | A `char` is a number underneath. Ordering anything else can be written as a function | 7, 6 |
| 3.2 | 31 | A `-` between two values subtracts, and a `-` where a value starts is the minus in front. `-` joins step 2 of the order, `/` and `%` join step 3 | The convention | 9 |
| 3.6 | 32 | A basic type is a name, or a name reached through a package (`geom.Box`). A type is written with the forms of 3.6 wherever a type is expected | "A name from another file is reached through a package" (11.1) | 7 |
| 3.6 | 33 | Two types are the same when their shape is the same. Names of parameters and fields, and `const` and `var`, are not part of a type | "A name for a type is the same type under another name" (4.1) | 7 |
| 3.7 | 34 | `==` compares exactly what `=` copies, for every type: values for a `^T`, addresses for an `&T` and an `@T`, part by part for arrays, structs and maps | One rule for every type, and a copy always equals its original | 7, 1 |
| 3.8 | 35 | The table that gives every form of value its type | The typing rules, gathered from the sections | 4 |
| 3.8 | 36 | Only a value written on the spot takes its type from its place. `[]`, `{}` and `null` need a place | It is how `5` and `[3, 10]` already work (3.2, 5.8) | 4 |
| 3.8 | 37 | A call to a function that returns nothing has no value | `void` means there is nothing to use | 4 |

### Types, generics, `convert` and `any`

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 4.2 | 38 | A type that decides the type of something is written out. A type worked out another way (`x.T`, `wholeTypes[1]`) can only be compared, stored and passed | Every value's type is known before the program runs (4.2) | 4, 8 |
| 4.2 | 39 | A type lasts as long as the program | A type never changes, so it is never an argument that ends too soon | 7 |
| 4.3 | 40 | In a function type, a name that isn't declared stands for a type the function is given: the first such name is the first `type` parameter, and so on. The name is not part of the type | Jack's `(type)([]T)T` has to stand without a value beside it | 6, 1 |
| 4.3 | 41 | The type handed in is written out whenever the function's type uses it | The result's type has to be known before the program runs | 8 |
| 4.3 | 42 | A generic function is checked once. A `T` can be used only in the ways every type allows | The type then says everything, so a generic function can be held in a variable and still be checked | 1, 8 |
| 4.3 | 43 | Underneath, a `type` value is the address of a description: size, copy, free, compare | An `any` and a generic function need exactly that | 12 |
| 4.4 | 44 | More conversions: float to float, `char` and `bool` with every whole-number type, numbers and `bool`s with text. Every other pair is an error | A `bool` and a `char` are numbers underneath (3.1). Text reuses `convert` instead of new functions | 6, 2 |
| 4.4, 4.5 | 45 | The built-ins' own types: `convert` is `(type, type)(A)B`, `wrap` is `(type)(T)any`, `unwrap` is `(type)(any)T` | They are ordinary function values, so they have a type | 1, 11 |
| 4.5 | 46 | An `any` keeps the type of its first value. Another type fails while running | Its size comes from the value inside, and a value on the stack keeps its size, like an unsized array's length | 7 |
| 4.6 | 47 | A top-level type can use any top-level type, itself included, but holds itself only through `^`, `&`, `@` or a function type | `Node` already names itself (8.14). A value needs a fixed size | 8 |

### Arrays, text and maps

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 5.1 | 48 | An array's size is a written whole number, 0 or more. `[]` is the empty array value | A size is a number (5.1), and an empty text has to exist | 7 |
| 5.2 | 49 | An index is an `int` and it is checked: by the compiler when it can see the answer, otherwise while running | The docs check a slice this way (5.9) | 7, 8 |
| 5.3 | 50 | A length that is only known while running is checked while running | The compiler can't always see both lengths | 8 |
| 5.3 | 51 | Every holder of a type with no size in it (a name, a field, an element, a heap value) takes its size from its own first value and keeps it | The rule of 5.3, said once for every holder and every such type | 7 |
| 5.3 | 52 | A fixed array becomes an unsized one through a slice: `primes[:]`, `&primes[:]` | They are different types, and the slice form already exists | 4, 6 |
| 5.5, 6.3 | 53 | A length is an `int` | `i == values^.length` then needs no conversion | 4 |
| 5.8 | 54 | A pair value has exactly two values, and a pair has no `.length` | A pair always has both values | 4 |
| 5.9 | 55 | Slice bounds are `int`s. A bound left out is 0 or the length | Like an index | 7 |
| 5.10 | 56 | Text is written `"Jack"`, which is the array value `['J', 'a', 'c', 'k']`. There is no text type and no end marker | "Text is an array of chars" (5.1). Length, indexing, slices, `==` and copying come from arrays | 6, 1 |
| 5.10 | 57 | A text is written on one line, and a character outside ASCII is held as its UTF-8 bytes | One `char` is one byte | 12 |
| 5.10 | 58 | `convert` goes between a number or a `bool` and `[]char`, with fixed forms. A text that isn't written right, or doesn't fit, fails while running | Both types are written, and it fails like every conversion that doesn't fit | 6, 2 |
| 5.11 | 59 | `join(T)` joins two unsized arrays into a new one. A list that grows is a heap owner given a new array | An array never grows, so one built-in makes a longer one. The `^` shows the heap use | 11, 2 |
| 6.1 | 60 | A map key can have any type | A lookup needs only `==`, and every type has it | 7 |
| 6.1 | 61 | The keys of a map value all differ, and `{}` is the empty map value | A key leads to one value | 8 |
| 6.2 | 62 | A lookup is checked: by the compiler when the keys are in view and the key is written, otherwise while running | The same as an index | 7, 8 |
| 6.3 | 63 | A whole new value for a map needs the same keys | The keys never change (6.3), as an unsized array keeps its length | 7 |
| 6.4 | 64 | Entries are held in the order written, and a lookup compares its key with `==` | It says what a lookup does underneath | 12 |

### Functions

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 7.1 | 65 | "Every path reaches a `return`" means the body always returns. Nothing is written after a `return` | No statement branches, so a body is one straight line | 8 |
| 7.1 | 66 | A function value is checked against its type, and takes its return type from its place | The value doesn't write the return type | 4 |
| 7.2 | 67 | A call gives one argument of the right type for each parameter. The value in front of the `(` is worked out first | The usual rule, and left to right | 4, 9 |
| 7.3 | 68 | A call that has no value can't be given back: `return hello();` is an error | There is no value to give | 4 |
| 7.7 | 69 | What the second address of a function value is, case by case. Each place leads on to the one around it | It shows that one address is always enough | 12 |
| 7.8 | 70 | One call has one set of parameters: every function a call gives back uses the same ones. **This settles the open point of 8.17** | A function borrows the variables it uses, and a kept parameter is one of them | 7, 6 |
| 7.8 | 71 | Copying a function copies its two addresses, so the copy uses the same variables and kept parameters | "Copying a function value copies the address" (7.7), and a copy of a borrower borrows the same thing | 1 |
| 7.8 | 72 | A function that a built-in gives back keeps nothing | `convert(float, int)` has nothing of a call to hold on to | 12 |

### Ownership

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 8.5 | 73 | A struct, array, pair, map or `any` that holds a borrower, an ownership or a function can't outlive what that part points at | Field by field (8.8): the rules for a whole value are the rules for its parts | 7 |
| 8.6 | 74 | A `const` ownership can only read | One rule for every `const` | 7 |
| 8.8 | 75 | What `=` copies for a function, a `type`, an `any` and a `package` | It completes the table that 3.7 builds on | 1 |
| 8.9, 8.18 | 76 | `@` works on a map entry | An entry is a holder like an element, and it can already be borrowed | 7 |
| 8.10 | 77 | `null` fits a `^T`, an `&T` and an `@T`, and nothing else. Following any of them when `null` fails | `null` is "no address", and all three hold an address (8.18) | 1 |
| 8.11 | 78 | When a kept heap value is freed: with the function that owns it, or at the end of the block the call was written in | It makes the last row of the 8.11 table exact | 12 |
| 8.13 | 79 | A result whose type can hold a pointer or a function counts like one at the call | The caller sees only the type, as with a borrower that comes back | 7 |
| 8.16 | 80 | Running out of heap at `^v` fails while running | It is the one thing `^v` can't always do | 8 |
| 8.17 | 81 | Following a borrower kept in a heap value, after its target is gone, isn't checked. What happens isn't defined | The docs leave it to the programmer. A check would run something the code doesn't show | 2 |

### Structs

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 9.2 | 82 | A struct value has one field for each type of its shape, in order, with that type written. A field's value can use the fields above it | The block is a block of declarations | 7 |
| 9.3 | 83 | A function can't change the fields of a `const` struct. A function field the compiler can't see can't be called through a `const` struct | A `const` never changes. The worst case is the docs' rule for calls (8.12) | 8 |
| 9.6 | 84 | A function read out of a struct uses the struct it was read from, and can't outlive it | The borrower rule for a function that uses names around it (7.8) | 7 |
| 9.6 | 85 | Underneath, a function in a struct keeps the struct's address, and a copy or a move re-aims it at the new struct | It is how "a copy is complete, functions included" comes out in memory | 12 |
| 9.9 | 86 | A field name is a label for a position in a struct type. The compiler gathers the labels from every struct value of that type in the program. One name can't label two positions, and one position can have several names | The type lists only types, so names can't be part of it. A struct in a parameter or on the heap has no block in view | 7, 12 |

### Repeating, files and running

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 10.4 | 87 | A program that calls deeper than the stack allows fails like any other check | One behaviour for every failure | 7 |
| 10.4 | 88 | The last call takes over the stack of the function that makes it, when nothing in the call needs that function's names | Repeating is a function calling itself, so repeating without end has to be possible | 5, 12 |
| 11.2 | 89 | `import` takes one text written out in the code, the path without `.my`, and is always called by its own name | The compiler has to know every file before the program runs | 8 |
| 11.2 | 90 | An import runs where it is written, and it runs only the file's top-level declarations, from the top down | An import is a call that makes a new value | 2 |
| 11.3 | 91 | A file's top-level names keep their own `const` and `var`. The word on the package says only whether the whole package can be replaced or moved | Jack's examples change `geom.made` and `a.count` through a `const` package | 2 |
| 11.3 | 92 | The compiler follows every package from its `import`. A name has to be in every file that can be there, with one type. Through a function it can't see, the worst case counts | It is how the checks of 11.5 and 11.6 are made | 8 |
| 11.4 | 93 | A file's names are set up from the top down. Function values written out and types are ready from the start. A top-level call can't reach any other name declared below it | A body can use any top-level name, so a call could otherwise read a name with no value yet | 8 |
| 11.4 | 94 | A top-level `main` in the file you run has to be a `const` of type `()int`. The status is from 0 to 255. With no `main` it is 0 | It completes the `main` rule. The status reuses the conversion rule | 6 |
| 11.5 | 95 | Underneath, a package value is the address of its file's table of names, then its variables. A `var` package has room for the biggest file it is given | It is how `shapes.area` works with nothing looked up while running | 12 |
| 11.6 | 96 | The `const` rules hold for a package: a `var` borrower and an ownership need a `var` package | "Like any other variable" | 7 |
| 11.7 | 97 | A failed check stops the program at once, writes `error: FILE:LINE: REASON` to the error output, and ends with status 1 | One behaviour for every check. 0 means everything went well | 7, 9 |
| 11.7 | 98 | A failure can't be caught. A program that isn't sure tests first | The tests already exist and are in the code | 6, 2 |
| 11.8 | 99 | `print: ([]char)void` writes exactly the `char`s it is given | Nothing is added out of sight | 2, 11 |
| 11.8 | 100 | `read: ()[]char` gives the next line with its `\n`, and an empty text at the end of the input | An empty result can then only mean the end | 6 |
| 11.8 | 101 | `exit: (int)void` ends the program with a status | The same status rule as for `main` | 6, 11 |
| App. A | 102 | The full grammar, with the order of operations built in, and the rules that tell the forms apart | One place for whoever writes the parser | 12 |
| App. B | 103 | Every check the compiler makes and every check made while running, with the reason each one prints | One place for whoever writes the compiler | 8 |

---

## 3. Adjusted rules

Every existing rule that changed. Each is marked **(adjusted)** in `DOCS.md`, and
`DECISIONS.md` section 13 has the same list.

| # | DOCS | Old wording | New wording | Why |
|---|---|---|---|---|
| A1 | 7.8 | "It can't outlive what was passed in… An argument handed over with `@` doesn't count." | The same, and also: "A function given back can't outlive the block its call was written in", unless it keeps only a heap value it took with `@` | Kept parameters need a place to live that isn't hidden heap use. **This is the bigger change: section 4** |
| A2 | 8.6, 8.17 | Under "What the compiler doesn't catch": "A borrower of a `const` given to a function that is held in a variable… only the type is known." | "A `const` is lent only where the compiler can see that it is only read." `fs[0](&limit)` is an error | A `const` could otherwise change. 8.12 already counts every `&` argument of such a call as one that may change, so this is that rule applied again |
| A3 | 8.17 | "Not settled yet: Two functions given back by one call that use the same `var` parameter… Whether they share the kept parameter or each keep their own." | "Nothing is left unsettled… They share it: one call has one set of parameters (7.8)." | The docs' own open point, decided by rule 70 |
| A4 | 5.9 | "`&a[:]` \| the whole array, the same as `&a`" | "the whole array. When `a` is unsized it is the same as `&a`. When `a` is fixed, `&a` is an `&[3]T` and `&a[:]` is the `&[]T`" | An `&[3]T` and an `&[]T` are different types, and a value never changes type by itself |
| A5 | 13 | "Four checks happen while the program runs:" | "These checks happen while the program runs. A check that fails stops the program (11.7), and Appendix B lists all of them" | The table had five rows already, and the filled-in rules add more |
| A6 | 3.2, 13 | The order of operations, step 2: "`!` and the marks". Step 3: "`*`" | Step 2: "`!`, `-` and the marks". Step 3: "`*` `/` `%`" | The new operators needed a place in the order. Nothing that was there moved |

Tables that only gained rows (3.2, 8.8, 8.16, 8.18 and the tables of section 13) are not
listed here: each new row is marked **(filled in)** where it stands.

---

## 4. Bigger changes

### 4.1 How far a function given back can travel (DOCS 7.8, A1)

**The gap.** 7.8 says a function given back keeps the parameters it uses, and that they "live
as long as it does". 7.7 says every function value is two addresses. The docs say nothing
goes on the heap unless the code says `^`. Nothing said where the kept parameters are, so a
compiler writer could not build `makeAdder`.

**The options.**

| Option | What it means | Against the philosophy |
|---|---|---|
| a. A hidden heap block for kept parameters | Every `makeAdder(five)` quietly allocates, and something has to free it | Heap use the code doesn't show. It breaks "explicit" at its centre |
| b. Function values that carry their kept parameters inside | A function value has a different size for each function | It undoes Jack's decision that every function value is two addresses, and with it arrays and maps of functions |
| c. Only Jack's rule: the function can't outlive the arguments | No new restriction | It can't be built. `keep = makeAdder(five);` inside a function passes the rule when `five` and `keep` are both top-level names, yet the kept `step` has nowhere to live once that function ends |
| d. Kept parameters stay on the stack in the block the call was written in, and the function can't outlive that block. A function that keeps only one heap value owns it and can go anywhere | One new restriction, and one exception that the heap makes possible | Nothing hidden. It is the borrower rule once more: what a function points at has to outlive it |

**Picked: d.** It keeps every rule Jack wrote and every example in the docs. `makeAdder`,
`makeCounter`, `steps`, `makeLookup`, `countFrom` and `newCounter` all work as written.
`newCounter` works because of the exception: the function `countFrom` gives back keeps only
the heap value it took with `@`, so it can be handed on.

**What it rejects that the old wording allowed.**

```go
const install: ()void = (){
    keep = makeAdder(five);    // the step would have to outlive this call of install
};
```

A call that hands over two heap values with `@`, or a heap value and a plain one, also gives
back a function that stays in its block.

**How to write those cases.** Put everything the function needs into one struct on the heap,
and hand it over with `@`. The `^` then shows the heap use, which is what the docs ask for.

**What Jack may want to change.** If functions given back should travel freely with any kept
parameters, the honest way is option a with a mark that shows the allocation. That would be
a new piece of syntax, close to idea 1 in `UNDECIDED.md`, so I did not go there.

---

## 5. Waiting on `UNDECIDED.md`

Nothing waits. Every gap could be filled with the rules that are decided.

Two filled-in rules would read more simply if an undecided idea were accepted. Both were
written with today's rules instead, and neither depends on the idea:

| Rule | Today | With the undecided idea |
|---|---|---|
| 4.1 above, keeping several values in a function that travels | one struct on the heap, handed to a helper with `@` | idea 1 (kept names after a `;`) would write it without the helper |
| 65 (DOCS 7.1), "every path reaches a `return`" | a body is one straight line, so the rule is "the body always returns" | idea 2 (`if` and `else`) would bring branches back, and the rule would need its branch cases again |

