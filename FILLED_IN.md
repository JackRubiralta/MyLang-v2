# Filled in: what was added to MyLang, and why

`DOCS.md` now holds every rule needed to write the compiler and the runtime. This file lists
what Claude added to get there, so Jack can review it. Every added rule is marked
**(filled in)** in `DOCS.md` and in `DECISIONS.md`, and every changed rule is marked
**(adjusted)**. A rule with no mark is Jack's. In `DOCS.md`, a mark in a heading covers the
whole section, and a mark at the start of a paragraph covers that paragraph and the example,
list or table that follows it.

Nothing in `UNDECIDED.md` was decided, used or built on. One point there needs Jack's eye all
the same: rule 79 and A1 say how the kept parameters of today's functions behave, and idea 1
describes them the other way round. Section 5 says more.

---

## 1. Summary

`DOCS.md` had the heart of the language: declarations, types, ownership, structs, choosing,
files. It did not yet say how the source is written, what most operators do at their edges,
what several rules mean for the compiler, or what happens when something goes wrong while the
program runs. Those gaps are filled now:

- **128 rules were filled in.** They are listed in section 2, each with its reason and the
  principle that decided it.
- **16 rules of Jack's were adjusted.** They are listed in section 3.
- **2 of those are bigger changes,** and they are the first things to review. They are in
  section 4: where the parameters that a function keeps live, and which calls the compiler
  can see into, with what a call through a variable counts as doing.
- **Nothing waits on `UNDECIDED.md`** (section 5).
- **Four rules are Jack's own decisions,** not mine: a `var` can hold a type, a `const` never
  changes, a struct type has its field names, and a package is a pointer to its file. Section
  6 lists them with the rules I added so that they work.

What was added, in short: the source text and every literal; a full grammar (Appendix A);
`/`, `%` and a minus in front; one rule for results that don't fit; equality for every type;
the type of every value; the rules for names; every form of assignment; generics checked
once, with every type known before the program runs; text as an array of `char`s with `join`
and `convert`; map keys; what stays read-only, so that a `const` never changes; how a function
in a `const` struct is checked; what a call counts as doing when the compiler can't see into
it, and which owners it can't tell apart; the order in which files are set up, and `main`; what
a failed check does; `print`, `read` and `exit`; the last-call rule for the stack; and one list
of every check (Appendix B).

The philosophy (1.2 and 1.3) and the gap list (1.4) were written first, before any decision.

### 1.1 Where to look first

After the two bigger changes in section 4, these are the rules where I had to make a real
choice. Each one has the options I weighed in its row of `DECISIONS.md`:

| Rule | The choice |
|---|---|
| 10 (DOCS 2.8) | no shadowing at all, where most languages allow it. It also holds for the fields of a struct value: a block with a field `count` can't be written where another `count` can be seen, and the field's name comes from the type, so the other name has to change. One struct value inside another is the exception: the inner one doesn't see the fields of the outer one, so both can use the names their types give them (rule 103) |
| 28, 29 (3.2) | a result that doesn't fit fails, and there are no special float values |
| 36 (3.7) | `==` works on every type and compares what `=` copies |
| 44, 46, 47 (4.3) | what the name of a type parameter can do in a function type; a generic function checked once, so `+` on a `T` is an error; and a type parameter that is never handed an ownership type `@X` itself |
| 48 (4.3) | every type is known before the program runs, so a generic function can't hand itself ever bigger types |
| 79 (7.8) | copying a function shares what it keeps: `var again: ()int = next;` is not a second counter. A function value is two addresses, and the kept parameters stay with the call that holds them (section 4.1), so a copy leads to the same ones. Jack's rule for a package has the same shape: a package is an address, and a copy of it holds the same file. Still, a counter is state that the line seems to copy, and nothing in the line says `&`, so this one is Jack's to confirm. A copy with kept parameters of its own would fit "a copy is complete" better, but it needs a place for them that the code doesn't show. Idea 1 in `UNDECIDED.md` describes kept names that are copied with the function and live as long as it does, which is the opposite of this rule and of A1: section 5 says more |
| 83, 84 (8.5) | what comes in through a parameter can't be stored where it outlasts the call. That holds for a function too, also one that keeps nothing, because a function's type doesn't say what it keeps. A function the compiler sees is stored like any value. And the line that stores a borrower behind a pointer is checked: something of this call can't go into a place that outlasts the call, and a name that holds it behind a pointer can't be given back |
| 85 to 87 (8.6), A12 | a borrower that can only read stays that way, also inside another value and behind a `^`. That is why `copyLinks` now borrows its list as a `var`. In a generic function, a `T` read from a `const` stays in a `const` |
| 93 to 97, 100 (8.12, 8.13) | what the compiler counts where it can't see. A call does what its function does to the variables it uses from outside itself, and a function that is handed over counts as called. A statement is still using what it has worked out, so a call later in the same statement can't free it. A pointer that function bodies share is still in use when a body ends, so the owner it may point into can be changed only after the pointer is given another target. Two pointers that came in from outside may lead to one owner. What a call gave back can still change what it was lent. These rules turn away some code that is safe, as 8.12 already did |
| 106 (9.3) | how a function read out of a `const` struct is checked while running: the check follows the function fields it reads, so a function that only reads can call another one |
| 107 to 109 (9.6), A10 | how "the copy it is called through" works. A function of a struct belongs to the struct's type. Stored in a struct of that type it is the struct's own function and finds the struct it sits in, so copying a struct is copying its fields and nothing more, and `b.add = counter.add;` is one line of `b = counter;`. Read out, the function keeps the struct it came from. One corner is the price: a function that a line stores through a pointer stays bound to the struct it was read out of. It can't also use a name of a function call around the struct value, and the top-level names of its file stay open to it |
| 111 (10.4) | the last call takes over the stack |
| 112 (11.2), A11 | `import` is the one built-in that can't be passed around |
| 113 (11.2, 11.4) | the exact order in which the files are set up. Jack's rule says once, and before any file that imports it. I added that it all happens when the program starts, in the order in which each file's text first names its imports, and that an `import` inside a function counts whether or not it runs |
| 119 (8.12, 11.6) | what one instance of each file means for ownership: a file's variable is one owner through whichever package it is reached, and where a package can hold several files, a name counts as that name in each of them |

### 1.2 The philosophy, as the two files show it

These are the principles `DOCS.md` and `DECISIONS.md` show, in my own words. Every gap below
was decided against them.

| # | Principle | Where the files show it |
|---|---|---|
| 1 | **Everything is a value, under one set of rules.** A function, a type, a struct and a package are declared, stored, passed and returned like an `int`. A type can be held in a `var` like any value | DOCS 1, 4.1, 4.2, 11.6; DECISIONS 1 |
| 2 | **Explicit.** Reading the code tells you what happens: the heap only at `^`, sharing only at `&`, a change of owner only at `@`, a change of type only at `convert`, `wrap`, `unwrap` | DOCS top, 3.2, 8.7; DECISIONS 1 |
| 3 | **The default stays unmarked.** A plain `T` is owned in place and nothing says so | DOCS 8.1; DECISIONS 1, 5.1 |
| 4 | **Strict types.** A value never changes type by itself, and a type describes its whole shape: a struct type has its field names | DOCS 3.2, 8.3, 9.1; DECISIONS 1, 4.4 |
| 5 | **Declarative.** One shape for every declaration, two kinds of name, no loops, a choice is a lookup. A `const` truly never changes | DOCS 2.1, 8.6, 10; DECISIONS 3, 5.7, 7 |
| 6 | **Reuse before adding.** Generics are functions that take a type. A slice reuses `&` and `&[]T`. An import is an ordinary declaration | DECISIONS 4.2, 4.3, 9 |
| 7 | **One rule, no special cases.** "The same rule as for a borrower", "field by field", a package "held like a function", a `const` package that is "the same as a const function that changes a variable outside itself" | DOCS 7.8, 8.8, 11.3, 11.5 |
| 8 | **The compiler checks what it can.** A check while running is kept only where the answer isn't known before, and it always sits at something written in the code | DOCS 4.4, 4.5, 8.6, 8.12, 8.16 |
| 9 | **Otherwise, do what most languages do.** "Do whatever the conventions are for most programming languages" | DECISIONS 8 |
| 10 | **Symbols for operators, modifiers in front.** What makes a value goes in front, what uses one goes behind | DOCS 3.3, 8.2; DECISIONS 4.1, 5.2, 8 |
| 11 | **A type is never called.** What acts is a function, and built-ins are ordinary function values | DOCS 4.1, 4.4; DECISIONS 4.2 |
| 12 | **Say what happens underneath.** A `bool` is a number, a pair is an array of two, a function is an address with a note beside its code, a package is the address of its file | DOCS 3.1, 5.7, 5.8, 7.7, 8.18, 11.5 |

### 1.3 How the files decided real cases

Reading the decided cases side by side shows the patterns behind them. I used these patterns
as the test for every new rule.

| Case | How it was decided | The pattern |
|---|---|---|
| Packages | A file is reached through a value of type `package`, made by an ordinary declaration. The value is the file's address, held like a function value, and there is one instance of each file | A new thing is an old thing again: a value, a declaration, an address |
| Pairs | `?T` is an array of two underneath, the way a `bool` is a number. The types keep them apart | One representation underneath, and strict types on top |
| Slices | `&a[i:j]` reuses the mark `&`, the type `&[]T` and every borrowing rule | No new type and no new rule where an old one fits |
| A `const` package | The name always holds the same file, and the file's names follow their own `const` or `var`: "the same as a const function that changes a variable outside itself" | A new case is an old case again, so no rule gains an exception |
| Ownership | The type tells the three cases apart, and a struct is handled field by field | The type carries the meaning, and a compound value follows its parts |
| `=` | It means the same for every type: a complete copy | One meaning everywhere, even when it costs work |
| Arrays and maps | The length and the keys come from the first value and never change, because a value on the stack has a fixed size | A size that isn't in the type comes from the first value, and stays |
| Conversions | Written out with both types, and checked while running that the value fits | A value always fits its type, and the check sits where the code shows it |
| Types in a `var` | `var t: type = int;` holds a type as a value. Only a type name declares things | No value is left out of the rules for values |
| Struct types | A struct type has its field names, and a value is checked against the names and the types | A type describes its whole shape |
| A `const` | It never changes. The compiler rejects what it sees, and a call through a variable that lends a `const` is checked at the call, with a note stored beside the function's code | The compiler when it can see, the running program when it can't, and nothing left unchecked |
| Other calls through a variable | The compiler looks at a function called by its own name, and assumes the worst for one it can't see | Sound without a new notation: when in doubt, count it as the worst case |
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
7. **By its own name, the compiler looks. Through a variable, it counts the worst that a
   function of that type could do.**
   Where the worst case would turn away too much, the check is made while the program runs:
   an index, a map key, `convert`, and whether a `const` would change.
8. **A `const` never changes.** What a `const` name reaches can't be changed through that
   name, whatever it is: a number, a struct or a borrower. A `const` function and a `const`
   package are addresses, so the name always holds the same code or the same file.

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

#### Group H: structs (touches C, D and L)
- whether two struct types with the same field types and other names are one type
- whether a struct value has to keep the order of its type, and can use the fields above it
- how a function read out of a `const` struct is checked, and what "could change" means
- reading a function out of a struct: which copy it uses
- what "uses the fields of the copy it is called through" is underneath

#### Group I: packages (touches D, F, H and L)
- how the compiler checks names through a package in a `var`, a parameter or an array
- a function that takes a package, called through a variable
- what `import` takes, where it may be written, and what it runs
- which file is set up first when neither of two files imports the other
- where a file's variables live, and how a name is found through a package that can hold
  more than one file
- what one instance of each file means for the borrowing rules

#### Group J: types and generics (touches C, D and K)
- where the name of a type parameter can be used in a function type, and whether it is part
  of the type
- what a generic body may do with a `T`
- which types can be handed to a function that takes a `type`
- which names of type `type` are type names, now that a `var` can hold a type
- whether every type can be known before the program runs
- how long a type lives, for a function given back that keeps one
- what a `type` value is underneath, so that a `var` can be given any other type
- where a type may be written; a type reached through a package
- a `var` holder of a type with no size in it: `any`, a `[]T`, a map
- arrays whose elements have no size in their type

#### Group K: functions (touches F, J and L)
- the type of a function value that has no place to take its type from
- two functions given back by one call that use the same `var` parameter (8.17)
- what copying a function value does to what it keeps
- where kept parameters live, with no heap use the code doesn't show
- how far a function given back can travel
- what the note beside a function's code has to hold

#### Group L: ownership (touches E, H, I and K)
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

#### Group M: for the compiler writer
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
| 2.7 | 2 | A source file is UTF-8, and its name ends in `.my` | `import("geom")` has to lead to exactly one file | 2 |
| 2.7 | 3 | Only ASCII outside comments and text | A `char` is 8 bits, so one character of code is one byte | 12 |
| 2.7 | 4 | Spaces, tabs and line breaks only separate things. A line break is `\n`, alone or after `\r` | Every statement already ends with `;` (2.5) | 2 |
| 2.7 | 5 | `//` is the only comment | One form does the job | 6 |
| 2.7 | 6 | A name is letters, digits and `_`, not starting with a digit. Case matters | The convention | 9 |
| 2.7 | 7 | Seven reserved words: `const var return void true false null` | Only the words the grammar itself needs | 6 |
| 2.7 | 8 | The built-in types and functions are ordinary names, already declared when a file starts | `int` is "an ordinary name" (3.1) and built-ins are ordinary values (4.4). With rule 10 they can't be declared again, so no second reserved list is needed | 1, 6 |
| 2.7 | 9 | The list of symbols, and the longest symbol is the one that is read | The convention. `&&` is already read this way (3.2) | 9 |
| 2.8 | 10 | A name can't be declared where the same name can already be seen: no second declaration, no shadowing, for names, parameters, the fields of a struct value, type parameters in a function type, and built-in names. No other name in a file is spelled like one of its top-level names, wherever that one is declared | "One name means one thing." The docs' examples already avoid it (`o`, `k`, `back`). One rule settles declaring twice, shadowing and the built-in names | 2, 7 |
| 2.8 | 11 | A name is used only below its declaration, with five exceptions: a function body (top-level names, its own name), type names (anywhere at the top level, and in their own declaration), and a function in a struct value (any field of its block) | Nothing may run before its names exist. A body runs later and a type never runs | 8 |
| 2.8 | 12 | A name is looked for in its block, the blocks around it, the file's top level, then the built-in names. Never in another file | "You can always tell where a name comes from" (11.1) | 2 |
| 2.9 | 13 | Five kinds of statement: declaration, assignment, a value on its own, `return`, block. A `return` only inside a function | They are the forms the examples use | 5 |
| 2.9 | 14 | Any value can stand on its own as a statement | The docs already write `4;` and `makeBig(1);`. No list of allowed forms | 7 |
| 2.9 | 15 | A place is a `var` name, an element of an array or a pair, a map entry, a field, a `var` of a file reached through a package, or what a pointer leads to. What a `const` name reaches can't be changed through it | The `const` rule of 5.2, 6.2, 8.3, 8.6 and 9.3 said once | 7 |
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
| 3.2 | 30 | The compiler makes these checks itself when every number is a written value | As it does for written slice bounds (5.9) | 8 |
| 3.2 | 31 | `==` and `!=` work on every type. `<` `>` `<=` `>=` work on numbers and `char`s, and they don't chain | A `char` is a number underneath. Ordering anything else can be written as a function | 7, 6 |
| 3.2 | 32 | A `-` between two values subtracts, and a `-` where a value starts is the minus in front | The convention | 9 |
| 3.2, 4.1 | 33 | In the rules for `(`, "a type" is a type written out. After `()`, a `{` that holds fields written `name: Type`, or one type, makes a function type, and anything else is a function body. A mark in front of a type written out makes a type: `^int` is never a heap value made from the type `int`. (The first sentence changes how Jack's rule for `(` is read: adjusted, A16) | `(){x: int, y: int}` and `(){ return 1; }` start alike, and `^int` has to mean one thing | 8 |
| 3.6, 11.3 | 34 | A basic type is a type name, or one reached through a package that can hold only one file. Through a package that can hold several files, `g.Box` only holds a type. A type is written with the forms of 3.6 wherever a type is expected | "A name from another file is reached through a package" (11.1) | 7 |
| 3.6, 9.1 | 35 | Two types are the same when their shape is the same. A struct's field names and their order are part of its shape, and a struct type names each field once, never like a built-in name. The names of parameters, `const` and `var`, and the name a function type gives a type parameter are not part of a function type. Types that name themselves are the same when no difference shows however far they are followed | "A name for a type is the same type under another name" (4.1). Fields are reached by name from outside, and a parameter's name is used only inside | 7, 4 |
| 3.7 | 36 | `==` compares exactly what `=` copies, for every type: values for a `^T`, addresses for an `&T` and an `@T`, part by part for arrays, structs and maps | One rule for every type, and a copy always equals its original | 7, 1 |
| 3.8 | 37 | The table that gives every form of value its type | The typing rules, gathered from the sections | 4 |
| 3.8 | 38 | Only a value written on the spot takes its type from its place. `[]`, `{}` and `null` need a place | It is how `5` and `[3, 10]` already work (3.2, 5.8) | 4 |
| 3.8 | 39 | A call to a function that returns nothing has no value | `void` means there is nothing to use | 4 |

### Types, generics, `convert` and `any`

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 4.2, 4.3 | 40 | A type name is a built-in type, a `const` declared with a type written out, or a `const` `type` parameter that its function's type names. Every other name of type `type` only holds a type: a `var`, a `const` whose value is worked out, a `type` parameter the function's type gives no name | The compiler has to know which names are types before the program runs. A name that can be handed a type while the program runs can't be one of them | 8, 7 |
| 4.2 | 41 | A type that decides the type of something is written out with type names. A type that is held or worked out another way (a `var`, `x.T`, `wholeTypes[1]`) can only be compared, stored and passed | Every value's type is known before the program runs (4.2) | 4, 8 |
| 4.2 | 42 | A top-level type name can be used anywhere in its file. The compiler collects those names first, imported files before the files that import them | A type never runs, so the order can't matter | 8 |
| 4.2 | 43 | A type never ends too soon: a `type` value is the address of something that is part of the program | A type never changes, so it is never an argument that a function given back could outlive | 7 |
| 4.3 | 44 | The name of a type parameter in a function type can be used from its parameter to the end of that function type. Only a `type` parameter has a name, and only when something uses it. The name is not part of the type. The function value declares the parameter itself, as a `const`, and a type parameter can share one parameter list with the values of its type. A function value written where nothing asks for a type has no type to name the parameter, so there it only holds a type | It is a name like any other: used after it is declared, never spelled like a name that can be seen. A name nothing uses would give one type two spellings | 7, 6 |
| 4.3 | 45 | The type handed in is written out whenever the function's type uses it | The result's type has to be known before the program runs | 8 |
| 4.3 | 46 | A generic function is checked once. A `T` can be used only in the ways every type allows. For lifetimes it counts as possibly holding a borrower, an ownership and a function, and for the read-only rule as a value with a borrower inside it | The type then says everything, so a generic function can be held in a variable and still be checked | 1, 8 |
| 4.3, 4.5 | 47 | A type parameter that the function's type names is never handed an ownership type `@X` itself, and neither is `wrap`. A type with an ownership inside it is fine | A parameter is never of the type `@X` itself (8.13). A `T` that stood for one would hold an ownership without taking it, and a function type could no longer promise that an `@` argument is taken | 7, 8 |
| 4.3 | 48 | Every type is known before the program runs: the compiler follows each written type through the calls that hand it on. It is an error when a function is reached again, through its own calls, with a type parameter handed a type that has its earlier type strictly inside it | A type parameter that is a type name is always handed a written type, so every type can be found. Types that never end can't be known before the program runs | 8, 4 |
| 4.3 | 49 | Underneath, a `type` value is the address of a description that the compiler writes into the program, one for each type. Size, copying, moving, freeing, comparing and finding the pointers inside come from it. Two type values are equal when they are the same address, and a type value has one size | An `any` and a generic function need exactly that. One size is what lets a `var` be given any other type | 12 |
| 4.4 | 50 | All the pairs `convert` has: two number types; a `char` or a `bool` and a whole-number type; a number or a `bool` and text; each of these types to itself. Every other pair is an error, and so is a pair with a type parameter | A `bool` and a `char` are numbers underneath (3.1). Text reuses `convert` instead of new functions | 6, 2 |
| 4.4, 4.5, 5.11 | 51 | The built-ins' own types: `convert` is `(A: type, B: type)(A)B`, `wrap` is `(T: type)(T)any`, `unwrap` is `(T: type)(any)T`, `join` is `(T: type)([]T, []T)[]T`. `convert` reached through a variable with a pair it has no conversion for fails while running | They are ordinary function values, so they have a type. A check the compiler can't make moves to the running program | 1, 11, 8 |
| 4.5 | 52 | An `any` keeps the type of its first value, and the value inside keeps its own size. The compiler rejects another type when it sees both `wrap` calls, and otherwise the check is made while running | Its size comes from the value inside, and a value on the stack keeps its size, like an unsized array's length | 7 |
| 4.6 | 53 | A type can name itself, at the top level or inside a function, but holds itself only through `^`, `&`, `@` or a function type | `Node` already names itself (8.14). A value needs a fixed size | 8 |

### Arrays, text and maps

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 5.1 | 54 | An array's size is a written whole number, 0 or more. `[]` is the empty array value | A size is a number (5.1), and an empty text has to exist | 7 |
| 5.2 | 55 | An index is an `int` and it is checked. The compiler knows a length when it is in the type, or when the array is a name declared with an array value written out. It then checks a written index itself, and otherwise the check is made while running | The docs check a slice this way (5.9), and they already reject `a = [1, 2, 4, 5];` for such a name | 7, 8 |
| 5.3 | 56 | Giving an unsized array a value of another length is rejected when the compiler knows both lengths, and otherwise fails while running | The compiler can't always know both lengths | 8 |
| 5.3 | 57 | Every holder of a type with no size in it (a name, a field, an element, a map entry, a value of a pair, a heap value) takes its size from its own first value and keeps it. Elements of such an array each hold their own size | The rule of 5.3, said once for every holder and every such type | 7, 12 |
| 5.3 | 58 | A fixed array becomes an unsized one through a slice: `primes[:]`, `&primes[:]` | They are different types, and the slice form already exists | 4, 6 |
| 5.5, 6.3 | 59 | A length is an `int` | `i == values^.length` then needs no conversion | 4 |
| 5.8 | 60 | A pair value has exactly two values, a pair has no `.length`, and its two values can be assigned and borrowed like elements | A pair always has both values, and underneath it is an array of two | 4, 7 |
| 5.9 | 61 | Slice bounds are `int`s. A bound left out is 0 or the length, and the start can't be below 0 | Like an index | 7 |
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
| 7.2 | 74 | "Called by its own name" is defined: a `const` declared with a function value written out; a function value called where it is written; a built-in or what it gives back; a function value that a seen function gives back; a `const` declared with any of these; the same through a one-file package or a `const` struct written out; and a lookup in a `const` array, pair or map whose functions are all written out. Everything else is a call through a variable, which counts the worst that any function of that type does. Whether a `const` would change is checked while running instead | Many rules use the two phrases. The line between them has to be exact, and a user function such as `first(int)` is seen like a built-in. Counting the worst case for a `const` would turn away every call that is lent one | 8, 7 |
| 7.3 | 75 | A call that has no value can't be given back: `return hello();` is an error | There is no value to give | 4 |
| 7.7 | 76 | What the second address of a function value is, case by case: nothing, the struct it was read out of, a call's place, a call's kept parameters. A top-level name needs no address. A struct's own function holds how far its place lies from the start of its struct instead. A function of a struct belongs to the type of that struct | It shows that one address is always enough, and it makes `==` on functions exact. There is one instance of each file, so the program knows where every top-level name is. "Belongs to" is what storing a function into a struct and the check for a `const` struct need | 12 |
| 7.7 | 77 | The note beside a function's code says for every parameter whether it is a `var`, not only for a `&T`. It also says which struct type the function belongs to, whether it could change such a struct, and which function fields it reads | It is Jack's note carrying what the checks of 8.6 and 9.3 need. It is stored with the code, so a function value stays two addresses | 6, 12 |
| 7.8 | 78 | One call has one set of parameters: every function a call gives back uses the same ones. **This settles the open point of 8.17** | A function borrows the variables it uses, and a kept parameter is one of them | 7, 6 |
| 7.8 | 79 | Copying a function copies its two addresses, so the copy uses the same variables and kept parameters. A second counter comes from a second call, and state that is copied along is a struct with fields | "Copying a function value copies the address" (7.7). The kept parameters stay with the call that holds them, so a copy is a second value for the same function, as a copy of a borrower borrows the same value | 1, 12 |
| 7.8 | 80 | Which results keep parameters: the compiler looks at a function it sees, and through a variable a result that can hold a function (a `T` included) counts as keeping every parameter. (This narrows Jack's rule that a function given back comes from every argument: adjusted, A1) | `pick(double)` and `wrap(int)(3)` keep nothing, so no rule should hold them | 8 |
| 7.8 | 81 | A result that is or holds a function comes from the arguments it keeps, from the function that was called, and from what an argument came from when the result is that argument. It can be given back when all of that lasts beyond the call or goes along: a parameter, which is kept with it (this part is adjusted: A1), an outside name, kept parameters this call holds. Not a name declared in the function, and not an argument made on the spot | "A function given back keeps the parameters it uses", and "a local is never kept". What a called function gives back can reach that function's kept parameters, so they count too | 7 |

### Ownership

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 8.5 | 82 | A value holds a borrower, an ownership or a function when it is one or has one inside, at any depth. An `any` and a `T` count as holding all three, a package holds none, and what lies behind a pointer doesn't count. A value can't outlive what a part it holds points at | Field by field (8.8). Behind a pointer the compiler can't follow, which is the docs' own limit (8.17) | 7 |
| 8.5 | 83 | What comes in through a parameter is only known to last as long as the call: a borrower, an ownership, a function, or a value that holds one. It can't be put in a variable that outlasts the call. That holds for a function that keeps nothing too, and a function the compiler sees is stored like any value | A borrower can't outlive what it borrows (8.5), and a function can't outlive what it keeps (7.8) | 8 |
| 8.5 | 84 | The line that stores a borrower, an ownership or a function behind a pointer is checked. In a new heap value, or in one that a name owns, the value or the name counts as holding it from then on. Through a pointer that came in from outside, it can only be something that lasts as long as the program, or something that came in from outside itself. It can't be a name of this call or of a function call or a block around the function, a heap value that such a name owns, or kept parameters. A pointer that a call gave back counts like what was lent to that call. A borrower of a heap value can be stored in a heap value that the same name owns | "The check is made every time a borrower is given something to borrow" (8.5). At the line that stores it both ends are in view. Without the check, `return ^&local;` would be accepted while `return &local;` is an error | 8, 7 |
| 8.6 | 85 | A borrower that can only read stays that way: one that is made from, or read from, a place that can't be assigned, every copy of it, and the result of a call that was lent one. It goes only into a `const`, a `const` parameter or back as a result. A value with a borrower anywhere in what `=` copies follows the same rule: in a field, an element or an entry, or in a heap value it owns | Otherwise a borrower that can only read could be turned into one that changes, and a `const` would change after all. A copy carries the borrowers behind a `^` along, so they count | 8, 7 |
| 8.6 | 86 | The check at a call through a variable covers everything read-only that the call hands over: `&limit`, a read-only borrower held in a name, and a value with one inside it | It is Jack's check, for each form a read-only value can take | 7 |
| 8.6 | 87 | Nothing read-only goes into an `any`: the parameter of `wrap` counts as `var`. Every other built-in only reads, so its parameters count as `const`. A `T` counts as a value with a borrower inside it, so in a generic function a `T` read from a `const` stays in a `const`. The check while running goes by the type the `T` really stands for. A borrower unwrapped from a `const` `any` can still change the `var` it borrows | The compiler can't follow a borrower inside an `any`, and `wrap` is the one way in, so every `any` can be copied freely. The `any` itself never changes. A generic function is checked once, so for a `T` the compiler counts the worst case, as it does for lifetimes | 8, 7 |
| 8.6 | 88 | An ownership is given only by a place that can be assigned, and a `const` ownership can only read | One rule for every `const` | 7 |
| 8.8 | 89 | What `=` copies for a function, a `type`, an `any` and a `package`. For a package it is the address of its file | It completes the table that 3.7 builds on | 1 |
| 8.9, 8.18 | 90 | `@` works on a map entry | An entry is a holder like an element, and it can already be borrowed | 7 |
| 8.10 | 91 | `null` fits a `^T`, an `&T` and an `@T`, and nothing else. Following any of them when `null` fails, and so does taking a value through a `null` ownership | `null` is "no address", and all three hold an address (8.18) | 1 |
| 8.11 | 92 | A kept parameter lives until the call that holds it ends, and a heap value it owns is freed then. (The table rows in 2.6 and 8.11 are adjusted: A1) | It makes the last row of the 8.11 table exact | 12 |
| 8.12 | 93 | A variable is outside a function when the function's body uses it without declaring it: a top-level `var` of any file, a name of a function or a block around it, a parameter it keeps, a field of its struct. The compiler reads a function for the outside variables it changes or takes, the pointers among them it gives a new target, and the pointers among them it uses. A call counts as doing all of that at its line. A function that keeps a place reached by following an owner, or a pointer that followed one, counts as a pointer itself, and the arguments and the function that is called are in use for everything the call does. A call that takes `x` with `@` counts for every owner inside `x`. (That a call through a variable counts only the functions of its type is adjusted: A2. That a call below to a function that uses a pointer is a use of it widens Jack's "still used below": adjusted, A15) | "The compiler reads which outside variables a function changes" has to cover a pointer that a function re-aims or follows, and the variables that a caller can't see | 7 |
| 8.12 | 94 | A statement is still using what it has worked out. A pointer, a place or a function that an earlier part of a statement reached by following an owner stays in use until the statement has used it, so a call in a later part can't change that owner. A value is read when it is worked out | The parts of a statement are worked out one after another (2.9), so "later in the text is later in time" holds inside a statement too | 8, 7 |
| 8.12 | 95 | A function that is handed to a call may be called by it: the call counts as a call of the handed function too | A handed function can use names of the function that hands it over, and no other rule counts those for the function that receives it | 8 |
| 8.12 | 96 | A pointer in an outside variable is shared. It counts as made at the line of a call that gives it a new target. It is still in use when a body ends, as long as it lives on, and at the first line of a body that doesn't declare it, it counts as having followed every owner that any line of the program has it follow. The top level of a file counts as a body here. (This widens Jack's "still used below": adjusted, A15) | Any function that reaches it may run later. A caller can't see the parameters that functions given back keep, and can call those functions in any order, so the check is made where they are written | 8, 7 |
| 8.12 | 97 | An owner reached through a pointer that came in from outside (a parameter, a pointer read out of a value that came in or from behind a pointer, a pointer in an outside variable) counts as the same owner as every owner of its type that is reached through another such pointer, and as every outside owner of that type. An owner that a called function changes and the caller can't name counts as one reached through a pointer from outside. Two elements of one array, or two entries of one map, count as one owner unless written values tell them apart, also when they are reached from one pointer. Owners of different types are never the same owner | `f(&p, &p)` hands one owner to two parameters, and nothing inside the function shows it. A function that is handed over can change a name of the call it was written in, which the function that calls it can't name. Strict types keep the worst case small | 8, 4 |
| 8.13 | 98 | An argument that holds borrowers counts as lending what they borrow. An ownership inside another value is lent too, not taken: only an ownership that is the whole argument goes to a parameter that asks for it | A borrower taken out of such an argument and given back needs an origin, and an ownership inside an array is an address like a borrower | 7 |
| 8.13 | 99 | A result that holds a pointer counts like a borrower that came back. A pointer behind a `^` in the result's type doesn't count. A borrower that is given back always comes from a parameter, never from a variable outside the function | The caller sees only the type. Behind a `^` the compiler doesn't follow, so what `copyLinks` gives back stays a value of its own | 7 |
| 8.13 | 100 | What came back can still change what was lent. An owner reached through a `var` borrower that a call gave back counts as what was lent, or any owner inside it. A call of a function that came back counts as lending the same things again, and a function read out of a struct counts as lent that struct | A result "comes from everything that was lent" (8.13). That has to hold for what is done through it later, not only for how long it lives | 7 |
| 8.3, 8.16 | 101 | Running out of heap fails while running: at `^v`, and where `=` copies a heap owner | It is the one thing a new heap value can't always get | 8 |
| 8.17, 8.18 | 102 | What the two cases the compiler doesn't catch do while running: a pointer or a function whose target is gone, or has gone to another owner, reaches memory that isn't defined. Every borrower, ownership or function that lies behind a pointer belongs to the first case, from the line that stored it on. A move or a `return` of a value whose borrowers point into itself belongs to the second, where the compiler can't see them. The two are also the only way left to reach a `const`. And what a link to a map or an `any` holds | The docs leave these to the programmer. Saying what happens tells a compiler writer what may be assumed | 2, 12 |

### Structs

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 9.2 | 103 | A struct value has every field of its type, in the type's order. A field's value can use the fields above it. Inside the block a field is a plain name, so it can't be spelled like a name that can already be seen there, or like a top-level name of the file. A struct value written inside another one doesn't see the fields of the block around it, so the two can use the same field names | The block is a block of declarations, and one order means one layout. Two struct values take their field names from their types, so neither name could give way | 7 |
| 9.2 | 104 | A call in a field's value can run only when every field its function uses is written above the call | A function in a struct value can use a field below it, so a call could otherwise read a field with no value yet. It is the rule for a file's top level (11.4) | 7, 8 |
| 9.3 | 105 | In a struct declared `const` with its block, the fields are `const` inside the block too, so the compiler rejects a function there that changes them | A `const` never changes, and here the compiler sees the function | 8 |
| 9.3 | 106 | The check while running for a function read out of a struct through a `const` name or a read-only borrower: it fails when the function works on a struct that the place it is read from lies in, and could change that struct. A struct's own function works on the struct it sits in, and a function that was read out works on the struct it keeps. "Could change" means it changes something the fields reach, also through a `^`, an `&` or an `@` held there, or reads a field with a function in it that could. A function that works on another struct, or on none, passes. Calling and reading out are checked alike, for a function read out by itself and for each function inside a value that is read out | It is Jack's "same kind of check", made exact: "could change" is everything a `const` name would not allow. Following the fields a function reads keeps `show` usable when it calls `area`. A function that works on another struct can't change this one | 8, 7 |
| 9.6 | 107 | A function that uses the fields of its struct is a function of the struct, and it belongs to the struct's type. Stored in a struct of that type it is the struct's own function, and it uses the fields of the struct it sits in. It is stored that way by the block that writes it, by every copy of its struct, and by a line that puts it into a place which the line reaches from such a struct with `.` and `[ ]`. Underneath it holds how far its place lies from the start of its struct, so nothing is done to it when the struct is copied, moved or given back. Every other function is copied with what it keeps, also one that a line stored through a pointer | "A function inside a struct uses the fields of the copy it is called through" (9.6), and a struct is copied field by field (8.8). An own function that finds the struct it sits in keeps both: `b.add = counter.add;` is one line of `b = counter;` | 7, 12 |
| 9.6 | 108 | Such a function is written as a field's value, or inside an array, pair or map in a field. It can use every field of its block, above or below, and the top-level names of its file, but no name of a function call around the struct value (both limits are adjusted: A10). A function in a struct value that uses no field is an ordinary function. A function written inside the body of a function that uses the fields reaches them through it. Inside its own block, a function of the struct can be called by its name or copied into another field, and it can't be read out to anywhere else | A function value has one address for what it keeps, and a function of a struct needs it for the struct. A top-level name needs no address. A block has no holder yet that a function could borrow | 12 |
| 9.6 | 109 | An own function that is read out, by itself or inside a part of its struct, keeps the struct it sat in, and borrows it. For the lifetime rules, a value that is read out of a struct counts as borrowing the struct when a function can sit in it within reach of `.` and `[ ]`, except where the line puts it straight into the same place of another struct of the same type | The borrower rule for a function that uses names around it (7.8). Reading it out is what lets it be a value like any other. The compiler sees types, and not which function a field holds | 7, 1 |

### Repeating, files and running

| DOCS | # | The rule | Why | Principle |
|---|---|---|---|---|
| 10.4 | 110 | A program that calls deeper than the stack allows fails like any other check, and so does a declaration that needs more stack than is left | One behaviour for every failure | 7 |
| 10.4 | 111 | The last call takes over the stack of the function that makes it, when nothing the call uses lives in that call: its names, the heap values they own, its kept parameters, the values made on the spot. An ownership that is handed over doesn't count | Repeating is a function calling itself, so repeating without end has to be possible | 5, 12 |
| 11.2 | 112 | `import` takes one text written out in the code, the path without `.my`. A text that leads to no file is an error, two texts that lead to one file name the same file, and a file can't import itself. An `import` inside a function counts like one at the top level. (That `import` is always called by its own name is adjusted: A11) | The compiler has to know every file before the program runs | 8 |
| 11.2, 11.4 | 113 | An `import` only gives the address of its file, and nothing of the file runs at that line. The program starts by setting up every file, each one after the files it imports, in the order in which the importing file's text first names them. An `import` inside a function counts, whether or not it runs, and the file you run comes last. So a name reached through a package is always set up | Jack's rule says once, and before any file that imports it. A compiler also has to know which of two files comes first when neither imports the other, because a declaration's value can be a call that prints | 2, 8 |
| 11.3 | 114 | The compiler works out which files each place can hold: an `import` is exactly its file, a name is its own place, each field of a struct is its own place, and the elements of an array, the values of a pair and the entries of a map count together. A pointer shares the place it points at, a seen call's result is looked at, and a heap value, an `any` and an unseen result are one shared place. A name has to be in every file of the place, with one type | It is how the checks of 11.5 and 11.6 are made, exactly enough that every compiler accepts the same programs. Fields are told apart by name, so a struct can hold two different files side by side. A place the compiler can follow is not mixed with the ones it can't | 8, 2 |
| 11.4 | 115 | A file's names are set up from the top down. A `const` whose value is a function value written out, a written value, or an array, pair or map value made only of written values, and a type name, are ready from the start. A top-level call can't reach any other name declared below it. Only the names of the file that is being set up can be missing | A body can use any top-level name, so a call could otherwise read a name with no value yet | 8 |
| 11.4 | 116 | A top-level `main` in the file you run has to be a `const` of type `()int`. The status is from 0 to 255. With no `main` it is 0 | It completes the `main` rule. The status reuses the conversion rule | 6 |
| 11.5 | 117 | Underneath, a package value is the address of its file's table of names, one table for each file, and every name reached through a package has one fixed position in all the tables. What a file works out stays in the file's own place until the program ends: every `var`, every worked-out `const`, and the kept parameters of top-level calls. That place is on the stack, below everything that runs afterwards | It is how `shapes.area` works with nothing looked up while running, and one address is one size. A file's variables are values like any other, so nothing goes on the heap out of sight | 12 |
| 11.6 | 118 | The `const` rules hold for the name that holds a package: a `var` borrower and an ownership need a `var` package, and through a `const` borrower the package can't be given another file. The file's own names follow their own `const` or `var` through all of them | "Like any other variable", and what such a name holds is the address | 7 |
| 8.5, 8.12, 11.6 | 119 | A file lasts as long as the program. A borrower of a file's variable and a function of a file can go anywhere, a package holds nothing that can end, and what is put into a file's variable has to last as long as the program. For 8.12, a file's variable is one owner through whichever package it is reached, and a package is not an owner. Through a package that can hold several files, a name counts as that name in each of them | "One instance of each file", carried into the ownership rules. Where the compiler can't tell which file a package holds, it counts the worst case | 7, 8 |
| 11.1 | 120 | The file you run is a file like every other: one instance, set up when the program starts, after the files it imports. No file of the program can import it | "A package is a file", with no special case for the first one. An import of it would close a circle | 7 |
| 11.7 | 121 | A failed check stops the program at once, writes `error: FILE:LINE: REASON` to the error output, and ends with status 1 | One behaviour for every check. 0 means everything went well | 7, 9 |
| 11.7 | 122 | A failure can't be caught. A program that isn't sure tests first | The tests already exist and are in the code | 6, 2 |
| 11.8 | 123 | `print: ([]char)void` writes exactly the `char`s it is given, at once | Nothing is added out of sight | 2, 11 |
| 11.8 | 124 | `read: ()[]char` gives the next line with its `\n`, and an empty text at the end of the input | An empty result can then only mean the end | 6 |
| 11.8 | 125 | `exit: (int)void` ends the program with a status | The same status rule as for `main` | 6, 11 |
| 11.8 | 126 | `print` and `read` fail like any other check when the output or the input can't be used | One behaviour for every failure | 7 |
| App. A | 127 | The full grammar, with the order of operations built in, and the rules that tell the forms apart | One place for whoever writes the parser | 12 |
| App. B | 128 | Every check the compiler makes and every check made while running, with the reason each one prints | One place for whoever writes the compiler | 8 |

Left out on purpose, and so not rules: reading and writing files, program arguments, and bit
operations. The language is complete without them, and each can come later as one more
built-in function.

---

## 3. Adjusted rules

Every existing rule that changed. Each is marked **(adjusted)** in `DOCS.md`, and
`DECISIONS.md` section 13 has the same list.

| # | DOCS | Old wording | New wording | Why |
|---|---|---|---|---|
| A1 | 7.8, 8.11, 2.6. Also marked in 8.5, 8.16, 13 | "It can't outlive what was passed in… An argument handed over with `@` doesn't count: … nothing is left to die." "So they live as long as it does." "The value is freed when the function given back ends." In 2.6: a parameter lives "as long as a function the call gives back uses it". Nothing said where kept parameters are | Kept parameters stay on the stack with the function call that made the call, and go along when the function is given back again. In no other way can the function leave that call: it can't be put in a variable that outlasts it. They end, and a heap value they own is freed, when the call that holds them ends. What a value handed over with `@` holds still counts. A result also comes from the function that was called, and whether a result keeps parameters at all is read off the function when the compiler sees it. A parameter also counts as kept when the function given back is made from it by another call, so `return makeAdder(by);` is fine for a parameter `by` | Kept parameters need a place to live that isn't hidden heap use. **This is a bigger change: section 4.1** |
| A2 | 8.12, 7.2 | "For one called through a variable, an array or a map, every `&` argument counts as one that may change." "A call through a variable, an array or a map counts as changing every outside variable." | "Through a variable (7.2)" in both places. A lookup in a `const` array, pair or map whose functions are all written out is a call the compiler sees. It counts the worst of those functions. A call through a variable counts as changing every outside variable that a function of its type changes or takes, not every outside variable | Every choice is such a lookup, so the old wording made every function that chooses count as changing everything. And a variable holds only a function of its type, so only those functions can run. **This is a bigger change: section 4.2** |
| A3 | 8.17 | The heading "…and what isn't settled", and "Not settled yet: Two functions given back by one call that use the same `var` parameter… Whether they share the kept parameter or each keep their own." | "Nothing here is left unsettled. Two functions given back by one call share the parameters they keep (7.8)." | The docs' own open point, decided by rule 78 |
| A4 | 5.9 | "`&a[:]` \| the whole array, the same as `&a`" | "the whole array. When `a` is unsized it is the same as `&a`. When `a` is fixed, `&a` is an `&[3]T` and `&a[:]` is the `&[]T`" | An `&[3]T` and an `&[]T` are different types, and a value never changes type by itself |
| A5 | 5.9 | The comment "checked while running: fails if n is over 4" on `&scores[1:n]` | "fails if n is below 1 or over 4" | The slice starts at 1, and the rule above the example says a start after the end fails |
| A6 | 13 | "Four checks happen while the program runs:" | "These checks happen while the program runs. A check that fails stops the program (11.7), and Appendix B lists all of them" | The table had five rows already, and the filled-in rules add more |
| A7 | 3.2, 13 | The order of operations, step 2: "`!` and the marks". Step 3: "`*`" | Step 2: "`!`, `-` and the marks". Step 3: "`*` `/` `%`" | The new operators needed a place in the order. Nothing that was there moved |
| A8 | 11.4 | "Nothing runs because a file was imported, so the order the program runs in never depends on imports." | "All that runs for an imported file is the working out of those declarations' values, once, when the file is set up. Nothing runs at the line of an `import`. So the order the program runs in is fixed before it starts: every imported file is set up first, and then the file you run goes from the top." | A declaration's value can be a call, so setting a file up does run something. No statement runs, and nothing runs because of where an `import` is written |
| A9 | 5.9 | "When both are written as numbers and the length is in the type, the compiler checks them." | "When both are written values and the compiler knows the length (5.2), the compiler checks them, and a bound that is left out counts as written." | The docs already reject `a = [1, 2, 4, 5];` for an unsized `a` declared with three elements, so the compiler knows that length. Under the old wording an assignment and an index would use it, and a slice of the same array would not |
| A10 | 9.6, which narrows a sentence of 7.8. Also marked in 13 | 7.8: "a block sees the names of the blocks around it". 9.6: "A function declared inside a struct is in the struct's scope, so it uses the struct's fields directly." | The same, but only a function that is written as a field's value, or inside an array, a pair or a map in a field, can use the fields. And such a function can't also use a name of a function call around the struct value: a parameter or a name of the function that the struct value is written in | A function of a struct works on the struct it sits in, and only a function that is written as a field's value sits in the struct. A function value has one address for what it keeps, and a function of a struct needs it for the struct, so it has none left for the call around it (9.8: a struct holds only its fields). The top-level names of its file need no address, so it can still use those |
| A11 | 13, 4.4 | In 13: "Built-ins are first-class values like everything else: they can be named, stored and passed." In 4.4: "Like every built-in, it's an ordinary first-class value" | The same, with one exception: `import` is always called by its own name with its text written out. In 4.4: "Like every built-in except `import` (11.2)" | The compiler has to know every file of a program before it runs |
| A12 | 8.15 | "`const copyLinks: (&^Link)^Link = (const from: &^Link){`" | "`(var from: &^Link){`", with a sentence that says why | The copy's `prev` links borrow nodes of the original until `linkBack` has set them. Through a read-only borrower, that would put borrowers of a list that may be a `const` into a `var` (rule 85) |
| A13 | 5.9 | "and the array can't be given a new value or moved while the slice is used further down (8.12)" | "and an array on the heap can't be freed or moved away while a slice of it is used further down (8.12)" | 8.12 protects what an owner on the heap holds. A plain array stays where it is when it is given a new value, and a slice then sees the new elements, as `b^` sees `count = 8;` in 8.4 |
| A14 | 4.5 | "Everything that doesn't touch an `any` is checked before the program runs." | "The type of everything that doesn't touch an `any` is checked before the program runs." The checks about values are listed in Appendix B | The docs already had checks while running that touch no `any`: a `null` that is followed, a map key, a conversion that doesn't fit, a slice. The sentence is about types, and now it says so |
| A15 | 8.12 | "'Still used below' means written anywhere further down the function." | The same, with two things added: a call further down to a function that uses the pointer is a use of it, and a pointer in an outside variable is still in use when the function ends | A function can use a pointer in a variable from outside itself, and a pointer that function bodies share can be followed after the function that aimed it has ended. With the old wording alone, freeing what such a pointer points at would go unnoticed (rules 93 and 96) |
| A16 | 3.2 | "The compiler knows which values are types, because their type is `type` (4.1). So `(a)` starts a function type when `a` is a type, and a group when it isn't." | The same sentences, followed by: in these rules "a type" means a type that is written out, with type names and the forms of 3.6. A value that only holds a type, such as a `var` of type `type` or `x.T`, is an ordinary value, so `(x.T)` is a group | The type of `x.T` and of a `var` that holds a type is `type` too, and neither can be written where a type is expected. Only a type that is written out is known before the program runs (rule 33) |

Several existing tables also gained rows or cells (2.6, 3.2, 5.9, 8.5, 8.8, 8.9, 8.11, 8.12,
8.16, 8.18 and the tables of section 13). Each addition of mine is marked
**(filled in)** or **(adjusted)** where it stands.

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

**Picked: e.** It keeps every example in the docs, and every rule except the wordings that A1
lists. `makeAdder`, `makeCounter`, `steps`, `makeLookup`, `countFrom` and `newCounter` all work
as written, and so does `first(int)` given back from a function.

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
  (rule 80). So `pick(double)` and `wrap(int)(3)` are held by nothing.
- Kept parameters end when the call that holds them ends, and a heap value they own is freed
  then (rule 92). The docs said "when the function given back ends", which is not one moment
  once the function has been copied.
- A result that comes from a parameter can be handed on (rule 81). So `return makeAdder(by);`
  is fine when `by` is a parameter, and an error when it is a name declared in the function.
  The docs said a function given back can't outlive any argument, and read alone that would
  turn the line away. But "a function given back keeps the parameters it uses", and "a local
  is never kept": the parameter that the function is made from is kept with it, and for the
  caller the result is limited by the arguments of the function that handed it on. Generic
  code needs this, as `sumWith` in 4.3 shows.
- A result also comes from the function that was called (rule 81). `adder3(five)(five)` gives
  back a function that reaches the kept parameters of both calls, so both sets go along when
  it is given back.
- Stored behind a pointer in a place that outlasts the call, a function given back is turned
  away at the line that stores it (rule 84). What the compiler can't see there belongs to the
  list in 8.17 (rule 102), like the borrower the docs already had there.
- Kept parameters stay with the call that holds them, and a function value holds their
  address. So a copy of such a function uses the same kept parameters (rule 79), the way a
  copy of a borrower borrows the same value. It is the one place where `=` on a value with
  state doesn't give a second, independent state, and nothing in the line says `&`.
- Kept parameters move when the function is given back again. A kept parameter that borrows
  another kept parameter itself would still point at the old place, and the compiler can't
  see it from the caller. That case is on the list in 8.17 (rule 102).

**A cost to know about.** Kept parameters end when the call that holds them ends. At the top
level of a file, that is the end of the program, so a heap value that a kept
parameter owns stays that long, even after the last function that used it is gone. Ending
them with the block the call is written in would free sooner, but it would stop a function
given back from being stored in a variable of an outer block of the same function.

**What Jack may want to change.** If a function given back should be storable anywhere, or if
a copy of it should get kept parameters of its own, the honest way is option a with a mark
that shows the allocation. That would be a new piece of syntax, close to idea 1 in
`UNDECIDED.md`, so I did not go there.

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
needs to know what a call does uses the same list: 4.3, 4.4, 4.5, 7.8, 8.5, 8.6, 8.12, 9.2,
9.3, 11.3 and 11.4.

**The second half: what a call through a variable counts.** The docs said such a call "counts
as changing every outside variable, because the compiler can't tell which function runs". A
function can also give a pointer in an outside variable a new target, or follow one (rules 93
and 96). With all three counted for every outside variable, any file with an owner and a
pointer at its top level would turn away three calls of a struct's functions in a row: the
first could aim the pointer at what the owner holds, the second could free it, and the third
could follow the pointer.

| Option | What it means | Against the philosophy |
|---|---|---|
| a. Every outside variable, as the docs said | The simplest worst case | Calls of functions in fields, the everyday way to use a struct, stop working beside a top-level pointer |
| b. Only what the functions of that type do | A variable holds only a function of its type, so the compiler reads those, in every file | A function added somewhere else in the program can change whether a call through a variable is accepted beside a pointer in use. 4.3 and 11.4 already count this way |
| c. Only the functions that are ever used as a value | Finer still | It needs a second notion, and a reader can no longer tell from a type what a call counts |

**Picked: b.** What Jack wanted still holds: when the compiler can't tell which function runs,
it counts the worst. It now counts the worst of the functions that could run. It stops short
of option b in the first table: the compiler doesn't follow where each function value
travels, it only asks which functions have the type.

**What Jack may want to change.** The need for this half comes from my own rules 93 and 96,
which follow pointers in outside variables, and not from a gap in the docs' sentence. If Jack
prefers his worst case as it stood, the price is the example above: three calls of a struct's
functions in a row are turned away beside a top-level owner and pointer.

**What it changes.** Calls that the old wording treated as unknown are now read exactly. One
kind of line is judged more strictly for it: a `const` that is lent to a lookup the compiler
sees is an error when any of the listed functions could change it, also when the function that
is picked would not.

---

## 5. Waiting on `UNDECIDED.md`

Nothing waits. Every gap could be filled with the rules that are decided.

Two filled-in rules would read more simply if an undecided idea were accepted. Both were
written with today's rules instead, and neither depends on the idea:

| Rule | Today | With the undecided idea |
|---|---|---|
| 4.1 above, a function that has to be stored for longer than the call that made it | it is made at the top level of its file, or given back to where it is needed | idea 1 (kept names after a `;`) would be the place to say how a function keeps a value of its own |
| 71 (DOCS 7.1), "every path reaches a `return`" | a body is one straight line, so the rule is "the body runs into a `return`" | idea 2 (`if` and `else`) would bring branches back, and the rule would need its branch cases again |

One thing to know before deciding idea 1. Its text says that a kept name "lives as long as the
function", and that "copying the function copies its kept names". For the kept parameters
that exist today, rule 79 and A1 say the opposite: a copy of a function shares them, and they
end with the call that holds them. Both follow from 7.7, where every function value is two
addresses. I did not use the idea to decide either rule. If Jack accepts idea 1 as written,
rule 79 and A1 have to be looked at again together with it.

---

## 6. Jack's own changes

Four rules in `DOCS.md` are decisions Jack made himself. They carry no mark there, and each
has a row in `DECISIONS.md` in his words. They are listed here apart from my work, with the
rules I added so that each one works.

| Jack's rule | Where in DOCS | What I added for it |
|---|---|---|
| **A `var` can hold a type:** `var t: type = int;`. The variable holds the type as a value. It is not a type name, so it can't declare anything | 4.2 | Rule 40 says which names are type names. Rule 48 makes every type known before the program runs, and rule 49 makes a `type` value one address, so a `var` has room for any type. Rule 41 keeps a held type out of every place that decides a type |
| **A `const` never changes.** A call through a variable that lends a `const` is checked at the call, while the program runs, with a note beside the function's code that says which parameters are `var`. A function called through a `const` struct is checked in the same way. Both are off the list in 8.17, and both have a row in Appendix B.9 | 8.6, 7.7, 8.17, 9.3 | Rules 85 to 87 keep everything read-only from reaching a `var` where the compiler can see, so the check at the call is the only one needed. Rules 77 and 106 say what else the note holds and how the check for a struct works. Rule 102 says what is left in 8.17. One adjustment follows from the rule: A12 (`copyLinks`) |
| **A struct type has its field names:** `const Point: type = {x: int, y: int};`. A value is checked against the names and the types. A function type keeps no names, except a type parameter that a later part of the type uses: `(T: type, []T)T` | 3.3, 3.6, 4.3, 9.1, 9.2 | Rule 35 makes the names and their order part of what makes two struct types the same. Rule 44 says where a type parameter's name can be used. Rule 103 says what a struct value has to hold |
| **A package is a pointer to its file: one instance of each file.** A package value is the file's address, held like a function value. `const` on a package means the name always holds the same file, and the file's own names follow their own `const` or `var`. A file is set up once, before any file that imports it, and imports can't go in a circle | 11.2, 11.3, 11.5, 11.6, with 2.9, 7.7, 8.6 and 9.6 | Rule 113 fixes the order in which the files are set up, and says that an `import` only gives the address. Rule 117 says what the address points at and where a file's variables live. Rule 118 says what `const` and `&` mean for the name that holds a package, and rule 119 carries "one instance" into the ownership rules. Rule 76 follows it for functions: a function keeps no address for its file. Two adjustments stand beside the rule. A8 says when the declarations of an imported file run. A10 keeps a function in a struct from using the names of a call around it, and leaves the top-level names of its file open to it, because a top-level name needs no address |
