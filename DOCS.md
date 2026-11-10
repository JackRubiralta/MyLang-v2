# MyLang documentation

MyLang is a **declarative, explicit, first-class** language.

- **First-class:** everything is a value. Numbers, arrays, maps, functions, structs and
  even types are all declared, stored, passed and returned the same way.
- **Explicit:** reading the code tells you what happens. A value is shared only when you
  write `ref`. Otherwise every value is copied when it's declared, assigned or passed.
- **Declarative:** every name is declared in one shape that states what kind of name it
  is, what it's called, what type it has and what value it starts with. There are no loops
  and no `if` statements: a choice is a lookup, and repeating is a function calling itself.

This file documents the core of the language: values, names, types, arrays, maps,
functions, refs, structs, and choosing and repeating.

**Reading the examples**

- Code blocks are marked `go` only for colouring. The code is MyLang.
- Most examples use `int`, `float` and `bool`. Every built-in type is listed in 3.1.
- `//` starts a comment.
- `// ERROR` marks a line the compiler rejects.
- `...` stands for code that's left out.

## Contents

1. [Everything is a first-class value](#1-everything-is-a-first-class-value)
2. [Names and declarations](#2-names-and-declarations)
3. [Types](#3-types)
4. [Types are values](#4-types-are-values)
5. [Arrays](#5-arrays)
6. [Maps](#6-maps)
7. [Functions](#7-functions)
8. [`ref`: another name for a variable](#8-ref-another-name-for-a-variable)
9. [Structs](#9-structs)
10. [Choosing and repeating](#10-choosing-and-repeating)
11. [Putting it together](#11-putting-it-together)
12. [Quick reference](#12-quick-reference)

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
| `kind` | what sort of name it is: `const`, `var` or `ref` |
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

### 2.2 The three kinds of name

| Kind | Can it be given a new value? | Section |
|---|---|---|
| `const` | no | 2.3 |
| `var` | yes | 2.3 |
| `ref` | it *is* another variable: assigning to it changes that variable | 8 |

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

To get a second name for the *same* variable instead of a copy, use `ref` (section 8).

### 2.5 Statements end with `;`

Every statement ends with `;`, including a declaration whose value is a function body or
a struct block:

```go
const add: (int, int)int = (const x: int, const y: int){
    return x + y;
};                         // the ; ends the declaration
```

### 2.6 Where values live, and for how long

Every value lives on the stack. A value on the stack has a fixed size, which is why the
length of an array (5.3) and the keys of a map (6.3) never change.

A **block** is the code between `{` and `}`: a function body, or a block written on its
own. How long something lives depends on where it's written:

| What | It lives until… |
|---|---|
| a name | the end of the block it's declared in |
| a parameter | its function call ends |
| a value that isn't given a name | the end of its statement |

```go
var a: int = 1;
{
    var b: int = 2;        // b starts here
    a = a + b;             // a is now 3
}                          // b ends here
                           // a is still here

4;                         // a value with no name: it's gone once the statement is done
```

When a name ends, its value is removed from the stack.

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

Three everyday names stand for the 64-bit types:

| Name | Is the same type as |
|---|---|
| `int` | `int64` |
| `uint` | `uint64` |
| `float` | `float64` |

These are ordinary names for those types, the same as naming a type yourself (4.1), so an
`int` and an `int64` are the same type.

Two more built-in types have their own sections: `type`, the type of a type (4.1), and
`any`, a value of any type (4.5).

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

### 3.3 Every modifier goes in front

A type is a basic type, or a modifier followed by a type. A basic type is a built-in type
(3.1) or the name of a type (section 4).

| Kind | Form | Example | Read as |
|---|---|---|---|
| fixed array | `[n]T` | `[3]int` | array of 3 ints |
| unsized array | `[]T` | `[]int` | array of ints |
| choice array | `[bool]T` | `[bool]int` | two ints: one for `false`, one for `true` |
| map | `{K}V` | `{int}bool` | map from int keys to bool values |
| function | `(P1, P2)R` | `(int, int)int` | takes two ints, returns an int |
| function that returns nothing | `(P1, P2)void` | `(int)void` | takes an int, returns nothing |
| struct | `{T1, T2}` | `{bool, int}` | a struct with a bool field and an int field |

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
```

Swapping two modifiers swaps the meaning, and the order shows it:

| Type | Meaning |
|---|---|
| `[3](int)int` | array of 3 functions |
| `(int)[3]int` | function returning an array of 3 |
| `[2]{int}bool` | array of 2 maps |
| `{int}[2]bool` | map whose values are arrays of 2 |

No type ever needs brackets to group it.

### 3.5 Types read in the order you use the value

```go
// f: (int)(bool)[3]int
f(1)(true)[0]          // call with an int, call with a bool, index → an int
```

The type says *call, call, index, int*, and the use says *call, call, index*: the same
order.

### 3.6 Grammar

```
Type      = BasicType
          | "[" [ NUMBER ] "]" Type           (array: fixed or unsized)
          | "[" "bool" "]" Type               (choice array: indexed by a bool)
          | "{" Type "}" Type                 (map: {key}value)
          | "{" TypeList "}"                  (struct: its field types, section 9)
          | "(" [ TypeList ] ")" Return       (function: parameter types, then the return type)
Return    = Type | "void"                     (void = returns nothing)
TypeList  = Type { "," Type }
```

A function type always ends with its return type. A function that returns nothing ends
with `void`: `(int)void` takes an int and returns nothing, and `()void` takes nothing and
returns nothing.

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

`first(int)` gives back a function made for `int`, so what each call works with is written
right there in the code.

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

An array holds values of one type, and that can be any type:

```go
const word: [4]char = ['J', 'a', 'c', 'k'];     // text is an array of chars
const flags: [2]bool = [true, false];
```

### 5.2 Reading and changing elements

Elements are numbered from 0:

```go
primes[0]              // 2
scores[2]              // 30
scores[0] = 15;        // scores is now [15, 20, 30]
primes[0] = 7;         // ERROR: primes is const
```

An element can be changed when the array is a `var`.

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

### 5.6 Passing an array to a function

There are two ways, and the code says which one is happening:

```go
const sumCopy: ([]int)int = (const v: []int){ ... };       // v is a copy of the array
const sumRef: ([]int)int = (ref v: []int){ ... };          // v is the caller's array itself

sumCopy(scores);       // copies all of scores, like every declaration (2.4)
sumRef(scores);        // no copy: v is another name for scores (8.3)
```

Both functions have the same type, `([]int)int`. Only the parameter says `ref`.

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

### 5.8 A choice array: `[bool]T`

`[bool]T` is an array with exactly two slots that is indexed with a `bool` instead of a
number. The first slot is for `false` and the second is for `true`:

```go
const sizes: [bool]int = [3, 10];      // the slot for false, then the slot for true
sizes[false]                           // 3
sizes[true]                            // 10

const big: bool = count > 10;
sizes[big]                             // 3 or 10, picked by big
sizes.length                           // 2
```

| Type | Slots | Indexed by |
|---|---|---|
| `[3]T` | 3 | `0`, `1`, `2` |
| `[bool]T` | 2 | `false`, `true` |

The two kinds of index don't mix:

```go
sizes[1]                               // ERROR: a [bool] array is indexed with a bool
primes[true]                           // ERROR: a [3] array is indexed with a number
```

Underneath, a `[bool]T` is laid out exactly like a `[2]T`: `false` is the first slot and
`true` is the second. Its main use is choosing (section 10).

---

## 6. Maps

### 6.1 Making a map

The type is `{KeyType}ValueType`, and the value lists `key: value` pairs:

```go
var seen: {int}bool = {1: true, 4: false};
const pairs: {int}[2]int = {1: [1, 2], 2: [3, 4]};
```

### 6.2 Reading and changing entries

```go
seen[4]                // false
seen[4] = true;        // change an existing entry
pairs[2][0]            // 3: the entry for key 2, then its first element
pairs[1] = [0, 0];     // ERROR: pairs is const
```

An entry can be changed when the map is a `var`.

### 6.3 A map has a fixed size

A map's keys are set by its first value and never change, just like an array's length:

```go
seen[1] = false;       // fine: 1 is a key
seen[7] = true;        // ERROR: 7 isn't a key, and the map can't grow
seen.length            // 2: the number of keys
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

### 7.2 Calling a function

```go
add(1, 2)              // 3
```

Each argument is copied into its parameter (2.4), unless the parameter is a `ref` (8.3).

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

### 7.4 Parameters are declared like names

Each parameter is `const`, `var` or `ref` with a name and a type, like a declaration:

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
| `ref x: int` | the caller's variable itself (8.3) |

`const`, `var` and `ref` on a parameter don't appear in the function's type. The type only
lists the parameter types.

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

---

## 8. `ref`: another name for a variable

### 8.1 Declaring a ref

`ref` is a kind of name. A ref is another name for an existing variable: reading it reads
that variable, and assigning to it changes that variable.

```go
var b: int = 3;
ref r: int = b;
r                      // 3
r = 5;                 // b is now 5
b = 8;                 // r is now 8 too: they are the same variable
```

A ref always refers to the variable it was declared with. Assigning to a ref never makes
it refer to something else:

```go
var c: int = 10;
r = c;                 // copies c's value into b: b is now 10, and r still refers to b
```

### 8.2 What a ref can refer to

A ref can refer to a variable, an array element or a map entry, but never to a plain value:

```go
var scores: []int = [10, 20, 30];
ref top: int = scores[0];
top = 99;              // scores is now [99, 20, 30]

var seen: {int}bool = {1: true, 4: false};
ref four: bool = seen[4];
four = true;           // seen[4] is now true

ref bad: int = 3;      // ERROR: 3 isn't a variable
```

### 8.3 `ref` parameters

A function can take a parameter as a `ref`, so it can change the caller's variable. Only
the parameter in the function's value says `ref`. The function's type stays plain:

```go
const bump: (int)void = (ref x: int){
    x = x + 1;
};

var n: int = 3;
bump(n);               // n is now 4
bump(scores[1]);       // scores[1] goes up by 1
bump(3);               // ERROR: 3 isn't a variable

const swap: (int, int)void = (ref a: int, ref c: int){
    var t: int = a;
    a = c;
    c = t;
};

var left: int = 1;
var right: int = 2;
swap(left, right);     // left is now 2, right is now 1
```

A `ref` parameter also passes a big value without copying it (5.6).

### 8.4 A ref can't outlive what it refers to

A ref can only refer to something that lives at least as long as the ref itself. If what
it refers to would be gone while the ref is still there, the compiler raises an error.

```go
var x: int = 3;
ref a: int = x;            // fine: a and x end together, at the same }
{
    ref y: int = x;        // fine: y ends at this block's }, before x does
    y = 4;                 // x is now 4
}

bump(x);                   // fine: bump's ref parameter ends when the call ends,
                           // and x is still there

ref top: int = three(7)[0];    // ERROR: the array three(7) gives back is gone at the end
                               // of this line, but top would still be there
```

(`three` is the function from 7.6 that returns an array.)

| The ref refers to… | Allowed? |
|---|---|
| a variable declared in the same block | yes: they end together |
| a variable declared in an outer block | yes: the ref ends first |
| the caller's variable, as a `ref` parameter | yes: the parameter ends when the call ends |
| a value with no name, which is gone after its statement (2.6) | no: the ref would outlive it |

### 8.5 A ref is not a type

`ref` never appears in a type, not even in a function's parameter list. A ref can't be put
in an array, a map or a struct, and can't be returned, so these don't exist:

```go
[3]ref int             // ERROR: not a type
(int)ref int           // ERROR: not a type
(ref int)void          // ERROR: not a type; write (int)void
```

### 8.6 Underneath: a ref is a saved address

A ref just holds the **address** of what it refers to: the place in memory of the variable,
array element or map entry. It works like a pointer in disguise that the language looks
after for you. Reading or assigning the ref goes to that address, so it reads and changes
the original:

```go
var scores: []int = [10, 20, 30];
ref top: int = scores[0];      // top holds the address of scores[0]
top = 99;                      // writes 99 at that address
```

You never see the address, and you can't change it: a ref can't be pointed somewhere else,
stored or returned. That, together with rule 8.4, is what keeps the address valid for as
long as the ref exists.

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

### 9.7 A struct is copied, like every value

Every declaration and every assignment copies the whole block:

```go
var q: Point = p;      // a full copy: q gets its own block
q.x = 5;               // p.x doesn't change
```

To share a struct instead of copying it, use a `ref` (section 8):

```go
ref same: Point = p;
same.x = 9;            // p.x is now 9
```

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

---

## 10. Choosing and repeating

MyLang has no `if` statement, no `switch` and no loops. A choice is a lookup: the possible
results are values, and you pick one by its index or key. Repeating is a function calling
itself.

### 10.1 Choosing between two: `[bool]T`

A choice array (5.8) holds one value for `false` and one for `true`, and a `bool` picks
between them:

```go
const sizes: [bool]int = [3, 10];      // false, then true
const size: int = sizes[count > 10];   // 10 when count is over 10, otherwise 3
```

Both values in a choice array are worked out before the lookup. To run code for only the
chosen side, put **functions** in the slots: look up the function, then call it.

```go
const abs: (int)int = (const n: int){
    const cases: [bool](int)int = [
        (const k: int){ return k; },           // false: n is not below 0
        (const k: int){ return 0 - k; }        // true: n is below 0
    ];
    return cases[n < 0](n);            // pick the function for the answer, then call it
};

abs(7)                 // 7
abs(0 - 7)             // 7
```

`cases[n < 0]` picks one of the two functions, and `(n)` calls it. The other function is
never called.

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

A function can call itself by its own name. A choice array decides when to stop:

```go
const factorial: (int)int = (const n: int){
    const cases: [bool](int)int = [
        (const k: int){ return k * factorial(k - 1); },    // false: keep going
        (const k: int){ return 1; }                         // true: n is 0, so stop
    ];
    return cases[n == 0](n);
};

factorial(4)           // 24
```

The same shape goes through an array, one index at a time:

```go
const sumFrom: ([]int, int)int = (ref values: []int, const i: int){
    const cases: [bool]([]int, int)int = [
        (ref v: []int, const k: int){ return v[k] + sumFrom(v, k + 1); },   // false: add this one, go on
        (ref v: []int, const k: int){ return 0; }                           // true: past the end, so stop
    ];
    return cases[i == values.length](values, i);
};

var scores: []int = [10, 20, 30];
sumFrom(scores, 0)     // 60
```

`sumFrom` takes the array as a `ref`, so no call copies it (5.6).

---

## 11. Putting it together

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

// a function that takes a function and a ref
const applyTo: (Op, int, int)void = (const f: Op, ref target: int, const by: int){
    target = f(target, by);
};

applyTo(ops[1], table[0][1], 10);   // table[0][1] is now 12
applyTo(ops[2], table[1][0], 2);    // table[1][0] is now 6

// a struct type, a function that makes one, and a ref to share it
const Point: type = {int, int};

const makePoint: (int, int)Point = (const a: int, const b: int){
    var p: Point = { x: int = a; y: int = b; };
    return p;
};

var corner: Point = makePoint(table[0][1], table[1][0]);   // x is 12, y is 6
ref c: Point = corner;
c.x = 0;                            // corner.x is now 0
```

---

## 12. Quick reference

### Kinds of name

| Kind | Declares | New value later? |
|---|---|---|
| `const` | a value that never changes | no |
| `var` | a value that can change | yes |
| `ref` | another name for an existing variable | assigning changes that variable |

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
| `[n]T` | fixed array of `n` `T`s |
| `[]T` | unsized array of `T`s: the length comes from the first value |
| `[bool]T` | choice array: one `T` for `false`, one for `true` |
| `{K}V` | map from `K` keys to `V` values |
| `(P1, P2)R` | function taking `P1` and `P2`, returning `R` |
| `(P1)void` | function returning nothing |
| `{T1, T2}` | struct with fields of types `T1` and `T2` |

### Operators

| Operators | Work on | Give |
|---|---|---|
| `+` `-` `*` | two numbers of the same type | a number of that type |
| `==` `!=` `<` `>` `<=` `>=` | two values of the same type | a `bool` |

### Choosing and repeating

| Instead of | You write |
|---|---|
| `if` / `else` | a choice array of functions: `cases[n < 0](n)` (10.1) |
| `switch` | a map of functions: `cases[op](a, b)` (10.2) |
| a loop | a function that calls itself (10.3) |

### Built-in functions

| You write | Its type | What it does |
|---|---|---|
| `convert(float, int)` | `(float)int` | converts a value from the first type to the second (4.4) |
| `wrap(int)` | `(int)any` | wraps a value in an `any` (4.5) |
| `unwrap(int)` | `(any)int` | takes the value back out of an `any`, checked while running (4.5) |

Built-ins are first-class values like everything else: they can be named, stored and
passed.

### What copies and what shares

| You write | What happens |
|---|---|
| `var b: int = a;` | `b` is a copy |
| `b = a;` | `a`'s value is copied into `b` |
| `f(a)` with a `const` or `var` parameter | the parameter is a copy |
| `ref r: int = a;` | `r` is another name for `a`: no copy |
| `f(a)` with a `ref` parameter | the parameter is the caller's `a`: no copy |

### What never changes

| Thing | Fixed by |
|---|---|
| a `const` | its declaration |
| a named type | its declaration |
| the type of a value | its declaration |
| the length of an array | the type `[n]T`, or the first value of a `[]T` |
| the keys of a map | its first value |
| what a ref refers to | its declaration |

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
| `sizes[1]` where `sizes` is a `[bool]int` | a choice array is indexed with a `bool` |
| `a = [1, 2, 4, 5];` where `a` has length 3 | an unsized array keeps its first length |
| `scores.length = 4;` | a length never changes |
| `seen[7] = true;` | `7` isn't a key, and a map can't grow |
| `ref bad: int = 3;` and `bump(3);` | `3` isn't a variable |
| `ref top: int = three(7)[0];` | the ref would outlive what it refers to |
| `[3]ref int`, `(int)ref int`, `(ref int)void` | `ref` is not a type |
| `Point(1, 2)` | a type is never called |
| `var p: Point = Point;` | `Point` is a type, not a value of type `Point` |
| `origin.x = 1;` | `origin` is `const`, so none of its fields can change |

Three checks happen while the program runs:

| Line | Fails when |
|---|---|
| `convert(int, int8)(n)` and other conversions to a smaller or differently signed type | the value doesn't fit |
| `unwrap(bool)(x)` and other unwraps of an `any` | `x` holds a different type |
| `cases[op]` and other map lookups | the key isn't in the map |
