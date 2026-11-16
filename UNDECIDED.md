# Undecided

Ideas for MyLang that are not decided yet. None of them is in `DOCS.md`, and the language
works without them.

---

## 1. A shortcut for keeping a local: kept names after a `;`

**Status:** undecided.

**What it is for.** In `DOCS.md` 7.8, a function given back can keep a local only through a
helper function. The local goes on the heap, and the helper's parameter takes it over with `@`:

```go
const countFrom: (@^int)()int = (var @count: ^int){
    return (){
        count~ = count~ + 1;
        return count~;
    };
};

const newCounter: ()()int = (){
    var total: ^int = ^0;
    return countFrom(@total);          // total is handed over and left null
};
```

The shortcut does the same job without the helper. The function declares what it keeps after
a `;` in its own brackets:

```go
const newCounter: ()()int = (){
    var total: ^int = ^0;
    return (; var @count: ^int = @total){     // count takes total's heap int as the function
        count~ = count~ + 1;                  // is made; total is left null
        return count~;
    };
};
```

**Everything else works the same.** It is only a shorter way to write the helper: a kept name
is a parameter filled in when the function is made, instead of at a call.

- Each kept name is an ordinary declaration with a value (2.1), and it runs once, when the
  function is made. The body still runs at each call, from the top.
- The `= value` tells a kept name from a parameter, because a parameter never has one.
- The function's type doesn't show kept names: `newCounter` above still gives back a `()int`.
- The declaration says how the value is kept: `= x` keeps a copy, `= &x` a borrow (the function
  can't outlive `x`), and `= @x` takes the value over.
- A kept name lives as long as the function. A `var` one keeps its changes between calls, and
  copying the function copies its kept names (8.8).

C++ lambda init-capture (`[count = std::move(total)]`) and Swift capture lists work the same
way.

---

## 2. `if` and `else`

**Status:** undecided.

**What `DOCS.md` does instead.** A choice is a lookup: a pair `?T` picks one of two
values or functions, and a map of functions picks one of many (`DOCS.md` 10.1, 10.2). Recursion
stops with a pair too (10.3).

**What `if` would add.** A choice written as a statement, so only the chosen side runs and a
`return` inside it ends the function. These examples in `DOCS.md` use a pair now and
would read more simply with an `if`: the null test and `valueOr` (8.10), `length` (8.14),
`linkBack` (8.15), and `factorial` and `sumFrom` (10.3). `return;` (7.3) would also become
useful for stopping a function early, and the rule that every path reaches a `return` (7.1)
would need the `if` rules below.

### The text that was in DOCS.md 10.1

```go
if (count > 10) {
    size = 10;
} else {
    size = 3;
}
```

| Part | What it says |
|---|---|
| `(count > 10)` | the condition, in brackets. It has to be a `bool` |
| the first `{ }` | runs when the condition is `true` |
| `else if (…) { }` | another test, tried only when every test above it was `false`. There can be several |
| `else { }` | runs when every test was `false`. It can be left out |

Only the chosen side runs. Each `{ }` is a block (2.6), so a name declared inside it ends at
its `}`. An `if` doesn't end with `;`, the same as a block on its own.

```go
const abs: (int)int = (const n: int){
    if (n < 0) {
        return 0 - n;
    } else {
        return n;
    }
};

abs(7)                 // 7
abs(0 - 7)             // 7
```

`return` inside an `if` ends the whole function.

A function that returns a value has to reach a `return` on every path (7.1). An `if` with no
`else` leaves a path without one:

```go
const cap: (int)int = (const n: int){
    if (n > 10) {
        return 10;
    }
};                     // ERROR: when n isn't over 10, the function ends without a return
```

Give the `if` an `else` that returns, as `abs` does, or put a `return` after it:

```go
const cap: (int)int = (const n: int){
    if (n > 10) {
        return 10;
    }
    return n;          // fine: every path reaches a return
};

cap(12)                // 10
cap(7)                 // 7
```

The compiler doesn't work out the conditions. It counts both sides of every `if` as
possible, so two `if`s that cover every case between them still need a `return` after them.

With `else if`, the tests are tried from the top, and the first one that is `true` runs:

```go
if (n < 0) {
    sign = 0 - 1;
} else if (n == 0) {
    sign = 0;
} else {
    sign = 1;
}
```

`else if` is the same as an `if` written inside the `else` block, with one pair of braces
fewer.

The condition is never anything but a `bool`, because a value never changes type by itself
(3.2):

```go
if (count) { ... }         // ERROR: count is an int, not a bool: write count != 0
```

Several tests go in one condition with `&&`, `||` and `!` (3.2):

```go
if (count > 0 && count < 10) { ... }       // both tests have to be true
if (p == null || p~ == 0) { ... }          // p~ is followed only when p isn't null
if (!ready) { ... }                        // runs when ready is false
```

---

## 3. Packages: a program of more than one file

**Status:** undecided.

**What `DOCS.md` does instead.** A program is one file (`DOCS.md` section 11). Every name is
declared in that file or is a built-in, and the file runs from the top.

**What packages would add.** A file could bring in another file with `import`, and reach its
top-level names through a value, a package, whose type is the file: `package("geom")`. A
program could then be split over several files.

**How to read the text below.** It is the text that was in `DOCS.md`, kept whole, with the
reasons that were in `DECISIONS.md`. Section numbers are the ones `DOCS.md` had then. Three of
those sections are still there under new numbers: 11.4 (order, where statements go, and `main`)
is 11.1 now, 11.7 (a failed check) is 11.2, and 11.8 (output, input and ending the program) is
11.3. **(filled in)** marks a rule that Claude added to Jack's rules, and **(adjusted)** marks
one that Claude changed, as in `DOCS.md`. Jack has not reviewed those parts.

### The text that was in DOCS.md section 11, "Files and packages"

#### 11.1 A file is a package

Every file is a package. Its top-level declarations are the package's names, and a file sees
only its own names until it imports another file. So you can always tell where a name comes
from: it is declared in this file, or it is reached through a package this file imported.

Inside a file, `{ }` blocks still make smaller scopes (2.6).

**(filled in)** The file you run is a file like every other. There is one instance of it
(11.2), and it is set up when the program starts, after the files it imports (11.4). No file of
the program can import it: those imports would go in a circle.

#### 11.2 Using another file: `import`

A file brings in another file with an ordinary declaration:

```go
// the file geom
const Point: type = {x: float, y: float};
const area: (float)float = (const side: float){
    return side * side;
};
var made: int = 0;
```

```go
// the file main
const geom: package("geom") = import("geom");

geom.area(2.0)                         // 4.0
const Point: type = geom.Point;        // a type from geom, named here too
geom.made = geom.made + 1;             // made is a var in the file geom, so it can change (11.3)
```

| Piece | What it is |
|---|---|
| `import("geom")` | a built-in that gives the file `geom` as a value |
| `package("geom")` | the type of that value: the type of the file `geom` |
| `geom.area` | one of the package's names, reached with `.` like a struct field |

- **`import` is a built-in, like `convert` and `wrap` (4.4, 4.5),** so the line is an ordinary
  declaration in the one shape every name uses (2.1).
- **A package is a pointer to its file.** There is one instance of each file, and a package
  value is that file's address, held like a function value (11.5). Importing the same file
  twice gives the same file both times, so both names reach the same variables (11.3).
- **A package's type is its file.** `package("geom")` is the type of the file `geom`: its
  top-level names, each with its `const` or `var` and its type. It is a type form, written
  where a type goes, like `[3]int` or `?int`. It is not a call: a type is still never called
  (4.1). The text has to be written out, the same as for `import`, so the type is known before
  the program runs. There is no plain type `package`: every package has its own exact type.
- **A package can be a `const` or a `var`,** like a function. `const` means that the name
  always holds the same file. The file's own names follow their own `const` or `var` (11.3).
- **A package value is always the address of the one instance of its file, so a copy of it
  reaches the same file.** `var p: package("geom") = geom;` is fine: the type only ever means
  "that file", and a file can't exist twice, so nothing is hidden. This is different from
  copying a function with kept state (7.8), where the type `()int` doesn't say that anything
  would be shared.
- **The name in `import` is the file's path from the importing file's folder:**
  `import("shapes/geom")`. A folder only organises files.
- **A file is set up once, before any file that imports it** (11.4). Two files can't import
  each other, directly or through other files, so that order always exists.

**(filled in) What `import` takes.** `import` takes one text, and the text is written out in
the code, so the compiler knows every file of a program before it runs:

```go
const deep: package("shapes/geom") = import("shapes/geom");          // shapes/geom.my
const util: package("../common/util") = import("../common/util");    // ../common/util.my: .. is the folder above
const which: package("geom") = import(fileName);                     // ERROR: the file has to be written out
```

- The text is the file's path without the `.my` (2.7). Its parts are joined with `/`. A text
  that leads to no file is an error.
- Two texts that lead to the same file name the same file, however they are written:
  `"geom"` and `"shapes/../geom"`.
- The text in a package type is read in the same way, from the folder of the file that writes
  it.
- A file can't import itself, at its top level or inside a function: it is the smallest
  circle of imports.
- An `import` that is written inside a function counts like one at the top level, for the
  circle rule and for the order in which the files are set up (11.4). It makes no difference
  whether that line ever runs.
- `import("geom")` has the type `package("geom")`. So `import` has no single type of its own.
  Unlike the other built-ins, it is always called by its own name with its text written out,
  because a file that is only named while the program runs couldn't be checked.

**(filled in) What an import does when it runs.** An import is a call, and all it does is give
the address of its file. Nothing of the file runs at that line: every file is set up when the
program starts, before the first line of the file you run (11.4). So an `import` can be written
at the top level or inside a function, and it can run any number of times. Each time it gives
the same file.

#### 11.3 What a file can use from another

Everything at the top level of an imported file can be used: its types, functions and
variables. A file's variables live as long as the program, and a call that changes them counts
as changing them (8.12).

**There is one instance of each file.** Every package that holds a file reaches the same
variables, and a function of the file uses those, whoever calls it:

```go
// the file counter
var count: int = 0;
const bump: ()void = (){
    count = count + 1;
};
```

```go
// the file you run
const a: package("counter") = import("counter");
const b: package("counter") = import("counter");

a.bump();
a.bump();
a.count                // 2
b.count                // 2: a and b hold the same file, so it is the same count
a == b                 // true: the same file (3.7)

const readCount: ()int = (){
    const c: package("counter") = import("counter");   // an import inside a function gives the same file again
    return c.count;
};
readCount()            // 2
```

**`const` on a package means the name always holds the same file.** The file's own names follow
their own `const` or `var`. A `var` of the file can be given a new value through any package
that holds the file, and a `const` of the file through none:

```go
a.count = 5;                   // fine: count is a var in the file counter
a.bump = a.bump;               // ERROR: bump is a const in the file counter
a = import("counter");         // ERROR: a is const, so it always holds the same file
```

It is the same as a `const` function that changes a variable outside itself (7.8): the function
never changes, and neither does the package. So this is no exception to the rule that a `const`
never changes (8.6).

**Two package types are the same when the files have the same names, kinds and types.** So
giving a `var` package another file is an ordinary type check, and a function can take a
package of a given shape:

```go
var shapes: package("geom") = import("geom");
shapes = import("geomFast");      // fine only if geomFast has exactly geom's names, kinds and types
shapes.area(2.0)                  // checked from the type alone

const draw: (package("geom"))void = (const g: package("geom")){ ... };
draw(shapes);                     // fine: shapes has the type the parameter asks for
```

**A package type can be named once and reused,** as any type can:

```go
const Geom: type = package("geom");
var fast: Geom = import("geomFast");
```

**(filled in) What "the same names, kinds and types" covers.** Two files have the same package
type when each top-level name of the one is a top-level name of the other, with the same
`const` or `var` and the same type (3.6). A type name also has to name the same type in both.
The order of the declarations makes no difference, and neither do the values. So `g.name` is
checked from the type of `g` alone: `name` has to be a top-level name of that file, it has the
type it has there, and it can be assigned when it is a `var` there. `g.Point` is the type that
`Point` names in the file, whichever file of that type `g` holds.

**(filled in)** A package type can name any file, also one that the file doesn't import. The
compiler reads such a file for its names, and the file is set up only when some `import` names
it (11.4). Two files can name each other's types: only imports can't go in a circle.

#### 11.4 Order, where statements go, and `main`

- **A function's body can use any top-level name of its own file,** wherever in the file it is
  declared. A body runs only when the function is called, so two functions can call each
  other. The value in a top-level declaration still uses only names declared above it.
- **Statements that run, such as `countUp();`, go only in the file you run.** An imported file
  holds only declarations. **(adjusted)** All that runs for an imported file is the working
  out of those declarations' values, once, when the file is set up (below). Nothing runs at
  the line of an `import`. So the order the program runs in is fixed before it starts: every
  imported file is set up first, and then the file you run goes from the top.

```go
// the file geom
area(1.0);             // ERROR: geom is imported, so it holds only declarations
```

**It is good practice to put what the program does in a `const main: ()int`.** It isn't required,
and code can still go outside it:

```go
// the file you run
const geom: package("geom") = import("geom");

const main: ()int = (){
    const size: float = geom.area(2.0);
    return 0;
};
```

When the file you run declares `main`, the program runs that file's statements from the top,
then calls `main`. The `int` it returns is the program's result, `0` when everything went well.

**(filled in) When a file's names are set up.** The top-level declarations of a file are
worked out in order, from the top down, each value when its line is reached. That is the
file's set-up, and it happens once (11.2). A program is the file you run and every file that is
reached from it through imports. It starts by setting up all of them:

1. A file's imports are all the `import`s written in it, at its top level and inside its
   functions (11.2).
2. Before a file is set up, the files it imports are set up, in the order in which its text
   first names them in an `import`. A file that is already set up is left as it is.
3. The file you run comes last. Its set-up is its top-level lines, declarations and statements
   together, in the order written.

```go
// the file you run
const a: package("counter") = import("counter");   // counter is set up before this file, and so is every file it imports
const g: package("geom") = import("geom");         // geom is set up after counter, because this file names it second
print("start\n");                                  // by the time this line runs, every file of the program is set up
```

**(filled in)** So every file is set up before any file that imports it, and all of them before
the first line of the file you run. A name that is reached through a package is always set up
already.

**(filled in)** Three kinds of top-level name are ready from the start, because nothing has to
run to make them: a `const` whose value is a function value written out, a `const` whose value
is a written value (3.2) or an array, pair or map value made only of written values, such as
`const limit: int = 10;` and `const primes: [3]int = [2, 3, 5];`, and a type name (4.2). Every
other top-level name is set up when its line is reached: every `var`, and every `const` whose
value has to be worked out.

**(filled in)** A function's body can use top-level names declared below it, so a call could
reach a name that isn't set up yet. The compiler rejects that call: **a call at the top level
can run only when every top-level name its function uses is ready, or is declared above the
call.** The compiler reads the function for this, and the functions it calls, as in 8.12. Only
the names of the file that is being set up can be missing: the names of every other file that
the call can reach are set up already. A call through a variable (7.2) could run any function
of its type, so it counts as using what every function of that type uses in the file whose top
level is being set up.

```go
const readLate: ()int = (){ return late; };

const early: int = readLate();     // ERROR: readLate uses late, which is set up below this line
var late: int = 5;
const onTime: int = readLate();    // fine: late is set up by now
```

**(filled in) The rules for `main`.**

- `main` matters only in the file you run. If that file declares a top-level `main`, it has to
  be a `const` of type `()int`. In an imported file `main` is an ordinary name, and nothing
  calls it.
- The program sets up the imported files, runs the top-level lines of the file you run from
  the top down, and then calls `main`.
- The `int` that `main` returns is the program's status. A status is a number from 0 to 255:
  the `int` is converted the way `convert(int, uint8)` converts it (4.4), and a number outside
  that range fails in the same way (11.7).
- A file with no `main` ends after its last line, with status 0.
- `exit` ends a program before that (11.8).

#### 11.5 Underneath: a package value is an address

A package value is the address of its file, the way a function value is the address of its
code (7.7). There is one instance of each file: its variables are held once, for the whole
program, and every package that holds the file leads to them. Copying a package value copies
the address, and nothing else (8.8).

So a package can be held like a function. It can be a `const` or a `var`, passed to a function
or kept in an array, and a `var` one can be given another file of the same type, as a `var`
function can be given another function (7.5):

```go
var shapes: package("geom") = import("geom");
shapes.area(2.0)               // 4.0, from geom

shapes = import("geomFast");   // shapes now holds the file geomFast, and the file geom is untouched
shapes.area(2.0)               // 4.0, from geomFast
```

The new file has to have the type of `shapes`: exactly the names, kinds and types of `geom`
(11.3). So the compiler checks every `shapes.area` from the type, and nothing is looked up
while the program runs.

**(filled in)** This is how that works underneath. For each file the compiler writes a table
into the program, with one entry for each of the file's top-level names. A package value is the
address of that table, so every package value has the same size, whichever file it holds. Files
of one package type have their tables laid out alike, so `shapes.area` reads the entry at the
same position, whichever of them `shapes` holds. The entry leads to the name's value:

| The name | Where its value is |
|---|---|
| is ready from the start (11.4): a function value written out, a value made of written values, a type | in the program itself |
| is worked out when the file is set up: every `var`, and every `const` that isn't ready from the start | in the file's own place |

**(filled in)** The file's own place is the frame of the file's top level. It also holds the
kept blocks of the calls made there (7.8). It is on the stack, like every value whose type has
no `^` (2.6). The files are set up one after another when the program starts, and what each one
works out stays where it is put, below everything that runs afterwards, until the program ends.
So a file puts nothing on the heap out of sight, and a top-level value with no size in its type
(5.3) takes its size from its first value, like any other.

#### 11.6 Ownership and borrowing work on a package like on any variable

A package is a value like any other, so everything in section 8 applies to it. The value is the
file's address (11.5), so each of these passes the address around, and the file itself stays
where it is:

| You write | What happens, the same as for any variable |
|---|---|
| `var p: package("geom") = geom;` | `p` gets its own copy of the address, so `p` and `geom` hold the same file (8.8) |
| `var b: &package("geom") = &geom;` | `b` borrows the name `geom`: nothing is copied, and through `b` the name can be read, or given another file (8.4) |
| `var h: ^package("geom") = ^geom;` | a copy of the address on the heap (8.3) |
| `var @mine: package("geom") = @geom;` | `mine` takes `geom`'s value over. An address owns nothing on the heap, so this is the same as a copy (8.9) |
| `take(@geom)` | a parameter that asks for an ownership takes it at the call, in the same way (8.13) |

**(filled in)** The rules for `const` hold here too, for the name that holds the package. A
`var` borrower borrows only a `var` package (8.6), and only a `var` package has an ownership to
give (8.9). So the second, fourth and fifth lines need `geom` to be a `var`. Through a `const`
borrower the package can be read, and it can't be given another file. None of this touches the
file's own names: they follow their own `const` or `var` through every one of these (11.3).

Everything that holds a package shares the one file. So a function that is handed a package
changes the same variables as its caller, with no borrow:

```go
const counter: package("counter") = import("counter");

const bumpTwice: (package("counter"))void = (const p: package("counter")){
    p.bump();
    p.bump();
};

bumpTwice(counter);
counter.count                  // 2: p holds the same file, so it is the one count that went up
```

A borrower of a package is for the name itself. It lets a function give the caller's package
another file:

```go
const useFast: (&package("geom"))void = (var p: &package("geom")){
    p~ = import("geomFast");
};

var shapes: package("geom") = import("geom");
useFast(&shapes);              // shapes now holds the file geomFast
```

The borrowing rules hold too: a borrower of a package can't outlive the name it borrows (8.5).

**One element of a package is just an ordinary declaration,** with no new syntax:

```go
const area: (float)float = geom.area;                  // a function
const Point: type = geom.Point;                        // a type
const area2: (float)float = import("geom").area;       // in one line
const count: &int = &import("counter").count;          // a var, live: borrow it
var snapshot: int = import("counter").count;           // a copy of the value now
```

A file lasts the whole program, so borrowing its variable is always fine.

**(filled in) A file lasts as long as the program.** A name that is reached through a package
is a top-level name of a file, wherever the line that reaches it is written. So:

- A borrower of a file's variable never outlives what it borrows, and neither does a function
  of the file: the file outlasts both (7.8, 8.5). A function still can't give such a borrower
  back, because a borrower that is given back comes from a parameter (8.13). A package holds
  nothing that can end, so it can go anywhere.
- What is put into a file's variable has to last as long as the program, as for every top-level
  variable (8.5).
- For rule 8.12, a file's variable is one owner, through whichever package it is reached.
  Giving a `var` package another file frees nothing and changes no owner.

```go
var late: &int = null;
{
    const c: package("counter") = import("counter");
    late = &c.count;           // fine: count is a variable of the file counter, which is always there
}
late~                          // the count of the file counter: c has ended, and the file has not
```

### What the other sections of DOCS.md said about packages

**2.7, reserved words and built-in names (filled in).** There were eight reserved words:
`const var return void true false null package`. "`package` starts the type of a file,
`package("geom")` (11.2)." `import` was one of the built-in functions.

**2.8, names (filled in).** A function's body could use "any top-level name of its own file",
and a top-level name could be seen "in every function body of its file". On how a name is
found: "It never looks in another file: a name from another file is always reached through a
package, as in `geom.area` (11.2)."

**2.9, places (filled in).** The table of places had this row:

| Place | Example | It can be assigned when |
|---|---|---|
| a name of a package | `geom.made = 1;` | the name is a `var` in its file (11.3) |

"A name behind a package's `.` is a name in its own right: a top-level name of a file, with a
`const` or `var` of its own. The package in front only says which file it is in (11.5). So
`geom.made` is a place when `made` is a `var` in its file, whatever word the package was
declared with (11.3)."

**3.1.** "A file that is used from another file has a type of its own, written
`package("geom")` (11.2)."

**3.3, the table of type forms.**

| Kind | Form | Example | Read as |
|---|---|---|---|
| package | `package("file")` | `package("geom")` | the type of the file `geom` (11.2) |

**3.6, the grammar of types.** `Type` had one more line, and a basic type could be reached
through a package **(filled in: the last two lines)**:

```
          | "package" "(" TEXT ")"            (the type of a file, 11.2)
BasicType = NAME { "." NAME }                 (a type's name, or one reached through a package: geom.Point)
TEXT      = a text written in the code (5.10)
```

"A type reached through a package, `geom.Point`, is the type that `Point` names in the file of
the package's type (11.3)."

**3.7, what "equal" means (filled in).**

| Type | `=` copies | So two values are equal when |
|---|---|---|
| a `package` | the address of its file (11.5) | they hold the same file |

**3.8, the type of every value (filled in).**

| Form | What it needs | Its type |
|---|---|---|
| `g.x` | `g` has a package type, and `x` is a top-level name of that file | the type of that name (11.3) |
| `import("geom")` | the text is written out, and leads to a file | `package("geom")` (11.2) |

**4.2 (filled in).** The compiler collects the top-level type names of a file before it reads
anything else. "It does this for the imported files first."

**4.4 and 13, built-ins (adjusted).** In 4.4: "Like every built-in except `import` (11.2), it's
an ordinary first-class value". In 13: "The one exception is `import`, which is always called by
its own name with its text written out (11.2)."

**5.2 (filled in).** "A name that is reached through a package whose file the compiler sees
(8.12) counts like the name itself, here and for the keys of a map (6.2, 6.3) and the type
inside an `any` (4.5)."

**7.2, which function a call runs (filled in).** The table of calls the compiler sees had this
row, and a call through a variable "could run any function of that type, written in any file of
the program":

| The value in front of the `(` | The compiler sees |
|---|---|
| a name from these rows reached through an `import` that is written out, or through a `const` package that was declared with one: `import("geom").area(2.0)`, `geom.area(2.0)` (11.2) | that function |

**7.7 (filled in).** "A top-level name needs no address. There is one instance of each file
(11.2), so each of its top-level names has one place, and the whole program knows where it is
(11.5)."

**8.5 (filled in).** "A package holds none of them: it is the address of a file, and a file
lasts as long as the program (11.6)."

**8.6.** "A `const` function and a `const` package are addresses: the name always holds the
same code, or the same file. A function can still change a variable outside itself (7.8), and
the names of a file follow their own `const` or `var` (11.3). Neither is an exception to the
rule that a `const` never changes: the `const` itself, the address, stays the same."
**(filled in)** "The names of a file behind a package" were one of three things that are not
reached through a `const` name.

**8.8, what `=` copies.**

| Type of both sides | What the left side gets |
|---|---|
| a package | the address of the same file: a copy reaches the same file (11.2) |

**8.12 (filled in).** A call through a variable counted every function of its type "in every
file of the program". A shared pointer was in use "at the top level of every file", where "the
variables of the other files are outside it". The table of outside variables had this row:

| An outside variable of a function | Where a call of the function reaches it |
|---|---|
| a top-level `var` of a file: of its own file, or of another file that it reaches through a package | under the same name in that file, and through every package that holds the file (above) |

**(filled in) A file's variable is one owner, however it is reached.** A package is not an
owner in this rule: it is the address of a file, and a file is never freed (11.5), so giving a
`var` package another file changes no owner. The top-level variables of a file are owners like
any others, and there is one instance of each file. So `counter.list` and the `list` that the
functions of the file `counter` use are the same owner, through whichever package the line
reaches it. The compiler sees which file a package holds when the package is an `import` that
is written out, or a `const` that was declared with one. For any other package, `g.list` counts
as the `list` of every file of the program that has the package's type (11.3).

**9.6.** "A function in a file is different. It sees the file's top-level names, because there
is one instance of each file (11.3)."

**11.7 (filled in).** In the error line, "`FILE` is the name of the source file with its `.my`,
as its path from the folder of the file you run."

### The rows that were in the quick reference and the appendices

**13, type forms and built-in functions.**

| Form | Meaning |
|---|---|
| `package("geom")` | the type of the file `geom` (11.2) |

| You write | Its type | What it does |
|---|---|---|
| `import("geom")` | `package("geom")` | gives the file `geom` as a package (11.2) |

**13, what copies and what shares.**

| You write | What happens |
|---|---|
| `var p: package("geom") = geom;` | `p` holds the same file as `geom`: a package is the file's address, and a copy of it reaches the same file (11.2) |
| `import("geom")` twice | the same file both times: there is one instance of each file (11.2) |
| `const area: (float)float = geom.area;` | one element of a package, declared like any name (11.6) |
| `&geom`, `^geom`, `@geom` | borrows, puts on the heap or moves the address that a package holds, like any variable (11.6) |

**13, what never changes.**

| Thing | Fixed by |
|---|---|
| the file a `const` package holds | its declaration (11.3) |

**13, what the compiler rejects.**

| Line | Reason |
|---|---|
| `shapes = import("other");` where `other` doesn't have exactly the names, kinds and types of `geom` | the two package types differ, so it is an ordinary type error (11.3) |
| `geom = import("other");` where `geom` is a `const` package | a `const` package always holds the same file (11.3) |
| `a.bump = a.bump;` where `bump` is a `const` in its file | a name of a file follows its own `const` or `var`, whatever the package is (11.3) |
| two files that import each other | imports can't go round in a circle: a file is set up before any file that imports it (11.2) |
| a statement such as `countUp();` in a file another file imports | an imported file holds only declarations (11.4) |
| `import(fileName)` | the file is written out in the code (11.2) |

**Appendix A (filled in).** A.2 said `File = { Statement }`, with the note "an imported file
holds only Declarations, 11.4". In A.3, the line for `"." NAME` listed "a package's name", and
"A package type, `package("geom")`, is one of the forms of `Type`." In A.4, `g.Name` was "a type
when `g` has a package type and `Name` is a type name in that file (11.3)", and the table had
this row:

| What is read | It is |
|---|---|
| `package` followed by `(` and a text | the type of a file (3.6). `package` is a reserved word, so nothing else starts with it |

**Appendix B (filled in).** B.2, B.6 and B.8 had these rows:

| The compiler rejects | Example | Rule |
|---|---|---|
| a statement that isn't a declaration, in an imported file | `area(1.0);` in `geom` | 11.4 |
| an `import` whose text isn't written out, or whose file doesn't exist | `import(fileName)`, `import("missing")` | 11.2 |
| `import` used in any way but a call by its own name | `const load: ([]char)package("geom") = import;` | 11.2 |
| a package type whose text isn't written out, or leads to no file. `package` with nothing after it is no type | `var g: package = import("geom");` | 11.2 |
| files that import each other, directly or through other files, and a file that imports itself. An `import` inside a function counts | `a` imports `b`, and `b` imports `a` | 11.2 |
| a name that the file of a package's type doesn't have | `geom.volume` when the file `geom` declares no `volume` | 11.3 |
| a package given a file of another package type | `shapes = import("other");` when `other` doesn't have exactly the names, kinds and types of `geom` | 11.3 |
| a new value for a `const` name of a file | `a.bump = a.bump;` | 11.3 |
| a `const` package given another file | `a = import("counter");` for a `const a` | 11.3 |

| The compiler checks that | Rule |
|---|---|
| a top-level variable of a file counts as one owner, through whichever package it is reached. Where the compiler doesn't see which file a package holds, a name counts as that name in every file of the program that has the package's type | 8.12, 11.6 |
| what is put into a variable of a file, through a package or not, lasts as long as the program | 8.5, 11.6 |

### Why: the rows that were in DECISIONS.md

| Choice | Why |
|---|---|
| A package is a file | Jack: "I think a package should be a file." One file is one scope, so every name comes from this file or from a package it imported, and you can see which |
| `const geom: package("geom") = import("geom");` | The import is an ordinary declaration in the one shape every name uses. `import` is a built-in like `convert`, and the type of what it gives is written `package("geom")`. Zig works the same way |
| Everything at a file's top level can be used from outside, for now | The simplest start |
| Statements that run go only in the file you run; an imported file holds only declarations | Nothing runs because a file was imported, so the run order never depends on imports. **(adjusted)**: the values of the declarations do run, once, when the file is set up. See section 13 |
| A package is a pointer to the file: one instance of each file. A package value is just that file's address, held like a function value. `const` on a package means the name always holds the same file, and the file's own variables follow their own `const` or `var`. A file is set up once, before any file that imports it | Jack: "a package is a pointer to the file: one instance of each file". Importing a file twice gives the same file, so every package that holds it reaches the same variables, and an import inside a function makes nothing new. A `const` package whose file has a `var` is "the same as a const function that changes a variable outside itself, so it isn't an exception to 'a const never changes'". With one instance there is nothing to copy or to size, and a function keeps no address for its file. "Imports can't go in a circle, so that order always exists" |
| A package's type is its file: `package("geom")` | Jack: "a package's type is its file: package("geom")". It is a type form written in the type slot, like `[3]int`, and not a call: a type is still never called. The text is written out, so the type is known before the program runs, and the plain type `package` goes away. Two package types are the same when the files have the same names, kinds and types, so giving a `var` package another file is an ordinary type check, and a function can take a package of a given shape. A copy of a package reaches the same file, and nothing is hidden: the type only ever means "that file", and a file can't exist twice |
| Importing one element is an ordinary declaration | Jack: "importing one element is an ordinary declaration". `const area: (float)float = geom.area;` needs no new syntax, and a variable of a file is borrowed with `&`, which is always fine because a file lasts the whole program |
| A package can be a `const` or a `var` | Jack: underneath it is like a function pointer. A package value is the address of the file's names, so it is held like a function, and a `var` one can be given another file of the same type |
| Ownership and borrowing apply to a package like any other variable: copy, borrow, heap, move | Jack: "ownership rules and stuff like that, borrowing should all apply to a package, just like any other variable". The value is the file's address, so each of them passes the address around, and the file stays where it is |
| Two files can't import each other | A file is set up before any file that imports it (the row on a package), and in a circle no file could come first **(filled in)** |
| `import` takes one text written out in the code, and is always called by its own name. `import("geom")` has the type `package("geom")`, so `import` has no single type of its own. The text is the file's path without `.my`. A text that leads to no file is an error, two texts that lead to one file name the same file, and a file can't import itself | The compiler has to know every file before the program runs, or it couldn't check a single name. This is the one built-in that can't be passed around. *Principle: the compiler checks what it can.* **(filled in)**, and **(adjusted)** for the sentences in DOCS 13 and 4.4: section 13 |
| An `import` only gives the address of its file: nothing of the file runs at that line, so it can be written at the top level or inside a function, and run any number of times. The program starts by setting up every file, each one after the files it imports, in the order in which the importing file's text first names them. An `import` inside a function counts, whether or not it ever runs. The file you run comes last | "A file is set up once, before any file that imports it" (the row on a package) leaves open which of two files comes first when neither imports the other. A declaration's value can be a call that prints, so a compiler needs one order, and the order the imports are written in is the order the code shows. Counting every `import` in the text means the order is known before the program runs, and a name reached through a package is always set up. Weighed: setting a file up when its first `import` runs, which would put a check at every `import` and let the order depend on which lines run; this. *Principles: explicit; the compiler checks what it can.* **(filled in)**, and **(adjusted)** for the sentence in DOCS 11.4: see section 13 |
| Two files have the same package type when each top-level name of the one is a top-level name of the other, with the same `const` or `var` and the same type, and each type name names the same type in both. The order of the declarations and the values make no difference. A package type can name any file, also one that isn't imported, and two files can name each other's types | Jack's "same names, kinds and types", made exact. A type name has to name the same type, or `g.Point` would be two types under one name. The order can't matter, because a name is reached by its name. A file that is only named in a type is read for its names and set up only when it is imported, so only imports can go in a circle. Weighed: the order of the declarations as part of the type, which would make two files differ for a reason that no line using them can see; this. *Principles: strict types; the compiler checks what it can.* **(filled in)** |
| Underneath, a package value is the address of its file's table of names, one table for each file, and files of one package type have their tables laid out alike. What a file works out when it is set up stays in the file's own place until the program ends: every `var`, every `const` that isn't ready from the start, and the kept blocks of the calls made at its top level. That place is on the stack, below everything that runs afterwards | It is what makes `shapes.area` work for any file with nothing looked up while running. One address is one size, so a `var` package can be given any file of its type. A file's variables are values like any other, so they live on the stack unless their type says `^` (DOCS 2.6), and nothing goes on the heap out of sight. *Principle: say what happens underneath.* **(filled in)** |
| The `const` rules hold for the name that holds a package: a `var` borrower and an ownership need a `var` package, and through a `const` borrower the package can't be given another file. The file's own names follow their own `const` or `var` through all of them | "Just like any other variable" (the row above on ownership). What such a name holds is the address, so that is what its `const` protects. *Principle: one rule, no special cases.* **(filled in)** |
| The file you run is a file like every other: one instance, set up when the program starts, after the files it imports. No file of the program can import it | "A package is a file" (this section), with no special case for the first one. An import of it from inside its own program would close a circle. *Principle: one rule, no special cases.* **(filled in)** |
| A file lasts as long as the program. A borrower of a file's variable and a function of a file never outlive the file, though a function still can't give such a borrower back, and what is put into a file's variable has to last as long as the program. For the rule in 5.6, a file's variable is one owner through whichever package it is reached, and a package itself is not an owner. Where the compiler doesn't see which file a package holds, a name counts as that name in every file that has the package's type | "One instance of each file" (the row on a package), carried into the ownership rules. A name reached through a package is a top-level name of a file, so it follows the rules for a top-level name wherever the line is written. Giving a `var` package another file frees nothing. Where the compiler can't tell which file a package holds, it counts the worst case, as it does for a call through a variable (section 6). *Principles: one rule, no special cases; the compiler checks what it can.* **(filled in)** |
| A basic type is a name, or a name reached through a package: `geom.Point` can stand wherever a type is written | A name from another file is always reached through its package (section 9), and a type is a name like any other. Two files of one package type name the same type under each type name (section 9), so the type is known whichever file the package holds. *Principle: one rule, no special cases.* **(filled in)** |


In those rows, "section 13" is the list of adjusted rules in `DECISIONS.md`, which had these two
rows for packages:

| Where | Old wording | New wording | Why |
|---|---|---|---|
| DOCS 11.4 | "Nothing runs because a file was imported, so the order the program runs in never depends on imports." | "All that runs for an imported file is the working out of those declarations' values, once, when the file is set up. Nothing runs at the line of an `import`. So the order the program runs in is fixed before it starts: every imported file is set up first, and then the file you run goes from the top." | A declaration's value can be a call, so setting a file up does run something. What Jack wanted still holds: no statement runs, and nothing runs because of where an `import` is written |
| DOCS 13 and 4.4 | In 13: "Built-ins are first-class values like everything else: they can be named, stored and passed." In 4.4: "Like every built-in, it's an ordinary first-class value" | The same, with one exception: `import` is always called by its own name with its text written out. In 4.4: "Like every built-in except `import` (11.2)" | The compiler has to know every file of a program before it runs |

`package` was the eighth reserved word **(filled in)**, because `package("geom")` is a type form
and not a call, and a type can stand where a value is expected: as an ordinary built-in name,
`const Geom: type = package("geom");` would read as a call.
