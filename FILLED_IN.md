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

- **114 rules were filled in.** They are listed in section 2, each with its reason and the
  principle that decided it.
- **10 rules of Jack's were adjusted.** They are listed in section 3.
- **2 of those are bigger changes,** and they are the first things to review. They are in
  section 4: where the parameters that a function keeps live, and which calls the compiler
  can see into.
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

After the two bigger changes in section 4, these are the rules where I had to make a real
choice. Each one has the options I weighed in its row of `DECISIONS.md`:

| Rule | The choice |
|---|---|
| 10 (DOCS 2.8) | no shadowing at all, where most languages allow it |
| 28, 29 (3.2) | a result that doesn't fit fails, and there are no special float values |
| 36 (3.7) | `==` works on every type and compares what `=` copies |
| 44, 46 (4.3) | how a function type names its type parameters, and a generic function checked once, so `+` on a `T` is an error |
| 93 to 95 (9.6) | a function in a struct gets its struct from the call, so a function field can't be read out as a value |
| 96 (9.9) | field names are labels for positions, known from the struct values a file can see |
| 98 (10.4) | the last call takes over the stack |
| 99 (11.2) | `import` is the one built-in that can't be passed around |
| 101 (11.3) | a file's names keep their own `const` and `var` under a `const` package |
| 90 (8.17) | two more cases on the list of what the compiler doesn't catch, instead of rules that would reject safe code |

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
| top | 1 | Each example is its own piece of code. It can declare a name again, and an `// ERROR` line or a picture declares nothing | The existing examples reuse names such as `count` and `p`, and rule 10 would otherwise make them errors | 8 |
| 2.7 | 2 | A source file is UTF-8, and its name ends in `.my` | `import("geom")` has to lead to exactly one file | 2 |
| 2.7 | 3 | Only ASCII outside comments and text | A `char` is 8 bits, so one character of code is one byte | 12 |
| 2.7 | 4 | Spaces, tabs and line breaks only separate things. A line break is `\n`, alone or after `\r` | Every statement already ends with `;` (2.5) | 2 |
| 2.7 | 5 | `//` is the only comment | One form does the job | 6 |
| 2.7 | 6 | A name is letters, digits and `_`, not starting with a digit. Case matters | The convention | 9 |
| 2.7 | 7 | Seven reserved words: `const var return void true false null` | Only the words the grammar itself needs | 6 |
| 2.7 | 8 | The built-in types and functions are ordinary names, already declared when a file starts | `int` is "an ordinary name" (3.1) and built-ins are ordinary values (4.4). With rule 10 they can't be declared again, so no second reserved list is needed | 1, 6 |
| 2.7 | 9 | The list of symbols, and the longest symbol is the one that is read | The convention. `&&` is already read this way (3.2) | 9 |
| 2.8 | 10 | A name can't be declared where the same name can already be seen: no second declaration, no shadowing, for names, parameters, fields and built-in names | "One name means one thing." The docs' examples already avoid it (`o`, `k`, `back`). One rule settles declaring twice, shadowing and the built-in names | 2, 7 |
| 2.8 | 11 | A name is used only below its declaration, with four exceptions: a function body (top-level names, its own name) and type names (anywhere at the top level, and in their own declaration) | Nothing may run before its names exist. A body runs later and a type never runs | 8 |
| 2.8 | 12 | A name is looked for in its block, the blocks around it, the file's top level, then the built-in names. Never in another file | "You can always tell where a name comes from" (11.1) | 2 |
| 2.9 | 13 | Five kinds of statement: declaration, assignment, a value on its own, `return`, block. A `return` only inside a function | They are the forms the examples use | 5 |
| 2.9 | 14 | Any value can stand on its own as a statement | The docs already write `4;` and `makeBig(1);`. No list of allowed forms | 7 |
| 2.9 | 15 | A place is a `var` name, an element of an array or a pair, a map entry, a field, a `var` name of a package, or what a pointer leads to. What a `const` name reaches can't be changed through it | The `const` rule of 5.2, 6.2, 8.3, 8.6 and 9.3 said once | 7 |
| 2.9 | 16 | A value with no name, a slice and a length are not places | A place starts at a name or goes through a pointer | 8 |
| 2.9 | 17 | An assignment is a statement, not a value | Every change starts its own line | 2 |
| 2.9 | 18 | An assignment works out its right side first, then finds its place, frees what was there, and puts the value in. A move keeps the same order | A call already works out every argument before it sets a parameter (7.2). Values first, places after | 6 |
| 2.9 | 19 | Everything else is worked out from left to right | The order the code is read in | 2, 9 |

### Numbers, operators and equality

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 3.2 | 20 | Whole numbers in base ten, `0x` and `0b`. A float is `digits.digits` with an optional `e` power | The usual forms. The docs name a type `bitboard`, which needs the other bases | 9 |
| 3.2 | 21 | A "written value" is a number, a `char`, a text, `true` or `false` in the code | Several rules check a line before the program runs only when its values are written. One meaning makes every compiler reject the same lines | 8 |
| 3.2 | 22 | The escapes `\n \t \r \0 \\ \' \" \xHH`, and any other `\` is an error | The usual ones. `\xHH` reaches every value of a `char` | 9 |
| 3.2 | 23 | A `char` is one byte. A character outside ASCII doesn't fit in one | A `char` is 8 bits (3.1), and nothing changes size out of sight | 4, 12 |
| 3.2 | 24 | A written number takes its type from the nearest thing around it that has one. The type passes through brackets, `-`, the arithmetic operators and the marks `^` and `&` | It makes "the place it's written for" exact. `^5` is already a `^int` where one is asked for (8.3) | 4 |
| 3.2 | 25 | A written number with no place is an `int`, or a `float` with a decimal point | The everyday names are the defaults | 3 |
| 3.2 | 26 | `/` divides and `%` gives the remainder. Whole-number `/` drops the part after the point. `%` has the sign of the left side, works on whole numbers only, and fails only for a zero on the right | Symbols, in the usual step. `convert(float, int)` already drops that part | 10, 9, 6 |
| 3.2 | 27 | `-` in front of a signed whole number or a float. Directly in front of a written number it is part of the number, so `-128` is an `int8` | The docs already write `-7`. A minus for written numbers only would be a special case | 7 |
| 3.2 | 28 | A result always fits its type: an answer out of range, or a division by zero, fails while running. Nothing wraps | It is the rule for conversions (4.4) applied to arithmetic | 7, 2 |
| 3.2 | 29 | Floats follow IEEE 754, but an answer that is too big or isn't a number fails. There is no infinity and no "not a number". `0.0` equals `-0.0` | The same rule again. A float then always equals itself | 7 |
| 3.2 | 30 | The compiler makes these checks itself when every number is a written value | As it does for written slice bounds (5.9) | 8 |
| 3.2 | 31 | `==` and `!=` work on every type. `<` `>` `<=` `>=` work on numbers and `char`s. Comparisons don't chain | A `char` is a number underneath. Ordering anything else can be written as a function | 7, 6 |
| 3.2 | 32 | A `-` between two values subtracts, and a `-` where a value starts is the minus in front | The convention | 9 |
| 3.2 | 33 | In the rules for `(`, "a type" is a type written out. After `()`, a `{` that holds types makes a function type, and anything else is a function body | `(){int, int}` and `(){ return 1; }` start alike | 8 |
| 3.6 | 34 | A basic type is a type name, or one reached through a package that can hold only one file. A type is written with the forms of 3.6 wherever a type is expected | "A name from another file is reached through a package" (11.1) | 7 |
| 3.6 | 35 | Two types are the same when their shape is the same. Names of parameters and fields, and `const` and `var`, are not part of a type. Types that name themselves are the same when no difference shows however far they are followed | "A name for a type is the same type under another name" (4.1) | 7 |
| 3.7 | 36 | `==` compares exactly what `=` copies, for every type: values for a `^T`, addresses for an `&T` and an `@T`, part by part for arrays, structs and maps | One rule for every type, and a copy always equals its original | 7, 1 |
| 3.8 | 37 | The table that gives every form of value its type | The typing rules, gathered from the sections | 4 |
| 3.8 | 38 | Only a value written on the spot takes its type from its place. `[]`, `{}` and `null` need a place | It is how `5` and `[3, 10]` already work (3.2, 5.8) | 4 |
| 3.8 | 39 | A call to a function that returns nothing has no value | `void` means there is nothing to use | 4 |

### Types, generics, `convert` and `any`

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 4.2 | 40 | A type name is a built-in type, a `const` declared with a type written out, or a `type` parameter | The compiler has to know which names are types before the program runs | 8 |
| 4.2 | 41 | A type that decides the type of something is written out with type names. A type worked out another way (`x.T`, `wholeTypes[1]`) can only be compared, stored and passed | Every value's type is known before the program runs (4.2) | 4, 8 |
| 4.2 | 42 | A top-level type name can be used anywhere in its file, and the compiler collects those names first | A type never runs, so the order can't matter | 8 |
| 4.2 | 43 | A type lasts as long as the program | A type never changes, so it is never an argument that ends too soon | 7 |
| 4.3 | 44 | In a function type, a name that isn't declared stands for a type the function is given. Each `type` parameter opens a slot, and each new name takes the first free slot to its left. A name with no slot is an error, and the name is not part of the type | Jack's `(type)([]T)T` has to stand without a value beside it | 6, 1 |
| 4.3 | 45 | The type handed in is written out whenever the function's type uses it | The result's type has to be known before the program runs | 8 |
| 4.3 | 46 | A generic function is checked once. A `T` can be used only in the ways every type allows, and for lifetimes it counts as possibly holding a borrower and a function | The type then says everything, so a generic function can be held in a variable and still be checked | 1, 8 |
| 4.3 | 47 | Underneath, a `type` value is the address of a description: size, copy, free, compare | An `any` and a generic function need exactly that | 12 |
| 4.4 | 48 | More conversions: float to float, `char` and `bool` with every whole-number type, numbers and `bool`s with text, each of these types to itself. Every other pair is an error | A `bool` and a `char` are numbers underneath (3.1). Text reuses `convert` instead of new functions | 6, 2 |
| 4.4, 4.5, 5.11 | 49 | The built-ins' own types: `convert` is `(type, type)(A)B`, `wrap` is `(type)(T)any`, `unwrap` is `(type)(any)T`, `join` is `(type)([]T, []T)[]T`. `convert` reached through a variable with a pair it has no conversion for fails while running | They are ordinary function values, so they have a type. A check the compiler can't make moves to the running program | 1, 11, 8 |
| 4.5 | 50 | An `any` keeps the type of its first value, and the value inside keeps its own size. Another type fails while running | Its size comes from the value inside, and a value on the stack keeps its size, like an unsized array's length | 7 |
| 4.6 | 51 | A type can name itself, at the top level or inside a function, but holds itself only through `^`, `&`, `@` or a function type | `Node` already names itself (8.14). A value needs a fixed size | 8 |

### Arrays, text and maps

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 5.1 | 52 | An array's size is a written whole number, 0 or more. `[]` is the empty array value | A size is a number (5.1), and an empty text has to exist | 7 |
| 5.2 | 53 | An index is an `int` and it is checked: by the compiler when the index is a written number and the length is in the type, otherwise while running | The docs check a slice this way (5.9) | 7, 8 |
| 5.3 | 54 | Giving an unsized array a value of another length is rejected when both are written out in the code, and otherwise fails while running | The compiler can't always know both lengths | 8 |
| 5.3 | 55 | Every holder of a type with no size in it (a name, a field, an element, a heap value) takes its size from its own first value and keeps it. Elements of such an array each hold their own size | The rule of 5.3, said once for every holder and every such type | 7, 12 |
| 5.3 | 56 | A fixed array becomes an unsized one through a slice: `primes[:]`, `&primes[:]` | They are different types, and the slice form already exists | 4, 6 |
| 5.5, 6.3 | 57 | A length is an `int` | `i == values^.length` then needs no conversion | 4 |
| 5.8 | 58 | A pair value has exactly two values, a pair has no `.length`, and its two values can be assigned and borrowed like elements | A pair always has both values, and underneath it is an array of two | 4, 7 |
| 5.9 | 59 | Slice bounds are `int`s. A bound left out is 0 or the length | Like an index | 7 |
| 5.10 | 60 | Text is written `"Jack"`, which is the array value `['J', 'a', 'c', 'k']`. There is no text type and no end marker | "Text is an array of chars" (5.1). Length, indexing, slices, `==` and copying come from arrays | 6, 1 |
| 5.10 | 61 | A text is written on one line, and a character outside ASCII is held as its UTF-8 bytes | An escape already writes a line break, and one `char` is one byte | 6, 12 |
| 5.10 | 62 | `convert` goes between a number or a `bool` and `[]char`, with fixed forms. A text that isn't written right, or doesn't fit, fails while running | Both types are written, and it fails like every conversion that doesn't fit | 6, 2 |
| 5.11 | 63 | `join(T)` joins two unsized arrays into a new one. A list that grows is a heap owner given a new array | An array never grows, so one built-in makes a longer one. The `^` shows the heap use | 11, 2 |
| 6.1 | 64 | A map key can have any type | A lookup needs only `==`, and every type has it | 7 |
| 6.1 | 65 | The keys of a map value all differ, and `{}` is the empty map value | A key leads to one value | 8 |
| 6.2 | 66 | A lookup is checked: by the compiler when the map was declared with written keys and the key is a written value, otherwise while running | The same as an index | 7, 8 |
| 6.3 | 67 | A whole new value for a map needs the same keys, and the map keeps its keys where they are | The keys never change (6.3), as an unsized array keeps its length | 7 |
| 6.4 | 68 | Entries are held in the order written, and a lookup compares its key with `==` | It says what a lookup does underneath | 12 |

### Functions

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 7.1 | 69 | "Every path reaches a `return`" means the body runs into one. Code after a `return` is accepted and never runs | No statement branches, so a body is one straight line. `return;` "stops a function early", as in most languages | 9 |
| 7.1 | 70 | A function value is checked against its type, and takes its return type from its place | The value doesn't write the return type | 4 |
| 7.2 | 71 | A call gives one argument of the right type for each parameter. The value in front of the `(` is worked out first | The usual rule, and left to right | 4, 9 |
| 7.2 | 72 | "Called by its own name" is defined: a `const` declared with a function value written out (also through a one-file package or a `const` struct written out), a built-in or what it gives back, and a lookup in a `const` array, pair or map whose functions are all written out. Everything else is a call through a variable | Many rules use the two phrases. The line between them has to be exact | 8, 7 |
| 7.3 | 73 | A call that has no value can't be given back: `return hello();` is an error | There is no value to give | 4 |
| 7.7 | 74 | What the second address of a function value is, case by case. Each place leads on to the one around it | It shows that one address is always enough | 12 |
| 7.8 | 75 | One call has one set of parameters: every function a call gives back uses the same ones. **This settles the open point of 8.17** | A function borrows the variables it uses, and a kept parameter is one of them | 7, 6 |
| 7.8 | 76 | Copying a function copies its two addresses, so the copy uses the same variables and kept parameters | "Copying a function value copies the address" (7.7), and a copy of a borrower borrows the same thing | 1 |
| 7.8 | 77 | Which results keep parameters: the compiler looks at a function it sees, and through a variable a result that can hold a function counts as keeping every parameter | `pick(double)` and `wrap(int)(3)` keep nothing, so no rule should hold them | 8 |

### Ownership

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 8.5 | 78 | A struct, array, pair, map, `any` or package that holds a borrower, an ownership or a function can't outlive what that part points at | Field by field (8.8): the rules for a whole value are the rules for its parts | 7 |
| 8.5 | 79 | What a parameter borrows is only known to last as long as the call, so it can't be put in a variable that outlasts the call | A borrower can't outlive what it borrows (8.5) | 8 |
| 8.6 | 80 | A `var` borrower is made only from a `var` name or another `var` borrower. A copy of a `const` borrower can only read | Otherwise a borrower that can only read could be turned into one that changes | 8 |
| 8.6 | 81 | An ownership is given only by a `var`, and a `const` ownership can only read | One rule for every `const` | 7 |
| 8.8 | 82 | What `=` copies for a function, a `type`, an `any` and a `package` | It completes the table that 3.7 builds on | 1 |
| 8.9, 8.18 | 83 | `@` works on a map entry | An entry is a holder like an element, and it can already be borrowed | 7 |
| 8.10 | 84 | `null` fits a `^T`, an `&T` and an `@T`, and nothing else. Following any of them when `null` fails | `null` is "no address", and all three hold an address (8.18) | 1 |
| 8.11 | 85 | A heap value owned by a kept parameter is freed when the call that holds the kept parameters ends | It makes the last row of the 8.11 table exact | 12 |
| 8.12 | 86 | A call counts as a use of the pointers its function uses | "Still used below" has to include a function that reads the pointer when it is called | 7 |
| 8.13 | 87 | An argument that holds borrowers counts as lending what they borrow | A borrower taken out of such an argument and given back needs an origin | 7 |
| 8.13 | 88 | A result whose type can hold a pointer counts like a borrower that came back | The caller sees only the type | 7 |
| 8.16 | 89 | Running out of heap at `^v` fails while running | It is the one thing `^v` can't always do | 8 |
| 8.17 | 90 | What the cases the compiler doesn't catch do while running: a borrower whose target is gone reads memory that isn't defined, and a `const` reached through a function the compiler can't see is simply changed. Two cases join the list: a borrower stored through a pointer, and a function in a `const` struct that came from elsewhere | The docs leave these to the programmer. Saying what happens tells a compiler writer what may be assumed | 2 |

### Structs

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 9.2 | 91 | A struct value has one field for each type of its shape, in order, with that type written. A field's value can use the fields above it | The block is a block of declarations | 7 |
| 9.3 | 92 | In a struct declared `const` with its block, the fields are `const` inside the block too, so a function there can't change them | A `const` never changes where the compiler can see it | 8 |
| 9.6 | 93 | A function field is called through its struct and can be given a new value, but it can't be read out as a value | "It uses the copy it is called through": on its own it has no fields to work on. A function that calls it shows the borrow | 2 |
| 9.6 | 94 | A function in a struct value uses the fields of its own block, when it is itself the value of a field. It can use the names around the struct, but not the fields of a struct its own is written inside | A call hands over one struct only | 12 |
| 9.6 | 95 | Underneath, a call through a struct hands the function the address of that struct. Nothing of the struct is in the function value | It is how "a copy is complete, functions included" needs no extra work | 12 |
| 9.9 | 96 | A field name is a label for a position in a struct type. A file knows the labels from struct values in the file itself and in the files it imports. A name known for two positions of one type can't be used in that file, and one position can have several names | The type lists only types, so names can't be part of it. A struct in a parameter or on the heap has no block in view | 7, 2 |

### Repeating, files and running

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 10.4 | 97 | A program that calls deeper than the stack allows fails like any other check | One behaviour for every failure | 7 |
| 10.4 | 98 | The last call takes over the stack of the function that makes it, when nothing the call uses is on that function's stack | Repeating is a function calling itself, so repeating without end has to be possible | 5, 12 |
| 11.2 | 99 | `import` takes one text written out in the code, the path without `.my`, and is always called by its own name. A text that leads to no file is an error | The compiler has to know every file before the program runs | 8 |
| 11.2 | 100 | An import runs where it is written, and it runs only the working out of the file's top-level declarations, from the top down | An import is a call that makes a new value | 2 |
| 11.3 | 101 | A file's top-level names keep their own `const` and `var`. The word on the package says only whether the whole package can be replaced or moved | Jack's examples change `geom.made` and `a.count` through a `const` package | 2 |
| 11.3 | 102 | The compiler works out which files each place can hold: a name or parameter is its own place, a pointer shares the place it points at, everything else is one shared place. A name has to be in every file of the place, with one type | It is how the checks of 11.5 and 11.6 are made, exactly enough that every compiler accepts the same programs | 8 |
| 11.4 | 103 | A file's names are set up from the top down. Function values written out and type names are ready from the start. A top-level call can't reach any other name declared below it | A body can use any top-level name, so a call could otherwise read a name with no value yet | 8 |
| 11.4 | 104 | A top-level `main` in the file you run has to be a `const` of type `()int`. The status is from 0 to 255. With no `main` it is 0 | It completes the `main` rule. The status reuses the conversion rule | 6 |
| 11.5 | 105 | Underneath, a package value is the address of its file's table of names, then everything that has to be worked out. A copy or a move re-aims the functions held in it. A `var` package has room for the biggest file its place can hold | It is how `shapes.area` works with nothing looked up while running | 12 |
| 11.6 | 106 | The `const` rules hold for a package: a `var` borrower and an ownership need a `var` package | "Like any other variable" | 7 |
| 11.7 | 107 | A failed check stops the program at once, writes `error: FILE:LINE: REASON` to the error output, and ends with status 1 | One behaviour for every check. 0 means everything went well | 7, 9 |
| 11.7 | 108 | A failure can't be caught. A program that isn't sure tests first | The tests already exist and are in the code | 6, 2 |
| 11.8 | 109 | `print: ([]char)void` writes exactly the `char`s it is given | Nothing is added out of sight | 2, 11 |
| 11.8 | 110 | `read: ()[]char` gives the next line with its `\n`, and an empty text at the end of the input | An empty result can then only mean the end | 6 |
| 11.8 | 111 | `exit: (int)void` ends the program with a status | The same status rule as for `main` | 6, 11 |
| 11.8 | 112 | `print` and `read` fail like any other check when the output or the input can't be used | One behaviour for every failure | 7 |
| App. A | 113 | The full grammar, with the order of operations built in, and the rules that tell the forms apart | One place for whoever writes the parser | 12 |
| App. B | 114 | Every check the compiler makes and every check made while running, with the reason each one prints | One place for whoever writes the compiler | 8 |

Left out on purpose, and so not rules: reading and writing files, program arguments, and bit
operations. The language is complete without them, and each can come later as one more
built-in function.

---

## 3. Adjusted rules

Every existing rule that changed. Each is marked **(adjusted)** in `DOCS.md`, and
`DECISIONS.md` section 13 has the same list.

| # | DOCS | Old wording | New wording | Why |
|---|---|---|---|---|
| A1 | 7.8 | "It can't outlive what was passed in… An argument handed over with `@` doesn't count: … nothing is left to die." Nothing said where kept parameters are | Kept parameters stay on the stack with the function call that made the call, and go along when the function is given back again. In no other way can the function leave that call. What a value handed over with `@` holds still counts. Whether a result keeps parameters at all is read off the function when the compiler sees it | Kept parameters need a place to live that isn't hidden heap use. **This is a bigger change: section 4.1** |
| A2 | 8.12, 7.2 | "For one called through a variable, an array or a map, every `&` argument counts as one that may change." "A call through a variable, an array or a map counts as changing every outside variable" | The same, except that a lookup in a `const` array, pair or map whose functions are all written out is a call the compiler sees. It counts the worst of those functions | Every choice is such a lookup, so the old wording made every function that chooses count as changing everything. **This is a bigger change: section 4.2** |
| A3 | 8.17 | The heading "…and what isn't settled", and "Not settled yet: Two functions given back by one call that use the same `var` parameter… Whether they share the kept parameter or each keep their own." | "Nothing here is left unsettled. Two functions given back by one call share the parameters they keep (7.8)." | The docs' own open point, decided by rule 75 |
| A4 | 5.9 | "`&a[:]` \| the whole array, the same as `&a`" | "the whole array. When `a` is unsized it is the same as `&a`. When `a` is fixed, `&a` is an `&[3]T` and `&a[:]` is the `&[]T`" | An `&[3]T` and an `&[]T` are different types, and a value never changes type by itself |
| A5 | 5.9 | The comment "checked while running: fails if n is over 4" on `&scores[1:n]` | "fails if n is below 1 or over 4" | The slice starts at 1, and the rule above the example says a start after the end fails |
| A6 | 13 | "Four checks happen while the program runs:" | "These checks happen while the program runs. A check that fails stops the program (11.7), and Appendix B lists all of them" | The table had five rows already, and the filled-in rules add more |
| A7 | 3.2, 13 | The order of operations, step 2: "`!` and the marks". Step 3: "`*`" | Step 2: "`!`, `-` and the marks". Step 3: "`*` `/` `%`" | The new operators needed a place in the order. Nothing that was there moved |
| A8 | 11.4 | "Nothing runs because a file was imported, so the order the program runs in never depends on imports." | "All that runs when a file is imported is the working out of those declarations' values, at the line where the `import` is written. So the order the program runs in is always the order the code shows." | A declaration's value can be a call, so importing does run something. No statement runs, and the order is still the one the code shows |
| A9 | 11.5 | "Functions and `const`s never change, so they are never really copied." | "Function values written out and types never change and need nothing worked out, so they are never copied. A `const` whose value is worked out when the file is set up is held with the variables." | A worked-out `const` can differ between two imports of one file, and it can hold a function that keeps its own package |
| A10 | 4.2 | "A type can still be held as a value: … inside an `any` (4.5). It is always held as a `const`." | The rule is checked where a type shows it: a declaration whose written type has `type` in it. A type inside an `any` or under a `T` is not covered | The compiler can't see inside an `any` or a `T`, and such a type can never declare anything |

Several existing tables also gained rows or cells (3.2, 8.5, 8.8, 8.16, 8.18 and the tables of
section 13). Each addition is marked **(filled in)** or **(adjusted)** where it stands.

---

## 4. Bigger changes

### 4.1 Where kept parameters live, and how far a function given back can go (DOCS 7.8, A1)

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
| d. Kept parameters stay on the stack of the caller, and the function can never leave that call | The simplest rule that can be built | It breaks Jack's own `newCounter`, which gives back what `countFrom` gave it |
| e. As d, but kept parameters go along when the function is given back again | One restriction: a function given back can't be *stored* where it outlasts the call it is in | Nothing hidden. A value that is given back is always taken by the caller (8.8), and the kept parameters are taken with it |

**Picked: e.** It keeps every rule Jack wrote and every example in the docs. `makeAdder`,
`makeCounter`, `steps`, `makeLookup`, `countFrom` and `newCounter` all work as written, and so
does `first(int)` given back from a function.

**What it rejects that the old wording allowed.**

```go
const install: ()void = (){
    keep = makeAdder(five);    // the step would have to outlive this call of install
};
```

`keep` is a top-level `var`. Written at the top level of the file, the same line is fine.

**What else changed with it.**

- A value handed over with `@` "doesn't count" as before, but a borrower or a function inside
  it still does. Without this, a function given back could keep a borrower of a local that
  was handed over inside a struct.
- Whether a result keeps parameters is read off the function when the compiler sees it
  (rule 77). So `pick(double)` and `wrap(int)(3)` are held by nothing.
- Kept parameters end when the call that holds them ends, and a heap value they own is freed
  then (rule 85). The docs said "when the function given back ends", which is not one moment
  once the function has been copied.

**What Jack may want to change.** If a function given back should be storable anywhere, the
honest way is option a with a mark that shows the allocation. That would be a new piece of
syntax, close to idea 1 in `UNDECIDED.md`, so I did not go there.

### 4.2 A choice by lookup is a call the compiler sees (DOCS 7.2 and 8.12, A2)

**The gap.** The docs use "called by its own name" and "called through a variable, an array
or a map" in many rules, and never say exactly where the line is. Read as written, every
`cases[n == 0](n)` is a call through an array. 8.12 then counts it as changing every outside
variable and as changing every `&` argument. Since every choice in MyLang is such a lookup,
every function that chooses would count as changing everything, and none could be called
while a pointer into a list is in use.

**The options.**

| Option | What it means | Against the philosophy |
|---|---|---|
| a. Leave the wording | Any function with a pair of cases counts as the worst case | It makes choosing by lookup, the language's only way to choose, cost what the compiler knows |
| b. Follow every function value through the whole program | The most programs accepted | A reader could no longer tell from a line what the compiler knows about it |
| c. A lookup in a `const` array, pair or map whose functions are all written out is seen. The compiler counts the worst of those functions | One more case in a short list | None: the functions are all there in the code, next to the lookup |

**Picked: c.** The list of calls the compiler sees is in 7.2, and 8.12 points to it. The docs'
own examples (`valueOr`, `length`, `sumFrom`, `calc`) are all of this kind. Every rule that
needs to know what a call does uses the same list: 7.8, 8.6, 8.12, 9.3, 11.3 and 11.4.

**What it changes.** Nothing that was accepted is rejected. Calls that the old wording treated
as unknown are now read exactly.

---

## 5. Waiting on `UNDECIDED.md`

Nothing waits. Every gap could be filled with the rules that are decided.

Two filled-in rules would read more simply if an undecided idea were accepted. Both were
written with today's rules instead, and neither depends on the idea:

| Rule | Today | With the undecided idea |
|---|---|---|
| 4.1 above, a function that has to be stored for longer than the call that made it | it is made at the top level of its file, or given back to where it is needed | idea 1 (kept names after a `;`) would be the place to say how a function keeps a value of its own |
| 69 (DOCS 7.1), "every path reaches a `return`" | a body is one straight line, so the rule is "the body runs into a `return`" | idea 2 (`if` and `else`) would bring branches back, and the rule would need its branch cases again |

