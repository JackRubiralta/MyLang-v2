# Filled in: what was added to MyLang, and why

`DOCS.md` now holds every rule needed to write the compiler and the runtime. This file lists
what Claude added to get there, so Jack can review it. Every added rule is marked
**(filled in)** in `DOCS.md` and in `DECISIONS.md`, and every changed rule is marked
**(adjusted)**. A rule with no mark is Jack's. In `DOCS.md`, a mark in a heading covers the
whole section, and a mark at the start of a paragraph covers that paragraph and the examples,
lists and tables that follow it, up to the next paragraph. A mark inside a paragraph covers the
sentence it stands in and what follows, up to the next mark or the end of the paragraph. A mark
in a table covers its row or its cell, and a mark in a comment covers its line.

Nothing in `UNDECIDED.md` was decided, used or built on. Section 5 says where an undecided idea
touches a rule.

---

## 1. Summary

`DOCS.md` had the heart of the language: declarations, types, ownership, structs, choosing and
repeating. It did not yet say how the source is written, what most operators do at their edges,
what several rules mean for the compiler, or what happens when something goes wrong while the
program runs. Those gaps are filled now:

- **118 rules were filled in.** They are listed in section 2, each with its reason and the
  principle that decided it.
- **13 rules of Jack's were adjusted.** They are listed in section 3.
- **1 of those is a bigger change,** and it is the first thing to review. It is in section 4:
  what a function given back can't outlive.
- **Nothing waits on `UNDECIDED.md`** (section 5).
- **Eleven decisions are Jack's own,** not mine: a `var` can hold a type, a `const` never
  changes, a struct type has its field names, copying a function that keeps parameters is an
  error, a struct value is just a bundle of values, a field name never clashes with another
  name, a function with kept state is a borrower of its kept block, packages are undecided for
  now and a program is one file, `~` follows a pointer, a borrower unwrapped from a `const`
  `any` is a `const` borrower, and a call through a variable is checked while the program runs.
  Section 6 lists them with the rules I added so that they work.
- **Six rules of mine are accepted as they stand,** and they are no longer counted above: the
  `.my` ending, the function that keeps one borrower, how far the copy rule reaches, the helper
  case left open in 8.17, and two adjustments of mine to Jack's sentences in 9.6 and 8.13.
  Section 6 lists them too.

What was added, in short: the source text and every literal; a full grammar (Appendix A); `/`,
`%` and a minus in front; one rule for results that don't fit; equality for every type; the type
of every value; the rules for names; every form of assignment; generics checked once, with every
type known before the program runs; text as an array of `char`s with `join` and `convert`; map
keys; what stays read-only, so that a `const` never changes; what the second address of a
function is; what a call counts as doing, the exact check at a call through a variable, and
which owners the compiler can't tell apart; when the top-level names are set up,
and `main`; what a failed check does; `print`, `read` and `exit`; the last-call rule for the
stack; and one list of every check (Appendix B).

The philosophy (1.2 and 1.3) and the gap list (1.4) were written first, before any decision.

### 1.1 Where to look first

After the bigger change in section 4, these are the rules where I had to make a real choice.
The options I weighed are in their rows of `DECISIONS.md`:

| Rule | The choice |
|---|---|
| 10 (DOCS 2.8) | no shadowing at all, where most languages allow it. Fields are no part of it: by Jack's rule a field is only reached with a dot, so it never clashes with a name |
| 28, 29 (3.2) | a result that doesn't fit fails, and there are no special float values |
| 36 (3.7) | `==` works on every type and compares what `=` copies |
| 44, 46, 47 (4.3) | what the name of a type parameter can do in a function type; a generic function checked once, so `+` on a `T` is an error; and a type parameter that is never handed an ownership type `@X` itself |
| 48 (4.3) | every type is known before the program runs, so a generic function can't hand itself ever bigger types. It is the one check at a call through a variable that can't wait for the program to run, so the compiler counts the worst case there: through a variable a type parameter is handed only a type parameter as it is, or a type with none in it |
| 80 (7.8) | a function that a kept function gives back borrows the kept block of the function that made it. So `return plusFive(five);` is an error when `plusFive` is a name of this function that keeps parameters: what comes back would point at a block that stays in this frame |
| 84, 85 (8.5) | what comes in through a parameter can't be stored where it outlasts the call. That holds for a function too, also one that keeps nothing, because a function's type doesn't say what it keeps. A function the compiler sees is stored like any value. And the line that stores a borrower behind a pointer is checked: something of this call can't go into a place that outlasts the call, and a name that holds it behind a pointer can't be given back. After a call, a variable that was lent for change counts as given what the other arguments were, so a helper function can't store into it what a line of its own would be turned away for |
| 86 to 88 (8.6), A7 | a borrower that can only read stays that way, also inside another value and behind a `^`. That is why `copyLinks` now borrows its list as a `var`. An `any`, and in a generic function a `T`, counts as a value with a borrower inside it, so one read from a `const` stays in a `const`, and Jack's rule for `unwrap` is the same rule again. `wrap` then only reads, like every other built-in |
| 93, 95 to 98, 101 (8.12, 8.13) | what the compiler counts where it can't see. A call does what its function does to the variables it uses from outside itself, and a function that is handed over counts as called. A statement is still using what it has worked out, so a call later in the same statement can't free it. Two pointers that came in from outside may lead to one owner. What a call gave back can still change what it was lent. These rules turn away some code that is safe, as 8.12 already did |
| 94 (8.12) | the exact cases in which Jack's check at a call through a variable fails: the function frees or takes what a pointer in use depends on, aims a top-level pointer somewhere the caller doesn't know of, or uses a pointer the caller let go stale. And what a fail-safe note counts: everything outside the function, on top of what its own lines do. A function that came in through a parameter doesn't make a note fail safe, because its caller counts that function already |
| 97, 78 (8.12, 7.8) | how a pointer that function bodies share is checked with nothing written somewhere else read. A name of a function and a kept parameter are reached only by lines inside one function, and those are read together. A pointer in a top-level variable is protected by the top level at its own calls, the way a body protects a pointer at a call of `reset`. So renewing an owner inside a function is fine, and the call that renews it while a top-level pointer points into it is the error. To leave nothing that only a reading of the whole program could catch, a kept parameter can't borrow a variable outside its function, as a borrower that is given back can't |
| 107 (10.4) | the last call takes over the stack |

### 1.2 The philosophy, as the two files show it

These are the principles `DOCS.md` and `DECISIONS.md` show, in my own words. Every gap below
was decided against them.

| # | Principle | Where the files show it |
|---|---|---|
| 1 | **Everything is a value, under one set of rules.** A function, a type and a struct are declared, stored, passed and returned like an `int`. A type can be held in a `var` like any value | DOCS 1, 4.1, 4.2; DECISIONS 1 |
| 2 | **Explicit.** Reading the code tells you what happens: the heap only at `^`, sharing only at `&`, a change of owner only at `@`, a change of type only at `convert`, `wrap`, `unwrap` | DOCS top, 3.2, 8.7; DECISIONS 1 |
| 3 | **The default stays unmarked.** A plain `T` is owned in place and nothing says so | DOCS 8.1; DECISIONS 1, 5.1 |
| 4 | **Strict types.** A value never changes type by itself, and a type describes its whole shape: a struct type has its field names | DOCS 3.2, 8.3, 9.1; DECISIONS 1, 4.4 |
| 5 | **Declarative.** One shape for every declaration, two kinds of name, no loops, a choice is a lookup. A `const` truly never changes | DOCS 2.1, 8.6, 10; DECISIONS 3, 5.7, 7 |
| 6 | **Reuse before adding.** Generics are functions that take a type. A slice reuses `&` and `&[]T`. A function with kept state reuses the borrower rules | DECISIONS 4.2, 4.3, 6.1 |
| 7 | **One rule, no special cases.** "The same rule as for a borrower", "field by field", a function with kept state under "the ordinary borrower rules", one mark, `~`, for following every kind of pointer | DOCS 7.8, 8.2, 8.8 |
| 8 | **The compiler checks what it can.** A check while running is kept only where the answer isn't known before, and it always sits at something written in the code | DOCS 4.4, 4.5, 8.6, 8.12, 8.16 |
| 9 | **Otherwise, do what most languages do.** "Do whatever the conventions are for most programming languages" | DECISIONS 8 |
| 10 | **Symbols for operators, modifiers in front.** What makes a value goes in front, what uses one goes behind, and each symbol has one meaning: `^` is the heap, and `~` follows a pointer | DOCS 3.3, 8.2; DECISIONS 4.1, 5.2, 8 |
| 11 | **A type is never called.** What acts is a function, and built-ins are ordinary function values | DOCS 4.1, 4.4; DECISIONS 4.2 |
| 12 | **Say what happens underneath.** A `bool` is a number, a pair is an array of two, a function is two addresses with a note beside its code, a kept block sits in a frame | DOCS 3.1, 5.7, 5.8, 7.7, 8.18 |

### 1.3 How the files decided real cases

Reading the decided cases side by side shows the patterns behind them. I used these patterns
as the test for every new rule.

| Case | How it was decided | The pattern |
|---|---|---|
| Pairs | `?T` is an array of two underneath, the way a `bool` is a number. The types keep them apart | One representation underneath, and strict types on top |
| Slices | `&a[i:j]` reuses the mark `&`, the type `&[]T` and every borrowing rule | No new type and no new rule where an old one fits |
| Following a pointer | `~` behind a value follows any pointer: a heap owner, a borrower or an ownership. `^` only means the heap | One symbol, one meaning |
| Ownership | The type tells the three cases apart, and a struct is handled field by field | The type carries the meaning, and a compound value follows its parts |
| `=` | It means the same for every type: a complete copy | One meaning everywhere, even when it costs work |
| Arrays and maps | The length and the keys come from the first value and never change, because a value on the stack has a fixed size | A size that isn't in the type comes from the first value, and stays |
| Conversions | Written out with both types, and checked while running that the value fits | A value always fits its type, and the check sits where the code shows it |
| Types in a `var` | `var t: type = int;` holds a type as a value. Only a type name declares things | No value is left out of the rules for values |
| Struct types | A struct type has its field names, and a value is checked against the names and the types | A type describes its whole shape |
| Struct values | A bundle of values. A field is not a name in scope, and a function in a struct takes the struct as a parameter | Nothing reaches a value that the code doesn't name |
| Functions given back | A function value is two addresses, and its kept block sits in the frame that receives it. The function is a borrower of that block, so the borrower rules decide where it can go, and `^` keeps it longer | A new case is an old case again, with no rule of its own |
| Copying a function that keeps parameters | An error: the copy would share them with no `&` in the line. The compiler rejects what it sees, and otherwise the copy fails while running | Sharing is written with `&`, and never happens out of sight |
| A `const` | It never changes. The compiler rejects what it sees, and a call through a variable that lends a `const` is checked at the call, with a note stored beside the function's code | The compiler when it can see, the running program when it can't, and nothing left unchecked |
| Other calls through a variable | The compiler looks at a function called by its own name. A call through a variable is checked while the program runs, by the note beside the function's code, and a function written somewhere else never turns a line away | The compiler when it can see, the running program when it can't, and nothing turned away for code that the line doesn't reach |
| `&&` / `\|\|`, brackets, the order of operations | The usual ones | Convention, where the philosophy has nothing to say |

Eight working rules follow from these, and most of the filled-in rules are one of them
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
7. **By its own name, the compiler looks. Through a variable, the check is made while the
   program runs,** by the note beside the function's code: whether a `const` would change,
   whether a function that is copied keeps something, what the call does to the variables
   outside it, and whether a top-level name is set up yet. What the line and the type decide
   by themselves, such as a `&` argument that may change, is still counted as the worst case
   before the program runs, and so is the rule that every type is known before it starts. An
   index, a map key and `convert` follow the same pattern: the compiler checks them when it
   can see, and otherwise they are checked while running.
8. **A `const` never changes.** What a `const` name reaches can't be changed through that
   name, whatever it is: a number, a struct or a borrower. A `const` function is an address,
   so the name always holds the same code.

When two principles pulled different ways, I took the option that kept the language smallest
and most uniform, and the row for that rule says so.

---

### 1.4 The gaps, sorted into groups that touch each other

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
  functions, types and `any`
- which types a map key can have
- the same key twice in a map value
- comparing with `null`

#### Group D: names and scope (touches A, H and I)
- declaring a name twice in one block
- shadowing a name from an outer block, a parameter or a built-in
- the order of declarations in a block, and using a name above its declaration
- a type that names itself (`Node`), and two types that name each other
- calling a function at the top level before a name it uses is set up
- how a name is found: block, file, built-ins

#### Group E: statements and assignment (touches F and K)
- the full list of statement forms
- which expressions can stand alone
- every form of assignment: a name, an element, an entry, a field, through `~`, and `@`
- whether an assignment is a value
- what can stand on the left of `=`
- code after a `return`; `return f();` in a function that returns nothing

#### Group F: while the program runs (touches B, E, G and J)
- what a failed check does: what stops, what is printed, the exit status
- the full list of checks made while running (an index out of range is missing from the docs)
- the order in which operands, arguments and the sides of `=` are worked out
- when the top-level names are set up, and a top-level call that reaches one too early
- where the top-level names live
- the exact rule for `main`, and the exit status with and without it
- running out of stack, and repeating without end in a language with no loops
- running out of heap

#### Group G: what every program needs (touches A, B and F)
- printing
- reading input
- text: length, joining, comparing
- text to number and back
- ending the program with a status

#### Group H: structs (touches C, D and K)
- whether two struct types with the same field types and other names are one type
- whether a struct value has to keep the order of its type
- whether `counter.add(&counter)` passes the borrowing rules, and why

#### Group I: types and generics (touches C, D and J)
- where the name of a type parameter can be used in a function type, and whether it is part
  of the type
- what a generic body may do with a `T`
- which types can be handed to a function that takes a `type`
- which names of type `type` are type names, now that a `var` can hold a type
- whether every type can be known before the program runs
- how long a type lives, for a function given back that keeps one
- what a `type` value is underneath, so that a `var` can be given any other type
- where a type may be written
- a `var` holder of a type with no size in it: `any`, a `[]T`, a map
- arrays whose elements have no size in their type

#### Group J: functions (touches F, I and K)
- the type of a function value that has no place to take its type from
- two functions given back by one call that use the same `var` parameter (8.17)
- what counts as a copy of a function, and which functions keep something
- what the second address is for each kind of function
- which frame receives the kept block when the call is an argument, or stands under a `^`
- a function given back by a function that was given back itself
- whether an argument that was copied can still limit a function given back
- what the note beside a function's code has to hold

#### Group K: ownership (touches E, H and J)
- whether an ownership `@T` can be `null`
- what a `const` ownership may do
- `@` on a map entry
- a copy of a `const` borrower put in a `var`, alone or inside another value
- a read-only borrower handed to a generic function, or wrapped in an `any`
- the lifetime rules for a struct, an array or an `any` that holds a borrower or a function
- what happens when a borrower kept inside a heap value is followed after its target is gone
- two pointers handed to one function that lead to the same owner
- a pointer that several functions share, when one of them frees what it points at
- what a call through a variable can do to variables the caller can't see

#### Group L: for the compiler writer
- the type of every expression, in one place
- a complete grammar, in one place
- every check the compiler makes, and every check made while running, in one place

---

## 2. Every filled-in rule

The last column names the principle that decided the rule, by its number in the table in 1.2:
1 first-class, 2 explicit, 3 the default stays unmarked, 4 strict types, 5 declarative,
6 reuse before adding, 7 one rule and no special cases, 8 the compiler checks what it can,
9 do what most languages do, 10 symbols and marks, 11 a type is never called, 12 say what
happens underneath.

### Writing the source, names and statements

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| top | 1 | Each example is its own piece of code. It can use names that other examples declare, and it can declare a name again. An `// ERROR` line or a picture declares nothing. A line that ends without a `;` shows a value and what it is | The existing examples reuse names such as `count` and `p`, and rule 10 would otherwise make them errors | 8 |
| 2.7 | 2 | The source file is plain text in UTF-8 | Comments and text can then hold any character, and the code itself stays ASCII (rule 3), which reads the same in UTF-8 | 9 |
| 2.7 | 3 | Only ASCII outside comments and text | A `char` is 8 bits, so one character of code is one byte | 12 |
| 2.7 | 4 | Spaces, tabs and line breaks only separate things. A line break is `\n`, alone or after `\r` | Every statement already ends with `;` (2.5) | 2 |
| 2.7 | 5 | `//` is the only comment | One form does the job | 6 |
| 2.7 | 6 | A name is letters, digits and `_`, not starting with a digit. Case matters | The convention | 9 |
| 2.7 | 7 | Seven reserved words: `const var return void true false null` | Only the words the grammar itself needs | 6 |
| 2.7 | 8 | The built-in types and functions are ordinary names, already declared when a file starts | `int` is "an ordinary name" (3.1) and built-ins are ordinary values (4.4). With rule 10 they can't be declared again, so no second reserved list is needed | 1, 6 |
| 2.7 | 9 | The list of symbols, and the longest symbol is the one that is read | The convention. `&&` is already read this way (3.2) | 9 |
| 2.8 | 10 | A name can't be declared where the same name can already be seen: no second declaration, no shadowing, for names, parameters, type parameters in a function type, and built-in names. No other name in a file is spelled like one of its top-level names, wherever that one is declared | "One name means one thing." The docs' examples already avoid it (`o`, `k`, `back`). One rule settles declaring twice, shadowing and the built-in names | 2, 7 |
| 2.8 | 11 | A name is used only below its declaration, with four exceptions: a function body can use any top-level name and its own name, and a type name can be used anywhere at the top level and in its own declaration | Nothing may run before its names exist. A body runs later and a type never runs | 8 |
| 2.8 | 12 | A name is looked for in its block, the blocks around it, the file's top level, then the built-in names | "You can always tell where a name comes from" (section 11): it is declared in this file, or it is a built-in | 2 |
| 2.9 | 13 | Five kinds of statement: declaration, assignment, a value on its own, `return`, block. A `return` only inside a function | They are the forms the examples use | 5 |
| 2.9 | 14 | Any value can stand on its own as a statement | The docs already write `4;` and `makeBig(1);`. No list of allowed forms | 7 |
| 2.9 | 15 | A place is a `var` name, an element of an array or a pair, a map entry, a field, or what a pointer leads to. What a `const` name reaches can't be changed through it | The `const` rule of 5.2, 6.2, 8.3, 8.6 and 9.3 said once | 7 |
| 2.9 | 16 | A value with no name, a slice and a length are not places | A place starts at a name or goes through a pointer | 8 |
| 2.9 | 17 | An assignment is a statement, not a value | Every change starts its own line | 2 |
| 2.9 | 18 | An assignment works out its right side first, then finds its place, frees what was there, and puts the value in. A move keeps the same order | A call already works out every argument before it sets a parameter (7.2). Values first, places after | 6 |
| 2.9 | 19 | Everything else is worked out from left to right | The order the code is read in | 2, 9 |

### Numbers, operators and equality

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 3.2 | 20 | Whole numbers in base ten, `0x` and `0b`. A float is `digits.digits` with an optional `e` power | The usual forms. The docs name a type `bitboard`, which needs the other bases | 9 |
| 3.2 | 21 | A "written value" is a number, a `char`, a text, `true` or `false` in the code, also with a `-` directly in front of a number | Several rules check a line before the program runs only when its values are written. One meaning makes every compiler reject the same lines | 8 |
| 3.2 | 22 | The escapes `\n \t \r \0 \\ \' \" \xHH`, and any other `\` is an error | The usual ones. `\xHH` reaches every value of a `char` | 9 |
| 3.2 | 23 | A `char` is one byte. A character outside ASCII doesn't fit in one | A `char` is 8 bits (3.1), and nothing changes size out of sight | 4, 12 |
| 3.2 | 24 | A written number takes its type from the nearest thing around it that has one. The type passes through brackets, `-`, the arithmetic operators and the marks `^`, `&` and `@` | It makes "the place it's written for" exact. `^5` is already a `^int` where one is asked for (8.3) | 4 |
| 3.2 | 25 | A written number with no place is an `int`, or a `float` with a decimal point | The everyday names are the defaults | 3 |
| 3.2 | 26 | `/` divides and `%` gives the remainder. Whole-number `/` drops the part after the point. `%` has the sign of the left side, works on whole numbers only, and fails only for a zero on the right | Symbols, in the usual step. `convert(float, int)` already drops that part | 10, 9, 6 |
| 3.2 | 27 | `-` in front of a signed whole number or a float. Directly in front of a written number it is part of the number, so `-128` is an `int8` | The docs already write `-7`. A minus for written numbers only would be a special case | 7 |
| 3.2 | 28 | A result always fits its type: an answer out of range, or a division by zero, fails while running. Nothing wraps | It is the rule for conversions (4.4) applied to arithmetic | 7, 2 |
| 3.2 | 29 | Floats follow IEEE 754, with ties going to the even value, but an answer that is too big or isn't a number fails. There is no infinity and no "not a number". `0.0` equals `-0.0`. A written float too big for its type is an error | The same rule again. A float then always equals itself | 7 |
| 3.2 | 30 | The compiler makes these checks itself when every number is a written value, and when a number is divided by a written `0` or `0.0` | As it does for written slice bounds (5.9) | 8 |
| 3.2 | 31 | `==` and `!=` work on every type. `<` `>` `<=` `>=` work on numbers and `char`s, and they don't chain | A `char` is a number underneath. Ordering anything else can be written as a function | 7, 6 |
| 3.2 | 32 | A `-` between two values subtracts, and a `-` where a value starts is the minus in front | The convention | 9 |
| 3.2, 4.1 | 33 | In the rules for `(`, "a type" is a type written out. After `()`, a `{` that holds fields written `name: Type`, or one type, makes a function type, and anything else is a function body. A mark in front of a type written out makes a type: `^int` is never a heap value made from the type `int`. (The first sentence changes how Jack's rule for `(` is read: adjusted, A11) | `(){x: int, y: int}` and `(){ return 1; }` start alike, and `^int` has to mean one thing | 8 |
| 3.6 | 34 | The grammar of types gains its last two lines: a basic type is the name of a type, and `NUMBER` is a whole number written in the code. A type is written with the forms of 3.6 wherever a type is expected | The grammar used both words without saying what they are | 8 |
| 3.6, 9.1 | 35 | Two types are the same when their shape is the same. A struct's field names and their order are part of its shape, and a struct type names each field once. The names of parameters, `const` and `var`, and the name a function type gives a type parameter are not part of a function type. Types that name themselves are the same when no difference shows however far they are followed | "A name for a type is the same type under another name" (4.1). Fields are reached by name from outside, and a parameter's name is used only inside | 7, 4 |
| 3.7 | 36 | `==` compares exactly what `=` copies, for every type: values for a `^T`, addresses for an `&T` and an `@T`, part by part for arrays, structs and maps | One rule for every type, and a copy always equals its original | 7, 1 |
| 3.8 | 37 | The table that gives every form of value its type | The typing rules, gathered from the sections | 4 |
| 3.8 | 38 | Only a value written on the spot takes its type from its place. `[]`, `{}` and `null` need a place | It is how `5` and `[3, 10]` already work (3.2, 5.8) | 4 |
| 3.8 | 39 | A call to a function that returns nothing has no value | `void` means there is nothing to use | 4 |

### Types, generics, `convert` and `any`

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 4.2, 4.3 | 40 | A type name is a built-in type, a `const` declared with a type written out, or a `const` `type` parameter that its function's type names. Every other name of type `type` only holds a type: a `var`, a `const` whose value is worked out, a `type` parameter the function's type gives no name | The compiler has to know which names are types before the program runs. A name that can be handed a type while the program runs can't be one of them | 8, 7 |
| 4.2 | 41 | A type that decides the type of something is written out with type names. A type that is held or worked out another way (a `var`, `x.T`, `wholeTypes[1]`) can only be compared, stored and passed | Every value's type is known before the program runs (4.2) | 4, 8 |
| 4.2 | 42 | A top-level type name can be used anywhere in the file. The compiler collects those names first | A type never runs, so the order can't matter | 8 |
| 4.2, 7.7 | 43 | A type never ends too soon: a `type` value is the address of something that is part of the program. A type parameter that is a type name takes no room in a kept block, so a function that keeps nothing else has no kept block: `first(int)` is copied and stored like `double` | A type never changes, so it is never something that a function given back could outlast. The function that comes back is simply the one made for that type | 7 |
| 4.3 | 44 | The name of a type parameter in a function type can be used from its parameter to the end of that function type. Only a `type` parameter has a name, and only when something uses it. The name is not part of the type. The function value declares the parameter itself, as a `const`, and a type parameter can share one parameter list with the values of its type. A function value written where nothing asks for a type has no type to name the parameter, so there it only holds a type | It is a name like any other: used after it is declared, never spelled like a name that can be seen. A name nothing uses would give one type two spellings | 7, 6 |
| 4.3 | 45 | The type handed in is written out whenever the function's type uses it | The result's type has to be known before the program runs | 8 |
| 4.3 | 46 | A generic function is checked once. A `T` can be used only in the ways every type allows. For lifetimes it counts as possibly holding a borrower, an ownership and a function, and for the read-only rule as a value with a borrower inside it | The type then says everything, so a generic function can be held in a variable and still be checked | 1, 8 |
| 4.3, 4.5 | 47 | A type parameter that the function's type names is never handed an ownership type `@X` itself, and neither is `wrap`. A type with an ownership inside it is fine | A parameter is never of the type `@X` itself (8.13). A `T` that stood for one would hold an ownership without taking it, and a function type could no longer promise that an `@` argument is taken | 7, 8 |
| 4.3 | 48 | Every type is known before the program runs: the compiler follows each written type through the calls that hand it on. It is an error when a function is reached again, through its own calls, with a type parameter handed a type that has its earlier type strictly inside it. A call through a variable counts the worst case here, without looking at the functions of its type: it hands a type parameter only a type parameter as it is, or a type with none in it, and a call that hands a bigger type is an error when the function it calls hands a type parameter on through a variable | A type parameter that is a type name is always handed a written type, so every type can be found. Types that never end can't be known before the program runs. This is the one check at a call through a variable that can't wait for the program to run, and the worst case keeps a function written somewhere else from turning a line away | 8, 4 |
| 4.3 | 49 | Underneath, a `type` value is the address of a description that the compiler writes into the program, one for each type. Size, copying, moving, freeing, comparing and finding the pointers inside come from it. Two type values are equal when they are the same address, and a type value has one size | An `any` and a generic function need exactly that. One size is what lets a `var` be given any other type | 12 |
| 4.4 | 50 | All the pairs `convert` has: two number types; a `char` or a `bool` and a whole-number type; a number or a `bool` and text; each of these types to itself. Every other pair is an error, and so is a pair with a type parameter | A `bool` and a `char` are numbers underneath (3.1). Text reuses `convert` instead of new functions | 6, 2 |
| 4.4, 4.5, 5.11 | 51 | The built-ins' own types: `convert` is `(A: type, B: type)(A)B`, `wrap` is `(T: type)(T)any`, `unwrap` is `(T: type)(any)T`, `join` is `(T: type)([]T, []T)[]T`. `convert` reached through a variable with a pair it has no conversion for fails while running | They are ordinary function values, so they have a type. A check the compiler can't make moves to the running program | 1, 11, 8 |
| 4.5 | 52 | An `any` keeps the type of its first value, and the value inside keeps its own size. The compiler rejects another type when it sees both `wrap` calls, and otherwise the check is made while running | Its size comes from the value inside, and a value on the stack keeps its size, like an unsized array's length | 7 |
| 4.6 | 53 | A type can name itself, at the top level or inside a function, but holds itself only through `^`, `&`, `@` or a function type | `Node` already names itself (8.14). A value needs a fixed size | 8 |

### Arrays, text and maps

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 5.1 | 54 | An array's size is a written whole number, 0 or more. `[]` is the empty array value | A size is a number (5.1), and an empty text has to exist | 7 |
| 5.2 | 55 | An index is an `int` and it is checked. The compiler knows a length when it is in the type, or when the array is a name declared with an array value written out. It then checks a written index itself, and it rejects a written index below 0 whatever the length. Otherwise the check is made while running | The docs check a slice this way (5.9), and they already reject `a = [1, 2, 4, 5];` for such a name | 7, 8 |
| 5.3 | 56 | Giving an unsized array a value of another length is rejected when the compiler knows both lengths, and otherwise fails while running | The compiler can't always know both lengths | 8 |
| 5.3 | 57 | Every holder of a type with no size in it (a name, a field, an element, a map entry, a value of a pair, a heap value) takes its size from its own first value and keeps it. Elements of such an array each hold their own size | The rule of 5.3, said once for every holder and every such type | 7, 12 |
| 5.3 | 58 | A fixed array becomes an unsized one through a slice: `primes[:]`, `&primes[:]` | They are different types, and the slice form already exists | 4, 6 |
| 5.5, 6.3 | 59 | A length is an `int` | `i == values~.length` then needs no conversion | 4 |
| 5.8 | 60 | A pair value has exactly two values, a pair has no `.length`, and its two values can be assigned and borrowed like elements | A pair always has both values, and underneath it is an array of two | 4, 7 |
| 5.9 | 61 | Slice bounds are `int`s. A bound left out is 0 or the length, and the start can't be below 0. A written start below 0, or after a written end, is rejected whatever the length | Like an index | 7 |
| 5.10 | 62 | Text is written `"Jack"`, which is the array value `['J', 'a', 'c', 'k']`. There is no text type and no end marker | "Text is an array of chars" (5.1). Length, indexing, slices, `==` and copying come from arrays | 6, 1 |
| 5.10 | 63 | A text is written on one line, and a character outside ASCII is held as its UTF-8 bytes | An escape already writes a line break, and one `char` is one byte | 6, 12 |
| 5.10 | 64 | `convert` goes between a number or a `bool` and `[]char`, with fixed forms. A text that isn't written right, or doesn't fit, fails while running | Both types are written, and it fails like every conversion that doesn't fit | 6, 2 |
| 5.11 | 65 | `join(T)` joins two unsized arrays into a new one. A list that grows is a heap owner given a new array | An array never grows, so one built-in makes a longer one. The `^` shows the heap use | 11, 2 |
| 6.1 | 66 | A map key can have any type | A lookup needs only `==`, and every type has it | 7 |
| 6.1 | 67 | The keys of a map value all differ, and `{}` is the empty map value | A key leads to one value | 8 |
| 6.2 | 68 | A lookup is checked: by the compiler when the map was declared with written keys and the key is a written value, otherwise while running | The same as an index | 7, 8 |
| 6.3 | 69 | A whole new value for a map needs the same keys, and the map keeps its keys where they are | The keys never change (6.3), as an unsized array keeps its length | 7 |
| 6.4 | 70 | Entries are held in the order written, and a lookup compares its key with `==` | It says what a lookup does underneath | 12 |

### Functions

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 7.1 | 71 | "Every path reaches a `return`" means the body runs into one. Code after a `return` is accepted and never runs | No statement branches, so a body is one straight line. `return;` "stops a function early", as in most languages | 9 |
| 7.1 | 72 | A function value is checked against its type, and takes its return type from its place | The value doesn't write the return type | 4 |
| 7.2 | 73 | A call gives one argument of the right type for each parameter. The value in front of the `(` is worked out first, and an argument that is a call runs in its turn: "one argument can't change what another argument reads" is about the taking | The usual rule, and left to right | 4, 9 |
| 7.2 | 74 | "Called by its own name" is defined: a `const` declared with a function value written out; a function value called where it is written; a built-in or what it gives back; a function value that a seen function gives back; a `const` declared with any of these; and the same through a `const` struct written out. With Jack's lookup (section 6), everything else is a call through a variable | Many rules use the two phrases. The line between them has to be exact, and a user function such as `first(int)` is seen like a built-in | 8, 7 |
| 7.3 | 75 | A call that has no value can't be given back: `return hello();` is an error | There is no value to give | 4 |
| 7.7 | 76 | What the second address of a function value is, case by case: empty for a function that uses only its own parameters and names, built-ins, types, top-level names and names of a block written at the top level; the frame of a call around it that is still running; or its kept block. A top-level name needs no address, so a function that uses only top-level names keeps nothing. A frame or a kept block also holds the address of the frame or the block around it, when the function uses names from there | It shows that one address is always enough. A top-level name has one place for the whole program (rule 110), so every function knows where it is | 12 |
| 7.7 | 77 | The note beside a function's code says for every parameter whether it is a `var`, not only for a `&T`, how big the function's kept block is, and which top-level names the function uses, and it counts what the functions it calls by name do. A block with a kept parameter whose type has no size in it holds its own size. A note is fail safe when the function calls through a variable a function that didn't come to it through a parameter, or calls by name a function whose note is fail safe: on top of what its own lines do, it changes, takes and follows every variable outside it, of every type, and uses every top-level name | It is Jack's note, carrying what the checks need: a value with a borrower inside it follows the rule for a borrower (8.6), a frame has to make room for a kept block that comes back from a call through a variable, and a call through a variable at the top level has to know what it reaches (11.1). Fail safe is the worst the function could do, while what it does itself stays exact. A function that came in through a parameter is counted by the caller already (rule 96). It is stored with the code, so a function value stays two addresses | 6, 12 |
| 7.8 | 78 | A kept parameter can't borrow a local or another parameter of its own call, and neither can a borrower, an ownership or a function inside it. The compiler rejects the function value that keeps such a parameter. No line can give it a target in a variable outside the function that keeps it, a line of a function given back included. A heap value that a kept parameter owns can be borrowed | The kept block is handed back to another place, so such a borrower would point at the old one. It is the rule for a borrower that is given back (8.13), applied to what goes back inside the block, and such a borrower never comes from a variable outside the function. The block goes where nothing sees those variables, so only a reading of the whole program could keep them from being changed under it | 8 |
| 7.8 | 79 | One call has one kept block: every function that a call gives back uses the same kept parameters. **This settles the open point of 8.17** | A function borrows the variables it uses, and a kept parameter is one of them. Each function keeping its own copy would make a counter and its reader disagree | 7, 6 |
| 7.8 | 80 | A function that a kept function gives back keeps the parameters of its own call in a block of its own, and borrows the kept block of the function that made it. So it can't outlast that block, and a result that is or holds a function counts as borrowing what the called function keeps | "A function borrows the variables it uses from around it": the kept parameters of the function around it are such variables. Copying them into the new block would give two functions each a copy of one variable | 7 |
| 7.8 | 81 | Which results have a kept block: the compiler looks at a function it sees. A function written in the body that uses the parameters has one, and so does a value that holds such a function, and a result that the function was given back itself and hands on. Any other result keeps nothing, whatever its type. Through a variable, a result whose type can hold a function (a `T` included) counts as having a kept block in the frame that receives it | `wrap(int)(3)` gives back an `any` that holds no function, so no rule should hold it. Through a variable the compiler counts the worst case, as everywhere | 8 |
| 7.8 | 82 | Where the block of a new function goes: to the frame in which the call is written, also when the function goes straight on as an argument. With `^` in front of the call it is built on the heap with the function, and so are the blocks of the calls inside a value that is written under the `^` | It says what "the call that receives the new function" is in each position. A parameter only points at a block in its caller's frame, which outlasts the call | 12 |

### Ownership

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 8.5 | 83 | A value holds a borrower, an ownership or a function when it is one or has one inside, at any depth. An `any` and a `T` count as holding all three, and what lies behind a pointer doesn't count. A value can't outlive what a part it holds points at | Field by field (8.8). Behind a pointer the compiler can't follow, which is the docs' own limit (8.17) | 7 |
| 8.5 | 84 | What comes in through a parameter is only known to last as long as the call: a borrower, an ownership, a function, or a value that holds one. It can't be put in a variable that outlasts the call. That holds for a function that keeps nothing too. A function the compiler sees is stored like any value, by the rules of 7.8 | A borrower can't outlive what it borrows (8.5), and a function can't outlive what it keeps (7.8). A function's type doesn't say what it keeps | 8 |
| 8.5, 8.13 | 85 | The line that stores a borrower, an ownership or a function behind a pointer is checked. In a new heap value, or in one that a name owns, the value or the name counts as holding it from then on. Through a pointer that came in from outside, or that was read from behind a pointer, it can only be a borrower of a top-level name itself, or something that came in through a parameter of this call. It can't be a name of this call or of a function call or a block around the function, a heap value that a name owns, a pointer or a function from an outside variable, or a kept block. A pointer that a call gave back counts like what was lent to that call. After a call, a variable that was lent for change and holds a borrower, an ownership or a function counts as given what the other arguments were (8.13). A borrower of a heap value can be stored in a heap value that the same name owns | "The check is made every time a borrower is given something to borrow" (8.5). At the line that stores it both ends are in view. Without the check, `return ^&local;` would be accepted while `return &local;` is an error, and a helper could store into the caller's variable what a line of its own is turned away for | 8, 7 |
| 8.6 | 86 | A borrower that can only read stays that way: one that is made from, or read from, a place that can't be assigned, every copy of it, and the result of a call that was lent one. It goes only into a `const`, a `const` parameter or back as a result. A value with a borrower anywhere in what `=` copies follows the same rule: in a field, an element or an entry, or in a heap value it owns | Otherwise a borrower that can only read could be turned into one that changes, and a `const` would change after all. A copy carries the borrowers behind a `^` along, so they count | 8, 7 |
| 8.6 | 87 | The check at a call through a variable covers everything read-only that the call hands over: `&limit`, a read-only borrower held in a name, and a value with one inside it | It is Jack's check, for each form a read-only value can take | 7 |
| 8.6 | 88 | An `any` and a `T` count as a value with a borrower inside it: one that is read from a place that can't be assigned goes only into a `const`, a `const` parameter or back as a result. Every built-in only reads, `wrap` included, so its parameters count as `const`. The check while running goes by the value inside an `any`, and by the type that a `T` stands for | The compiler can't see inside an `any`, and a generic function is checked once, so it counts the worst case, as it does for lifetimes. Jack's rule that a borrower unwrapped from a `const` `any` is a `const` borrower is then this rule applied to `unwrap`, and a copy can't get round it. `wrap` needs nothing of its own: what it gives back from something read-only is read-only, by the rule for a result | 8, 7 |
| 8.6 | 89 | An ownership is given only by a place that can be assigned, and a `const` ownership can only read | One rule for every `const` | 7 |
| 8.8 | 90 | What `=` copies for a `type` and an `any`: the same type, and a copy of the type and the value inside. A parameter that a function returns is taken, like a local | It completes the table that 3.7 builds on. The row for a function is Jack's. A parameter ends with the call, as a local does | 1 |
| 8.9, 8.18 | 91 | `@` works on a map entry | An entry is a holder like an element, and it can already be borrowed | 7 |
| 8.10 | 92 | `null` fits a `^T`, an `&T` and an `@T`, and nothing else. Following any of them when `null` fails, and so does taking a value through a `null` ownership | `null` is "no address", and all three hold an address (8.18) | 1 |
| 8.12 | 93 | A variable is outside a function when the function's body uses it without declaring it: a top-level `var`, a name of a function or a block around it, a parameter it keeps. The compiler reads a function for the outside variables it changes or takes, the pointers among them it gives a new target, and the pointers among them it uses. A call counts as doing all of that at its line. A function that keeps a place reached by following an owner, or a pointer that followed one, counts as a pointer itself, and the arguments and the function that is called are in use for everything the call does. A call that takes `x` with `@` counts for every owner inside `x`. What the functions it calls by name do counts as done by it. (That a call below to a function that uses a pointer is a use of it widens Jack's "still used below": adjusted, A10) | "The compiler reads which outside variables a function changes" has to cover a pointer that a function gives a new target or follows, and the variables that a caller can't see | 7 |
| 8.12 | 94 | Jack's check at a call through a variable fails when the note of the function that the variable holds says that it changes or takes an owner that a pointer in use followed, gives a pointer in a top-level variable a new target that followed an owner the caller doesn't count it as having followed, or follows a pointer that may point at something gone. A function handed over that the compiler doesn't see is checked by its note too. With a fail-safe note, the call fails when a pointer in use followed any owner other than a name of the caller's own call that no function written inside the caller uses. It prints `call touches what a pointer depends on` | These are the three ways a function the compiler can't see could break what the caller counts on, so each is a row of its own. A new target is counted only for a top-level pointer, because for every other shared pointer the caller counts the worst case already (rule 97). A fail-safe function doesn't count as aiming a top-level pointer, because a call inside it that would do that fails where it is made | 8, 7 |
| 8.12 | 95 | A statement is still using what it has worked out. A pointer, a place or a function that an earlier part of a statement reached by following an owner stays in use until the statement has used it, so a call in a later part can't change that owner. A value is read when it is worked out | The parts of a statement are worked out one after another (2.9), so "later in the text is later in time" holds inside a statement too | 8, 7 |
| 8.12 | 96 | A function that is handed to a call may be called by it: the call counts as a call of the handed function too. Where the compiler doesn't see that function, the check is made while the program runs, with its note | A handed function can use names of the function that hands it over, and no other rule counts those for the function that receives it | 8 |
| 8.12 | 97 | A pointer in an outside variable is shared. It counts as made at the line of a call that gives it a new target. It is still in use when a body ends, as long as it lives on. A name of a function or a block, and a kept parameter, are reached only by lines written inside that function or block: at the first line of a body that doesn't declare one, it counts as having followed every owner that one of those lines has it follow, and so it does in the body that declares it after a call the compiler doesn't see into. A pointer in a top-level variable counts as having followed nothing at the first line of a function, and the top level checks every call it makes, `main` included. The top level of the file counts as a body here. (This widens Jack's "still used below": adjusted, A10) | Any function that reaches it may run later. A caller can't see the parameters that functions given back keep, and can call those functions in any order, so the check is made where they are written. A top-level variable can be reached from anywhere, and reading every line that aims it would let a function written somewhere else turn a line away, so the top level protects it at its own calls, the way a body protects a pointer at a call of `reset` | 8, 7 |
| 8.12 | 98 | An owner reached through a pointer that came in from outside (a parameter, a pointer read out of a value that came in, a pointer in an outside variable) counts as the same owner as every owner of its type that is reached through another such pointer, and as every outside owner of that type. An owner reached through a pointer that was read from behind a pointer counts as the same owner as every owner of its type, also one that the function names itself. An owner that a called function changes and the caller can't name counts as one reached through a pointer from outside, and for a call through a variable the note gives such owners by their type. Two elements of one array, or two entries of one map, count as one owner unless written values tell them apart, also when they are reached from one pointer. A pointer that is read out of an array, a pair or a map of pointers with an index or a key that isn't a written value counts as each pointer held there. Two owners reached from one pointer along ways that go only through owners and their fields, and differ by more than an index or a key, are two owners. Owners of different types are never the same owner | `f(&p, &p)` hands one owner to two parameters, and nothing inside the function shows it. A borrower that a line stored in a heap value can lead back to an owner that the same function names. A function that is handed over can change a name of the call it was written in, which the function that calls it can't name. Strict types keep the worst case small | 8, 4 |
| 8.13 | 99 | An argument that holds borrowers counts as lending what they borrow. An ownership inside an argument counts for the caller like a borrower inside it: the function can follow it, and a `var` parameter can change or take the variable's value through it | A borrower taken out of such an argument and given back needs an origin, and an ownership inside an array is an address like a borrower | 7 |
| 8.13 | 100 | A result that holds a pointer counts like a borrower that came back. A pointer behind a `^` in the result's type doesn't count. A borrower that is given back always comes from a parameter, never from a variable outside the function | The caller sees only the type. Behind a `^` the compiler doesn't follow, so what `copyLinks` gives back stays a value of its own | 7 |
| 8.13 | 101 | What came back can still change what was lent. An owner reached through a `var` borrower that a call gave back counts as what was lent, or any owner inside it. A call of a function that came back counts as lending the same things again | A result "comes from everything that was lent" (8.13). That has to hold for what is done through it later, not only for how long it lives | 7 |
| 8.3, 8.16 | 102 | Running out of heap fails while running: at `^v`, and where `=` copies a heap owner | It is the one thing a new heap value can't always get | 8 |
| 8.17, 8.18 | 103 | What the two cases the compiler doesn't catch do while running: a pointer or a function whose target is gone, or has gone to another owner, reaches memory that isn't defined. Every borrower, ownership or function that lies behind a pointer belongs to the first case, from the line that stored it on. A move or a `return` of a value whose borrowers point into itself belongs to the second, where the compiler can't see them, and so does a kept block that is handed back one frame. The two are also the only way left to reach a `const`. And what a link to a map or an `any` holds | The docs leave these to the programmer. Saying what happens tells a compiler writer what may be assumed | 2, 12 |

### Structs

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 9.2 | 104 | A struct value has every field of its type, in the type's order | One order for the type and the value means one layout | 7 |
| 9.6 | 105 | Why `counter.add(&counter)` is allowed. The function is worked out first, and it is an ordinary value that keeps nothing of the struct. The call uses it where it sits and doesn't copy it. `&counter` followed no owner, so nothing is in use that the call could free, and the function may even give its own field a new value while it runs. | Jack asked for the reason to be written down. What a function keeps never sits in the struct that holds it (7.7), and the code that is running is part of the program | 8 |

### Repeating, files and running

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 10.4 | 106 | A program that calls deeper than the stack allows fails like any other check, and so does a declaration that needs more stack than is left | One behaviour for every failure | 7 |
| 10.4 | 107 | The last call takes over the stack of the function that makes it, when nothing the call uses lives in that call: its names, the heap values they own, the kept blocks in its frame, the values made on the spot. An ownership that is handed over doesn't count. When the last call gives back a function that keeps parameters, its kept block goes to the frame of the caller of the function that made the last call. (That the names of such a function end before the last call changes Jack's lifetimes: adjusted, A12) | Repeating is a function calling itself, so repeating without end has to be possible | 5, 12 |
| 11.1 | 108 | The top-level lines of the file are worked out from the top down, declarations and statements together. A `const` whose value is a function value written out, a written value, or an array, pair or map value made only of written values, and a type name, are ready from the start. A top-level call can't reach any other name declared below it: the compiler checks a function it sees, and a call through a variable is checked while running, by the note, with the reason `name used before it is set up`. The call of `main` counts as one that the top level makes after its last line | A body can use any top-level name, so a call could otherwise read a name with no value yet. At a call through a variable, a check that has to know the function waits for the program to run (7.2) | 8 |
| 11.1 | 109 | A top-level `main` has to be a `const` of type `()int`. The status is from 0 to 255. With no `main` it is 0 | It completes the `main` rule. The status reuses the conversion rule | 6 |
| 11.1 | 110 | The top level of the file has a frame of its own, on the stack, below everything that runs afterwards. Every top-level `var`, every `const` that isn't ready from the start, and every kept block of a call made at the top level live there until the program ends. A name that is ready from the start is part of the program itself | A top-level variable is a value like any other, so nothing goes on the heap out of sight. One place for the whole program is why a function needs no address for a top-level name | 12 |
| 11.2 | 111 | A failed check stops the program at once, writes `error: FILE:LINE: REASON` to the error output, and ends with status 1 | One behaviour for every check. 0 means everything went well | 7, 9 |
| 11.2 | 112 | A failure can't be caught. A program that isn't sure tests first | The tests already exist and are in the code | 6, 2 |
| 11.3 | 113 | `print: ([]char)void` writes exactly the `char`s it is given, at once | Nothing is added out of sight | 2, 11 |
| 11.3 | 114 | `read: ()[]char` gives the next line with its `\n`, and an empty text at the end of the input | An empty result can then only mean the end | 6 |
| 11.3 | 115 | `exit: (int)void` ends the program with a status | The same status rule as for `main` | 6, 11 |
| 11.3 | 116 | `print` and `read` fail like any other check when the output or the input can't be used | One behaviour for every failure | 7 |
| App. A | 117 | The full grammar, with the order of operations built in, and the rules that tell the forms apart | One place for whoever writes the parser | 12 |
| App. B | 118 | Every check the compiler makes and every check made while running, with the reason each one prints | One place for whoever writes the compiler | 8 |

Left out on purpose, and so not rules: reading and writing files, program arguments, and bit
operations. The language is complete without them, and each can come later as one more
built-in function.

---

## 3. Adjusted rules

Every existing rule that changed. Each is marked **(adjusted)** in `DOCS.md`, and `DECISIONS.md`
section 13 has the same list.

| # | DOCS | Old wording | New wording | Why |
|---|---|---|---|---|
| A1 | 8.17 | The heading "…and what isn't settled", and "Not settled yet: Two functions given back by one call that use the same `var` parameter… Whether they share the kept parameter or each keep their own." | The heading is "What the compiler doesn't catch", and the section says: "Two functions given back by one call that use the same `var` parameter are not on this list either. They share the kept parameter (7.8)." | The docs' own open point, decided by rule 79 |
| A2 | 5.9 | "`&a[:]` \| the whole array, the same as `&a`" | "the whole array. When `a` is unsized it is the same as `&a`. When `a` is fixed, `&a` is an `&[3]T` and `&a[:]` is the `&[]T`" | An `&[3]T` and an `&[]T` are different types, and a value never changes type by itself |
| A3 | 5.9 | The comment "checked while running: fails if n is over 4" on `&scores[1:n]` | "fails if n is below 1 or over 4" | The slice starts at 1, and the rule above the example says a start after the end fails |
| A4 | 13 | "Four checks happen while the program runs:" | "These checks happen while the program runs. A check that fails stops the program (11.2), and Appendix B lists all of them" | The table had five rows already, and the filled-in rules add more |
| A5 | 3.2, 13 | The order of operations, step 2: "`!` and the marks". Step 3: "`*`" | Step 2: "`!`, `-` and the marks". Step 3: "`*` `/` `%`" | The new operators needed a place in the order. Nothing that was there moved |
| A6 | 5.9 | "When both are written as numbers and the length is in the type, the compiler checks them." | "When both are written values and the compiler knows the length (5.2), the compiler checks them, and a bound that is left out counts as written." | The docs already reject `a = [1, 2, 4, 5];` for an unsized `a` declared with three elements, so the compiler knows that length. Under the old wording an assignment and an index would use it, and a slice of the same array would not |
| A7 | 8.15 | "`const copyLinks: (&^Link)^Link = (const from: &^Link){`" | "`(var from: &^Link){`", with a sentence that says why | The copy's `prev` links borrow nodes of the original until `linkBack` has set them. Through a read-only borrower, that would put borrowers of a list that may be a `const` into a `var` (rule 86) |
| A8 | 5.9 | "and the array can't be given a new value or moved while the slice is used further down (8.12)" | "and an array on the heap can't be freed or moved away while a slice of it is used further down (8.12)" | 8.12 protects what an owner on the heap holds. A plain array stays where it is when it is given a new value, and a slice then sees the new elements, as `b~` sees `count = 8;` in 8.4 |
| A9 | 4.5 | "Everything that doesn't touch an `any` is checked before the program runs." | "The type of everything that doesn't touch an `any` is checked before the program runs." The checks about values are listed in Appendix B | The docs already had checks while running that touch no `any`: a `null` that is followed, a map key, a conversion that doesn't fit, a slice. The sentence is about types, and now it says so |
| A10 | 8.12 | "'Still used below' means written anywhere further down the function." | The same, with two things added: a call further down to a function that uses the pointer is a use of it, and a pointer in an outside variable is still in use when the function ends | A function can use a pointer in a variable from outside itself, and a pointer that function bodies share can be followed after the function that aimed it has ended. With the old wording alone, freeing what such a pointer points at would go unnoticed (rules 93 and 97) |
| A11 | 3.2 | "The compiler knows which values are types, because their type is `type` (4.1). So `(a)` starts a function type when `a` is a type, and a group when it isn't." | The same sentences, followed by: in these rules "a type" means a type that is written out, with type names and the forms of 3.6. A value that only holds a type, such as a `var` of type `type` or `x.T`, is an ordinary value, so `(x.T)` is a group | The type of `x.T` and of a `var` that holds a type is `type` too, and neither can be written where a type is expected. Only a type that is written out is known before the program runs (rule 33) |
| A12 | 10.4, which changes the lifetimes of 2.6 and 8.11 for one case | In 2.6, a name lives until "the end of the block it's declared in". In 8.11, a heap value is freed when "its owner ends": at "the `}` of the block the owner was declared in, or the end of the call for a parameter" | The same, with one case added in 10.4: when the last thing a function does is a call that uses nothing of that function, its names and parameters end before the call starts, and a heap value that one of them owns is freed then. The other values made on the spot in that statement are gone by then as well, which is sooner than "the end of its statement" in 2.6 | Repeating is a function calling itself, so a repeat without end must not grow the stack. The names have to end before the last call for that, which is sooner than the `}` (rule 107) |
| A13 | 7.8. Also marked at `makeLookup` in 7.8 | "It can't outlive what was passed in. At the call, the function given back counts as coming from every argument, the same as a borrower that comes back (8.13). If an argument dies before the function, the compiler raises an error. An argument handed over with `@` doesn't count…" Two examples were errors by it: `addFive = makeAdder(six);` in a block where `six` ends first, and `var addTen: (int)int = makeAdder(10);`. At `makeLookup`: "so nothing in the caller has to outlive the function" | "It can't outlive what was lent." The function given back counts as coming from everything that was lent to the call: every `&` argument, and every argument that holds a borrower, an ownership or a function. A parameter that was copied sets no limit, so `makeAdder(10)` can be kept in a variable. An argument handed over with `@` still doesn't count, but a borrower, an ownership or a function inside its value does. At `makeLookup`: "and the kept block then owns it" | The kept block holds its own copy of each parameter that was copied, so an argument that was copied can end without harm. What can still end too soon is what was lent, as for a borrower that comes back. **This is a bigger change: section 4.1** |

Several existing tables also gained rows or cells (3.2, 5.9, 8.5, 8.8, 8.9, 8.12, 8.16, 8.18 and
the tables of section 13). Each addition of mine is marked **(filled in)** or **(adjusted)**
where it stands.

---

## 4. Bigger changes

### 4.1 What a function given back can't outlive (DOCS 7.8, A13)

**The rule that changed.** 7.8 said: "It can't outlive what was passed in. At the call, the
function given back counts as coming from every argument… If an argument dies before the
function, the compiler raises an error." Two of its examples were errors by that rule:

```go
{
    var six: int = 6;
    addFive = makeAdder(six);           // was an ERROR: six dies at this block's }, before addFive
}

var addTen: (int)int = makeAdder(10);   // was an ERROR: the 10 is gone at the end of this line
```

**Why it had to be looked at.** Jack's rule for kept parameters puts them in a kept block, in
the frame of the call that receives the function, and makes the function a borrower of that
block under "the ordinary borrower rules", "with no new ones". A parameter that was copied is
then the block's own: the kept `step` holds its own 6 or 10. The argument it was copied from has
nothing left to protect.

**The options.**

| Option | What it means | Against the philosophy |
|---|---|---|
| a. Keep the sentence as it stood | `makeAdder(10)` can't be kept in a variable, and `makeAdder(six)` not beyond `six` | A rule with no reason left: the block holds its own copy, so nothing can point at a value that is gone. It would be one more rule beside the borrower rules, where Jack's decision says "no new ones" |
| b. Drop the rule | Only the kept block limits the function | A kept `&T` parameter still borrows the caller's variable, and an argument that holds a borrower, an ownership or a function still points at something of the caller. The function could outlive it |
| c. Narrow it to what was lent | The function counts as coming from every `&` argument, and from every argument that holds a borrower, an ownership or a function. A parameter that was copied sets no limit | None: it is the rule for a borrower that comes back (8.13), which the old sentence already pointed to |

**Picked: c.** Both examples above are now fine. What can go wrong is still turned away:

```go
var keep: (int)int = double;
{
    var local: int = 5;
    keep = adderOn(&local);             // ERROR: local was lent, and it ends at this }, before keep
}
```

(`adderOn` and `double` are from DOCS 7.8 and 7.6.) Storing a function with a kept block where
it outlasts the frame that holds the block is an error by Jack's own rule, as in `install`. That
rule and this one are two different limits: the block, and what was lent.

**What else changed with it.**

- The sentence at `makeLookup` said that with a value handed straight over with `@`, "nothing in
  the caller has to outlive the function". The function is tied to the frame that holds its kept
  block, so the sentence now ends "and the kept block then owns it".
- An argument handed over with `@` still "doesn't count", in Jack's words. A borrower, an
  ownership or a function inside its value does count: it points at something that was only
  lent.

**What Jack may want to change.** If the old sentence should stand as it was written (option a),
the two examples are errors again, and every `makeAdder(10)` needs a variable for the 10 that
lasts as long as the function.

---

## 5. Waiting on `UNDECIDED.md`

Nothing waits. Every gap could be filled with the rules that are decided.

Packages are undecided now, by Jack's decision (item 3 there), and a program is one file. The
rules I had filled in for packages are kept with that item, marked as in `DOCS.md`: what
`import` takes, the order in which files are set up, what makes two package types the same,
what a package value is underneath, and what one instance of each file means for the ownership
rules. None of them is in `DOCS.md` or counted in this file.

One filled-in rule would read more simply if an undecided idea were accepted. It was written
with today's rules instead, and it doesn't depend on the idea:

| Rule | Today | With the undecided idea |
|---|---|---|
| 71 (DOCS 7.1), "every path reaches a `return`" | a body is one straight line, so the rule is "the body runs into a `return`" | idea 2 (`if` and `else`) would bring branches back, and the rule would need its branch cases again |

One thing to know before deciding idea 1. Its text says that a kept name "lives as long as the
function", and that "copying the function copies its kept names". For the kept parameters that
exist today, Jack's own rules say something else: copying such a function is an error, and the
kept block ends with the frame or the heap value that holds it (section 6). I did not use the
idea to decide anything. If Jack accepts idea 1 as written, it has to be read together with
those two rules.

The text of item 3, packages, was written when a call through a variable counted every function
of its type, and when a pointer that function bodies share counted every line of the program
that aims it. In `DOCS.md` the first is now Jack's check while the program runs, and the second
reads only the function that declares the pointer (DOCS 7.2, 8.12). The item itself is
unchanged. If Jack accepts packages, those parts of its text have to be read with the rules as
they are now.

---

## 6. Jack's own decisions, and the rules he accepted

Eleven decisions are Jack's own. The rules they made carry no mark in `DOCS.md`, and each has
a row in `DECISIONS.md` in his words. They are listed here apart from my work, with the rules I
added so that each one works.

| Jack's rule | Where in DOCS | What I added for it |
|---|---|---|
| **A `var` can hold a type:** `var t: type = int;`. The variable holds the type as a value. It is not a type name, so it can't declare anything | 4.2 | Rule 40 says which names are type names. Rule 48 makes every type known before the program runs, and rule 49 makes a `type` value one address, so a `var` has room for any type. Rule 41 keeps a held type out of every place that decides a type |
| **A `const` never changes.** A call through a variable that lends a `const` is checked at the call, while the program runs, with a note beside the function's code that says which parameters are `var`. It is off the list in 8.17, and it has a row in Appendix B.9 | 8.6, 7.7, 8.17 | Rules 86 to 88 keep everything read-only from reaching a `var` where the compiler can see, so the check at the call is the only one needed. Rule 77 says what else the note holds. Rule 103 says what is left in 8.17. One adjustment follows from the rule: A7 (`copyLinks`) |
| **A struct type has its field names:** `const Point: type = {x: int, y: int};`. A value is checked against the names and the types. A function type keeps no names, except a type parameter that a later part of the type uses: `(T: type, []T)T` | 3.3, 3.6, 4.3, 9.1, 9.2 | Rule 35 makes the names and their order part of what makes two struct types the same. Rule 44 says where a type parameter's name can be used. Rule 104 says what a struct value has to hold |
| **Copying a function that keeps parameters is an error; to share it, borrow it.** `var again: ()int = next;` is an error, and `var again: &()int = &next;` is the way. A function that keeps nothing copies as normal. The compiler rejects the copy where it can see the function, and otherwise the copy fails while running, using the note beside the function's code | 7.8, with 7.2, 7.7, 8.8, 8.16, 13 and Appendix B.5 and B.9 | The details of the check, which are accepted (below), say which functions keep something, what counts as a copy, and where the compiler sees it. Rule 43 lets a function that keeps only a type name copy as normal |
| **A struct value is just a bundle of values.** Fields aren't names in scope: not for a function written in the struct, and not for the other fields. A function that works on its struct takes the struct as a parameter, as in `counter.add(&counter)`. Copying a struct copies its fields, nothing more, and a `const` struct is protected by the normal `const` rule | 9.2, 9.3, 9.6 | Rule 105 says why `counter.add(&counter)` passes the borrowing rules, as Jack asked. Rule 104 keeps the fields of a value in the type's order. Nothing else was needed: a function in a struct is an ordinary function, and the check for a `const` struct is the one of 8.6 |
| **A field name never clashes with another name:** fields are only reached with a dot | 9.2, 2.8 | Nothing. Rule 10 (no shadowing) is about names in scope, and a field is not one |
| **A function value is two addresses; its kept block lives in the frame of the call that receives it,** and a function with kept state is a borrower of that block, so the borrower rules decide where it can go. Returning it hands the block back one frame, and `^` keeps it longer: `^makeAdder(five)` puts the function on the heap, kept block included | 7.7, 7.8, with 2.6, 8.5, 8.11, 8.16, 10.4 and 13 | Rule 76 says what the second address is for each kind of function, and the accepted rule on a function that keeps one borrower (below) makes `keep = adderOn(&topStep);` work. Rule 82 says which frame receives the block when the call is an argument or stands under a `^`. Rule 79 gives one call one block, rule 80 covers a function that a kept function gives back, and rule 81 says which results have a block. Rule 78 keeps a kept parameter from borrowing a name of its own call, because the block moves. Rule 77 puts the block's size in the note. One adjustment follows from the rule, and it is a bigger change: A13, in section 4.1 |
| **Packages move to `UNDECIDED.md`, and a program is one file.** The language doesn't support packages for now: there is no `import`, no `package` type and no rule about several files. What a one-file program needs stays: where statements go, the order the file runs in, `main`, failed checks, and `print`, `read` and `exit` | 11, with 2.7, 2.8, 2.9, 3.6, 7.2, 7.7, 8.12, 13 and the appendices | Rule 108 says when the top-level names are set up, rule 109 completes `main`, and rule 110 says where the top-level names live. The rules I had filled in for packages went to `UNDECIDED.md` with the text (section 5) |
| **`~` follows a pointer, and `^` only means the heap.** `count~` is what `count` leads to, and `c~.count` reaches a field through a pointer. `^int` is still the type, and `^5` still makes one. `~` is the use-mark behind a value for following any pointer: a heap owner, a borrower or an ownership | 8.2, with 2.7, 3.2, 3.5, 8.3, 8.4, 8.7, 8.9, 8.18, 13 and Appendix A | Nothing. No rule changed: every follow is written with `~` |
| **A borrower unwrapped from a `const` `any` is a `const` borrower,** so it can only read. That follows from "a `const` never changes" | 8.6, with 4.5, 8.16, 13 and Appendix B.6 and B.9 | Rule 88 counts an `any` as a value with a borrower inside it, the same as a `T`. The read-only rule then gives Jack's rule for `unwrap`, and a copy of a `const` `any` can't get round it. `wrap` only reads, like every other built-in. The `tryInt` example in 4.5 takes a `var` parameter for it |
| **A call through a variable is checked while the program runs.** The note beside each function's code also says which outside variables it changes, takes, gives a new target to or follows, and the call reads the note of the function the variable really holds and fails if it touches something a pointer still used below depends on. A function written somewhere else never makes a line a compile error, and for a function that itself calls through a variable the note is fail safe. A lookup in a `const` array, pair or map whose functions are all written out is a call the compiler sees, and it counts the worst of those functions | 7.2, 7.7, 8.12, with 4.3, 7.8, 11.1, 8.16, 13 and Appendix B.3, B.6 and B.9 | Rule 94 gives the cases in which the call fails, and what a fail-safe note counts. Rule 77 puts the rest in the note: every parameter, the top-level names for rule 108, and what the functions called by name do. Rule 96 checks a function that is handed over the same way, and rule 98 gives owners that a caller can't name by their type. Rule 97 reads a shared pointer only inside the function that declares it, and the top level protects its own pointers at its calls. Rule 78 keeps a kept parameter from borrowing a variable outside its function, so nothing is left that only a reading of the whole program could catch. Rule 48 counts the worst case for types, the one check that can't wait for the program to run. Rule 74 keeps the list of calls the compiler sees |

Six rules of mine are accepted as they stand. Like the rules that were accepted before this
work, they carry no mark in `DOCS.md`, and their rows in `DECISIONS.md` keep the
**(filled in)** mark with no *Principle* note:

| The accepted rule | Where in DOCS |
|---|---|
| The source file's name ends in `.my` | 11 |
| A function given back that keeps only one borrower has no kept block: its second address is the address that was lent, so `adderOn(&topStep)` can go wherever a borrower of `topStep` can, while `sumOn`, which keeps two, has a block. The conditions are exact: one kept parameter and no name of a call around it, a borrower of one address, never pointed at something else | 7.7, 7.8 |
| The copy rule covers every function whose second address isn't empty, so a function that uses names of a call around it can't be copied either, and it is passed with `&`. What counts as a copy, what doesn't, and where the compiler sees one | 7.8 |
| A borrower that a helper function puts behind a `^` in what it gives back is not caught: `return boxUp(&local);` | 8.17 |
| A function in a struct value that is written inside a function can also use that function's names | 9.6 |
| An ownership inside another value is lent, not taken: only an ownership that is the whole argument goes to a parameter that asks for it | 8.13 |
