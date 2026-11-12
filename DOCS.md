# MyLang documentation

MyLang is a **declarative, explicit, first-class** language.

- **First-class:** everything is a value. Numbers, arrays, maps, functions, structs and
  even types are all declared, stored, passed and returned the same way.
- **Explicit:** reading the code tells you what happens. A value goes on the heap only when
  its type says `^`, it is shared only when you write `&`, and it changes owner only where
  you write `@`. Otherwise every value is copied when it's declared, assigned or passed.
- **Declarative:** every name is declared in one shape that states what kind of name it
  is, what it's called, what type it has and what value it starts with. There are no loops:
  repeating is a function calling itself. A choice is a lookup.

This file documents the language: values, names, types, arrays, maps, functions, ownership
(the heap, borrowing and moving), structs, choosing and repeating, and files.

Why each rule is the way it is is in `DECISIONS.md`.

**Reading the examples**

- Code blocks are marked `go` only for colouring. The code is MyLang.
- Most examples use `int`, `float` and `bool`. Every built-in type is listed in 3.1.
- `//` starts a comment.
- `// ERROR` marks a line the compiler rejects.
- `...` stands for code that's left out.
- **(filled in)** Each example is its own piece of code. An example can use names from the
  examples before it, and it can also declare a name again that an earlier example declared.
  In one real file that second declaration would be an error (2.8).
- **(filled in)** marks a rule that Claude added, and **(adjusted)** marks one that Claude
  changed. `FILLED_IN.md` lists them all.

## Contents

1. [Everything is a first-class value](#1-everything-is-a-first-class-value)
2. [Names and declarations](#2-names-and-declarations)
3. [Types](#3-types)
4. [Types are values](#4-types-are-values)
5. [Arrays](#5-arrays)
6. [Maps](#6-maps)
7. [Functions](#7-functions)
8. [Ownership: plain, heap and borrowed values](#8-ownership-plain-heap-and-borrowed-values)
9. [Structs](#9-structs)
10. [Choosing and repeating](#10-choosing-and-repeating)
11. [Files and packages](#11-files-and-packages)
12. [Putting it together](#12-putting-it-together)
13. [Quick reference](#13-quick-reference)

---

## 1. Everything is a first-class value

Functions, arrays, maps, structs and types are values, exactly like numbers. Any value
can be:

| You can… | Example |
|---|---|
| give it a name | `const add: (int, int)int = …;` |
| put it in an array | `const ops: [2](int, int)int = [add, sub];` |
| put it in a map | `const byKey: {int}(int, int)int = {1: add, 2: sub};` |
| pass it to a function | `twice(double, 3)` |
| return it from a function | `return f;` |

There are no special cases. An array of functions, a map of arrays and a function that
returns a function all use the same rules, applied again.

---

## 2. Names and declarations

### 2.1 One shape for every declaration

```go
kind name: Type = value;
```

| Part | What it says |
|---|---|
| `kind` | what sort of name it is: `const` or `var` |
| `name` | the name |
| `Type` | what the value is (section 3) |
| `value` | the value it starts with |

```go
const limit: int = 10;
var count: int = 3;
const add: (int, int)int = (const x: int, const y: int){ return x + y; };
const scores: [3]int = [10, 20, 30];
```

A number, a function and an array are all declared the same way.

One mark can go in front of the name: `@`, when the declaration takes a value over from
another variable, as in `var @q: ^int = @p;` (8.9).

### 2.2 The two kinds of name

| Kind | Can it be given a new value? |
|---|---|
| `const` | no |
| `var` | yes |

Where a value lives and who owns it is not a kind of name. It is part of the type
(section 8).

### 2.3 `const` and `var`

A `const` keeps the value it's declared with. A `var` can be given a new value of the same
type as often as you like:

```go
const limit: int = 10;
limit = 11;                // ERROR: limit is const

var count: int = 3;
count = 4;                 // fine: count is var
count = 5;                 // and again
```

### 2.4 Declaring makes a new value

A declaration always gets its own value, a copy of what it's given:

```go
var a: int = 3;
var b: int = a;            // b is a new value, a copy of a
b = 7;                     // a is still 3
```

This holds for every type. Declaring an array, a map, a function or a struct from another
one copies the whole value:

```go
var first: [3]int = [1, 2, 3];
var second: [3]int = first;    // a copy of all three elements
second[0] = 9;                 // first is still [1, 2, 3]
```

To use the *same* value instead of a copy, borrow it with `&` (section 8).

### 2.5 Statements end with `;`

Every statement ends with `;`, including a declaration whose value is a function body or
a struct block:

```go
const add: (int, int)int = (const x: int, const y: int){
    return x + y;
};                         // the ; ends the declaration
```

### 2.6 Where values live, and for how long

Every value lives on the stack, unless its type says `^`, which puts it on the heap
(section 8). A value on the stack has a fixed size, which is why the length of an array
(5.3) and the keys of a map (6.3) never change.

A **block** is the code between `{` and `}`: a function body, or a block written on its
own. How long something lives depends on where it's written:

| What | It lives until… |
|---|---|
| a name | the end of the block it's declared in |
| a parameter | its function call ends, or as long as a function the call gives back uses it (7.8) |
| a value that isn't given a name | the end of its statement |
| a value on the heap | its owner ends, or gives it up (8.11) |

```go
var a: int = 1;
{
    var b: int = 2;        // b starts here
    a = a + b;             // a is now 3
}                          // b ends here
                           // a is still here

4;                         // a value with no name: it's gone once the statement is done
```

When a name ends, its value is removed from the stack. If the name owns a value on the
heap, that value is freed too.

### 2.7 Writing the source

**(filled in)** A source file is plain text in UTF-8, and its file name ends in `.my`.

**Characters.** Outside comments and text, a file uses only the letters, digits and symbols
of ASCII, with spaces, tabs and line breaks between them. A comment or a piece of text (5.1)
can hold any character.

**Spaces, tabs and line breaks only separate things.** They can go between any two words,
numbers or symbols, and a statement can run over several lines. A statement ends at its `;`
(2.5), never at a line break.

**Comments.** `//` starts a comment, and the comment runs to the end of the line. There is
no other kind of comment.

**Names.** A name starts with a letter or `_`. After that come letters, digits and `_`.
Case matters, so `point` and `Point` are two different names:

```go
count   total2   _spare   makePoint   Point      // five names
2total                                           // ERROR: a name can't start with a digit
```

**Reserved words.** Seven words belong to the language and are never names:

```
const   var   return   void   true   false   null
```

**Built-in names.** The built-in types and functions are ordinary names that are already
declared when a file starts:

| Built-in names | What they are |
|---|---|
| `int8` `int16` `int32` `int64` `uint8` `uint16` `uint32` `uint64` `float32` `float64` `int` `uint` `float` `bool` `char` `type` `any` `package` | types (3.1) |
| `convert` `wrap` `unwrap` `import` `print` `read` `join` `exit` | functions (13) |

A name is declared only once (2.8), so a file can't declare any of them again:

```go
const int: type = int8;        // ERROR: int is already declared
var print: int = 3;            // ERROR: print is already declared
```

**Symbols.** These are all the symbols:

```
=   ;   :   ,   .   (   )   [   ]   {   }
+   -   *   /   %   ==   !=   <   >   <=   >=   &&   ||   !
^   &   @   ?   "   '   //
```

Where two symbols could be read as one longer symbol, the longer one is read: `<=` is one
symbol, not `<` and then `=`. `&&` is always read as one symbol, and where it stands says
whether it means *and* or two borrow marks (3.2).

How numbers, `char`s and text are written is in 3.2 and 5.1.

### 2.8 A name is declared once

**(filled in)** A name can be seen from its declaration to the end of its block (2.6): in
that block, and in every block and function body inside it (7.8).

**A name can't be declared where the same name can already be seen.** So a name means one
thing wherever it is read. This holds for every way of declaring a name: a `const`, a `var`,
a parameter (7.4), a field in a struct value (9.2), and the built-in names (2.7).

```go
var count: int = 3;
var count: int = 4;            // ERROR: count is already declared in this block
{
    var count: int = 5;        // ERROR: the count of the outer block can be seen here
    var inner: int = 1;
}
var inner: int = 2;            // fine: the first inner ended at its }

const twice: (int)int = (const count: int){ return count * 2; };   // ERROR: count can be seen here
const triple: (int)int = (const n: int){ return n * 3; };          // fine
const square: (int)int = (const n: int){ return n * n; };          // fine: triple's n ended with its body
```

Two names can be spelled the same when neither can be seen from the other, like the two
`n`s above. A name behind a `.` (`p.x`, `geom.area`, `a.length`) is found inside the value in
front of the `.`, so it never clashes with a name declared somewhere else.

**A name is used only below its declaration.** The value of a declaration uses names declared
above it. There are three exceptions, and each is safe because nothing runs too early:

| Exception | Why it is safe |
|---|---|
| a function's body can use any top-level name of its own file (11.4) | the body runs only when the function is called |
| a function's body can use the function's own name (7.8) | the same reason |
| a type can use any top-level type of its own file, itself included (4.6) | a type is known before the program runs, so nothing runs at all |

A top-level name can be seen in every function body of its file. So no parameter and no name
inside a function can be spelled like a top-level name of the same file.

**How a name is found.** The compiler looks for a name in the block where it is used, then
in each block around it, out to the top level of the file, and last among the built-in names.
The first rule above means it finds at most one. It never looks in another file: a name from
another file is always reached through a package, as in `geom.area` (11.2).

### 2.9 Statements

**(filled in)** A block holds statements, and so does the top level of the file you run
(11.4). There are five kinds:

| Statement | Form | Example |
|---|---|---|
| a declaration | `kind name: Type = value;` | `var count: int = 3;` (2.1) |
| an assignment | `place = value;` | `count = 4;` |
| a value on its own | `value;` | `countUp();` |
| a return | `return value;` or `return;` | `return x + y;` (7.1, 7.3) |
| a block | `{ … }` | a smaller scope (2.6) |

Every statement ends with `;` (2.5), except a block, which ends at its `}`.

**Any value can stand on its own as a statement.** It is worked out, and then it is gone
(2.6). A call is the usual case. `4;` is allowed too, and does nothing.

**An assignment gives a place a new value.** A place is somewhere a value is held:

| Place | Example | It can be assigned when |
|---|---|---|
| a name | `count = 4;` | the name is a `var` (2.3) |
| an element | `scores[0] = 15;` | the array can be assigned (5.2) |
| a map entry | `seen[4] = true;` | the map can be assigned, and the key is in it (6.3) |
| a field | `p.x = 5;` | the struct can be assigned (9.3) |
| what a pointer leads to | `p^ = 6;` | `p` is a `var` heap owner, a `var` borrower or a `var` ownership (8.3, 8.6, 8.9) |
| an ownership | `@q = @p;` | `q` can be assigned. This is a move (8.9) |

The forms combine, read from left to right: `list^.next^.value = 3;`, `table[0][1] = 12;`,
`both[0]^ = 7;`. The rule for all of them is one rule: **what a `const` name reaches can't be
changed through that name.** So the left side of an assignment starts at a `var` name, or at
a value with no name that is then followed with `^`:

```go
second(&list^)^.value = 5;     // fine: the call gives a borrower, and the place is behind it (8.13)
makePoint(1, 2).x = 5;         // ERROR: the struct has no name, so nothing could ever read the 5
scores.length = 4;             // ERROR: a length never changes (5.5)
```

**An assignment is a statement, not a value.** It can't be written inside another value:

```go
a = b = 3;                     // ERROR: b = 3 is a statement, not a value
```

**An assignment works out its right side first.** Then it finds the place on the left, frees
what the place owned (8.8), and puts the new value in. A call uses the same order: every
argument is worked out before any parameter is set (7.2). Values come first, and places
after.

```go
var scores: []int = [10, 20, 30];
var i: int = 0;
const next: ()int = (){
    i = i + 1;
    return i;
};

scores[i] = next();            // next() runs first and makes i 1, so this sets scores[1] to 1
```

**Everything else is worked out from left to right:**

| In | The order |
|---|---|
| `a + b`, `a < b` and every operator with two sides | the left side, then the right side |
| `a && b`, `a \|\| b` | the left side, then the right side only when it's needed (3.2) |
| `[a, b, c]`, a struct value, a map value | the parts in the order they are written. In a map value, each key before its value |
| `a[i]`, `a[i:j]` | `a`, then `i`, then `j` |
| `f(a, b)` | `f`, then `a`, then `b`, and then the parameters are set (7.2) |

---

## 3. Types

### 3.1 Basic types

These types are built in:

| Type | Holds | Size |
|---|---|---|
| `int8` | whole numbers from -128 to 127 | 8 bits |
| `int16` | whole numbers from -32,768 to 32,767 | 16 bits |
| `int32` | whole numbers from -2,147,483,648 to 2,147,483,647 | 32 bits |
| `int64` | whole numbers from -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 | 64 bits |
| `uint8` | whole numbers from 0 to 255 | 8 bits |
| `uint16` | whole numbers from 0 to 65,535 | 16 bits |
| `uint32` | whole numbers from 0 to 4,294,967,295 | 32 bits |
| `uint64` | whole numbers from 0 to 18,446,744,073,709,551,615 | 64 bits |
| `float32` | numbers with a decimal point, to about 7 digits | 32 bits |
| `float64` | numbers with a decimal point, to about 15 digits | 64 bits |
| `bool` | `true` or `false` | 8 bits |
| `char` | one character | 8 bits |

A `u` in front means unsigned: the number is never negative.

Underneath, a `bool` is a number: `false` is 0 and `true` is 1. Its type still keeps it apart
from the numbers, so it is never used as one without a conversion (3.2).

Three everyday names stand for the 64-bit types:

| Name | Is the same type as |
|---|---|
| `int` | `int64` |
| `uint` | `uint64` |
| `float` | `float64` |

These are ordinary names for those types, the same as naming a type yourself (4.1), so an
`int` and an `int64` are the same type.

Three more built-in types have their own sections: `type`, the type of a type (4.1),
`any`, a value of any type (4.5), and `package`, a file used from another file (11.2).

### 3.2 Writing basic values

```go
const total: int = 42;
const price: float = 9.5;
const ready: bool = true;
const letter: char = 'a';
```

| You write | It is |
|---|---|
| `42`, `-7` | a whole number |
| `9.5`, `3.0` | a number with a decimal point |
| `true`, `false` | a `bool` |
| `'a'` | a `char` |

**(filled in)** The exact forms:

| Kind | Written as | Examples |
|---|---|---|
| a whole number in base ten | digits | `42`, `0`, `1000` |
| a whole number in base sixteen | `0x`, then digits and the letters `a` to `f`, in either case | `0xFF`, `0x1f` |
| a whole number in base two | `0b`, then `0`s and `1`s | `0b1010` |
| a number with a decimal point | digits, a point, digits, and then optionally `e` and a power of ten | `9.5`, `3.0`, `1.5e3`, `2.0e-3` |
| a `char` | one character between two `'`, or one escape | `'a'`, `'\n'` |
| a text | characters and escapes between two `"` (5.1) | `"Jack"` |

- A number with a decimal point has a digit on both sides of the point: `0.5` and `3.0`,
  never `.5` or `3.`.
- `e` multiplies by a power of ten: `1.5e3` is `1500.0`. The `e` can be written `E`, and the
  power can carry a `+` or a `-`.
- A number can't run straight into a name: `2total` is an error (2.7).
- `true` and `false` are the two `bool` values, and `null` is the pointer that points at
  nothing (8.10). All three are reserved words (2.7).

An escape writes a character that can't be typed between the quotes:

| Escape | The character |
|---|---|
| `\n` | new line (number 10) |
| `\t` | tab (number 9) |
| `\r` | carriage return (number 13) |
| `\0` | the character with number 0 |
| `\\` | `\` |
| `\'` | `'` |
| `\"` | `"` |
| `\x41` | the character whose number is written with two base-sixteen digits: here 65, `'A'` |

A `\` followed by anything else is an error. A `char` is 8 bits (3.1): one byte, holding a
number from 0 to 255. The characters of ASCII have the numbers 0 to 127. A character outside
ASCII takes more than one byte in UTF-8, so it doesn't fit in a `char`:

```go
const e: char = 'é';           // ERROR: é is two bytes, and a char holds one
```

In a text it is simply two `char`s (5.1).

**A number written in the code takes the type of the place it's written for.** A whole
number fits any whole-number type it's in range for, and a number with a decimal point
fits either float type:

```go
var small: int8 = 5;           // 5 is an int8 here
var level: uint8 = 200;        // 200 is a uint8 here
var ratio: float32 = 0.5;      // 0.5 is a float32 here

var big: uint8 = 300;          // ERROR: 300 doesn't fit in a uint8
var below: uint8 = -1;         // ERROR: a uint8 is never negative
var half: int = 2.5;           // ERROR: 2.5 isn't a whole number
var rate: float = 3;           // ERROR: write 3.0
```

**(filled in)** The place a written number takes its type from is the nearest thing around it
that has a type: the declaration or assignment it's the value of, the parameter it's passed
to, the `return` it's given back by, the array, pair, map or struct value it's written in, or
the other side of an operator. The type passes inwards through brackets, through a `-` in
front and through `+`, `-`, `*`, `/` and `%`:

```go
var small: int8 = (2 + 3) * 4;     // 2, 3 and 4 are all int8s here
const isFive: bool = small == 5;   // 5 takes the type of the other side: an int8
```

**(filled in)** A written number with no place to take a type from is an `int`, or a `float`
when it has a decimal point: in `4;` the `4` is an `int`, and in `3 > 2` both numbers are
`int`s.

**A value never changes type by itself.** Once a value has a type, using it as another
type takes a conversion that you write out (4.4):

```go
var count: int = small;                       // ERROR: small is an int8
var count: int = convert(int8, int)(small);   // fine: the conversion is written
```

**`+`, `-` and `*` work on two numbers of the same type** and give a number of that type:

```go
var sum: int = count + 2;                             // 2 is an int here
var mixed: int = count + small;                       // ERROR: an int and an int8
var fixed: int = count + convert(int8, int)(small);   // fine
```

**(filled in) `/` divides and `%` gives the remainder.** Like `*`, each works on two numbers
of the same type and gives a number of that type:

| You write | It gives | Examples |
|---|---|---|
| `a / b` on whole numbers | how many whole times `b` goes into `a`. The part after the decimal point is dropped, as in a conversion (4.4) | `7 / 2` is `3`, and `-7 / 2` is `-3` |
| `a % b` on whole numbers | what is left over, `a - (a / b) * b`. It has the sign of `a` | `7 % 2` is `1`, and `-7 % 2` is `-1` |
| `a / b` on floats | `a` divided by `b` | `7.0 / 2.0` is `3.5` |

`%` works on whole numbers only.

**(filled in) `-` in front of a number gives its negative.** It works on a signed whole
number or a float, and never on an unsigned number. A `-` in front of a written number is
part of that number when its type is checked, so `-128` fits in an `int8` although `128`
doesn't:

```go
var low: int8 = -128;          // fine: -128 is in range for an int8
var flipped: int = -count;     // the negative of count
var up: int = -(count - 10);   // brackets first, then the -
var off: uint8 = -level;       // ERROR: level is a uint8, and a uint8 is never negative
```

**(filled in) A result always fits its type.** `+`, `-`, `*`, `/`, `%` and a `-` in front
give a number of the type that went in. When the true answer doesn't fit in that type, the
program fails while it runs (11.7), the same as a conversion that doesn't fit (4.4). A number
never wraps around, and it never turns into another type.

| Line | Fails while running when |
|---|---|
| `a + b`, `a - b`, `a * b`, `-a` | the answer is outside the range of the type |
| `a / b`, `a % b` on whole numbers | `b` is 0, or the answer is outside the range: `-128 / -1` in `int8`s |
| any of them on floats | the answer is too big for the type, or isn't a number at all: `1.0 / 0.0`, `0.0 / 0.0` |

So a float is always an ordinary number: there is no infinity and no "not a number" value,
and dividing by `0.0` fails like dividing by `0`. Floats follow the IEEE 754 standard that
processors use: a `float32` is its 32-bit format and a `float64` its 64-bit format. An answer
that can't be held exactly is rounded to the nearest value that can, and one too small to
tell from zero becomes zero.

The compiler makes the check itself where it already knows the answer: when every number is
written in the code, and when a whole number is divided by a written `0`:

```go
var over: int8 = 100 + 100;    // ERROR: 200 doesn't fit in an int8
var none: int = count / 0;     // ERROR: division by zero
```

**Comparing two values of the same type gives a `bool`:**

| Operator | `true` when |
|---|---|
| `==` | the two values are equal |
| `!=` | the two values are different |
| `<` | the left value is smaller |
| `>` | the left value is bigger |
| `<=` | the left value is smaller or equal |
| `>=` | the left value is bigger or equal |

```go
const isBig: bool = count > 10;
const isFive: bool = small == 5;       // 5 is an int8 here
const wrong: bool = count == small;    // ERROR: an int and an int8
```

**(filled in)** Which types each comparison works on:

| Operators | Work on |
|---|---|
| `==` `!=` | two values of the same type, whatever the type. What "equal" means for each type is in 3.7 |
| `<` `>` `<=` `>=` | two numbers of the same type, or two `char`s, which compare by their numbers |

A comparison gives a `bool`, and a `bool` has no order, so comparisons don't chain:

```go
const between: bool = 0 < count < 10;         // ERROR: 0 < count is a bool, and < doesn't work on a bool
const inside: bool = 0 < count && count < 10; // fine
const early: bool = 'a' < 'b';                // true: 97 is less than 98
```

**`&&`, `||` and `!` work on `bool`s and give a `bool`:**

| You write | Read it as | It is `true` when |
|---|---|---|
| `a && b` | `a` and `b` | both are `true` |
| `a \|\| b` | `a` or `b` | at least one of them is `true` |
| `!a` | not `a` | `a` is `false` |

```go
const inRange: bool = count > 0 && count < 10;
const stop: bool = !ready || count > 10;
const mixed: bool = count && ready;    // ERROR: count is an int, not a bool: write count != 0
const empty: bool = !count;            // ERROR: count is an int, not a bool: write count == 0
```

**`&&` and `||` work out the left side first, and the right side only when it's needed.** A
`false` on the left of `&&` already makes the answer `false`, and a `true` on the left of
`||` already makes it `true`, so in those two cases the right side is skipped:

```go
const big: bool = p != null && p^ > 10;    // p^ is followed only when p isn't null (8.10)
```

`&&` between two values is *and*. In front of a value or inside a type it is two borrow
marks: `&&int` is a borrower of a borrower (3.4). Where it stands tells the two apart.

**The order of operations** is the usual one, as in maths and most programming languages.
Each row of this table goes before the rows below it:

| Step | What | Examples |
|---|---|---|
| 1 | what is behind a value: following, indexing, calling, a field (8.2) | `p^` `a[0]` `f(1)` `pt.x` |
| 2 | what is in front of a value: `!`, `-` and the marks (8.2) **(filled in: `-`)** | `!ready` `-count` `^v` `&x` `@x` |
| 3 | `*` `/` `%` **(filled in: `/` and `%`)** | `limit * 2` `total / 2` |
| 4 | `+` `-` | `count + 1` |
| 5 | the comparisons | `==` `!=` `<` `>` `<=` `>=` |
| 6 | `&&` | `a && b` |
| 7 | `\|\|` | `a \|\| b` |

Operators in the same step go from left to right.

```go
count + 1 > limit * 2 && !ready
// && joins two tests: count + 1 > limit * 2, and !ready
// >  compares two numbers: count + 1, and limit * 2

a || b && c            // a, or both of b and c
10 - 4 - 3             // 3: 10 - 4 goes first
!seen[4]               // the opposite of seen[4]: the index goes first
```

**(filled in)** A `-` between two values subtracts. A `-` where a value starts is the minus in
front:

```go
a - -b                 // a, minus the negative of b
-a * b                 // the negative of a, times b: what is in front goes before *
10 / 4 * 2             // 4: 10 / 4 is 2, then 2 * 2
```

**Brackets change the order.** What is inside a pair of brackets is worked out first:

```go
(count + 1) * 2        // the sum first, then * 2
(a || b) && c          // a or b, and c as well
!(ready && count > 0)  // true unless both tests are true
```

The order says how a line is read, not when each part runs: the right side of `&&` and `||`
is still skipped when it isn't needed.

**What a `(` starts.** Brackets also write function types (3.3), function values (7.1) and
calls (7.2). Where a `(` stands tells them apart:

| A `(` … | Starts | Example |
|---|---|---|
| right after a value | a call | `f(1)`, `first(bool)([true, false])` |
| holding parameters, each with `const` or `var`, or empty, then followed by `{` | a function value | `(const x: int){ … }` |
| holding types, or nothing, then a return type | a function type | `(int, int)int`, `()void` |
| holding one value that isn't a type | a group | `(count + 1) * 2` |

The compiler knows which values are types, because their type is `type` (4.1). So `(a)`
starts a function type when `a` is a type, and a group when it isn't. A type is never put in
a group: in a type, `(` always starts a function type, and no type needs grouping (3.4).

### 3.3 Every modifier goes in front

A type is a basic type, or a modifier followed by a type. A basic type is a built-in type
(3.1) or the name of a type (section 4).

| Kind | Form | Example | Read as |
|---|---|---|---|
| fixed array | `[n]T` | `[3]int` | array of 3 ints |
| unsized array | `[]T` | `[]int` | array of ints |
| pair | `?T` | `?int` | two ints: one for `false`, one for `true` |
| map | `{K}V` | `{int}bool` | map from int keys to bool values |
| function | `(P1, P2)R` | `(int, int)int` | takes two ints, returns an int |
| function that returns nothing | `(P1, P2)void` | `(int)void` | takes an int, returns nothing |
| struct | `{T1, T2}` | `{bool, int}` | a struct with a bool field and an int field |
| on the heap | `^T` | `^int` | an int on the heap, owned by this holder (section 8) |
| borrowed | `&T` | `&int` | an int that another holder owns (section 8) |
| ownership | `@T` | `@int` | the ownership of an int variable (8.9) |

### 3.4 Modifiers stack in any combination

Read every type from left to right, one modifier at a time:

```go
[3](int)int            // array of 3 → functions taking int → returning int
(int)[]int             // function taking int → returning array of ints
(int)(int)int          // function taking int → returning a function (int)int
((int)int, int)int     // function taking a function (int)int and an int → returning int
{int}(int)int          // map from int → to functions (int)int
{int}[3]int            // map from int → to arrays of 3 ints
[3]{int}bool           // array of 3 → maps from int to bool
[2][3]int              // array of 2 → arrays of 3 ints
[3]^int                // array of 3 → ints on the heap
^[3]int                // on the heap → an array of 3 ints
[2]&int                // array of 2 → borrowed ints
(&int)void             // function taking a borrowed int → returning nothing
(@^int)void            // function taking the ownership of a heap int → returning nothing
(int)^[3]int           // function taking int → returning an array of 3 ints on the heap
```

Swapping two modifiers swaps the meaning, and the order shows it:

| Type | Meaning |
|---|---|
| `[3](int)int` | array of 3 functions |
| `(int)[3]int` | function returning an array of 3 |
| `[2]{int}bool` | array of 2 maps |
| `{int}[2]bool` | map whose values are arrays of 2 |
| `[3]^int` | three ints, each on the heap |
| `^[3]int` | one array of 3 ints on the heap |

No type ever needs brackets to group it, and none is grouped: in a type, `(` always starts a
function type (3.2).

### 3.5 Types read in the order you use the value

```go
// f: (int)(bool)[3]int
f(1)(true)[0]          // call with an int, call with a bool, index → an int
```

The type says *call, call, index, int*, and the use says *call, call, index*: the same
order.

The heap and borrow marks work the same way (8.2):

```go
// p: ^[3]int
p^[0]                  // follow, index → an int

// ps: [3]^int
ps[0]^                 // index, follow → an int
```

### 3.6 Grammar

```
Type      = BasicType
          | "[" [ NUMBER ] "]" Type           (array: fixed or unsized)
          | "?" Type                          (pair: one Type for false, one for true)
          | "{" Type "}" Type                 (map: {key}value)
          | "{" TypeList "}"                  (struct: its field types, section 9)
          | "^" Type                          (a value on the heap, section 8)
          | "&" Type                          (a borrowed value, section 8)
          | "@" Type                          (the ownership of a variable, section 8)
          | "(" [ TypeList ] ")" Return       (function: parameter types, then the return type)
Return    = Type | "void"                     (void = returns nothing)
TypeList  = Type { "," Type }
```

A function type always ends with its return type. A function that returns nothing ends
with `void`: `(int)void` takes an int and returns nothing, and `()void` takes nothing and
returns nothing.

**(filled in)** Two more lines complete it:

```
BasicType = NAME { "." NAME }                 (a type's name, or one reached through a package: geom.Box)
NUMBER    = a whole number written in the code (3.2)
```

A type is written with these forms wherever a type is expected: after the `:` of a
declaration, a parameter or a field, and inside another type. The full grammar of the
language is in Appendix A.

**(filled in)** Two types are the same type when they are the same once every type name is
replaced by what it names (4.1). Only the shape counts: the names of parameters and fields,
and `const` and `var`, are not part of a type (7.4, 9.9).

### 3.7 What "equal" means

**(filled in) `==` compares exactly what `=` copies (8.8).** So a copy is always equal to the
value it was copied from, and that one rule says what "equal" means for every type:

| Type | `=` copies | So two values are equal when |
|---|---|---|
| a number, a `bool`, a `char` | the value | they are the same value |
| an array `[n]T` or `[]T`, a text | every element | they have the same length, and every element is equal |
| a pair `?T` | both values | both values are equal |
| a struct | every field (8.8) | every field is equal, field by field |
| a map | every key and its value | they have the same keys, and the values under each key are equal |
| a heap owner `^T` | the value on the heap | the values they own are equal, or both are `null` |
| a borrower `&T`, an ownership `@T` | the address | they point at the same place, or both are `null`. Two `&[]T` also need the same length |
| a function | its two addresses (7.7), and the heap value it owns, if it owns one (7.8) | they are the same code, and what they keep is the same place on the stack, or equal heap values |
| a `type` | the type | they are the same type |
| an `any` | the type inside and the value inside | the types inside are the same, and the values are equal |
| a `package` | the file's variables (11.5) | they are the same file, and every variable is equal |

`!=` is the opposite of `==`. Comparing copies nothing and changes nothing.

```go
var a: [3]int = [1, 2, 3];
var b: [3]int = a;
a == b                         // true: a copy equals its original
b[0] = 9;
a == b                         // false: the first elements differ

var p: ^int = ^5;
var q: ^int = p;               // q owns its own heap int, holding 5
p == q                         // true: = copied the value, so == compares the values
var r: &int = &p^;
var s: &int = &q^;
r == s                         // false: two different places
r^ == s^                       // true: both places hold 5

"Jack" == "Jack"               // true: two texts with the same chars
p == null                      // false: p owns a value
```

- The two sides always have the same type (3.2). A `[3]int` and a `[4]int` are different
  types, so comparing them is an error. Two `[]int` of different lengths are the same type,
  and they are simply not equal.
- A `null` takes the pointer type of the other side.
- Two floats are equal when they are the same number. Every float is an ordinary number
  (3.2), so a float always equals itself.
- Comparing two big values looks at every part, the way `=` copies every part. To ask
  whether two borrowers lead to equal values, compare what they lead to: `r^ == s^`.

### 3.8 The type of every value

**(filled in)** The compiler gives every value a type before the program runs. This table
says how, form by form. "Its place" is where the value is written: what it is declared as,
assigned to, passed to, returned as, or held in (3.2).

| Form | What it needs | Its type |
|---|---|---|
| a written number | to fit the type | the number type of its place. With no place: `int`, or `float` with a decimal point (3.2) |
| `true`, `false` | | `bool` |
| `'a'` | | `char` |
| `null` | a place | the `^T`, `&T` or `@T` of its place (8.10) |
| `[a, b, c]`, `"abc"` | every element is a `T` | the `[n]T` or `[]T` of its place, or its `?T` when there are two elements. With no place: `[n]T`, a fixed array of what the elements are (5.1) |
| `{ x: int = 1; y: int = 2; }` | each value has the type written in its field | the struct type with those field types, in order (9.2) |
| `{k1: v1, k2: v2}` | every key is a `K`, every value a `V` | the `{K}V` of its place. With no place: `K` and `V` are what the keys and values are (6.1) |
| `(const x: P1, var y: P2){ … }` | every `return` gives an `R` | `(P1, P2)R`, where `R` is the return type of its place. With no place: `R` is what its `return`s give, or `void` (7.1) |
| a name | | the type it was declared with |
| a type, such as `int`, `[3]int` or `Point` | | `type` (4.1) |
| `(a)` | `a` isn't a type | the type of `a` |
| `-a` | `a` is a signed whole number or a float | the type of `a` |
| `!a` | `a` is a `bool` | `bool` |
| `a + b`, `a - b`, `a * b`, `a / b` | two numbers of the same type | that type |
| `a % b` | two whole numbers of the same type | that type |
| `a == b`, `a != b` | two values of the same type | `bool` |
| `a < b`, `a > b`, `a <= b`, `a >= b` | two numbers of the same type, or two `char`s | `bool` |
| `a && b`, `a \|\| b` | two `bool`s | `bool` |
| `a[i]` | `a` is an `[n]T` or `[]T`, and `i` is an `int` | `T` (5.2) |
| `a[i]` | `a` is a `?T`, and `i` is a `bool` | `T` (5.8) |
| `a[i]` | `a` is a `{K}V`, and `i` is a `K` | `V` (6.2) |
| `a[i:j]` | `a` is an `[n]T` or `[]T`. `i` and `j` are `int`s, and either can be left out | `[]T`, a copy. With `&` in front: `&[]T`, a slice (5.9) |
| `a.length` | `a` is an array or a map | `int` (5.5, 6.3) |
| `s.x` | `s` is a struct, and `x` names one of its fields | the type of that field (9.9) |
| `g.x` | `g` is a `package`, and `x` is a top-level name of its file | the type of that name (11.2) |
| `x.T` | `x` is an `any` | `type` (4.5) |
| `f(a, b)` | `f` is a `(P1, P2)R`, `a` is a `P1` and `b` is a `P2` | `R` (7.2) |
| `^v` | `v` is a `T` | `^T` (8.3) |
| `&x` | `x` is a `T` | `&T` (8.4) |
| `@x` | `x` is a `T` | `@T` (8.9) |
| `p^` | `p` is a `^T`, an `&T` or an `@T` | `T` (8.2) |

Three rules go with the table:

- **Only a value written on the spot takes its type from its place:** a number, a `null`, an
  array or text value, a map value and a function value. Every other value already has a type,
  and it has to be exactly the type its place asks for (3.2).
- **A value with no place has to say enough by itself.** `[]` and `{}` hold nothing that says
  what `T`, `K` or `V` is, and a `null` doesn't say what it points at, so each needs a place.
- **A call to a function that returns nothing has no value.** It can only stand on its own as
  a statement (2.9):

```go
show(5);                       // fine
var x: int = show(5);          // ERROR: show returns nothing
```

---

## 4. Types are values

### 4.1 `type` is a type

A type is a value, and the type of that value is `type`. `type` is a type like any other,
so a type can be named, stored and passed around like every other value.

**Naming a type** uses `const`:

```go
const Pair: type = [2]int;
const Table: type = {int}[2]int;
const Op: type = (int, int)int;
const bitboard: type = uint;         // a basic type can be named too

var p: Pair = [1, 2];
const add: Op = (const x: int, const y: int){ return x + y; };
const ops: [2]Op = [add, add];       // an array of Op
var board: bitboard = 0;
```

A struct type is named the same way: `const Point: type = {int, int};` (9.1).

A named type can be used anywhere a type can, including inside other types (`[2]Op`). A
name for a type is the same type under another name, so a `bitboard` can be used wherever
a `uint` can:

```go
var plain: uint = board;             // fine: bitboard and uint are the same type
```

**`type` goes wherever a type goes,** including inside other types:

```go
const wholeTypes: [2]type = [int8, int];          // an array of types
const byCode: {int}type = {1: int, 2: bool};      // a map whose values are types

wholeTypes[1]                // int
byCode[2]                    // bool
wholeTypes.length            // 2
```

A function can take a type as a parameter too, which is how generics work (4.3).

**A type is a value, not a function, so it is never called:**

```go
int(3.0)                     // ERROR: int is a type, not a function; use convert (4.4)
```

### 4.2 Types never change

A named type is always a `const`, so it can't be given a new value. Every
value declared with it has a type that's known before the program runs:

```go
var T: type = int;                   // ERROR: a type can't be a var
var kinds: [2]type = [int, bool];    // ERROR: an array of types can't be a var either
```

A type can still be held as a value: in an array or a map (4.1), as a function's parameter
(4.3) or inside an `any` (4.5). It is always held as a `const`.

A value's type is the one it was declared with. The type inside an `any` is read with
`x.T` (4.5).

**(filled in) A type that decides the type of something is written out.** After a `:`,
inside another type, and as an argument that a function's type uses (4.3), a type is written
as a type: a name, or one of the forms in 3.6. A type that is worked out any other way, such
as `wholeTypes[1]` or `x.T`, is a value like any other. It can be compared, stored and
passed, but it can't declare anything:

```go
var n: wholeTypes[1] = 5;      // ERROR: write the type itself: int
x.T == int                     // fine: comparing two types gives a bool (3.7)
```

**(filled in) A type lasts as long as the program.** It is fixed before the program runs,
wherever it is written. So a type that is handed to a function is never an argument that ends
too soon (7.8): `first([3]int)` is as good as `first(int)`.

### 4.3 Generics: functions that take a type

A type is a value, so it can be passed to a function like any other value. A generic
function is just a function that takes a type as a parameter:

```go
const first: (type)([]T)T = (const T: type){
    return (const v: []T){
        return v[0];
    };
};

const firstInt: ([]int)int = first(int);      // call first with the type int: gives a function for int arrays
firstInt([7, 8])                              // 7: the first element

const firstBool: ([]bool)bool = first(bool);  // the same for bool arrays
firstBool([true, false])                      // true: the first element

first(bool)([true, false])                    // true: the same, in one line
```

The last line is two calls in a row: `first(bool)` makes the function for bool arrays, and
`([true, false])` calls that function straight away.

The function `first` gives back uses `T`, a parameter of `first`, so it keeps `T` (7.8).

`first(int)` gives back a function made for `int`, so what each call works with is written
right there in the code.

**(filled in) How a function type names the type it is given.** In `(type)([]T)T` the name
`T` isn't declared anywhere. Inside a function type, a name that isn't declared stands for a
type the function is given. The first such name is the function's first `type` parameter,
the second such name is its second, and so on:

```go
(type)([]T)T                   // T is the type handed to the type parameter
(type, type)(A)B               // A is the first type handed in, and B is the second
```

- The name is used only to the right of the `type` it stands for, up to the end of that
  function type.
- The name itself is not part of the type: `(type)([]T)T` and `(type)([]E)E` are the same
  type (3.6).
- The function value names its `type` parameters itself, `(const T: type)`, and they match
  the type by position, like every parameter (7.1). The code reads best when the two names
  agree, as in `first`.

**(filled in) Calling it.** At a call, the compiler puts the type that was handed in wherever
its name stands in the function's type: `first(int)` is a `([]int)int`. So the type handed in
is written out as a type (4.2) whenever the function's type uses it. Where the function's type
doesn't use it, any value of type `type` will do:

```go
const sameType: (type, type)bool = (const a: type, const b: type){
    return a == b;
};

sameType(x.T, int)             // fine: sameType's type doesn't use a or b
first(x.T)                     // ERROR: the type of first(x.T) would only be known while running
```

**(filled in) What a generic function can do with a `T`.** The compiler checks the function
once, for every type at the same time. Inside it, `T` is the name of a type, so it goes
wherever a type is written: `var x: T = v[0];`. A value of type `T` can be used in the ways
every type allows: it can be declared, copied, moved, borrowed, put on the heap, passed,
returned, wrapped (4.5), compared with `==` and `!=` (3.7), and held in arrays, structs and
maps. What only some types allow is an error. To add two `T`s, the caller passes the function
that adds them:

```go
const sumBad: (type)(T, T)T = (const T: type){
    return (const a: T, const b: T){ return a + b; };   // ERROR: + doesn't work on every type
};

const sumWith: (type)(T, T, (T, T)T)T = (const T: type){
    return (const a: T, const b: T, const plus: (T, T)T){
        return plus(a, b);     // fine: plus is a function for this T
    };
};

sumWith(int)(2, 3, add)        // 5: add is the (int, int)int from 7.1
```

**(filled in) Underneath, a `type` value is an address.** For every type the program uses,
the compiler writes a small description into the program: the type's size, and how to copy,
free and compare a value of it. A `type` value is the address of that description. An `any`
carries one (4.5), and a generic function reads one to handle a `T` whose size it learns only
when it is called.

### 4.4 Converting a value: `convert`

A value never changes type by itself (3.2). To get a value of another type you call
`convert`, a built-in function. It takes the type you convert **from** and the type you
convert **to**, and gives back the function that does the conversion:

```go
convert(float, int)(3.0)               // 3
convert(int, float)(3)                 // 3.0
var whole: int = convert(float, int)(2.0);
```

`convert(float, int)` is a function of type `(float)int`. Like every built-in, it's an
ordinary first-class value, so it can be named, stored and passed like any other function:

```go
const toInt: (float)int = convert(float, int);
toInt(2.0)                             // 2
```

Both types are always written, so the code shows exactly what changes into what.

What a conversion between the basic types gives:

| Conversion | Example | Result |
|---|---|---|
| a smaller whole-number type to a bigger one | `convert(int8, int)(small)` | the same number: it always fits |
| a whole number to a float | `convert(int, float)(3)` | `3.0` |
| a float to a whole number | `convert(float, int)(3.7)` | `3`: the part after the decimal point is dropped |
| a bigger type to a smaller one | `convert(int, int8)(100)` | `100`: checked when the line runs that it fits |
| between signed and unsigned | `convert(int8, uint8)(5)` | `5`: checked when the line runs that it fits |
| a `char` to its number, and back | `convert(char, uint8)('a')`, `convert(uint8, char)(97)` | `97`, `'a'` |

**(filled in)** The rest of the table:

| Conversion | Example | Result |
|---|---|---|
| a whole number to a float that can't hold it exactly | `convert(int, float32)(16777217)` | the nearest float: `16777216.0` |
| a float to a whole number it is out of range for | `convert(float, int8)(300.0)` | fails while running: checked that it fits, like the rows above |
| a `float32` to a `float64` | `convert(float32, float)(ratio)` | the same number: it always fits |
| a `float64` to a `float32` | `convert(float, float32)(0.1)` | the nearest `float32`, checked when the line runs that it isn't too big |
| a `char` to any whole-number type, and back | `convert(char, int)('a')`, `convert(int, char)(97)` | `97`, `'a'`: a `char` converts as the `uint8` it is underneath |
| a `bool` to any whole-number type, and back | `convert(bool, int)(true)`, `convert(int, bool)(0)` | `1`, `false`: a `bool` converts as the number it is underneath (3.1). Going back, only 0 and 1 fit |
| a number or a `bool` to text, and back | `convert(int, []char)(42)`, `convert([]char, int)("42")` | `"42"`, `42` (5.10) |
| a type to itself | `convert(int, int)(3)` | `3` |

`convert` works between the number types, `bool` and `char`, and between those and the text
type `[]char`, as the two tables list. Any other pair of types is an error:

```go
convert([3]int, int)                   // ERROR: there is no conversion from an array to an int
convert(bool, float)                   // ERROR: a bool converts to a whole number only
```

`convert` itself is a function that takes two types, so its own type is `(type, type)(A)B`
(4.3).

A conversion whose value doesn't fit fails while the program runs:

```go
convert(int, int8)(300)                // fails while running: 300 doesn't fit in an int8
convert(int8, uint8)(-1)               // fails while running: a uint8 is never negative
```

Because the conversion is always written out, every line where that can happen is visible
in the code.

### 4.5 `any`: a value of any type

`any` is a built-in type for a value whose type is only known while the program runs.

#### Wrapping and unwrapping

Nothing takes a value of any type, so two built-in functions move a value into an `any`
and back out. Each takes a type first and gives back the function for that type:

| You write | Its type | What it does |
|---|---|---|
| `wrap(int)` | `(int)any` | wraps an `int` in an `any` |
| `unwrap(int)` | `(any)int` | takes the `int` back out, checked when the line runs |

```go
var x: any = wrap(int)(3);         // holds the type int and the value 3
var y: any = wrap(bool)(true);     // holds the type bool and the value true

x.T                                // int: the type inside
unwrap(int)(x)                     // 3: checked while running that x holds an int
unwrap(bool)(x)                    // fails while running: x holds an int
```

`wrap(int)` and `unwrap(int)` are ordinary function values, so they can be named:

```go
const wrapInt: (int)any = wrap(int);
var z: any = wrapInt(7);           // holds the type int and the value 7
```

`x` itself has the type `any`. The type inside it is `x.T`.

**(filled in)** `wrap` and `unwrap` are functions that take a type, so their own types are
`(type)(T)any` and `(type)(any)T` (4.3). Any type can be wrapped, an `any` included.

**(filled in) An `any` keeps the type of its first value.** Its size comes from the value
inside (see "Underneath" below), and a value on the stack keeps its size (2.6). So a `var`
`any` can be given a new value of the same type, and giving it a value of another type fails
while the program runs (11.7). It is the rule for an unsized array, which keeps its first
length (5.3):

```go
var x: any = wrap(int)(3);
x = wrap(int)(4);                  // fine: x still holds an int
x = wrap(bool)(true);              // fails while running: x holds an int
var y: any = wrap(bool)(true);     // fine: a new declaration, with its own first value
```

#### Passing an `any`

A function can take an `any`, so it can be called with a value of any type:

```go
const describe: (any)void = (const x: any){
    x.T                    // the type it was given
    ...
};

describe(wrap(int)(3));
describe(wrap(bool)(true));
describe(3);               // ERROR: describe takes an any; write wrap(int)(3)
```

#### Every wrap and unwrap is written out

A value never turns into an `any` by itself, and an `any` can't be used where a fixed type
is expected until you unwrap it:

```go
const double: (int)int = (const n: int){ return n * 2; };

double(x);                 // ERROR: x is an any
double(unwrap(int)(x));    // fine: the unwrap is written out
```

So every place where something could go wrong while running is visible in the code: each
wrap is `wrap(int)(…)` and each unwrap is `unwrap(int)(…)`, `unwrap(bool)(…)` and so on.
Everything that doesn't touch an `any` is checked before the program runs.

#### What `any` is for

- **Data whose shape you only learn while running:** a file or a message that says "the
  next value is a bool".
- **Choosing a type while running:** for example a small number type for small data and a
  big one for big data.
- **Reading the type inside:** `x.T` tells you what the `any` holds.

#### Underneath

An `any` is a struct of two things: the value's type and the value. Pictured as a struct:

```go
var x: any = wrap(int)(3);
// pictured as a struct: the type, then a value of that type
var x: {type, int} = {
    T: type = int;                // x.T
    value: int = 3;
};

var y: any = wrap(bool)(true);
// a different shape inside, because the value is a bool
var y: {type, bool} = {
    T: type = bool;
    value: bool = true;
};
```

The second field's type depends on the first, so every `any` can have a different shape
inside. An `any` sits on the stack like everything else. Its size depends on the value
inside, so it's known when the `any` is made or passed in, the same as an unsized array
whose length comes from its first value (5.3).

#### `any` or generics?

Both let one function work with many types:

| | generic: takes a type (4.3) | `any` |
|---|---|---|
| call | you pass the type: `first(int)` | the type travels with the value: `describe(wrap(int)(3))` |
| type known | before the program runs, when you write the type | only while it runs |
| mistakes show up | before the program runs | when an unwrap like `unwrap(int)(x)` runs |
| best for | the same code for many types | values whose type you only learn while running |

### 4.6 A type that names itself

**(filled in)** A type declared at the top level of a file can use any top-level type of that
file, wherever it is declared, itself included (2.8). That is how a list node names the next
node (8.14):

```go
const Node: type = {int, ^Node};       // fine: the next node is behind a ^
const Even: type = {int, ^Odd};        // fine: Odd is declared below, and both are top-level types
const Odd: type = {int, ^Even};
```

A value has a fixed size (2.6), so a type can't hold itself directly. It reaches itself
through a pointer (`^`, `&` or `@`) or a function type. Each of those has the same size
whatever it leads to:

```go
const Bad: type = {int, Bad};          // ERROR: a Bad would hold a Bad, which holds a Bad, without end
const Rows: type = [2]Rows;            // ERROR: the same
const Step: type = (int)Step;          // fine: a function that returns a function of its own type
```

A type declared inside a function can name itself in the same way. Like every other name
there, it is used only below its declaration (2.8).

---

## 5. Arrays

### 5.1 Fixed and unsized

```go
const primes: [3]int = [2, 3, 5];       // fixed: exactly 3 elements
var scores: []int = [10, 20, 30];       // unsized: the length comes from the first value (3)
```

| Type | Where the length comes from |
|---|---|
| `[3]int` | the type |
| `[]int` | the first value it's given |

The size between the brackets is a number, or nothing. A type can't go there:

```go
const sizes: [bool]int = [3, 10];       // ERROR: the size of an array is a number
```

An array holds values of one type, and that can be any type:

```go
const word: [4]char = ['J', 'a', 'c', 'k'];     // text is an array of chars
const flags: [2]bool = [true, false];
```

**(filled in)** The size is a whole number written in the code (3.2), 0 or more. An array can
be empty: `[]` is the array value with no elements, and it needs a place that says what its
element type is (3.8).

**(filled in)** Text has a shorter way to be written: `"Jack"` is the same array value as
`['J', 'a', 'c', 'k']` (5.10).

### 5.2 Reading and changing elements

Elements are numbered from 0:

```go
primes[0]              // 2
scores[2]              // 30
scores[0] = 15;        // scores is now [15, 20, 30]
primes[0] = 7;         // ERROR: primes is const
```

An element can be changed when the array is a `var`.

**(filled in) An index is an `int`, and it is checked.** The first element is at 0 and the
last is at `length - 1`. When the index is a written number and the length is in the type, the
compiler checks it. Otherwise the check is made while the program runs, and an index out of
range fails there (11.7):

```go
primes[3]              // ERROR: primes has length 3, so its last index is 2
scores[i]              // checked while running: fails if i is below 0 or above 2
scores[small]          // ERROR: small is an int8, and an index is an int (3.2)
```

### 5.3 An unsized array keeps its first length

The length of an unsized array is set by its first value and never changes:

```go
var a: []int = [1, 2, 3];
a = [4, 5, 6];                 // fine: still 3 long
a = [1, 2, 4, 5];              // ERROR: a has length 3
var b: []int = [1, 2, 3, 4];   // fine: a new declaration, length 4
```

Because of this, a function can take or return an unsized array of any length:

```go
const firstOf: ([]int)int = (const values: []int){
    return values[0];
};
firstOf([7, 8])        // 7
firstOf([1, 2, 3, 4])  // 1
```

**(filled in) When a length is only known while running, the check is made while running.**
The compiler rejects `a = [1, 2, 4, 5];` above because it can see both lengths. When it can't,
giving an unsized array a value of another length fails while the program runs (11.7):

```go
var part: []int = a[0:2];      // length 2
part = a[1:n];                 // checked while running: fails unless n is 3
```

**(filled in) Every holder of an unsized array has its own first value.** A name, a field, an
element and a value on the heap each take their length from the first value they hold, and
keep it. A parameter and a result get a new first value at every call. The same holds for
every type whose size isn't in the type: a map (6.3) and an `any` (4.5).

```go
var rows: [2][]int = [[1, 2, 3], [4]];     // two rows: one of length 3, one of length 1
rows[1] = [9];                             // fine: still 1 long
rows[1] = [9, 9];                          // ERROR: rows[1] has length 1
```

A fixed array and an unsized array are different types, and a value never changes type by
itself (3.2). A fixed array becomes an unsized one through a slice (5.9): `primes[:]` is a
copy of `primes` as a `[]int`, and `&primes[:]` borrows it as an `&[]int`.

### 5.4 Arrays inside arrays

```go
const grid: [2][3]int = [[1, 2, 3], [4, 5, 6]];
grid[1][2]             // 6: row 1, element 2
grid[0]                // [1, 2, 3]: the whole first row
```

### 5.5 Reading the length

`.length` reads an array's length. It can't be assigned, because the length never changes:

```go
const primes: [3]int = [2, 3, 5];
var scores: []int = [10, 20, 30];

primes.length          // 3: from the type [3]int
scores.length          // 3: from the first value
scores.length = 4;     // ERROR: a length never changes
```

**(filled in)** A length is an `int`.

### 5.6 Passing an array to a function

There are two ways, and both the type and the call say which one is happening:

```go
const sumCopy: ([]int)int = (const v: []int){ ... };        // v is a copy of the array
const sumBorrow: (&[]int)int = (const v: &[]int){ ... };    // v borrows the caller's array

sumCopy(scores);       // copies all of scores, like every declaration (2.4)
sumBorrow(&scores);    // no copy: v borrows scores (8.4)
```

Inside `sumBorrow` the array is `v^`: `v^[0]` is its first element and `v^.length` is its
length.

### 5.7 Underneath: an array is like a struct

An array is like a struct whose fields all have the same type and are reached by position
instead of by name:

```go
var a: [3]int = [10, 20, 30];     // like a struct {int, int, int}
a[1]                              // the second field, by position
```

- **A fixed array `[3]int`** is exactly that: three `int` fields one after another. Its
  length is in the type, so it isn't stored anywhere. `a.length` is just the 3 from `[3]`.
- **An unsized array `[]int`** is like a struct of its length followed by its elements. The
  type doesn't say how long it is, so the length is stored in the value, which is why
  `.length` reads like a field: for `[]int` it really is one.

Either way an array has a length you know, and it's fixed. Written out as structs, just to
picture them (you never write arrays this way, and the field names are only labels):

```go
var a: [3]int = [10, 20, 30];
// pictured as a struct: three int fields; the length 3 is fixed by the type [3]int
var a: {int, int, int} = {
    at0: int = 10;                // a[0]
    at1: int = 20;                // a[1]
    at2: int = 30;                // a[2]
};

var b: []int = [7, 8];
// pictured as a struct: the length is stored, then the elements
var b: {int, [2]int} = {
    length: int = 2;              // b.length: set by the first value, and it never changes
    elements: [2]int = [7, 8];
};
```

In both, the length is known and it never changes.

### 5.8 A pair: `?T`

`?T` is a pair: exactly two values of type `T`, one for `false` and one for `true`, looked up
with a `bool`. It is written like an array of two, the value for `false` first, and its type
comes from the declaration, the same way `5` takes the type of the place it's written (3.2):

```go
const sizes: ?int = [3, 10];           // the value for false, then the value for true
sizes[false]                           // 3
sizes[true]                            // 10

const big: bool = count > 10;
sizes[big]                             // 3 or 10, picked by big
```

A pair is always looked up with a `bool`, and an array with a number:

```go
sizes[1]                               // ERROR: a pair is looked up with a bool
primes[true]                           // ERROR: a [3] array is indexed with a number
```

A pair always has both values, so a lookup never fails.

**(filled in)** A pair value is written with exactly two values, and a pair has no `.length`,
because the answer is always two:

```go
const three: ?int = [1, 2, 3];         // ERROR: a pair has two values
sizes.length                           // ERROR: a pair has no length
```

Underneath, a pair is just an array of two, a `[2]T`, the same way a `bool` is just a number
underneath (3.1). Looking it up with `false` takes the first value and with `true` the second,
because `false` is 0 and `true` is 1. The types keep the two apart in the code.

Its main use is choosing (section 10).

### 5.9 Slices: borrowing part of an array

`&a[i:j]` borrows the elements of `a` from index `i` up to, but not including, index `j`. Its
type is `&[]T`, the same type as a borrower of a whole unsized array (5.6), so a function that
takes an `&[]int` takes either:

```go
var scores: []int = [10, 20, 30, 40];

const middle: &[]int = &scores[1:3];   // borrows 20 and 30
middle^[0]                             // 20
middle^.length                         // 2

sumBorrow(&scores[1:3])                // 50: no copy (5.6)
```

| You write | It borrows |
|---|---|
| `&a[1:3]` | elements 1 and 2 |
| `&a[1:]` | from element 1 to the end |
| `&a[:2]` | elements 0 and 1 |
| `&a[:]` | the whole array. When `a` is unsized it is the same as `&a`. When `a` is fixed, `&a` is an `&[3]T` and `&a[:]` is the `&[]T` **(adjusted)** |

A slice is a borrower, so everything in section 8 holds for it. A `var` slice can change the
elements it borrows, a slice of a `const` array is a `const` borrower (8.6), it can't outlive
the array (8.5), and the array can't be given a new value or moved while the slice is used
further down (8.12):

```go
var front: &[]int = &scores[:2];
front^[0] = 99;                        // scores is now [99, 20, 30, 40]
```

A slice of a slice borrows from the same array: `&middle^[1:]` borrows `30`.

**(filled in)** The two bounds are `int`s, like an index (5.2). A bound that is left out is
0 at the start and the length at the end.

**Without the `&`, it is a copy.** `a[1:3]` on its own gives the elements as a new unsized
array, the same way every `=` copies (8.8):

```go
var part: []int = scores[1:3];         // part is its own [20, 30]
part[0] = 5;                           // scores doesn't change
```

**The bounds are checked.** The start can't be after the end, and the end can't be past the
length. When both are written as numbers and the length is in the type, the compiler checks
them. Otherwise the check is made while the program runs, and a slice out of range fails
there:

```go
const primes: [3]int = [2, 3, 5];
&primes[2:1]                           // ERROR: the start is after the end
&primes[0:5]                           // ERROR: primes has length 3
&scores[1:n]                           // checked while running: fails if n is over 4
```

**Underneath, a slice is two numbers:** the address of its first element and how many
elements it borrows. Every `&[]T` is held this way, a borrower of a whole array too, so the
same function takes both. That is why `.length` works on it like on any unsized array, and why
nothing is copied.

### 5.10 Text

**(filled in)** Text is an array of `char`s (5.1). There is no separate text type, so
everything an array does, a text does.

**Writing it.** `"Jack"`, between two `"`, is the array value `['J', 'a', 'c', 'k']` written
shorter. Like every array value, it takes its type from its place (3.8):

```go
const word: [4]char = "Jack";          // the same as ['J', 'a', 'c', 'k']
var name: []char = "Jack";             // unsized: length 4, from the first value
const empty: []char = "";              // a text with no chars
const wrong: [3]char = "Jack";         // ERROR: "Jack" has 4 chars
```

- A text holds exactly the `char`s that are written. Nothing is added at the end: its length
  is in its type or stored with it, like every array's (5.7).
- A text is written on one line. A line break inside it is written `\n` and a `"` inside it
  is written `\"`, with the escapes from 3.2.
- A character outside ASCII is held as its UTF-8 bytes, one `char` each: `"é".length` is 2.

**Using it.** Every one of these is an array rule or a built-in function:

| You want | You write |
|---|---|
| its length | `name.length` (5.5) |
| one `char` of it | `name[0]` (5.2) |
| to change one `char` | `name[0] = 'M';` when it is a `var` (5.2) |
| a part of it | `name[1:3]` for a copy, `&name[1:3]` to borrow (5.9) |
| to compare two texts | `name == "Jack"` (3.7) |
| to compare two `char`s | `name[0] < 'a'` (3.2) |
| to join two texts | `join(char)(name, "!")` (5.11) |
| to print it | `print(name);` (11.8) |
| a number as text, and back | `convert(int, []char)(42)`, `convert([]char, int)("42")` |

**Numbers as text.** `convert` (4.4) goes between a number or a `bool` and the text type
`[]char`:

| Conversion | Gives |
|---|---|
| `convert(int, []char)(n)`, for any whole-number type | the number in base ten, with a `-` in front when it is negative: `"42"`, `"-7"`, `"0"` |
| `convert(float, []char)(f)`, for either float type | the shortest text that reads back as the same float, always with a decimal point: `"9.5"`, `"3.0"`, `"-0.25"` |
| `convert(bool, []char)(b)` | `"true"` or `"false"` |
| `convert([]char, int)(t)`, for any whole-number type | the number written in `t`: digits in base ten, with or without a `-` in front |
| `convert([]char, float)(t)`, for either float type | the float nearest to the number written in `t`: written like a float in the code (3.2), with or without a `-` in front, and the decimal point can be left out |
| `convert([]char, bool)(t)` | `true` for `"true"` and `false` for `"false"` |

A float whose size is `1.0e21` or more, or below `1.0e-6` and not zero, is written with a
power of ten: `"1.5e21"`, `"2.0e-7"`.

Going from text to a value is checked when the line runs, like every conversion that might
not fit (4.4). It fails when the text isn't written as the table says, or when the number
doesn't fit in the type:

```go
convert([]char, int)("12a")            // fails while running: "12a" isn't a whole number
convert([]char, int)(" 12")            // fails while running: a space isn't a digit
convert([]char, uint8)("300")          // fails while running: 300 doesn't fit in a uint8
convert([]char, int)(word[:])          // 4 chars that aren't digits: fails while running too
```

A text held in a fixed array is converted through a slice, `word[:]`, because the conversion
takes a `[]char` (5.3).

### 5.11 Joining two arrays: `join`

**(filled in)** An array never grows (5.3). `join` makes a new one. It is a built-in function
that takes a type and gives back the function that joins two unsized arrays of that type: the
result is a new unsized array holding the elements of the first, then the elements of the
second.

```go
join(char)("Jack", "son")              // "Jackson"
join(int)([1, 2], [3])                 // [1, 2, 3]

var name: []char = "Jack";
var hello: []char = join(char)("Hello, ", name);       // "Hello, Jack"
```

| You write | Its type | What it does |
|---|---|---|
| `join(char)` | `([]char, []char)[]char` | joins two texts |
| `join(int)` | `([]int, []int)[]int` | joins two arrays of `int`s |

- `join` itself has the type `(type)([]T, []T)[]T` (4.3). It works for every element type.
- Both arrays are copied into the new one, like every argument (7.2). The two originals don't
  change.
- A fixed array is given as a slice: `join(int)(primes[:], [7])`.

A `var` unsized array keeps its first length (5.3), so a longer result goes into a new
declaration. A list that grows is a heap owner that is given a new, longer array each time:

```go
var all: ^[]int = ^[1, 2];
all = ^join(int)(all^, [3]);           // a new heap array [1, 2, 3]: the old one is freed (8.8)
all^.length                            // 3
```

---

## 6. Maps

### 6.1 Making a map

The type is `{KeyType}ValueType`, and the value lists `key: value` pairs:

```go
var seen: {int}bool = {1: true, 4: false};
const pairs: {int}[2]int = {1: [1, 2], 2: [3, 4]};
```

**(filled in) A key can have any type.** A lookup finds its key with `==`, and every type can
be compared with `==` (3.7). So a `char`, a text, a struct or a `type` can be a key, as well
as a number.

**(filled in) The keys of a map value all differ.** The compiler checks keys that are written
values. Keys that are worked out while the program runs are checked then, and two equal keys
fail there (11.7):

```go
const doubled: {int}bool = {1: true, 1: false};    // ERROR: the key 1 is written twice
const mixed: {int}bool = {a: true, b: false};      // checked while running: fails if a equals b
```

A map can be empty: `{}` is the map value with no entries. It needs a place that says what its
key and value types are (3.8).

### 6.2 Reading and changing entries

```go
seen[4]                // false
seen[4] = true;        // change an existing entry
pairs[2][0]            // 3: the entry for key 2, then its first element
pairs[1] = [0, 0];     // ERROR: pairs is const
```

An entry can be changed when the map is a `var`.

**(filled in) A lookup is checked.** A map's keys never change (6.3). So when a map is a name
that was declared with its keys written out, the compiler knows them, and it rejects a written
key that isn't one of them. In every other case the check is made while the program runs, and
a key that isn't in the map fails there (10.2, 11.7):

```go
seen[7]                // ERROR: 7 isn't a key of seen
seen[k]                // checked while running: fails unless k is 1 or 4
```

### 6.3 A map has a fixed size

A map's keys are set by its first value and never change, just like an array's length:

```go
seen[1] = false;       // fine: 1 is a key
seen[7] = true;        // ERROR: 7 isn't a key, and the map can't grow
seen.length            // 2: the number of keys
```

**(filled in)** The length is an `int`, like an array's (5.5).

**(filled in) A whole new value needs the same keys.** A `var` map can be given a new map
value, and that value has to have exactly the keys the map already has, in any order. The
compiler checks it when it can see both sets of keys. Otherwise the check is made while the
program runs (11.7). It is the rule for an unsized array, which keeps its first length (5.3):

```go
seen = {4: true, 1: false};    // fine: the same two keys
seen = {1: false};             // ERROR: seen has the keys 1 and 4
```

### 6.4 Underneath: a map is like a struct too

A map is like a struct whose fields are its keys: `{1: true, 4: false}` holds a value for
key `1` and a value for key `4`, side by side on the stack. A struct's fields are fixed by
its shape, and a map's keys are fixed by its first value, so neither can gain a new one.

Pictured as a struct (field names only as labels):

```go
var seen: {int}bool = {1: true, 4: false};
// pictured as a struct: each key, then its value
var seen: {int, bool, int, bool} = {
    key0: int = 1;   value0: bool = true;     // seen[1]
    key1: int = 4;   value1: bool = false;    // seen[4]
};
```

The keys never change, and the values can when the map is a `var`. That's why
`seen[4] = true;` works but `seen[7] = true;` doesn't: there's no field for key 7.

**(filled in)** The entries are held in the order they were written. A lookup compares its
key with the map's keys (3.7), and takes the value that sits beside the key that is equal.

---

## 7. Functions

### 7.1 Writing a function

The **type** lists the parameter types, then the return type. The **value** lists the
parameters, each declared like a name, then the body:

```go
const add: (int, int)int = (const x: int, const y: int){
    return x + y;
};
```

| Piece | Here |
|---|---|
| type | `(int, int)int` |
| parameters | `(const x: int, const y: int)`: in the same order as the types |
| body | `{ return x + y; }` |

`return value;` ends the function and gives back the value.

**A function that returns a value has to reach a `return` on every path.** The compiler
rejects a function that can get to the end of its body without one. A function that returns
nothing ends at the end of its body, or earlier at a `return;` (7.3).

**(filled in) What "every path" comes down to.** No statement branches, so a body runs
straight down, into each block inside it. A block *always returns* when its last statement is
a `return`, or is a block that always returns. A function that returns a value needs a body
that always returns. Nothing is written after a `return`, or after a block that always
returns, because it could never run:

```go
const one: ()int = (){
    {
        return 1;
    }                          // this block always returns, so the body does too
};

const late: ()int = (){
    return 1;
    show(2);                   // ERROR: nothing after a return can run
};
```

**(filled in) A function value is checked against its type.** It has as many parameters as
the type lists, each with the type listed at its position, and every `return` in it gives a
value of the return type. A function value written where nothing asks for a type takes its
return type from what its `return`s give (3.8).

### 7.2 Calling a function

```go
add(1, 2)              // 3
```

Each argument is copied into its parameter (2.4). To let a function use the caller's value
itself, the parameter borrows it (8.4): `bump(&n)`.

A call works out all of its arguments first, from left to right. Then it sets the
parameters, from left to right. So one argument can't change what another argument reads
(8.13).

**(filled in)** A call gives exactly one argument for each parameter, and each argument has
the type of its parameter (3.8). The value in front of the `(` is worked out before the
arguments (2.9):

```go
add(1)                 // ERROR: add takes two ints
add(1, true)           // ERROR: the second argument is a bool, and add takes an int
```

### 7.3 A function that returns nothing

Its return type is `void`:

```go
const show: (int)void = (const value: int){
    ...
};
show(5);

const hello: ()void = (){      // takes nothing, returns nothing
    ...
};
hello();
```

`return;` on its own ends it where it stands:

```go
const greet: ()void = (){
    show(1);
    return;                    // greet ends here
};
```

A function that returns nothing never gives a value back, and a function that returns a
value always does:

```go
const nothing: ()void = (){
    return 5;                  // ERROR: nothing returns nothing
};

const half: (int)int = (const n: int){
    return;                    // ERROR: half returns an int: write the value
};
```

**(filled in)** A call to a function that returns nothing has no value (3.8), so it can't be
given back either. `return hello();` is an error in every function: write `hello();` and then
`return;`.

### 7.4 Parameters are declared like names

Each parameter is `const` or `var` with a name and a type, like a declaration:

```go
const countDown: (int)int = (var n: int){
    n = n - 1;         // fine: n is a var parameter, a copy of what was passed
    return n;
};

var start: int = 5;
countDown(start)       // 4
start                  // still 5: countDown changed its own copy
```

| Parameter | What the function gets |
|---|---|
| `const x: int` | a copy it can't change |
| `var x: int` | a copy it can change |
| `const x: &int` | the caller's value, borrowed to read (8.6) |
| `var x: &int` | the caller's value, borrowed to change (8.4) |
| `var x: ^int` | a heap value of its own: a copy of the caller's (8.13) |
| `var @x: ^int` | the caller's own heap value: the parameter takes it over at the call (8.13) |

`const` and `var` on a parameter don't appear in the function's type. Borrowing, the heap and
ownership do: `(&int)void`, `(^int)void`, `(@^int)void`. A parameter written `@x: ^int`
shows as `@^int`, because that is what the call has to give (8.13).

### 7.5 Swapping a function in a `var`

A function held in a `var` can be replaced by another function of the same type:

```go
var op: (int, int)int = (const x: int, const y: int){
    return x + y;
};
op(2, 3)               // 5
op = (const x: int, const y: int){
    return x * y;
};
op(2, 3)               // 6
```

### 7.6 Functions as values

A function goes everywhere any other value goes:

```go
const sub: (int, int)int = (const x: int, const y: int){
    return x - y;
};

// in an array
const ops: [2](int, int)int = [add, sub];
ops[1](5, 3)                   // 2

// in a map
const byKey: {int}(int, int)int = {1: add, 2: sub};
byKey[1](2, 2)                 // 4

// passed in
const twice: ((int)int, int)int = (const f: (int)int, const x: int){
    return f(f(x));
};
const double: (int)int = (const x: int){
    return x * 2;
};
twice(double, 3)               // 12

// returned
const pick: ((int)int)(int)int = (const f: (int)int){
    return f;
};
pick(double)(5)                // 10

// returning an array
const three: (int)[3]int = (const x: int){
    return [x, x, x];
};
three(7)[0]                    // 7
```

### 7.7 Underneath: a function value is an address

A function value is the address of the function's code. Storing a function in a name, an
array or a map stores that address, and calling it jumps there:

```go
const ops: [2](int, int)int = [add, sub];   // two addresses
ops[1](5, 3)                                // jumps to sub's code
```

Copying a function value copies the address, not the code.

A function that uses variables from around it (7.8) also keeps their addresses, and a function
given back by a call keeps the parameters it uses from that call. These are kept with the
function out of sight, and its type is the same either way.

So every function value has the same size: the address of its code, and the address of what
it keeps. A function with nothing kept leaves the second empty. That is why functions given
back go in arrays and maps like any other function of their type:

```go
var one: int = 1;
var two: int = 2;
const steps: [3](int)int = [makeAdder(one), makeAdder(two), double];   // makeAdder: 7.8

steps[0](5)            // 6
steps[1](5)            // 7
steps[2](5)            // 10
```

`steps` can't outlive `one` or `two`, because the functions in it can't (7.8).

**(filled in) What the second address is.** One address is always enough:

| The function | The address of what it keeps |
|---|---|
| uses nothing from outside itself | empty |
| uses names of a block around it (7.8) | that block's place on the stack |
| is written in a struct value and uses its fields (9.6) | the struct value |
| is written at the top level of a file (11.3) | the package value that holds the file's variables |
| was given back by a call and keeps parameters of it (7.8) | the place of that call's parameters |
| was given back and keeps only a heap value (7.8) | the heap value |

Each of these places holds the address of the place around it: a block leads to the block or
the struct value around it, and in the end to the package value of its file. So a function
inside a function inside a file reaches everything it can see through one address.

### 7.8 Using a variable from outside the function

A function's body is a block (2.6), and a block sees the names of the blocks around it. So
a function can use names declared outside it, and it can change a `var` from outside:

```go
var count: int = 0;

const countUp: ()void = (){
    count = count + 1;         // changes a variable declared outside the function
};

countUp();
countUp();
count                          // 2
```

**A function borrows the variables it uses from around it.** It uses the variable itself, not a
copy, so its changes reach the variable. That holds for a function made inside another
function too:

```go
const addTwo: (int, int)int = (const a: int, const b: int){
    var sum: int = 0;
    const add: (int)void = (const x: int){
        sum = sum + x;         // add borrows sum, so this changes addTwo's sum
    };
    add(a);
    add(b);
    return sum;
};

addTwo(4, 5)                   // 9
```

The call doesn't show it: nothing on the line `countUp();` says that `count` changes. A
change made through a parameter does show at the call, with `&` or `@` (8.9).

The compiler works out which outside variables a function changes or takes, and counts a
call to it as changing them (8.12).

**A variable that a function uses can't die before the function.** It is the same rule as
for a borrower (8.5). A top-level variable lasts as long as the program, so a function can
always use one. A variable declared inside a function dies when that call ends, so a
function that uses it can't be given back, or stored in something that lasts longer:

```go
const makeBroken: ()()int = (){
    var x: int = 5;
    return (){ return x; };    // ERROR: x dies when this call ends, before the function that uses it
};
```

**A function given back keeps the parameters it uses.** The parameters are what the caller
handed in, and the function given back is made from them, so they live as long as it does.

**It can't outlive what was passed in.** At the call, the function given back counts as coming
from every argument, the same as a borrower that comes back (8.13). If an argument dies
before the function, the compiler raises an error. An argument handed over with `@` doesn't
count: the call takes its value, and the caller's variable is left `null` (8.13), so nothing
is left to die.

```go
const makeAdder: (int)(int)int = (const step: int){
    return (const n: int){ return n + step; };   // fine: the function keeps step
};

var five: int = 5;
var addFive: (int)int = makeAdder(five);         // fine: addFive can't outlive five
addFive(2)                     // 7
makeAdder(1)(2)                // 3: the 1 lasts until the end of the line (2.6)

{
    var six: int = 6;
    addFive = makeAdder(six);  // ERROR: six dies at this block's }, before addFive
}

var addTen: (int)int = makeAdder(10);   // ERROR: the 10 is gone at the end of this line, before addTen
```

A `var` parameter it keeps can still change, and the change stays for the next call:

```go
const makeCounter: (int)()int = (var count: int){
    return (){
        count = count + 1;
        return count;
    };
};

var start: int = 0;
var next: ()int = makeCounter(start);
next()                         // 1
next()                         // 2
```

**(filled in) One call has one set of parameters.** When a call gives back more than one
function, in a struct, an array or a pair, they all use the same kept parameters. It is the
rule from the top of this section again: a function borrows the variables it uses, and here
the variable is the call's parameter. Every new call has parameters of its own.

```go
const makeTally: (int){()void, ()int} = (var total: int){
    var both: {()void, ()int} = {
        up: ()void = (){ total = total + 1; };
        get: ()int = (){ return total; };
    };
    return both;
};

var zero: int = 0;
var tally: {()void, ()int} = makeTally(zero);
tally.up();
tally.up();
tally.get()                    // 2: up and get use the same total
```

**(filled in) Copying a function doesn't copy what it keeps on the stack.** A function value
is two addresses (7.7), and `=` copies those two. So a copy uses the same variables and the
same kept parameters as the original, the way a copy of a borrower borrows the same value
(8.8). And like the original, the copy can't outlive them:

```go
var again: ()int = next;       // again and next use the same kept count
again()                        // 3
next()                         // 4
```

**A parameter that takes a heap value can be filled at the call.** The value can be made on the
spot and handed straight over with `@` (8.9), so nothing in the caller has to outlive the
function:

```go
const makeLookup: (@^[3]int)(int)int = (var @data: ^[3]int){
    return (const i: int){ return data^[i]; };    // fine: data is a parameter, so it's kept
};

var look: (int)int = makeLookup(@^[10, 20, 30]);  // the array is made on the spot and handed
look(2)                // 30                       // straight over
```

To keep a local, use a helper whose parameter takes the value:

```go
const countFrom: (@^int)()int = (var @count: ^int){
    return (){
        count^ = count^ + 1;
        return count^;
    };
};

const newCounter: ()()int = (){
    var total: ^int = ^0;
    return countFrom(@total);          // total is handed over and left null
};
```

A kept parameter is kept the way it was declared (7.4). A copy stays the function's own. A
borrowed parameter (`&T`) is still a borrow, so the function can't outlive what the caller lent
(8.13). A parameter that took the caller's heap value over (`@`) keeps that value, and the
value is freed when the function given back ends (8.11). `first` (4.3) keeps its `T` the same
way.

The rule is only for a function the call gives back with `return`. A local is never kept by
using it in the body: returning a function that does that is an error, as above.

**(adjusted) Where kept parameters live, and how far the function can go.** Nothing goes on
the heap unless the code says `^` (2.6). So the parameters a function keeps stay where the
call put them: on the stack, in the block the call was written in. The function given back
holds their address (7.7). They end when that block ends. Two rules follow.

**A function given back can't outlive the block its call was written in.** It can't be given
back from that block's function, and it can't be stored in something that lasts longer than
the block. This comes on top of the rule about the arguments above. The top level of a file is
a block as well, and it lasts as long as its package value (11.3):

```go
var keep: (int)int = double;

const install: ()void = (){
    keep = makeAdder(five);    // ERROR: the step is kept in this call of install, and keep lasts longer
};

const adderFor: (int)(int)int = (const by: int){
    return makeAdder(by);      // ERROR: the step is kept in this call of adderFor, which ends here
};

const adderFrom: (int)(int)int = (const by: int){
    return (const n: int){ return n + by; };   // fine: it keeps by, a parameter of adderFrom itself
};
```

**A function that keeps only a heap value can go anywhere.** Sometimes the one thing a
function given back uses from outside itself is a parameter that took a heap value over with
`@`. Then nothing it keeps is on the stack: the function holds the address of that heap value,
and owns it. It can go wherever a heap owner can: it can be given back again, stored, and
moved. The heap value is freed when the function ends (8.11), and copying the function copies
the heap value, as with every owner (8.8). `makeLookup` and `countFrom` above give back
functions like this, which is why `newCounter` can hand on what `countFrom` gave it.

A function that needs more than one value in this way takes them as one struct on the heap.

Inside its own body, a function's name is the function itself, so a function made inside
another can call itself (10.3).

---

## 8. Ownership: plain, heap and borrowed values

Every value has exactly one **owner**: the name, field or array element that holds it. The
owner decides how long the value lives. A value's type says which of three cases it is.

### 8.1 The three cases

| Written | Who owns the value | Where the value is | It lives until… |
|---|---|---|---|
| `T`, no mark | this holder | right here, inside the holder | the holder ends |
| `^T` | this holder | somewhere else, on the heap | the holder ends, gives it up, or is given something else (8.11) |
| `&T` | another holder | wherever its owner put it | its owner decides: a borrower never frees it |

```go
var count: int = 3;                 // plain: count holds the 3 itself
var big: ^[1000]int = ^[...];       // heap: big owns an array kept on the heap
var view: &[1000]int = &big^;       // borrowed: view uses big's array, and big still owns it
```

A type with no mark is always the first case. Nothing is left unsaid: the other two cases
are always marked, so no mark can only mean "mine, right here".

A plain value never changes owner. It lives inside its holder, so it can be copied but
never passed on. Only a value on the heap can go from one owner to another (8.9).

A fourth mark, `@T`, is not a fourth case. It is the **ownership** of a variable: a pointer
to the variable that is allowed to take its value (8.9).

### 8.2 The symbols

| Written | Where | Meaning |
|---|---|---|
| `^T` | in a type | a `T` on the heap, owned by this holder |
| `&T` | in a type | a `T` that another holder owns |
| `@T` | in a type | the ownership of a `T` variable (8.9) |
| `^v` | in front of a value | a new value on the heap, holding a copy of `v` |
| `&x` | in front of a variable | a borrower of `x` |
| `@x` | in front of a variable | the ownership of `x` |
| `p^` | behind a heap owner, a borrower or an ownership | what it leads to |

What **makes** a marked value goes in front, like the mark in its type: `^5` is a `^int`,
`&count` is an `&int`, and `@p` is an `@^int`. What **uses** one goes behind, like `a[0]` and
`f(1)`.

What is behind a name is read first, from left to right. What is in front is applied after
it. So a chain of marks never needs brackets:

```go
&big^[0]               // follow big, take element 0, then borrow it
^p^                    // follow p, then make a new heap value from what's there
list^.next^.value      // follow, field, follow, field
@list^.next            // follow, field, then the ownership of that field
```

The marks in a type and the `^`s in its use come in the same order (3.5):

| Type | It is | Used as |
|---|---|---|
| `[3]^int` | three ints, each on the heap | `a[0]^` |
| `^[3]int` | one array of three ints on the heap | `a^[0]` |
| `[2]&int` | two borrowers | `a[1]^` |
| `&[2]int` | one borrower of a whole array | `a^[1]` |
| `&^int` | a borrower of a heap owner | `a^^` |
| `@^int` | the ownership of a heap owner | `a^^` |

### 8.3 A value on the heap: `^T`

`^` in front of a value puts a copy of it on the heap. `p^` is the value there:

```go
var p: ^int = ^5;          // 5 is an int, and ^5 is a ^int
p^                         // 5
p^ = 6;                    // changes the value on the heap
var x: int = p^;           // x is a plain copy: 6

var big: ^[1000]int = ^[...];
big^[3] = 7;               // follow, then index
big^.length                // 1000
```

The types are strict. An `int` and a `^int` are different types, and a value never changes
type by itself (3.2):

```go
var bad: ^int = 5;         // ERROR: 5 is an int, not a ^int: write ^5
var y: int = p;            // ERROR: p is a ^int, not an int: write p^
```

`^v` always makes a copy. The value it's made from stays as it is:

```go
var n: int = 3;
var h: ^int = ^n;          // the heap gets its own 3
h^ = 5;                    // n is still 3
```

A value written on the spot, like `^5` or `^[1, 2, 3]`, has no original, so it's simply
built on the heap.

A heap value lives as long as its owner. When the owner ends, the value is freed:

```go
const work: (int)void = (const x: int){
    var three: ^[3]int = ^[x, x, x];
    three^[0] = 9;
};                         // work ends, so three ends, and its array is freed
```

A `const` owner keeps its value, like every `const`:

```go
const table: ^[3]int = ^[1, 2, 3];
table^[0] = 9;             // ERROR: table is const, so its value can't change
```

### 8.4 A borrowed value: `&T`

`&` in front of a variable makes a borrower of it. A borrower is the same memory as what it
borrows: `b^` reads that value, and assigning to `b^` changes it.

```go
var count: int = 3;
var b: &int = &count;      // b borrows count
b^                         // 3
b^ = 5;                    // count is now 5
count = 8;                 // b^ is now 8 too: it is the same memory
```

Nothing is copied, and nothing changes owner. `count` still owns its value, and `b` never
frees it.

A borrower can borrow a variable, an array element, a map entry, a struct field or a value
on the heap. A variable can't keep a borrower of a value made on the spot, because that
value has no owner and is gone at the end of its statement (2.6):

```go
var scores: []int = [10, 20, 30];
var top: &int = &scores[0];
top^ = 99;                 // scores is now [99, 20, 30]

var seen: {int}bool = {1: true, 4: false};
var four: &bool = &seen[4];
four^ = true;              // seen[4] is now true

var view: &[1000]int = &big^;      // the whole array big owns
var first: &int = &big^[0];        // one element of it
view^[1] = 2;                      // big's array changes

var bad: &int = &3;        // ERROR: the 3 is gone after this line, and bad would still be there
var worse: &int = count;   // ERROR: count is an int, not an &int: write &count
```

`&T` is a type, so a borrower goes wherever other values go: in an array, in a struct, or
into a function.

```go
var left: int = 1;
var right: int = 2;
var both: [2]&int = [&left, &right];
both[0]^ = 7;              // left is now 7
```

A function borrows through its parameter, and the call shows it:

```go
const bump: (&int)void = (var x: &int){
    x^ = x^ + 1;
};

var n: int = 3;
bump(&n);                  // n is now 4
bump(&scores[1]);          // scores[1] goes up by 1
bump(n);                   // ERROR: n is an int, not an &int: write &n

const swap: (&int, &int)void = (var a: &int, var c: &int){
    var t: int = a^;
    a^ = c^;
    c^ = t;
};
swap(&left, &right);       // the two values change places
```

**A value made on the spot can be borrowed as an argument.** It lives until the end of its
statement (2.6), and the call is over by then, so the parameter never outlives it (8.5):

```go
look(&makeBig(1)^);        // fine: look borrows the new array, which is freed at the end
                           // of this line (8.13)
bump(&3);                  // fine: bump changes a 3 that nothing else can read
```

`b = …` changes **what** a borrower borrows, and `b^ = …` changes **the value** it borrows:

```go
var b: &int = &left;
b^ = 10;                   // left is now 10
b = &right;                // b now borrows right; left is untouched
b^ = 20;                   // right is now 20
```

### 8.5 A borrower can't outlive what it borrows

A borrower can only borrow something that lives at least as long as the borrower itself. If
what it borrows would be gone while the borrower is still there, the compiler raises an
error.

```go
var x: int = 3;
var a: &int = &x;          // fine: a and x end together, at the same }
{
    var y: &int = &x;      // fine: y ends at this block's }, before x does
    y^ = 4;                // x is now 4
}

bump(&x);                  // fine: bump's parameter ends when the call ends,
                           // and x is still there

{
    var inner: int = 5;
    a = &inner;            // ERROR: inner ends at this }, but a is still there after it
}

var top: &int = &three(7)[0];      // ERROR: the array three(7) gives back is gone at the
                                   // end of this line, but top would still be there
```

(`three` is the function from 7.6 that returns an array.)

| The borrower borrows… | Allowed? |
|---|---|
| a variable declared in the same block | yes: they end together |
| a variable declared in an outer block | yes: the borrower ends first |
| the caller's variable, through a `&T` parameter | yes: the parameter ends when the call ends |
| a variable declared in an inner block | no: it ends before the borrower does |
| a value with no name, as an argument (8.4) | yes: the value lives until the end of its statement (2.6), after the call has ended |
| a value with no name, kept in a variable | no: the value is gone after its statement, and the borrower would outlive it |
| a variable a function uses from around it (7.8) | yes, while the variable lasts at least as long as the function. A function given back keeps the parameters it uses, and can't outlive the arguments of the call, except those handed over with `@` |

The check is made every time a borrower is given something to borrow: when it's declared,
and again each time it's pointed at something else with `b = &…`.

A borrower of a heap value is measured against the variable that owns it:

```go
var view: &[1000]int = &big^;      // fine: view ends no later than big
```

An ownership (8.9) holds a variable's address too, so the same rule applies to it.

**(filled in) A value that holds a borrower follows the rule for it.** A struct, an array, a
pair, a map or an `any` that has a borrower, an ownership or a function inside it can't
outlive what that part borrows or uses. It is the field-by-field rule again (8.8): the
lifetime rules for a whole value are the rules for each of its parts.

```go
var both: [2]&int = [&left, &right];   // fine: both ends no later than left and right
{
    var inner: int = 5;
    both[0] = &inner;                  // ERROR: inner ends at this }, but both is still there after it
}
```

The compiler can't see inside an `any` (4.5). So an `any` counts as holding everything that
the value wrapped into it held.

A value on the heap can also be made to end before its owner's `}`. Rule 8.12 covers that.

### 8.6 `const`, `var` and borrowing

Whether a borrower can change the value it borrows depends on how the borrower is declared:

| The owner is | `var b: &T` | `const b: &T` |
|---|---|---|
| a `var` | allowed: `b` can read and change the value | allowed: `b` can only read it |
| a `const` | **error** | allowed: `b` can only read it |

```go
const limit: int = 10;
var a: &int = &limit;      // ERROR: limit is const, so its borrower has to be const
const c: &int = &limit;    // fine
c^                         // 10
c^ = 11;                   // ERROR: c is a const borrower, so it can only read

var count: int = 3;
const look: &int = &count; // fine: look can read count but not change it
count = 4;                 // look^ is now 4
look^ = 5;                 // ERROR: look is a const borrower
look = &limit;             // ERROR: look is const, so it always borrows count
```

A parameter works the same way: `(const v: &[]int)` borrows an array to read it, and
`(var x: &int)` borrows a value to change it.

**(adjusted) A `const` is lent only where the compiler can see that it is only read.** That
is a `const` borrower, or a `const` parameter of a function called by its own name. A function
called through a variable, an array or a map shows only its type, and the type doesn't say
whether a `&T` parameter is read or changed (7.4). There every `&` argument counts as one that
may change (8.12), so lending a `const` is an error:

```go
const peek: (&int)int = (const x: &int){ return x^; };
const fs: [1](&int)int = [peek];

peek(&limit)               // fine: the compiler sees that peek only reads
fs[0](&limit)              // ERROR: limit is const, and the compiler can't see which function runs
fs[0](&count)              // fine: count is a var
```

**(filled in)** An ownership follows the same table: a `const` ownership can only read, and
changing or taking through an ownership needs a `var` one (8.9). So one rule covers every
`const`: what a `const` name reaches can't be changed through that name (2.9).

### 8.7 Converting between the three

Going from one case to another is always written out:

| From | To | Written | Is anything copied? |
|---|---|---|---|
| a plain value | a heap value | `^v` | yes: the heap gets its own copy |
| a plain value | a borrower | `&x` | no |
| a heap value | a plain value | `p^` | yes, when it's put in a plain holder |
| a borrowed value | a plain value | `b^` | yes, when it's put in a plain holder |
| a heap value | a borrower | `&p^` | no |
| a borrowed value | a heap value | `^b^` | yes: a borrower can't become an owner, so this makes a new heap copy |

```go
var p: ^int = ^5;
var b: &int = &p^;         // borrow the value p owns
var x: int = b^;           // a plain copy of it
var q: ^int = ^b^;         // a new heap value holding a copy

var bad1: ^int = 5;        // ERROR: write ^5
var bad2: int = p;         // ERROR: write p^
var bad3: &int = p;        // ERROR: write &p^
var bad4: &int = &p;       // ERROR: &p is an &^int, a borrower of the owner itself: write &p^
var bad5: ^int = b;        // ERROR: a borrower can't become an owner: write ^b^ for a copy
var bad6: int = x^;        // ERROR: x is a plain int, so there is nothing to follow
```

### 8.8 `=` always makes a copy

`=` means the same for every type: the left side gets its own copy, and the right side is
never changed.

| Type of both sides | What the left side gets |
|---|---|
| `T` | its own copy |
| `^T` | its own copy, in a new place on the heap |
| `&T` | a borrower of the same value: a borrower owns nothing, so there is nothing to copy |
| `@T` | an ownership of the same variable: it holds an address, like a borrower, and nothing moves |
| a function **(filled in)** | the same code, using the same variables and kept parameters (7.8). A heap value the function owns is copied |
| a `type` **(filled in)** | the same type |
| an `any` **(filled in)** | its own copy of the type and the value inside |
| a `package` **(filled in)** | its own copy of the file's variables (11.5) |

```go
var a: int = 3;
var b: int = a;            // b is a copy

var p: ^int = ^5;
var q: ^int = p;           // q gets its own heap int holding 5
q^ = 9;                    // p^ is still 5

var r: &int = &a;
var s: &int = r;           // s borrows a too
```

**A copy is complete.** Copying a value that owns other heap values copies those as well,
so the copy shares nothing with the original:

```go
var rows: [2]^int = [^1, ^2];
var copy: [2]^int = rows;  // two new heap ints
copy[0]^ = 9;              // rows[0]^ is still 1
```

Copying the first node of a linked list therefore copies every node behind it (8.14).

**What is done to a struct is done to each of its fields.** A struct is used like any other
variable: it can be copied, moved, put on the heap, borrowed, and have its ownership taken.
Each of those is the same thing done to every field, each field by the rule for its own
type. The same holds for an array and its elements. With `foo` and `boo` two structs that
have the fields `a` and `b`:

| You write | Which is, field by field |
|---|---|
| `foo = boo;` | `foo.a = boo.a; foo.b = boo.b;` |
| `@foo = @boo;` | `@foo.a = @boo.a; @foo.b = @boo.b;` (8.9) |
| `var h: ^Boo = ^boo;` | one new heap value, then `h^.a = boo.a; h^.b = boo.b;` |
| `var r: &Boo = &boo;` | `r^.a` is `boo.a`, and `r^.b` is `boo.b`: the same memory |
| `var o: @Boo = @boo;` | `o` is the ownership of `boo.a` and of `boo.b` together |

This is why a copy is complete. A field that is a heap owner gets its own heap value, and
the fields of that value are copied the same way, all the way down.

A field that is a borrower is copied as a borrower, so the copy borrows the same thing the
original borrows. For the list with a `prev` in 8.15, each copied node's `prev` therefore
still borrows a node of the original list, and those links have to be set again after the
copy.

**An owner that is given a new value frees the one it had:**

```go
q = p;                     // q's old heap int is freed, and q gets a new copy of p's
```

**A value that nothing else holds is taken, not copied.** That covers a value written on
the spot, the result of a call, and a local that a function returns (8.13):

```go
var one: ^int = ^5;                    // the new heap value has no other owner, so one takes it
var made: ^[1000]int = makeBig(1);     // the same for what a function gives back
```

To take over a value that another owner still holds, without copying it, move it (8.9).

### 8.9 Ownership as a type: `@T`, and moving

Every variable has an **ownership**: the right to its value. `@` in front of a variable
gives that ownership as a value of its own, with its own type.

| Written | What it is |
|---|---|
| `@p` | `p`'s ownership. When `p` is a `^int`, its type is `@^int` |
| `h^` | follow an ownership: the variable it is the ownership of |
| `@q = h;` | set `q`'s ownership: `q` takes the value over, and the old variable lets go |
| `var @q: ^int = h;` | the same, for a new variable |

```go
var p: ^int = ^5;

var h: @^int = @p;         // h is p's ownership. Nothing has moved: p still owns the 5
h^                         // p itself, a ^int
h^^                        // 5

var c: ^int = h^;          // a copy, like every = (8.8). This is not a move
var @q: ^int = h;          // the move: q owns the 5 now, and p is left null (8.10)
```

`@` is the third mark, and it is written and used like the other two:

| Type | Made with | Followed with | Following gives |
|---|---|---|---|
| `^T` | `^v` | `p^` | the value on the heap |
| `&T` | `&x` | `b^` | the variable it borrows |
| `@T` | `@x` | `h^` | the variable it is the ownership of |

It works on every variable, and on a field or an element, not only on heap owners:

```go
var x: int = 3;
var o: @int = @x;          // @x is an @int
o^                         // x itself: 3

@list^.next                // the ownership of a field: an @^Node (8.14)
```

**(filled in)** A map entry is a holder like an array element (6.4), so `@` works on it too:
`@seen[4]` is the ownership of that entry.

`@` in front of a value made on the spot hands that value straight over. Nothing is left
`null`, because nothing else held it:

```go
take(@makeBig(1));                 // the new array goes straight to take's parameter (8.13)
var h: @^Node = @makeNode(1);      // ERROR: the node has no variable, so h would outlive it (8.5)
```

**Moving.** A move sets one variable's ownership from another's. Nothing is copied: the
value stays where it is on the heap, and only its owner changes. Both sides are ownerships,
so both have the same type:

```go
@q = @p;                   // both are @^int: q takes over p's heap int, and p is left null
                           // the value q owned before is freed

var @r: ^int = @q;         // a new variable that takes q's value over

q = @p;                    // ERROR: q is a ^int and @p is an @^int
@q = p;                    // ERROR: @q is an @^int and p is a ^int
```

In `var @r: ^int = @q;` the type says what `r` is, a `^int`. The `@` on the name says that
`r`'s ownership is what is being set.

What a move does depends on what the variable holds. It mirrors a copy (8.8):

| The variable is | `=` copies | A move |
|---|---|---|
| a heap owner, `^T` | a new heap value | the same heap value: the old variable is left `null` |
| a struct or array that holds heap owners | every heap value inside is copied | the plain parts are copied, and every owner inside is handed over and left `null` |
| a plain value that owns nothing on the heap | a copy | the same as a copy: there is nothing to hand over |

**Look, change, take.** An ownership sits beside a borrower as one more level of what a
holder may do to someone else's variable:

| | read it | change it | take its value away |
|---|---|---|---|
| `const b: &T` | yes | no | no |
| `var b: &T` | yes | yes | no |
| `h: @T` | yes | yes | yes |

So the mark at a call shows what a function may do to a variable you pass it: `f(x)`
nothing, because it gets a copy, `f(&x)` read or change it, and `f(@x)` takes its value
(8.13). A function can also change a variable from outside itself without being passed it
(7.8), and then the call shows nothing.

The rules for an ownership and a move:

- **Nothing moves when `@p` is written.** `p` lets go only when its ownership is set into
  another variable. Until then `p` still owns its value.
- **Both sides of a move are ownerships.** A plain `=` never changes its right-hand side.
- **The old variable is left `null`,** so the value still has exactly one owner.
- **The new owner's old value is freed,** as with any owner that is given something new. The
  value is taken out of the old variable before that happens (8.14 uses this).
- **An ownership can't outlive its variable.** It holds the variable's address, so the rule
  for borrowers applies (8.5):

  ```go
  const bad: ()@^int = (){
      var local: ^int = ^5;
      return @local;         // ERROR: local ends here, but its ownership would still be there
  };
  ```

- **Only an owner can give an ownership.** A borrower can read and change, but not take:

  ```go
  var b: &^int = &p;
  var @t: ^int = @b^;        // ERROR: b only borrows p
  ```

- **A `const` has no ownership to give,** because an ownership allows changing it and
  emptying it:

  ```go
  const fixed: ^int = ^5;
  var @t: ^int = @fixed;     // ERROR: fixed is const
  ```

- **Copying an ownership copies the address,** like copying a borrower. `var h2: @^int = h;`
  is a second ownership of the same variable. The value still has one owner: the variable.
- **After a move, an ownership still points at the old variable,** which is now `null`.
  `h^ == null` is `true` after `var @q: ^int = h;`.
- **A value can be moved only when nothing still in use points at it** (8.12).
- **The new owner decides how long the value lives from then on:**

  ```go
  var data: ^[1000]int = ^[...];
  {
      var @inner: ^[1000]int = @data;
  }                          // inner ends here, so the array is freed
  data == null               // true
  ```

A heap owner `p` can be used in four ways, and each is written differently:

| Written | What it gives | `p` afterwards |
|---|---|---|
| `p^` | the value | unchanged |
| `&p^` | a borrower of the value | unchanged |
| `p` | a copy of the value, in a new place on the heap | unchanged |
| `@p` | `p`'s ownership | unchanged, until that ownership is set into another variable: then `null` |

A local that a function returns needs no ownership (8.13): it ends there anyway, so nothing
is left behind to be emptied.

### 8.10 `null`

A heap owner that owns nothing is `null`.

**How an owner becomes `null`:**

| How | What happens to the value it owned |
|---|---|
| its value is moved to another variable (8.9) | it lives on with the new owner |
| it is set to `null` | it is freed |
| it is declared with `null` | there was no value |

```go
var p: ^int = ^5;
var @q: ^int = @p;         // moved away: p is null, and q owns the 5

var r: ^int = ^7;
r = null;                  // set to null: the 7 is freed

var later: ^int = null;    // declared null: it owns nothing yet
```

**How it stops being `null`:** it is given a value, or takes one over.

```go
later = ^8;                // later owns a new heap int
@p = @later;               // p takes it over, and now later is null
```

**Testing for it:** `p == null` and `p != null` give a `bool`. Comparing doesn't copy
anything and doesn't change `p`. A pair (10.1) then picks what to do. A function that
is handed an owner to test borrows the owner itself:

```go
const valueOr: (&^int, int)int = (const owner: &^int, const fallback: int){
    const cases: ?(&^int, int)int = [
        (const o: &^int, const f: int){ return o^^; },     // false: it owns a value, so read it
        (const o: &^int, const f: int){ return f; }        // true: it is null
    ];
    return cases[owner^ == null](owner, fallback);
};

valueOr(&p, 0)             // p's value, or 0 when p is null
```

`&p` borrows the owner itself, so its type is `&^int`. `owner^` is the owner, and `owner^^`
is the value it owns.

**Following a `null` fails while the program runs:**

```go
var none: ^int = null;
none^                      // fails while running: none owns nothing
```

A copy of a `null` owner is `null` as well, because there is nothing to copy.

A borrower can be `null` too, meaning it borrows nothing yet: `var b: &int = null;`.
Following it fails in the same way.

**(filled in)** `null` is "no address" (8.18), so it fits all three pointer types: a `^T`, an
`&T` and an `@T`. Following any of them when it is `null` fails while the program runs
(11.7). Nothing else can be `null`:

```go
var h: @^int = null;       // fine: an ownership of nothing yet
var n: int = null;         // ERROR: null is a pointer, and an int isn't one
count == null              // ERROR: count is an int
```

### 8.11 When a heap value is freed

A heap value is freed at the moment it has no owner left:

| When | Example |
|---|---|
| its owner ends | the `}` of the block the owner was declared in, or the end of the call for a parameter |
| its owner is given a new value with `=` | `q = p;` frees what `q` owned before |
| its owner takes another value over in a move | `@q = @p;` frees what `q` owned before |
| its owner is set to `null` | `r = null;` |
| it has no name and nothing receives it | `makeBig(1);` on its own, or only lent as in `look(&makeBig(1)^);`: freed at the end of the statement (2.6) |
| the value that owns it is freed | a list node's `next`: freeing a node frees every node behind it |
| the function given back that keeps it ends | a heap value owned by a parameter that a function given back keeps (7.8) |

A heap value is **not** freed when it's moved (it has a new owner), and never by a
borrower.

**(filled in)** For the last row of the table, "the function ends" means one of two moments
(7.8). A function that keeps only a heap value owns it, so the value is freed when that
function value ends, like the value of any owner. Every other kept parameter sits on the
stack in the block the call was written in, and a heap value it owns is freed when that block
ends.

### 8.12 Nothing is freed while something still points at it

Rule 8.5 stops a borrower from outliving the variable it borrows from. A value on the heap
can also be made to end sooner than that: its owner can be given something new, set to
`null`, or have its value moved away. This rule covers those.

A **pointer** here is a borrower (`&T`) or an ownership (`@T`). To reach a value on the heap,
a pointer has to follow that value's owner.

**While a pointer that was made by following an owner is still used below, that owner can't
be changed.** The compiler raises the error at the line that would change it.

"Changed" is everything that frees or moves what the owner holds:

| The line | What it does to what `p` owns |
|---|---|
| `p = …;` and `p = null;` | frees it (8.8) |
| `@p = …;` | frees it |
| `@q = @p;` and `var @q: T = @p;` | moves it away (8.9) |
| `take(@p)` | moves it into the function (8.13) |
| a call to a function that can change `p` | may do any of these |

"Still used below" means written anywhere further down the function. There are no loops, so
later in the text is later in time.

**Which owners a pointer followed** is read off the line that made it:

| The pointer | Owners it followed | So these can't be changed while it is used below |
|---|---|---|
| `&p` | none: it points at `p` itself | none |
| `&p^` | `p` | `p` |
| `@list^.next` | `list` | `list` |
| `&list^.next^` | `list`, then `list^.next` | `list` and `list^.next` |
| `&b^.value`, where `b` is a borrower | the ones `b` followed | the same as for `b` |

```go
var p: ^int = ^5;
var b: &int = &p^;               // b followed p
p = ^6;                          // ERROR: b is used below, and this frees what p owns
b^
```

To change the value where it is, so that `b` sees it, write `p^ = 6;`.

```go
var h: @^Node = @list^.next;     // h followed list
list = makeNode(9);              // ERROR: h is used below, and this frees the first node
var @n: ^Node = h;
```

Taking the node out first makes both lines fine, because `h` isn't used after that:

```go
var h: @^Node = @list^.next;
var @n: ^Node = h;
list = makeNode(9);              // fine
```

(`Node`, `makeNode` and `list` are from 8.14.)

**An owner beyond where a pointer points was not followed,** so it can still be changed. A
pointer into a list can change what comes after it:

```go
var cur: &Node = &list^.next^;   // cur followed list and list^.next
cur^.next = makeNode(9);         // fine: cur^.next is beyond cur
cur^.value
```

**A value may be moved anywhere.** A move changes the owner it empties, so it is allowed
whenever nothing still in use followed that owner. Where the value goes makes no difference,
including into an owner that ends sooner:

```go
var data: ^[1000]int = ^[...];
var b: &[1000]int = &data^;          // b followed data
{
    var @inner: ^[1000]int = @data;  // ERROR: b is used below, and this moves what data owns
}
b^[0]
```

Without the last line it is fine: the array is freed at the `}`, and nothing leads to it any
more.

**Calls.** While a call runs, its pointer arguments are in use, and the function may do what
each argument lets it (8.9):

| The argument | The function may | So the call is an error when another pointer in use followed… |
|---|---|---|
| `f(x)`, a copy | nothing to the caller's value | never |
| `f(&x)` to a `const` parameter | read | never |
| `f(&x)` to a `var` parameter | change `x`, and every owner inside it | `x`, or an owner inside it |
| `f(@x)` | take `x`'s value | `x` |

"Another pointer in use" is one that is used below in the caller, or another argument of the
same call: `f(&list^.next^, @list)` hands the list over while the first argument points into
it.

A function's type doesn't say whether a `&T` parameter is read or changed (7.4). For a
function called by its own name, the compiler looks at the function. For one called through
a variable, an array or a map, every `&` argument counts as one that may change.

**A function that changes a variable from outside itself** (7.8) counts too. For a function
called by its own name, the compiler reads which outside variables it changes or takes, and
the call line counts as changing them:

```go
const reset: ()void = (){
    list = makeNode(9);
};

var b: &Node = &list^;           // b followed list
reset();                         // ERROR: b is used below, and reset changes list
b^.value
```

A call through a variable, an array or a map counts as changing every outside variable,
because the compiler can't tell which function runs.

### 8.13 Functions and ownership

A parameter is declared like any name (7.4), so the rules above apply to it unchanged. The
function's type shows `^`, `&` and `@`, and the call shows `&` and `@`:

```go
const use: (^[1000]int)void = (var a: ^[1000]int){ ... };       // a gets its own copy
const look: (&[1000]int)void = (const a: &[1000]int){ ... };    // a borrows, to read
const take: (@^[1000]int)void = (var @a: ^[1000]int){ ... };    // a takes the caller's array over

use(big);              // use gets its own copy on the heap, and big is unchanged
look(&big^);           // look borrows the array: nothing is copied and nothing changes owner
take(@big);            // take's parameter takes the array over, and big is left null
```

| Parameter type | The call | What the function gets |
|---|---|---|
| `int` | `f(n)` | its own copy |
| `^[1000]int` | `use(big)` | its own copy, in a new place on the heap |
| `&[1000]int` | `look(&big^)` | the caller's array, borrowed: it can read it, and change it when the parameter is a `var` |
| `@^[1000]int` | `take(@big)` | the caller's array itself: the parameter takes it over at the call, and the caller's variable is left `null` |

**A parameter that asks for an ownership takes it.** The parameter is written like the moving
declaration in 8.9, `var @a: ^[1000]int`, because it is one: the argument is what the
parameter's ownership is set from. The function's type lists `@^[1000]int`, which is what
the call has to give. So the move happens at the call itself:

- after `take(@big)`, `big` is always `null`;
- inside the function, `a` is an ordinary `^[1000]int` that owns the array, and `a^[0]` is
  its first element;
- when the call ends, `a` ends, and the array is freed unless the function moved it on,
  returned it, or gave back a function that uses `a` (7.8).

An ownership held in a variable is given the same way: `take(h)`, where `h` is an
`@^[1000]int`. A value made on the spot is handed straight over with `@` in front:
`take(@makeBig(1));` (8.9). A parameter is never declared with the type `@T` itself, so a
function that is given an ownership can't leave the value where it was.

A value made on the spot can be lent as well, with `&` in front: `look(&makeBig(1)^);`. The
parameter borrows the new array for the call, nothing takes it over, and it is freed at the
end of the statement (8.4).

**Every argument is worked out before any parameter takes.** A call works out all of its
arguments first, from left to right, and only then sets the parameters, from left to right
(7.2). So an argument that reads a variable and an argument that hands the same variable
over don't get in each other's way:

```go
const both: (@^int, int)void = (var @owned: ^int, const was: int){ ... };

var x: ^int = ^5;
both(@x, x^);
```

| Step | What happens |
|---|---|
| 1 | `@x` is worked out: it is `x`'s ownership. Nothing moves yet (8.9) |
| 2 | `x^` is worked out: it is 5 |
| 3 | `owned` is set: it takes the heap int over, and `x` is left `null` |
| 4 | `was` is set: it gets the 5 from step 2 |

If each parameter were set as soon as its argument was worked out, step 3 would come before
step 2, and `x^` would follow a `null`. With this order, `both(@x, x^)` and a function that
takes the same two things the other way round both work.

A function gives a heap value back by returning it. Its type ends in a `^`, so the caller
knows it will own what comes back:

```go
const makeBig: (int)^[1000]int = (const x: int){
    var made: ^[1000]int = ^[...];
    return made;                       // made ends here, so the caller takes the array: no copy
};

var mine: ^[1000]int = makeBig(1);     // mine owns the array
var flat: [1000]int = makeBig(1)^;     // flat is a plain copy; the heap array is freed at the
                                       // end of this line, because nothing owns it
makeBig(1);                            // nothing receives it, so it is freed right away
```

This is how a value outlives the scope that made it: it goes on the heap, and its ownership
passes to something that lives longer.

A function can't return the ownership of one of its own locals, because the local ends with
the call (8.9). It returns the value. So a function that changes what the caller owns takes
it, and gives the result back:

```go
const dropFirst: (@^Node)^Node = (var @first: ^Node){
    var @rest: ^Node = @first^.next;   // rest takes everything behind the first node
    return rest;                       // rest ends here, so the caller owns it
};                                     // first ends here too, and the first node is freed

list = dropFirst(@list);               // list is left null at the call, then gets the rest back
```

(`Node` is the list node from 8.14.)

**A function can return a borrower.** The borrower lives on after the call, so by rule 8.5
it can't come from anything that ends with the call: a local, a copied parameter, or a value
the function took over. It can come from a borrowed parameter, because what that points
into lives in the caller:

```go
const firstValue: (&Node)&int = (var n: &Node){
    return &n^.value;                // fine: it comes from n
};

const second: (&Node)&Node = (var n: &Node){
    return &n^.next^;                // fine: the node after the one n borrows
};

const bad: (&Node)&int = (var n: &Node){
    var local: int = n^.value;
    return &local;                   // ERROR: local ends with the call, and the result lives on
};
```

At the call, the result counts as coming from everything that was lent to the function,
because the caller only sees the type:

- it can't outlive any of the borrowed arguments (8.5);
- if any of them is a `const`, the result has to be a `const` borrower (8.6);
- it counts as having followed the owners on the way to what was lent, and every owner
  inside it, because the compiler doesn't know how deep the function went (8.12).

```go
var v: &int = firstValue(&list^);    // v comes from list's data
list = makeNode(9);                  // ERROR: v is used below
v^
```

A returned borrower can still change what lies beyond itself:

```go
var cur: &Node = second(&list^);
cur^.next = makeNode(9);             // fine: reached through cur itself
```

With two borrowed arguments the type can't say which one the result came from, so it counts
as coming from both.

The same holds when an argument borrows a value made on the spot (8.4). That value is freed
at the end of the statement, so the result can't be kept past it:

```go
var gone: &int = firstValue(&makeNode(1)^);   // ERROR: the new node is freed at the end of
                                              // this line, but gone would still be there
```

**(filled in) A result that can hold a pointer or a function counts like one.** The caller
sees only the type of a result. When that type has a `&` or an `@` in it, or is an `any`, the
result counts like a borrower that came back: it comes from everything that was lent to the
call. When the type has a function type in it, or is an `any`, the result also counts like a
function given back (7.8): it comes from every argument except those handed over with `@`,
and it can't outlive the block of the call. A result with none of these in its type is a
plain copy, and it lives as long as the holder it is put in.

### 8.14 A linked list, step by step

A node can't hold another node directly, because it would have no fixed size. It holds a
heap owner of the next node (struct types are in section 9):

```go
const Node: type = {int, ^Node};       // value, next

const makeNode: (int)^Node = (const v: int){
    var n: ^Node = ^{ value: int = v; next: ^Node = null; };
    return n;
};
```

**Building.** Each node is owned by the `next` in front of it, so the whole chain belongs
to `list`:

```go
var list: ^Node = makeNode(1);         // list owns the node holding 1
list^.next = makeNode(2);              // that node's next owns the node holding 2
list^.next^.next = makeNode(3);        // and that node's next owns the node holding 3

list^.next^.next^.value                // 3
```

When `list` ends, the first node is freed, and every node behind it with it.

**Removing a node.** The node holding 3 moves up into the place of the node holding 2:

```go
@list^.next = @list^.next^.next;
```

Both sides are the ownership of a `next` field, so both are `@^Node`.

1. The right side is taken first: the node holding 3 leaves the second node's `next`,
   which is left `null`.
2. `list^.next` takes it over. Its old value, the node holding 2, has no owner now, so it
   is freed. Its `next` is already `null`, so nothing else goes with it.
3. The list is now 1, 3.

The same line with no marks, `list^.next = list^.next^.next;`, ends with the same list but
does more work: it copies the node holding 3, and then frees the node holding 2 together
with the original node holding 3.

**Replacing a node.** An owner that is given a new value frees the one it had, and
everything behind it:

```go
list^.next = makeNode(9);              // the old second node, and all the nodes behind it, are freed
```

**Adding a node at the front.**

```go
var first: ^Node = makeNode(0);
@first^.next = @list;                  // first's next takes over the whole old list, and list is null
@list = @first;                        // list takes over the new first node, and first is null
```

**Copying the list.** `=` makes a complete copy (8.8):

```go
var other: ^Node = list;               // every node is copied; other shares nothing with list
```

**Walking the list.** A function borrows an owner, checks it for `null`, and calls itself
on the next one:

```go
const length: (&^Node)int = (const owner: &^Node){
    const cases: ?(&^Node)int = [
        (const o: &^Node){ return 1 + length(&o^^.next); },    // false: count this node, go on
        (const o: &^Node){ return 0; }                          // true: no node here, so stop
    ];
    return cases[owner^ == null](owner);
};

length(&list)                          // the number of nodes
```

`o^` is the owner and `o^^` is the node it owns, so `&o^^.next` borrows the next owner.
Nothing is copied and nothing changes owner on the way.

### 8.15 Two links to one value

Two links can lead to the same value when one **owns** it and the other **borrows** it.
That makes a second link possible: a list that can be walked backwards, or a tree node that
points back at its parent.

```go
const Link: type = {int, ^Link, &Link};        // value, next (owned), prev (borrowed)

var a: ^Link = ^{ value: int = 1; next: ^Link = null; prev: &Link = null; };
a^.next = ^{ value: int = 2; next: ^Link = null; prev: &Link = &a^; };

a^.next^.prev^.value                           // 1: forward to the second node, back to the first
```

The second node is owned by the first node's `next`, and its `prev` borrows the first
node. A value never has two owners: any second link is a borrower.

The rule from 8.5 still applies: the borrower can't outlive the owner. It holds here,
because the second node is freed whenever the first one is. Removing or replacing a node
that another node's `prev` still borrows is the programmer's job to get right (8.17).

**Copying a list like this takes a function of its own.** A plain `=` copies field by field
(8.8), so each copied node's `prev` would still borrow a node of the original. The copy
function makes the plain copy, then sets every `prev` inside it:

```go
const linkBack: (&^Link, &Link)void = (var owner: &^Link, var before: &Link){
    const cases: ?(&^Link, &Link)void = [
        (var o: &^Link, var back: &Link){
            o^^.prev = back;                  // this node's prev borrows the node before it
            linkBack(&o^^.next, &o^^);        // go on: the next owner, with this node before it
        },
        (var o: &^Link, var back: &Link){ }   // no node here, so stop
    ];
    cases[owner^ == null](owner, before);
};

const copyLinks: (&^Link)^Link = (const from: &^Link){
    var made: ^Link = from^;                  // field by field: every node is copied
    linkBack(&made, null);                    // set every prev inside the copy
    return made;
};

var b: ^Link = copyLinks(&a);                 // b is a separate list, with its own prev links
```

### 8.16 What the compiler rejects, and what is checked while running

**Rejected before the program runs:**

| Line | Reason |
|---|---|
| `var p: ^int = 5;` | `5` is an `int`, not a `^int`: write `^5` |
| `var x: int = p;` where `p` is a `^int` | write `p^` |
| `var b: &int = count;` and `bump(n);` | an `int` is not an `&int`: write `&count`, `&n` |
| `var b: &int = &p;` where `p` is a `^int` | `&p` borrows the owner itself (`&^int`): write `&p^` |
| `var q: ^int = b;` where `b` is an `&int` | a borrower can't become an owner: `^b^` makes a copy |
| `x^` where `x` is a plain `int` | there is nothing to follow |
| `var b: &int = &3;` | the `3` is gone after its statement, so it can be borrowed only as an argument (8.4) |
| `var top: &int = &three(7)[0];` | the borrower would outlive what it borrows |
| `a = &inner;` where `inner` is in an inner block | the borrower would outlive what it borrows |
| `var a: &int = &limit;` where `limit` is `const` | a borrower of a `const` has to be `const` |
| `c^ = 11;` where `c` is a `const` borrower | a `const` borrower can only read |
| `look = &limit;` where `look` is `const` | a `const` borrower always borrows the same thing |
| `table^[0] = 9;` where `table` is a `const` owner | a `const` owner's value can't change |
| `q = @p;` and `@q = p;` | the two sides have different types: a move has an ownership on both sides |
| `var @t: ^int = @fixed;` where `fixed` is `const` | a `const` has no ownership to give |
| `var @t: ^int = @b^;` where `b` is a borrower | a borrower can read and change, but not take |
| `return @local;` | the ownership would outlive its variable |
| `var h: @^Node = @makeNode(1);` | the value has no variable, so the ownership would outlive it |
| `p = ^6;` where a borrower made with `&p^` is used below | it would free what the borrower points at (8.12) |
| `var @inner: ^T = @data;` where a borrower of `data^` is used below | it would move what the borrower points at (8.12) |
| `take(@list);` or `reset();` where a pointer into `list` is used below | the call takes or changes what the pointer points at (8.12) |
| `return &local;` | the borrower would outlive the local (8.13) |
| `return (){ return x; };` where `x` is declared inside the function | `x` would die before the function that uses it (7.8) |
| `var addTen: (int)int = makeAdder(10);` | the 10 dies at the end of the line, before the function made from it (7.8) |
| `var v: &int = firstValue(&makeNode(1)^);` | the result would outlive the node made on the spot (8.13) |
| `take(big);` where the parameter asks for an ownership | `big` is a `^[1000]int`, not an `@^[1000]int`: write `@big` |
| `fs[0](&limit)` where `limit` is `const` **(adjusted)** | a `const` can't be lent to a function the compiler can't see (8.6) |
| `var @t: ^int = h;` where `h` is a `const` ownership **(filled in)** | a `const` ownership can only read (8.6) |
| `var n: int = null;` **(filled in)** | `null` fits only a `^T`, an `&T` or an `@T` (8.10) |
| `both[0] = &inner;` where `both` outlives `inner` **(filled in)** | a value that holds a borrower can't outlive what the borrower borrows (8.5) |
| `keep = makeAdder(five);` inside a function, where `keep` is a top-level `var` **(adjusted)** | a function given back can't outlive the block its call was written in (7.8) |

**Checked while the program runs:**

| Line | Fails when |
|---|---|
| `p^`, for a heap owner or a borrower | `p` is `null` |
| `h^`, for an ownership **(filled in)** | `h` is `null` (8.10) |
| `^v` **(filled in)** | there is no room left on the heap (11.7) |

Every check the compiler makes, and every check made while running, is listed in Appendix B.

### 8.17 What the compiler doesn't catch, and what isn't settled

Rule 8.12 follows a pointer that is held in a variable. It can't follow these:

- **A borrower kept inside a heap value** (the `prev` in 8.15) whose target is removed
  first. It isn't a variable, so the compiler can't see when it will be used. Removing or
  replacing a node that another node's `prev` still borrows is the programmer's job.
  **(filled in)** Following such a borrower afterwards is not checked while running either.
  It reads or changes whatever is at the old address, and what the program does then isn't
  defined. This is the one mistake that neither the compiler nor a check while running
  catches.
- **A copy of a value whose borrowers point back into itself,** like the `prev` list in
  8.15. Copying is field by field (8.8), so the copy's `prev` links borrow nodes of the
  original until they are set again.
- **(adjusted)** A third case used to be listed here: a borrower of a `const` given to a
  function that is held in a variable, as in `fs[i](&limit)`. The compiler now rejects that
  line (8.6), by the rule 8.12 already uses for such calls: every `&` argument counts as one
  that may change.

Rule 8.12 also turns away some code that is safe, such as two borrowers returned from the
same list when one of them is used to change the list. That code has to be reordered.

**(filled in)** Nothing is left unsettled. The one open point was two functions given back by
one call that use the same `var` parameter, for example in a struct. They share it: one call
has one set of parameters (7.8).

### 8.18 Underneath: addresses

`^T`, `&T` and `@T` are all pointers. They differ in what they may do with what they point
at:

| | `^T` | `&T` | `@T` |
|---|---|---|---|
| holds | an address | an address | an address |
| can point at | only a value on the heap | anything that has an owner: a variable, an array element, a map entry, a field, or a value on the heap | a variable, a field, an element or a map entry **(filled in: a map entry)** |
| frees what it points at | yes | never | never, but the value can be taken out through it |
| made with | `^v` | `&x` | `@x` |
| followed with | `p^` | `p^` | `p^` |

An owning pointer can only point at the heap, because a value on the stack already has an
owner: the holder it lives in.

- **A `^T` holder** has a place on the stack like every name. That place holds the
  **address** of the value on the heap, and `p^` goes to that address.
- **A `&T` holder** holds the address of what it borrows: a variable, an array element, a
  map entry, a field, or a value on the heap.
- **`null`** is "no address".
- **`=` between two heap owners** makes room on the heap and copies the value into it.
- **An `@T` holder** holds the address of the variable it is the ownership of. For
  `h: @^int` that is a pointer to a pointer, and each `^` in `h^^` is one jump.
- **A move** copies the heap address out of the old owner into the new one and sets the old
  owner to `null`. The value itself doesn't move.
- **A link to an unsized array** (`^[]int`, `&[]int`) holds two things: where the elements
  start, and how many there are. That is how `v^.length` works.

---

## 9. Structs

### 9.1 A struct type

A struct's type is its shape: the field types in braces. Like every type, it is named with
`const` (4.1):

```go
const Point: type = {int, int};
const Settings: type = {bool, int};
```

### 9.2 Making a struct value

A struct value is a block of fields, one per type in the shape and in the same order. Each
field is a name, a type and a starting value:

```go
var p: Point = {
    x: int = 1;
    y: int = 2;
};
```

A struct is declared with `const` or `var`, like any other value. The fields carry no
`const` or `var` of their own (9.3).

**(filled in)** The block has one field for each type in the shape, in the same order, and
the type written in each field is the type at its position. A field's value can use the
fields written above it, like the names above a declaration in any block (2.8):

```go
var q: Point = { x: int = 1; };                   // ERROR: a Point has two fields
var r: Point = { x: int = 1; y: bool = true; };   // ERROR: the second field of a Point is an int
var s: Point = { x: int = 2; y: int = x * 3; };   // fine: y uses x, so y is 6
```

This mirrors functions: the type lists only the types, and the value gives the names.

| | Type (after `:`) | Value (after `=`) |
|---|---|---|
| function | `(int, int)int` | `(const x: int, const y: int){ … }` |
| struct | `{int, int}` | `{ x: int = 1; y: int = 2; }` |

`Point` is a type. It is not a value of that type, and it is never called:

```go
var p: Point = Point;              // ERROR: Point is a type, not a value of type Point
var p: Point = Point(1, 2);        // ERROR: a type is never called
```

To make a struct from inputs, write an ordinary function that returns one:

```go
const makePoint: (int, int)Point = (const a: int, const b: int){
    var p: Point = { x: int = a; y: int = b; };
    return p;
};

var corner: Point = makePoint(3, 4);   // x is 3, y is 4
```

### 9.3 Fields follow the struct's `const` or `var`

`.` reaches a field. The word the struct is declared with covers every field in it:

| Declared with | Its fields |
|---|---|
| `var` | can all be changed |
| `const` | can never be changed |

```go
var p: Point = { x: int = 1; y: int = 2; };
const origin: Point = { x: int = 0; y: int = 0; };

p.x                    // 1
p.x = 5;               // fine: p is a var
origin.x = 1;          // ERROR: origin is const
```

Arrays and maps follow the same rule (5.2, 6.2).

**(filled in) A function inside a `const` struct can't change its fields either** (9.6). When
the struct is declared with its block, the compiler sees the function, and rejects the change
where it is written. When the struct value comes from somewhere else, the compiler can't tell
which function a field holds. Then calling that field through a `const` struct counts as a
call that may change the fields, as in 8.12, and it is an error:

```go
const fixed: {int, (int)void} = {
    count: int = 0;
    add: (int)void = (const n: int){
        count = count + n;     // ERROR: count is a field of a const struct
    };
};

const frozen: {int, (int)void} = counter;   // a copy of the var struct counter from 9.6
frozen.count                   // fine: reading a field
frozen.add(1);                 // ERROR: frozen is const, and add may change its fields
```

### 9.4 Setting a struct to another struct

A `var` struct can be given a whole new value of the same type. Every field is set:

```go
p = origin;                        // p.x is now 0 and p.y is now 0
p = { x: int = 7; y: int = 9; };   // a block works too
origin = p;                        // ERROR: origin is const
```

### 9.5 Singletons

A singleton is a value whose struct type has no name. It's declared directly, like any
other value:

```go
var settings: {bool, int} = {
    debug: bool = false;
    size: int = 3;
};

settings.size = 5;          // fine: settings is a var
```

### 9.6 Functions inside a struct

A function declared inside a struct is in the struct's scope, so it uses the struct's
fields directly. There is no `this`:

```go
var counter: {int, (int)void} = {
    count: int = 0;
    add: (int)void = (const n: int){
        count = count + n;     // count is the struct's own field
    };
};

counter.add(5);                // counter.count is now 5
counter.add(2);                // counter.count is now 7
```

**A function inside a struct uses the fields of the copy it is called through.** After a copy,
each copy's functions change its own fields:

```go
var other: {int, (int)void} = counter;     // a full copy, add included
other.add(1);                  // other.count is now 8, and counter.count is still 7
```

Files work the same way (11.3).

**(filled in) A function read out of a struct uses the struct it was read from.** It borrows
that struct's fields (7.8), so it can't outlive the struct, and a call to it counts as
changing the struct (8.12):

```go
const addToCounter: (int)void = counter.add;
addToCounter(3);               // counter.count goes up by 3
```

**(filled in) Underneath,** a function written in a struct value keeps the address of that
struct value (7.7). When the struct is copied or moved, each function in the new struct that
kept the address of the old struct keeps the address of the new one instead. That is how each
copy's functions come to use its own fields.

### 9.7 A struct is copied, like every value

Every declaration and every assignment copies the whole block:

```go
var q: Point = p;      // a full copy: q gets its own block
q.x = 5;               // p.x doesn't change
```

To use the same struct instead of a copy, borrow it (section 8):

```go
var same: &Point = &p;
same^.x = 9;           // p.x is now 9
```

A struct can hold heap owners and borrowers as fields. A linked list is built that way
(8.14).

Everything in section 8 works on a struct the way it works on any variable, field by field
(8.8).

A struct is a value like any other, so it goes in arrays and maps and in and out of
functions:

```go
const sumOf: (Point)int = (const pt: Point){
    return pt.x + pt.y;
};
sumOf(p)               // the function gets its own copy of p
```

### 9.8 Underneath: the variable is its block

Writing a value on its own puts it on the stack and removes it again: `4;` adds 4 to the
stack, and it's gone once the statement is done. A struct block works the same way.
Writing

```go
{ x: int = 1; y: int = 2; };
```

lays its fields out on the stack one after another, and they're removed again. Saving the
block in a variable keeps it: the variable's own place on the stack holds the whole block,
just as an `int` variable holds its number. A field is found from the start of that block
and its position in the shape (`{int, int}`: `x` first, `y` second).

### 9.9 Field names

**(filled in)** A struct type lists only types (9.1), and the struct value gives the names
(9.2). So the names are not part of the type: `{ x: int = 1; y: int = 2; }` and
`{ w: int = 3; h: int = 4; }` both have the type `{int, int}`, and either one goes wherever
that type is asked for (3.6).

**A field name is a label for a position in a struct type.** Every struct value labels the
positions of its type. `p.x` is the field at the position labelled `x`. The compiler gathers
the labels of a struct type from every struct value of that type in the program: the file you
run and every file it imports, directly or through other files.

```go
const sumOf: (Point)int = (const pt: Point){
    return pt.x + pt.y;        // x and y: the labels that struct values of {int, int} give
};
```

| Rule | Example |
|---|---|
| a name that no struct value of the type gives is an error | `p.z`, when no `{int, int}` value has a field `z` |
| one name can't label two positions of the same type | `{ x: int = 1; y: int = 2; }` and `{ y: int = 1; x: int = 2; }` in one program are an error at both |
| one position can have several names | with the two values at the top, `p.x` and `p.w` are the same field of any `{int, int}` |

A `Point` and a size made of two `int`s are the same type, so a function that takes one takes
the other, and reads its fields by either set of names. It is the picture from 5.7 again: the
fields of a struct are reached by position, and their names are labels for the positions.

Inside its own block, a field is also a plain name: for the fields below it (9.2) and for the
functions written in the block (9.6). Like every name, it can't be spelled like a name that
can already be seen there (2.8).

---

## 10. Choosing and repeating

A choice is made by a lookup. MyLang has no loops: repeating is a function calling itself.

### 10.1 Choosing by lookup: `?T`

A pair (5.8) holds one value for `false` and one for `true`, and a `bool` picks between them.

```go
const sizes: ?int = [3, 10];           // false, then true
const size: int = sizes[count > 10];   // 10 when count is over 10, otherwise 3
```

Both values in a pair are worked out before the lookup. To run code for only the
chosen side, put **functions** in the slots: look up the function, then call it.

```go
const absOf: ?(int)int = [
    (const k: int){ return k; },           // false: not below 0
    (const k: int){ return 0 - k; }        // true: below 0
];

absOf[n < 0](n)        // pick the function for the answer, then call it
```

The slot for `false` is written first, because `false` is the first slot.

### 10.2 Choosing between many: a map of functions

For more than two cases, put one function per case in a map, with the case as the key.
Looking up the key picks the function, and calling it runs that case. This does the job of
a `switch`:

```go
const calc: (char, int, int)int = (const op: char, const a: int, const b: int){
    const cases: {char}(int, int)int = {
        '+': (const x: int, const y: int){ return x + y; },
        '-': (const x: int, const y: int){ return x - y; },
        '*': (const x: int, const y: int){ return x * y; }
    };
    return cases[op](a, b);            // pick the function for op, then call it
};

calc('+', 2, 3)        // 5
calc('-', 2, 3)        // -1
calc('*', 2, 3)        // 6
```

Every case is a key of the map. Looking up a key that isn't in the map fails while the
program runs:

```go
calc('/', 2, 3)        // fails while running: '/' isn't a key of cases
```

The cases are ordinary values, so they can be named functions too:

```go
const cases: {int}(int, int)int = {1: add, 2: sub};
cases[2](5, 3)         // 2
```

### 10.3 Repeating: a function that calls itself

A function can call itself by its own name. A pair decides when to stop:

```go
const factorial: (int)int = (const n: int){
    const cases: ?(int)int = [
        (const k: int){ return k * factorial(k - 1); },    // false: keep going
        (const k: int){ return 1; }                         // true: n is 0, so stop
    ];
    return cases[n == 0](n);
};

factorial(4)           // 24
```

The same shape goes through an array, one index at a time:

```go
const sumFrom: (&[]int, int)int = (const values: &[]int, const i: int){
    const cases: ?(&[]int, int)int = [
        (const v: &[]int, const k: int){ return v^[k] + sumFrom(v, k + 1); },   // false: add this one, go on
        (const v: &[]int, const k: int){ return 0; }                            // true: past the end, so stop
    ];
    return cases[i == values^.length](values, i);
};

var scores: []int = [10, 20, 30];
sumFrom(&scores, 0)    // 60
```

`sumFrom` borrows the array, so no call copies it (5.6).

### 10.4 How deep a function can call itself

**(filled in)** Every call that hasn't ended holds its parameters and names on the stack
(2.6). The stack has a limit, set by the system the program runs on. A program that calls
deeper than the stack allows fails while it runs (11.7), like any other check. `sumFrom` above
has one call on the stack for each element, because every call still has an addition to do
when the next call comes back.

**The last call takes over the stack of the function that makes it.** When the last thing a
function does is a call, its own parameters and names aren't needed any more. So they end
before that call starts, and the call uses their place on the stack. A function that calls
itself as its last step can repeat any number of times, and the stack doesn't grow.

A call is the last thing a function does when both of these hold:

| Condition | In detail |
|---|---|
| nothing comes after it | it is the whole value of a `return`, as in `return f(a, b);`. Or the function returns nothing, and the call is the last statement of its body |
| it doesn't need the function's own names | no argument is, or holds, a borrower or an ownership of a parameter or a name declared in the function. And the function that is called doesn't use one of them (7.8) |

The arguments, and the function to call, are worked out before the names end (2.9). In every
other case the call is an ordinary one, and the names end after it, as in 2.6.

```go
const sumTo: (&[]int, int, int)int = (const values: &[]int, const i: int, const total: int){
    const cases: ?(&[]int, int, int)int = [
        (const v: &[]int, const k: int, const t: int){ return sumTo(v, k + 1, t + v^[k]); },   // false: go on
        (const v: &[]int, const k: int, const t: int){ return t; }                              // true: past the end
    ];
    return cases[i == values^.length](values, i, total);
};

sumTo(&scores, 0, 0)   // 60, and the stack stays as small for a million elements
```

Every call here is a last call. The running total is handed on as an argument, so nothing is
left to do when a call comes back. `values` and `v` borrow the caller's array, not a name of
`sumTo`, so they don't hold the stack either.

---

## 11. Files and packages

### 11.1 A file is a package

Every file is a package. Its top-level declarations are the package's names, and a file sees
only its own names until it imports another file. So you can always tell where a name comes
from: it is declared in this file, or it is reached through a package this file imported.

Inside a file, `{ }` blocks still make smaller scopes (2.6).

### 11.2 Using another file: `import`

A file brings in another file with an ordinary declaration:

```go
// the file geom
const Box: type = {int, int};
const area: (int, int)int = (const w: int, const h: int){
    return w * h;
};
var made: int = 0;
```

```go
// the file main
const geom: package = import("geom");

geom.area(3, 4)                        // 12
const Box: type = geom.Box;            // a type from geom, named here too
geom.made = geom.made + 1;             // geom's variable changes
```

| Piece | What it is |
|---|---|
| `import("geom")` | a built-in that gives the file `geom` as a value |
| `package` | the type of that value, built in like `type` and `any` |
| `geom.area` | one of the package's names, reached with `.` like a struct field |

- **`import` is a built-in, like `convert` and `wrap` (4.4, 4.5),** so the line is an ordinary
  declaration in the one shape every name uses (2.1).
- **A package can be a `const` or a `var`,** like a function (11.5).
- **The name in `import` is the file's path from the importing file's folder:**
  `import("shapes/geom")`. A folder only organises files.
- **Each import makes a new copy of the file's variables,** because every declaration makes a
  new value (2.4). Importing the same file twice gives two packages with separate variables. To
  share one copy, borrow it: `&geom` (8.4).
- **Two files can't import each other,** directly or through other files: each import would
  make a new copy, forever.

**(filled in) What `import` takes.** `import` takes one text, and the text is written out in
the code, so the compiler knows every file of a program before it runs:

```go
const geom: package = import("geom");            // the file geom.my, in this file's folder
const deep: package = import("shapes/geom");     // shapes/geom.my
const util: package = import("../common/util");  // ../common/util.my: .. is the folder above
const which: package = import(fileName);         // ERROR: the file has to be written out
```

- The text is the file's path without the `.my` (2.7). Its parts are joined with `/`.
- `import` has the type `([]char)package`. Unlike the other built-ins, it is always called by
  its own name with its text written out, because a file that is only named while the program
  runs couldn't be checked.

**(filled in) What an import does when it runs.** An import is a call, so it runs where it is
written. It makes a new package value: it works out the file's top-level declarations, from
the top down (11.4). That is all an import runs. As a top-level declaration it runs when its
own file is set up. Inside a function it runs at each call of that function, and makes a new
copy each time.

### 11.3 What a file can use from another

Everything at the top level of an imported file can be used: its types, functions and
variables. A package's variables live as long as the package value that holds them, and a call
that changes them counts as changing them (8.12).

**A function in a file uses the variables of the copy it is called through,** the same rule as
for a function inside a struct (9.6):

```go
// the file counter
var count: int = 0;
const bump: ()void = (){
    count = count + 1;
};
```

```go
// the file you run
const a: package = import("counter");
const b: package = import("counter");

a.bump();
a.bump();
a.count                // 2
b.count                // 0: b has its own count
```

**(filled in) A file's names keep their own `const` and `var`.** A struct's fields follow the
struct's word, because they have none of their own (9.3). A file's top-level names each have
one. So `a.count` can be given a new value because `count` is a `var` in its file, even though
`a` is a `const`, and a `const` name of a file never changes. The word on the package itself
says only whether the whole package can be given another file (11.5) or have its value moved
away (11.6):

```go
a.count = 5;                   // fine: count is a var in the file counter
a.bump = a.bump;               // ERROR: bump is a const in the file counter
a = import("counter");         // ERROR: a is const
```

**(filled in) How the compiler checks a name reached through a package.** An `import` names
its file in the code, and a package keeps its file wherever it goes: into a declaration, an
assignment, an argument, a result, an element or a field. The compiler follows every package
from its `import`, so for every place that holds a package it knows which files can be there.

| The place can hold | `g.name` is accepted when |
|---|---|
| one file | `name` is a top-level name of that file |
| more than one file: a `var` package given another file (11.5), a parameter that calls give different packages (11.6), an array of packages | `name` is a top-level name of every one of those files, with the same type in each. It can be assigned when it is a `var` in each |

Two places need the worst case, as in 8.12. A function called through a variable, an array or
a map could be any function of its type, so a package handed to it has to have the names that
every function of that type in the program uses through that parameter. And a package taken
out of an `any` (4.5) could be any file the program wraps into one.

### 11.4 Order, where statements go, and `main`

- **A function's body can use any top-level name of its own file,** wherever in the file it is
  declared. A body runs only when the function is called, so two functions can call each
  other. The value in a top-level declaration still uses only names declared above it.
- **Statements that run, such as `countUp();`, go only in the file you run.** An imported file
  holds only declarations. Nothing runs because a file was imported, so the order the program
  runs in never depends on imports.

```go
// the file geom
area(1, 1);            // ERROR: geom is imported, so it holds only declarations
```

**It is good practice to put what the program does in a `const main: ()int`.** It isn't required,
and code can still go outside it:

```go
// the file you run
const geom: package = import("geom");

const main: ()int = (){
    const size: int = geom.area(3, 4);
    return 0;
};
```

When the file you run declares `main`, the program runs that file's statements from the top,
then calls `main`. The `int` it returns is the program's result, `0` when everything went well.

**(filled in) When a file's names are set up.** The top-level declarations of a file are
worked out in order, from the top down, each value when its line is reached. For the file you
run, that happens when the program starts, together with its statements, in the order
written. For every other file it happens each time an `import` of it runs (11.2).

A function's body can use top-level names declared below it, so a call could reach a name
that isn't set up yet. The compiler rejects that call: **a call at the top level can run only
when every top-level name its function uses is declared above the call.** The compiler reads
the function for this, and the functions it calls, as in 8.12. A call through a variable, an
array or a map counts as using every top-level name of the file.

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
- The program runs the file's top-level lines from the top down, and then calls `main`.
- The `int` that `main` returns is the program's status. A status is a number from 0 to 255:
  the `int` is converted the way `convert(int, uint8)` converts it (4.4), and a number outside
  that range fails in the same way (11.7).
- A file with no `main` ends after its last line, with status 0.
- `exit` ends a program before that (11.8).

### 11.5 Underneath: a package value is an address

A package value holds the address of the file's code, the way a function value is the address
of its code (7.7), and its own copy of the file's variables. Copying a package value copies its
variables (8.8), and the code stays one address. Functions and `const`s never change, so they
are never really copied.

So a package can be held like a function. It can be a `const` or a `var`, passed to a function
or kept in an array, and a `var` one can be given another file, as a `var` function can be
given another function (7.5):

```go
var shapes: package = import("geom");
shapes.area(3, 4)              // 12, from geom

shapes = import("geomFast");   // shapes now holds a new copy of the file geomFast
shapes.area(3, 4)              // 12, from geomFast
```

The new file has to have the names the code uses through `shapes`, each with the same type, as
a function in a `var` keeps its type. So the compiler still checks every `shapes.area`, and
nothing is looked up while the program runs.

**(filled in)** This is how that works underneath. For each file the compiler writes a table
into the program, with one entry for each of the file's top-level names. A package value holds
the address of its file's table, and then its own copy of the file's variables. Every name
that the program reaches through a package has one fixed position in all these tables, so
`shapes.area` reads the entry at that position, whichever file `shapes` holds. A `var` package
has room for the variables of the biggest file it is ever given, and the compiler knows which
files those are (11.3).

### 11.6 Ownership and borrowing work on a package like on any variable

A package is a value like any other, so everything in section 8 applies to it. It is used like
a struct whose fields are the file's variables, field by field (8.8):

| You write | What happens, the same as for any variable |
|---|---|
| `var p: package = geom;` | `p` gets its own copy of `geom`'s variables (8.8) |
| `var b: &package = &geom;` | `b` borrows `geom`: one copy is shared, and nothing is copied (8.4) |
| `var h: ^package = ^geom;` | a copy of `geom` on the heap (8.3) |
| `var @mine: package = @geom;` | `mine` takes `geom`'s value over, field by field (8.9) |
| `take(@geom)` | a parameter that asks for an ownership takes it at the call (8.13) |

Borrowing is how two parts of a program share one copy:

```go
var shared: package = import("counter");

const bumpTwice: (&package)void = (var p: &package){
    p^.bump();
    p^.bump();
};

bumpTwice(&shared);
shared.count                   // 2: bumpTwice borrowed shared, so no copy was made
```

The borrowing rules hold too. A borrower of a package, or of one of its variables, can't
outlive it (8.5), and while it is used further down, the package can't be given another file
(8.12):

```go
var c: &int = &shared.count;
shared = import("counter");    // ERROR: c is used below, and this would replace what c points at
c^
```

A function that takes a package uses only the names that every package given to it has, each
with the same type. The compiler checks this at each call, as it does when a `var` package is
given another file (11.5).

---

## 12. Putting it together

```go
// name long types once
const Op: type = (int, int)int;
const Row: type = [2]int;

// functions are values
const add: Op = (const x: int, const y: int){ return x + y; };
const mul: Op = (const x: int, const y: int){ return x * y; };

// a map of functions, an array of arrays
const ops: {int}Op = {1: add, 2: mul};
var table: [2]Row = [[1, 2], [3, 4]];

// a function that takes a function and borrows an int
const applyTo: (Op, &int, int)void = (const f: Op, var target: &int, const by: int){
    target^ = f(target^, by);
};

applyTo(ops[1], &table[0][1], 10);  // table[0][1] is now 12
applyTo(ops[2], &table[1][0], 2);   // table[1][0] is now 6

// a struct type, a function that makes one, and a borrower to share it
const Point: type = {int, int};

const makePoint: (int, int)Point = (const a: int, const b: int){
    var p: Point = { x: int = a; y: int = b; };
    return p;
};

var corner: Point = makePoint(table[0][1], table[1][0]);   // x is 12, y is 6
var c: &Point = &corner;
c^.x = 0;                           // corner.x is now 0

// a value on the heap: copied with =, moved by setting an ownership
var big: ^Row = ^[1, 2];
var copy: ^Row = big;               // copy gets its own array on the heap
var @mine: ^Row = @big;             // mine takes big's array over, and big is left null
```

---

## 13. Quick reference

### Kinds of name

| Kind | Declares | New value later? |
|---|---|---|
| `const` | a value that never changes | no |
| `var` | a value that can change | yes |

### Type forms

| Form | Meaning |
|---|---|
| `int8` `int16` `int32` `int64` | signed whole numbers |
| `uint8` `uint16` `uint32` `uint64` | unsigned whole numbers |
| `float32` `float64` | numbers with a decimal point |
| `int`, `uint`, `float` | everyday names for `int64`, `uint64`, `float64` |
| `bool` | `true` or `false` |
| `char` | one character |
| `type` | the type of a type |
| `any` | a value of any type, with its type inside |
| `package` | a file used from another file (11.2) |
| `[n]T` | fixed array of `n` `T`s |
| `[]T` | unsized array of `T`s: the length comes from the first value |
| `&a[i:j]` | a slice: borrows elements `i` up to `j` of `a`, as an `&[]T` (5.9) |
| `?T` | pair: one `T` for `false`, one for `true` |
| `{K}V` | map from `K` keys to `V` values |
| `(P1, P2)R` | function taking `P1` and `P2`, returning `R` |
| `(P1)void` | function returning nothing |
| `{T1, T2}` | struct with fields of types `T1` and `T2` |
| `^T` | a `T` on the heap, owned by this holder |
| `&T` | a `T` borrowed from another holder |
| `@T` | the ownership of a `T` variable |

### Operators

| Operators | Work on | Give |
|---|---|---|
| `+` `-` `*` | two numbers of the same type | a number of that type |
| `==` `!=` `<` `>` `<=` `>=` | two values of the same type | a `bool` |
| `&&` `\|\|` | two `bool`s | a `bool`: and, or. The right side is worked out only when it's needed |
| `!` | one `bool` | a `bool`: not |
| `( )` around a value | any value that isn't a type | the same value, worked out first |

The order of operations is the usual one: what is behind a value (`p^`, `a[0]`, `f(1)`,
`pt.x`), then what is in front of it (`!`, `^`, `&`, `@`), then `*`, then `+` and `-`, then the
comparisons, then `&&`, then `||`. Operators in the same step go from left to right, and
brackets go first (3.2).

### Ownership

| You write | What it does |
|---|---|
| `^v` | puts a copy of `v` on the heap: a `^T` |
| `&x` | borrows `x`: an `&T` |
| `@x` | the ownership of `x`: an `@T` |
| `p^` | what a heap owner, a borrower or an ownership leads to |
| `n = p;` | copies: `n` gets its own value, and `p` is unchanged |
| `@n = @p;` | moves: `n` takes over what `p` owns, and `p` is left `null` |
| `var @n: ^int = @p;` | the same, for a new variable |
| `f(@p)` | the same, for a parameter that asks for an ownership: `p` is left `null` at the call |
| `p = null;` | frees what `p` owns |
| `p == null` | `true` when `p` owns nothing |

### Choosing and repeating

| You want | You write |
|---|---|
| to test several things at once | `count > 0 && count < 10` (3.2) |
| to stop a function that returns nothing early | `return;` (7.3) |
| to choose a value by a `bool` | a pair: `sizes[count > 10]` (10.1) |
| to choose between many, like a `switch` | a map of functions: `cases[op](a, b)` (10.2) |
| to repeat, like a loop | a function that calls itself (10.3) |

### Built-in functions

| You write | Its type | What it does |
|---|---|---|
| `convert(float, int)` | `(float)int` | converts a value from the first type to the second (4.4) |
| `wrap(int)` | `(int)any` | wraps a value in an `any` (4.5) |
| `unwrap(int)` | `(any)int` | takes the value back out of an `any`, checked while running (4.5) |
| `import("geom")` | `package` | gives the file `geom` as a package (11.2) |

Built-ins are first-class values like everything else: they can be named, stored and
passed.

### What copies and what shares

| You write | What happens |
|---|---|
| `var b: int = a;` | `b` is a copy |
| `b = a;` | `a`'s value is copied into `b` |
| `f(a)` with a `const` or `var` parameter | the parameter is a copy |
| `var q: ^int = p;` | `q` gets its own copy, in a new place on the heap |
| `var h: ^int = ^a;` | the heap gets a copy of `a` |
| `var x: int = p^;` | `x` is a plain copy of the heap value |
| `var b: &int = &a;` | `b` borrows `a`: no copy |
| `f(&a)` with an `&int` parameter | the parameter borrows the caller's `a`: no copy |
| `var s: &[]int = &a[1:3];` | `s` borrows two elements of `a`: no copy (5.9) |
| `var part: []int = a[1:3];` | `part` is a copy of two elements of `a` (5.9) |
| `look(&makeBig(1)^)` | the parameter borrows the value made on the spot: no copy, and it is freed at the end of the statement |
| `var h: @^int = @p;` | `h` is `p`'s ownership: no copy, and nothing moves |
| `var @r: ^int = @p;` | `r` takes over what `p` owned: no copy, and `p` is left `null` |
| a function uses `x` from around it | the function borrows `x`: no copy, and `x` can't die before the function (7.8) |
| a function given back uses a parameter `p` of the call | the function keeps `p`, and can't outlive the call's arguments, except those handed over with `@` (7.8) |
| keeping a local in a function given back | a helper whose parameter takes the value: `return countFrom(@total);` (7.8) |
| `var p: package = geom;` | `p` gets its own copy of `geom`'s variables; the code isn't copied (11.5) |
| `import("geom")` twice | two packages, each with its own variables (11.2) |
| `&geom`, `^geom`, `@geom` | borrows, puts on the heap or moves a package, like any variable (11.6) |

### What never changes

| Thing | Fixed by |
|---|---|
| a `const` | its declaration |
| a named type | its declaration |
| the type of a value | its declaration |
| the length of an array | the type `[n]T`, or the first value of a `[]T` |
| the keys of a map | its first value |
| what a `const` borrower borrows | its declaration |

### What the compiler rejects

| Line | Reason |
|---|---|
| `limit = 11;` | `limit` is `const` |
| `var big: uint8 = 300;` | `300` doesn't fit in a `uint8` |
| `var rate: float = 3;` | a float is written with a decimal point: `3.0` |
| `var count: int = small;` where `small` is an `int8` | a value never changes type by itself: write `convert(int8, int)(small)` |
| `count + small` | `+` needs two numbers of the same type |
| `var T: type = int;` | a type can't be a `var` |
| `int(3.0)` | a type is never called: write `convert(float, int)(3.0)` |
| `describe(3);` | `describe` takes an `any`: write `wrap(int)(3)` |
| `double(x);` where `x` is an `any` | write the unwrap: `double(unwrap(int)(x))` |
| `primes[0] = 7;` | `primes` is `const` |
| `sizes[1]` where `sizes` is a `?int` | a pair is looked up with a `bool` |
| `var sizes: [bool]int = [3, 10];` | the size of an array is a number (5.1) |
| `shapes = import("other");` where `other` lacks a name the code uses from `shapes` | a package in a `var` can only be given a file with the same names and types (11.5) |
| two files that import each other | imports can't go round in a circle (11.2) |
| a statement such as `countUp();` in a file another file imports | an imported file holds only declarations (11.4) |
| `count && ready` and `!count` where `count` is an `int` | `&&`, `\|\|` and `!` work on `bool`s: write `count != 0`, `count == 0` |
| a function of type `(int)int` that can reach the end of its body without a `return` | a function that returns a value has to reach a `return` on every path (7.1) |
| `return 5;` in a function of type `()void` | a function that returns nothing gives no value back (7.3) |
| `return;` in a function of type `(int)int` | a function that returns a value has to give one (7.3) |
| `a = [1, 2, 4, 5];` where `a` has length 3 | an unsized array keeps its first length |
| `scores.length = 4;` | a length never changes |
| `&primes[2:1]` and `&primes[0:5]` where `primes` is a `[3]int` | a slice's start can't be after its end, or its end past the length (5.9) |
| `seen[7] = true;` | `7` isn't a key, and a map can't grow |
| `var p: ^int = 5;` | `5` is an `int`, not a `^int`: write `^5` |
| `var x: int = p;` where `p` is a `^int` | write `p^` |
| `var b: &int = count;` and `bump(n);` | an `int` is not an `&int`: write `&count`, `&n` |
| `var b: &int = &3;` | the `3` is gone after its statement: it can be borrowed only as an argument (8.4) |
| `var top: &int = &three(7)[0];` | the borrower would outlive what it borrows |
| `var a: &int = &limit;` where `limit` is `const` | a borrower of a `const` has to be `const` |
| `q = @p;` and `@q = p;` | the two sides have different types: a move has an ownership on both sides |
| `var @t: ^int = @fixed;` where `fixed` is `const` | a `const` has no ownership to give |
| `p = ^6;` where a borrower made with `&p^` is used below | it would free what the borrower points at (8.12) |
| returning a function that uses a variable declared inside the function around it | that variable would die before the function (7.8) |
| `Point(1, 2)` | a type is never called |
| `var p: Point = Point;` | `Point` is a type, not a value of type `Point` |
| `origin.x = 1;` | `origin` is `const`, so none of its fields can change |

The full list for ownership is in 8.16.

Four checks happen while the program runs:

| Line | Fails when |
|---|---|
| `convert(int, int8)(n)` and other conversions to a smaller or differently signed type | the value doesn't fit |
| `unwrap(bool)(x)` and other unwraps of an `any` | `x` holds a different type |
| `cases[op]` and other map lookups | the key isn't in the map |
| `p^`, for a heap owner or a borrower | `p` is `null` |
| `&a[i:j]` with `i` or `j` worked out while running | the slice is out of range (5.9) |
