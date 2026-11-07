# MyLang

A small language where **everything is a first-class value**: numbers, arrays, maps,
functions and even types are all declared, stored, passed and returned the same way.

This file holds the rules decided so far, each with the reason behind it. Anything not
decided yet is left out on purpose and listed at the end.

## The idea behind MyLang

MyLang is **declarative and explicit: reading the code tells you what happens.** Nothing
happens behind your back:

- a value goes on the heap only when you write `heap`;
- a value is shared only when you write `ref` or use a pointer you made with `&`;
- otherwise every value lives on the stack and is copied when it's declared or assigned.

Every rule below follows from this. When a choice comes up, the option that keeps what
happens visible in the code wins.

**Reading the examples**
- Code blocks are marked `go` only for colouring. The code is MyLang.
- The basic types aren't decided yet, so the examples use `int`, `float` and `bool` as stand-ins,
  and simple sums like `x + y` to show values.
- `// ERROR` marks a line the rules don't allow.

## Contents

1. [Everything is a first-class value](#1-everything-is-a-first-class-value)
2. [Declaring a name](#2-declaring-a-name)
3. [Types](#3-types)
4. [Types are values too](#4-types-are-values-too)
5. [Arrays](#5-arrays)
6. [Maps](#6-maps)
7. [Functions](#7-functions)
8. [`ref`: another name for a variable](#8-ref-another-name-for-a-variable)
9. [The stack and the heap](#9-the-stack-and-the-heap)
10. [Pointers](#10-pointers)
11. [Structs](#11-structs)
12. [Putting it together](#12-putting-it-together)
13. [Not decided yet](#13-not-decided-yet)
14. [Problems to look at later](#14-problems-to-look-at-later) (incl. struct notes, 14.12)

---

## 1. Everything is a first-class value

Functions, arrays, maps and types are values, exactly like numbers. Any value can be:

| You can… | Example |
|---|---|
| give it a name | `const add: (int, int)int = …;` |
| put it in an array | `const ops: [2](int, int)int = [add, sub];` |
| put it in a map | `const byKey: {int}(int, int)int = {1: add, 2: sub};` |
| pass it to a function | `twice(double, 3)` |
| return it from a function | `return f;` |

**Why:** with no special cases, everything combines with everything else. An array of
functions, a map of arrays or a function that returns a function needs no extra rules,
just the same rules applied again.

---

## 2. Declaring a name

### 2.1 One shape for every declaration

```go
kind name: Type = value;
```

| Part | What it says |
|---|---|
| `kind` | what sort of name it is: `const`, `var`, `ref`, `heap` or `struct` |
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

**Why:** since everything is a value (rule 1), there is no reason for functions or arrays
to be declared differently from numbers.

### 2.2 The five kinds of name

| Kind | Can it be given a new value? | Where its value lives | Section |
|---|---|---|---|
| `const` | no | stack | 2.3 |
| `var` | yes | stack | 2.3 |
| `ref` | it *is* another variable: assigning changes that variable | wherever that variable is | 8 |
| `heap` | yes | heap | 9 |
| `struct` | no: a struct type never changes | – | 11 |

### 2.3 `const` and `var`

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

To get a second name for the *same* variable instead of a copy, use `ref` (section 8).

### 2.5 Statements end with `;`

Every statement ends with `;`, including a declaration whose value is a function body:

```go
const add: (int, int)int = (const x: int, const y: int){
    return x + y;
};                         // the ; ends the declaration
```

---

## 3. Types

### 3.1 Every modifier goes in front

A type is a basic type, or a modifier followed by a type:

| Kind | Form | Example | Read as |
|---|---|---|---|
| fixed array | `[n]T` | `[3]int` | array of 3 ints |
| unsized array | `[]T` | `[]int` | array of ints |
| map | `{K}V` | `{int}bool` | map from int keys to bool values |
| function | `(P1, P2)R` | `(int, int)int` | takes two ints, returns an int |
| function that returns nothing | `(P1, P2)null` | `(int)null` | takes an int, returns nothing |
| pointer | `*T` | `*int` | pointer to an int |

### 3.2 Modifiers stack in any combination

Read every type from left to right, one modifier at a time:

```go
[3](int)int            // array of 3 → functions taking int → returning int
(int)[]int             // function taking int → returning array of ints
(int)(int)int          // function taking int → returning a function (int)int
((int)int, int)int     // function taking a function (int)int and an int → returning int
{int}(int)int          // map from int → to functions (int)int
{int}[]int             // map from int → to arrays of ints
[3]{int}bool           // array of 3 → maps from int to bool
[2][3]int              // array of 2 → arrays of 3 ints
[3]*int                // array of 3 → pointers to int
*[3]int                // pointer to → an array of 3 ints
(int)*int              // function taking int → returning a pointer to int
```

Swapping two modifiers swaps the meaning, and the order shows it:

| Type | Meaning |
|---|---|
| `[3]*int` | array of 3 pointers |
| `*[3]int` | pointer to an array of 3 |
| `[3](int)int` | array of 3 functions |
| `(int)[3]int` | function returning an array of 3 |

### 3.3 Types read in the order you use the value

```go
// f: (int)(bool)[3]int
f(1)(true)[0]          // call with an int, call with a bool, index → an int
```

The type says *call, call, index, int*, and the use says *call, call, index*: same order.

### 3.4 Grammar

```
Type      = BasicType
          | "[" [ NUMBER ] "]" Type           (array: fixed or unsized)
          | "{" Type "}" Type                 (map: {key}value)
          | "{" TypeList "}"                  (struct: its field types, section 11)
          | "*" Type                          (pointer)
          | "(" [ TypeList ] ")" Return       (function: parameter types, then the return type)
Return    = Type | "null"                     (null = returns nothing)
TypeList  = Type { "," Type }
```

A function type always ends with its return type. A function that returns nothing ends
with `null`: `(int)null`, and `()null` takes nothing and returns nothing.

`{…}` starts both a map type (`{int}bool`) and a struct type (`{int, *Node}`); how a
one-field struct `{int}` is told apart from a map isn't decided yet (section 13).

### 3.5 Why

- **Why in front:** the four options (`int[]` or `[]int`, `int(int)` or `(int)int`) were
  compared on hard nested types and on 3,000 random ones. Only the all-in-front form gave
  every type exactly one meaning without brackets. Mixing front and back (like `int[]`
  with `(int)int`) makes types such as `(int)int[3]` mean two different things.
- **Why no brackets at all:** `(…)` always starts a function type, so a bracket used for
  grouping would silently change the meaning.
- **Why it reads well:** left to right is plain English, and it's the order you use the
  value in (3.3).
- **Why no `fn` keyword:** it isn't needed, so types stay short and clean.

### 3.6 How the type syntax was chosen

Four combinations were on the table: arrays written after the type (`int[]`) or before it
(`[]int`), and functions written return-first (`int(int, int)`) or parameters-first
(`(int, int)int`). Three independent checks compared them:

| | A: `int[]` + `int(int)` | B: `int[]` + `(int)int` | C: `[]int` + `int(int)` | D: `[]int` + `(int)int` |
|---|---|---|---|---|
| 21 hard nested types, read by a parser | 12 clean, 9 with two meanings | 9 clean, 4 with two meanings | 11 clean, 10 with two meanings | **21 clean, 0** |
| 3,000 random types: how many need brackets | 711 | 1,216 | 648 | **0** |
| 17 hard types written by hand: how many need brackets | 5 | 7 | 5 | **0** |

D, every modifier in front, was the only one with no double meanings and no brackets, and
it was chosen. What went wrong with the others:

- **Mixing sides needs a precedence rule.** In B, `(int)int[3]` can mean a function
  returning `int[3]` or an array of functions. TypeScript, which works like B, needs
  brackets for exactly this (`(() => number)[]`). C's mix of front and back is why C
  programmers need the "spiral rule" to read their types.
- **All-at-the-back (A) reads backwards:** the first argument you pass is written last, as
  in `int[3](int)(int)`.
- **Brackets clash with function types:** wherever A, B or C needed a grouping bracket, it
  could be read as a function type instead, silently changing the meaning.
- **Real languages built after C** that treat functions as values, like Go, Zig and Odin,
  put modifiers in front as well.

The same rule decided the other modifiers:

- **Maps are `{K}V`, in front.** A map written after its value type, `V{K}`, brings the
  double meanings back: `[3]bool{int}` could be an array of maps or a map of arrays.
  Writing a map as `[K]V`, like an array, doesn't work either: types are values, so
  `[N]int` couldn't tell an array size `N` from a key type `N`.
- **Pointers are `*int`, not `int*`.** With `int*` the pointer is the only modifier at the
  back, and `[3]int*` could mean an array of pointers or a pointer to an array. With `*`
  in front, `[3]*int` and `*[3]int` say which one.

---

## 4. Types are values too

### 4.1 Naming a type

A type is a value, and its type is `type`. So a type is named with `const`, like any
other value:

```go
const Pair: type = [2]int;
const Table: type = {int}[]int;
const Op: type = (int, int)int;

var p: Pair = [1, 2];
const add: Op = (const x: int, const y: int){ return x + y; };
const ops: [2]Op = [add, add];       // an array of Op
```

Struct types are the one exception: they have their own kind of name, `struct` (11.1).

### 4.2 Types never change

A named type is always a `const` (4.1) or a `struct` (11.1), so it can't be reassigned, and
every value declared with it has a type that's known before the program runs. A type can still be held as a value, as a generic's
parameter (4.3) or inside an `any` (4.5), but always as a `const`: there is no `var` type.

```go
var T: type = int;     // ERROR: a type can't be a var
```

### 4.3 Generics: functions that take a type

Because a type is a value, it can be passed to a function like any other value. So
generics aren't a separate feature: a generic function is just a function that takes a
type as a parameter.

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

`first(int)` gives back a function made for `int`, so what each call works with is
written right there in the code.

### 4.4 Calling a type

A type is a value, and calling it does three things.

**Calling a built-in type with a value converts the value to that type.** A built-in type
can be called with a value of any type, including an `any` (4.5):

```go
int(3.0)               // 3
float(3)               // 3.0
var whole: int = int(2.0);
int(x)                 // x is an any: gives back the int inside it
```

If the value's type is only known while the program runs, the conversion is checked
when that line runs.

**Calling a struct type makes a new value of it.** `Node(5)` runs the function after the
`=` in `Node`'s declaration and gives back the block it builds (11.1, 11.2).

**`type(value)` gives back the type of any value.** The result is a type, so it can be
stored and used like any other type:

```go
type(3)                // int
type(add)              // (int, int)int
type(scores)           // []int

const Num: type = type(3);   // Num is int
var x: Num = 5;        // x is an int
```

`type(x)` gives the type `x` was declared with, which never changes (4.2).

### 4.5 `any`: a value of any type

`any` is a built-in type for a value whose type is only known while the program runs.
Calling it wraps a value, just as calling `int` converts one:

```go
var x: any = any(3);       // holds the type int and the value 3
var y: any = any(true);    // holds the type bool and the value true

x.T                        // int: the type inside
int(x)                     // 3: checked while running that x holds an int
bool(x)                    // fails while running: x holds an int
```

Underneath, an `any` is a struct of two things: the value's type and the value, the same
kind of explanation as a fat pointer (10.5). Pictured as a struct:

```go
var x: any = any(3);
// pictured as a struct: the type, then a value of that type
var x: {type, int} = {
    const T: type = int;          // x.T
    const value: int = 3;
};

var y: any = any(true);
// a different shape inside, because the value is a bool
var y: {type, bool} = {
    const T: type = bool;
    const value: bool = true;
};
```

The second field's type depends on the first, so every `any` can have a different shape
inside. That's why its size is only known when it's made (see "Underneath, on the stack"
below), and why writing `any` itself as one struct type needs the "names inside types"
idea: `{T: type, T}` (section 13).

A function can take an `any`, so it can be called with a value of any type:

```go
const describe: (any)null = (const x: any){
    x.T                    // the type it was given
    ...
};

describe(any(3));
describe(any(true));
describe(3);               // ERROR: 3 is an int, not an any; write any(3)
```

`type(x)` gives `any`, because that's the type `x` was declared with (4.4); the type
inside is `x.T`.

#### What `any` is for

- **Data whose shape you only learn while running:** a file or a message that says "the
  next value is a bool".
- **Choosing a type while running:** for example a small number type for small data and a
  big one for big data.
- **Acting on the type:** reading `x.T` to treat an `int` and a `bool` differently (once
  `if` exists).

#### The rule that keeps it explicit

A value whose type is only known while the program runs can't be used where a fixed type
is expected until you write the conversion yourself:

```go
const double: (int)int = (const n: int){ return n * 2; };

double(x);             // ERROR: x is an any
double(int(x));        // fine: the conversion is written out, so you can see where it could fail
```

So every place where something could go wrong while running is visible in the code: each
wrap is `any(…)` and each unwrap is `int(…)`, `bool(…)` and so on. Everything that doesn't
touch an `any` is still checked before the program runs.

#### Underneath, on the stack

An `any` sits on the stack like everything else. Its size depends on the value inside, so
it's only known when the `any` is made or passed in, the same as an unsized array whose
length comes from its first value (5.3). The stack simply grows by that much at that
point; nothing is moved to the heap.

#### `any` or generics?

Both let one function work with many types (4.3):

| | generic: takes a type | `any` |
|---|---|---|
| call | you pass the type: `first(int)` | the type travels with the value: `describe(any(3))` |
| type known | before the program runs, when you write the type | only while it runs |
| mistakes show up | before the program runs | when a conversion like `int(x)` runs |
| best for | the same code for many types | values whose type you only learn while running |

### 4.6 Why

- **Why types are values:** it follows from rule 1, so types get names the same way
  everything else does, and a long type can be named once and reused.
- **Why types are named with `const`:** a type is a value of type `type`, so it needs no
  kind of name of its own, and a `const` can never change, so types are always known before
  the program runs.
- **Why converting is calling a type:** a type is a value, so it can be called like a
  function, and the code shows exactly where a value changes type: `int(3.0)`.
- **Why `any` is built in:** a function that turned any value into an `any` would need a
  parameter that already accepts any value, which is `any` itself. Built in, it's just a
  type you call, like `int`.
- **Why you write `any(3)`:** a value never turns into an `any` by itself, so the code
  shows every place a value is wrapped (`any(3)`) or unwrapped (`int(x)`).
- **Why `type(…)`:** types are values, so the type of a value can be handed back and used
  like any other value.
- **Why generics are just functions:** types are values, and functions already take
  values, so nothing new is needed. It follows from rule 1.
- **Why there's no `var` type:** if a type could change while the program runs, values
  declared with it couldn't be checked before running. `any` (4.5) covers the cases where a
  type is only known while running, and it shows them in the code.

---

## 5. Arrays

### 5.1 Fixed and unsized

```go
const primes: [3]int = [2, 3, 5];       // fixed: exactly 3 elements
var scores: []int = [10, 20, 30];       // unsized: the length comes from the first value (3)
```

### 5.2 Reading and changing elements

```go
primes[0]              // 2
scores[2]              // 30
scores[0] = 15;        // scores is now [15, 20, 30]
```

An element can be changed when the array is a `var` or a `heap` variable.

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

There are three ways, and the code says which one is happening:

```go
const sumCopy: ([]int)int = (const v: []int){ ... };       // v is a copy of the array
const sumRef: ([]int)int = (ref v: []int){ ... };          // v is the caller's array itself
const sumPtr: (*[]int)int = (const v: *[]int){ ... };      // v is a fat pointer to it (10.5)

sumCopy(scores);       // copies all of scores, like every declaration (2.4)
sumRef(scores);        // no copy: v is another name for scores
sumPtr(&scores);       // no copy: a pointer you made with &, which can also be stored
```

### 5.7 Underneath: an array is like a struct

An array is like a struct whose fields all have the same type and are reached by position
instead of by name:

```go
var a: [3]int = [10, 20, 30];     // like a struct {int, int, int}
a[1]                              // the second field, by position
```

- **A fixed array `[3]int`** is exactly that: three `int` fields one after another. Its
  length is in the type, so it isn't stored anywhere; `a.length` is just the 3 from `[3]`.
- **An unsized array `[]int`** is like a struct of its length followed by its elements. The
  type doesn't say how long it is, so the length is stored in the value, which is why
  `.length` reads like a field: for `[]int` it really is one.

Either way an array has a length you know, and it's fixed. Written out as structs, just to
picture them (you never write arrays this way, and the field names are only labels):

```go
var a: [3]int = [10, 20, 30];
// pictured as a struct: three int fields; the length 3 is fixed by the type [3]int
var a: {int, int, int} = {
    var at0: int = 10;            // a[0]
    var at1: int = 20;            // a[1]
    var at2: int = 30;            // a[2]
};

var b: []int = [7, 8];
// pictured as a struct: the length is stored, then the elements
var b: {int, [2]int} = {
    const length: int = 2;        // b.length: fixed by the first value, so const
    var elements: [2]int = [7, 8];
};
```

The length is `const` in both: it's known, and it never changes (5.3).

**Why fixed sizes:** arrays live on the stack (section 9), and a value on the stack has a
fixed size.

---

## 6. Maps

### 6.1 Making a map

The type is `{KeyType}ValueType`, and the value lists `key: value` pairs:

```go
var seen: {int}bool = {1: true, 4: false};
const rows: {int}[]int = {1: [1, 2], 2: [3, 4, 5]};
```

The values in `rows` have different lengths, which raises a layout question on the stack
that's still open (14.6).

### 6.2 Reading and changing entries

```go
seen[4]                // false
seen[4] = true;        // change an existing entry
```

An entry can be changed when the map is a `var` or a `heap` variable.

### 6.3 A map has a fixed size

A map's keys are set by its first value and never change, just like an array's length:

```go
seen[1] = false;       // fine: 1 is a key
seen[7] = true;        // ERROR: 7 isn't a key, and the map can't grow
seen.length            // 2: the number of keys
```

### 6.4 Underneath: a map is like a struct too

A map is like a struct whose fields are its keys: `{1: true, 4: false}` holds a value for key
`1` and a value for key `4`, side by side on the stack. A struct's fields are fixed by its
shape, and a map's keys are fixed by its first value, so neither can gain a new one.

Pictured as a struct (field names only as labels):

```go
var seen: {int}bool = {1: true, 4: false};
// pictured as a struct: each key is const, each value can change
var seen: {int, bool, int, bool} = {
    const key0: int = 1;   var value0: bool = true;     // seen[1]
    const key1: int = 4;   var value1: bool = false;    // seen[4]
};
```

That's why `seen[4] = true;` works (a value is `var`) but `seen[7] = true;` doesn't (there's
no field for key 7, and the keys are fixed).

**Why:** like arrays, maps live on the stack, so their size can't change.

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

### 7.3 A function that returns nothing

Its return type is `null`:

```go
const show: (int)null = (const value: int){
    ...
};
show(5);

const hello: ()null = (){      // takes nothing, returns nothing
    ...
};
hello();
```

### 7.4 Parameters are declared like names

Each parameter is `const` or `var` with a name and a type, like a declaration:

```go
const countDown: (int)int = (var n: int){
    n = n - 1;         // fine: n is a var parameter, a copy of what was passed
    return n;
};
```

`const`, `var` and `ref` on a parameter don't appear in the function's type: the type only
lists the parameter types.

### 7.5 Swapping a function in a `var`

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

The new function must have the same type.

### 7.6 Functions as values

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

### 7.7 Underneath: a function is a function pointer

A function value is just a **function pointer**: the address of the function's code. Storing
a function in a name, an array or a map stores that address, and calling it jumps there:

```go
const ops: [2](int, int)int = [add, sub];   // two function pointers
ops[1](5, 3)                                // jumps to sub's code
```

A function that uses a value from outside itself needs more than its code. In this
example the inner function uses `n`, which belongs to `adder` and is gone once `adder` ends:

```go
const adder: (int)(int)int = (const n: int){
    return (const x: int){
        return x + n;      // n comes from adder
    };
};
```

*Idea, not decided:* such a function value is a function pointer **plus a copy of the
values it uses**, kept together like a struct (`adder(5)` would hold the code and its own
copy of `5`). That follows the copy rule (2.4): the values are copied when the function is
made, so nothing is left pointing at `adder`'s ended scope. See section 13.

**Why:** parameters are declared exactly like names (section 2), the function's type
reads the same way as every other type (section 3), and functions go everywhere other
values go (section 1).

---

## 8. `ref`: another name for a variable

### 8.1 Declaring a ref

`ref` is a third kind of name. A ref is another name for an existing variable: reading it
reads that variable, and assigning to it changes that variable.

```go
var b: int = 3;
ref r: int = b;
r                      // 3
r = 5;                 // b is now 5
b = 8;                 // r is now 8 too: they are the same variable
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
the parameter in the function's value says `ref`; the function's type stays plain:

```go
const bump: (int)null = (ref x: int){
    x = x + 1;
};

var n: int = 3;
bump(n);               // n is now 4
bump(scores[1]);       // scores[1] goes up by 1
bump(3);               // ERROR: 3 isn't a variable

const swap: (int, int)null = (ref a: int, ref c: int){
    var t: int = a;
    a = c;
    c = t;
};
```

### 8.4 A ref is not a type

`ref` never appears in a type, not even in a function's parameter list. A ref can't be put
in an array or a map, and can't be returned, so these don't exist:

```go
[3]ref int             // ERROR: not a type
(int)ref int           // ERROR: not a type
(ref int)null          // ERROR: not a type; write (int)null
```

### 8.5 Underneath: a ref is a location

A ref just holds the **location** of what it refers to: the place in memory of the
variable, array element or map entry. Reading or assigning the ref goes to that place, so
it reads and changes the original:

```go
var scores: []int = [10, 20, 30];
ref top: int = scores[0];      // top holds the location of scores[0]
top = 99;                      // writes 99 at that location
```

### 8.6 Why

- **Why a kind of name, not a type:** a reference type couldn't go everywhere other types
  can (an array of references or a returned reference breaks), so it would break rule 1.
  As a kind of name, every type stays first-class.
- **Why it's safe:** a ref can't be stored or returned, so it can never outlive the place
  it refers to. That place is always a spot in memory, including an array element or a
  map entry.
- **Why `ref` stays out of types:** a ref is a kind of name, not a type, so it belongs
  with the name (`ref x: int`), not in the type.
- **The cost:** because the type doesn't show `ref`, a caller holding a function value
  (for example `ops[i](n)`) can't tell from the type whether `n` will be changed. That's
  written down as problem 14.1.
- **Why not a reference type, like C++:** C++ has reference types, and it can't put them in
  arrays or point at them, so they're second-class. Keeping `ref` out of types avoids that.

---

## 9. The stack and the heap

For now, every value lives on the stack, unless it's declared `heap`. That includes
functions: a function value on the stack is a function pointer (7.7).

### 9.1 `heap`

`heap` is the fourth kind of name. A heap variable works like a `var` (it can change),
but its value is kept on the heap instead of the stack:

```go
heap grid: [3][3]int = [[0, 0, 0], [0, 0, 0], [0, 0, 0]];
grid[1][1] = 5;
grid = [[1, 1, 1], [1, 1, 1], [1, 1, 1]];
```

### 9.2 A heap variable dies with its scope

A heap variable dies when its scope ends (the `{ }` it was declared in), exactly like a
stack variable:

```go
const work: (int)null = (const x: int){
    heap big: [3]int = [x, x, x];
    big[0] = 9;
};                     // work ends, so big and its value are gone
```

The one way out is `hand` (10.6), which passes a heap value to the caller.

### 9.3 Declaring makes a new heap value

Like every declaration (2.4), declaring a heap variable makes a new value:

```go
heap grid: [3]int = [1, 2, 3];
heap copy: [3]int = grid;      // a second heap value
copy[0] = 0;                   // grid is still [1, 2, 3]
```

### 9.4 Underneath: a heap variable is a pointer in disguise

A heap variable is just a **pointer in disguise**, and that pointer sits on the stack like
every other name. The pointer holds the address of the value on the heap, and every use of
the name goes through it automatically, so you never write `*` or `&` for it:

```go
heap big: [3]int = [1, 2, 3];
big[0] = 9;            // goes through the hidden pointer to the heap value
```

When the scope ends, the slot is removed and the heap value it points at is freed.

### 9.5 Why

- **Why a kind of name:** where a value lives is about the name, like `ref`. A heap
  `int` is still an `int`, so every function still works with it.
- **Why it can change:** a heap value that can't change has little point.
- **Why it dies with its scope:** it behaves like everything else, and nothing is handed
  from one name to another without you seeing it.

### 9.6 Other ways that were considered

- **`new(value)` giving back a pointer** (`var p: *int = new(5);`). Rejected: a value on the
  heap shouldn't have to be a pointer first.
- **`heap` as part of the type** (`var x: heap int = …`). Rejected: a `heap int` is still an
  `int`, and when a function handed one to `var y: heap int = make();` you couldn't tell
  from the code that `y` now owned something on the heap. Declaring `heap` should always
  make a new heap value.
- **`heap` on the value** (`var x: int = heap 5;`). Liked for how it reads, but the same
  problem: ownership could pass to a new name without the code showing it.
- **A heap value that can't change** (`heap const`). Dropped: a heap value that can't
  change has little point, so `heap` always behaves like `var`.
- **`heap` as a kind of name, dying with its scope** (`heap x: int = 5;`). Chosen. Getting
  heap data out of a function then needs `hand` (10.6).

---

## 10. Pointers

### 10.1 The pointer type

A pointer to a `T` has type `*T`, with `*` in front like every other modifier:

```go
*int                   // pointer to an int
[3]*int                // array of 3 pointers
*[3]int                // pointer to an array of 3 ints
(int)*int              // function returning a pointer
{int}*int              // map from int to pointers
```

### 10.2 Making and using a pointer

`&` gives a pointer to a variable. `*p` reads or changes what `p` points at:

```go
heap n: int = 5;
var p: *int = &n;      // p points at n
*p                     // 5
*p = 6;                // n is now 6
n                      // 6
```

A pointer is a value, so it can be copied, stored and passed like any other:

```go
var q: *int = p;       // a second pointer to n
*q = 7;                // n is now 7
```

### 10.3 `null`

`null` is a pointer value: a pointer of any pointer type (`*int`, `*[3]int`, …) can be
`null`, meaning it points at nothing.

```go
var p: *int = null;    // points at nothing yet
heap n: int = 5;
p = &n;                // now it points at n
p = null;              // and at nothing again
```

`null` is also how a function type says it returns nothing: `(int)null` (3.4).

### 10.4 Going through a pointer

Indexing and calling go straight through a pointer, with no `*` needed:

```go
heap nums: [3]int = [1, 2, 3];
var r: *[3]int = &nums;
r[0]                   // 1
r[0] = 10;             // nums is now [10, 2, 3]
```

Indexing and calling happen before `*`, so `*ps[0]` means "what the first pointer in
`ps` points at":

```go
heap a: int = 1;
heap b: int = 2;
var ps: [2]*int = [&a, &b];
*ps[0]                 // 1
```

### 10.5 Fat pointers: `*[]T`

A pointer to an unsized array, `*[]T`, is a **fat pointer**. It's a struct of two things:
a pointer to where the elements start, and the length.

```go
var a: [5]int = [1, 2, 3, 4, 5];
var v: *[]int = &a;    // a fat pointer to a's elements

v.length               // 5: the length part of the struct
v[0]                   // 1: indexing goes through it (10.4)
v[4] = 9;              // a is now [1, 2, 3, 4, 9]
```

Pictured as a struct (field names only as labels):

```go
var v: *[]int = &a;
// pictured as a struct: where the elements start, and how many there are
var v: {*int, int} = {
    var start: *int = &a[0];
    const length: int = 5;        // v.length
};
```

A pointer to a fixed array, `*[5]int`, doesn't need the length part, because its type
already says 5.

`str` is a fat pointer to characters:

```go
const str: type = *[]char;
```

### 10.6 `hand`: passing heap data out of a function

#### The problem `hand` solves

A heap variable dies when its scope ends (9.2). That's usually what you want, but it means
a function can't build something on the heap and give it back:

- made in a function, it dies when the function ends;
- made in a loop body, it dies at the end of that pass;
- made in a recursive call, it dies when that call ends.

So a growing structure like a linked list can't be built: every new node is gone before
anything can keep it. Something has to let heap data outlive the scope that made it, and
in MyLang that has to be written out in the code.

#### What `hand` does

`hand` is used instead of `return` to end a function. The difference is what happens to
the heap value it names: with `return` the function's heap values die as the function
ends, but with `hand` that value **doesn't die**. It's handed to whatever receives it in
the caller, and that receiver becomes its **owner**.

```go
const makeNode: (int)*Node = (const v: int){
    heap n: Node = { var value: int = v; var next: *Node = null; };
    hand &n;                       // n is handed out instead of dying
};
```

#### What can receive it

| Receiver | What happens | Owner |
|---|---|---|
| a pointer, `var p: *Node = makeNode(5);` | `p` points at the handed value, which stays on the heap | `p` |
| a `ref`, `ref b: [1000]int = makeBig(1);` | `b` becomes the name for the handed value, which stays on the heap | `b` |
| a plain `var`, `var c: [1000]int = makeBig(1);` | this works, but it just makes a copy (2.4); the handed value is freed straight away | nobody |

```go
const makeBig: (int)[1000]int = (const x: int){
    heap big: [1000]int = [...];
    hand big;                      // hands the value itself
};

ref b: [1000]int = makeBig(1);     // b owns the handed heap value: no copy
var c: [1000]int = makeBig(1);     // works, but c is a copy on the stack; the heap value goes
```

So handing a pointer or giving the value to a `ref` keeps it alive, and giving it to a
`var` keeps only a copy.

#### Ownership

- A handed value has exactly one owner, the pointer or `ref` that received it.
- **It dies with its owner:** when the owner's scope ends, the value is freed.
- **When an owner gets a new value, the value it owned before has no owner any more and is
  freed right there.** `hand` passes ownership, so nothing else is holding the old value.

#### A linked list, step by step

```go
var list: Node = { var value: int = 1; var next: *Node = null; };

list.next = makeNode(2);           // the node holding 2 is handed to list.next, which owns it
list.next.next = makeNode(3);      // the node holding 3 is owned by the node holding 2
list.next = makeNode(9);           // list.next now owns the node holding 9; the node holding 2
                                   // has no owner, so it's freed, and the node holding 3 with it
```

Each node is owned by the `next` that points at it, so the whole chain belongs to `list`.
When `list`'s scope ends, every node goes with it. This works the same in a loop or a
recursive function, because every new node is handed to something outside it.

#### Other ways that were considered

- **The value lives until you free it yourself** (like `delete` in C). Easy to forget, and
  a freed value can still be used by mistake.
- **The value lives as long as any pointer reaches it.** Turned down: the code doesn't show
  when anything is freed.
- **The value is handed to an outer scope.** Chosen: `hand` shows exactly where ownership
  moves, and the owner shows exactly when it ends.

### 10.7 Why

- **Why `hand`:** a heap value dies with its scope (9.2), so getting heap data out of a
  function needs its own word. You can see in the code exactly where ownership moves.
- **Why a pointer or a `ref` owns it but a `var` copies:** a pointer or a `ref` can stand
  for the handed value itself, while a `var` always holds its own value (2.4). Giving the
  handed value to a `var` therefore works, but only as a copy.
- **Why the old value is freed:** `hand` passes ownership, so a value whose owner takes
  something new has no owner left.
- **Why `*T`:** it goes in front, like every other modifier (section 3).
- **Why a fat pointer is a struct:** it's two values kept together, the start and the
  length, which is exactly what a struct is (section 11). Nothing about it is hidden: `*`
  shows it's a pointer, and `[]` shows the length comes with it.
- **Why `.length`:** the length is part of the value, so it's read like a field. Fixed
  arrays, unsized arrays, maps and fat pointers all read it the same way.
- **Why indexing goes straight through:** otherwise you'd need brackets: `(*r)[0]`.

---

## 11. Structs

### 11.1 Declaring a struct: `struct`

`struct` is the fifth kind of name, with the same shape as every other declaration:

```go
struct Name: Shape = (inputs){ fields };
```

- After the `:` comes the **shape**: the field types in braces, `{int, *Node}`.
- After the `=` comes a **function that makes the struct**: it takes inputs like any
  function, and its body is the block of fields with their starting values, which can use
  the inputs.
- A `struct` name can never be given a new value, so a struct type never changes.

```go
struct Node: {int, *Node} = (const v: int){
    var value: int = v;
    var next: *Node = null;
};
struct Settings: {bool, int} = (){
    const debug: bool = false;
    var size: int = 3;
};
```

### 11.2 Making a struct value: call the struct

A struct is made by calling it like a function. The call runs the function from its
declaration with the inputs given, and the block it builds becomes the new value:

```go
var n: Node = Node(5);             // value is 5, next is null
var s: Settings = Settings();      // no inputs
```

The block holds one declaration per field, in the same order as the shape, each with
`const` or `var`, a name, a type and a starting value. This mirrors functions: the type
lists only the types, and the value gives the names.

| | Type (after `:`) | Value |
|---|---|---|
| function | `(int, int)int` | `(const x: int, const y: int){ … }` |
| struct | `{int, *Node}` | the block `Node(…)` builds: `{ var value: int = …; var next: *Node = null; }` |

### 11.3 Fields

`.` reaches a field. A `var` field can change and a `const` field can't:

```go
n.value = 5;           // fine: value is var
```

### 11.4 Singletons

A singleton is just a value whose struct type has no name:

```go
var settings: {bool, int} = {
    const debug: bool = false;
    var size: int = 3;
};

settings.size = 5;          // fine: size is var
settings.debug = true;      // ERROR: debug is const
```

### 11.5 Functions inside a struct

A function declared inside a struct is in the struct's scope, so it uses the struct's
fields directly. There is no `this`:

```go
var counter: {int, (int)null} = {
    var count: int = 0;
    const add: (int)null = (const n: int){
        count = count + n;     // count is the struct's own field
    };
};

counter.add(5);                // counter.count is now 5
```

### 11.6 Underneath: the variable is its block

Writing a value on its own puts it on the stack and removes it again: `4;` adds 4 to the
stack, and it's gone once the statement is done. A struct block works the same way.
Writing

```go
{ var value: int = 0; var next: *Node = null; };
```

lays its fields out on the stack one after another, and they're removed again. Saving the
block in a variable keeps it: the variable's own place on the stack holds the whole block,
just as an `int` variable holds its number. A field is found from the start of that block
and its position in the shape (`{int, *Node}`: `value` first, `next` second).

A struct doesn't work like a pointer. Every declaration and every assignment copies the
whole block:

```go
var n: Node = Node(0);
var m: Node = n;       // a full copy: m gets its own block on the stack
m.value = 5;           // n.value is still 0
```

A `const` field can't be changed on its own, but a `var` struct variable can be given a
whole new value, which copies the new block into its place:

```go
var node: {int, *Node} = { const value: int = 0; var next: *Node = null; };
node.value = 1;        // ERROR: value is const
node = { const value: int = 7; var next: *Node = null; };   // fine: the new block is copied in
```

A heap variable (9.4) is the one that holds a hidden pointer; a struct variable holds its
block directly.

### 11.7 Why

- **Why `struct` is its own kind of name:** a struct type comes with the function that
  makes it, and `struct Name: Shape = (inputs){ fields };` gives that function a place after
  the `=`. Other types don't need one, so they're named with `const` (4.1).
- **Why making a struct is a call:** the part after the `=` is a function, so making a
  struct is calling it, the same as calling any function. Its inputs are written right there
  in the call (`Node(5)`), and nothing is filled in behind your back.
- **Why the type lists only the field types:** it works like a function's type, which
  lists only the parameter types; the names belong in the value.
- **Why singletons need nothing new:** a struct value can be declared directly with `var`
  or `const`, like any other value.
- **Why no `this`:** a function inside a struct is already in the struct's scope, so the
  fields are just names it can see.
- **Why a struct is copied, not pointed at:** every other value is copied when it's declared
  or assigned (2.4), and structs behave the same. Because the variable holds its own block,
  it can never be left pointing at a block that has been removed.

### 11.8 Other ways that were considered

- **A `type` kind for every named type** (`type Pair: [2]int = [0, 0];`). Replaced: only
  struct types need a function that makes them, so they get `struct`, and every other type is a `const` of
  type `type` (4.1).
- **A struct as a `const` type value** (`const Node: type = {int, *Node};`). Replaced by
  `struct`, which gives the struct its making function after the `=`.
- **A plain default block after the `=`**, copied by `Node()` with no inputs. Replaced by
  the making function, which can take inputs.
- **Two blocks, inputs then the rest** (`const Node: type = {var value: int}{ var doubled:
  int = 2 * value; };`). Replaced by the function form, which reuses how functions already
  take inputs.
- **`var` types** (`var T: type`). Dropped: a type can never change (4.2).
- **Field names in the struct type** (`{value: int, next: *Node}`). Not decided yet: it's
  the "names inside types" question in section 13.
- **A struct variable as a fat pointer** to its block on the stack (where the block starts
  and ends). Dropped: with every declaration copying the whole block, the variable can
  simply hold the block, and nothing can be left pointing at a removed block.
- **A `this` for functions inside a struct.** Not needed: they're in the struct's scope.

---

## 12. Putting it together

```go
// name long types once
const Op: type = (int, int)int;
const Row: type = []int;

// functions are values
const add: Op = (const x: int, const y: int){ return x + y; };
const mul: Op = (const x: int, const y: int){ return x * y; };

// a map of functions, an array of arrays
const ops: {int}Op = {1: add, 2: mul};
var table: [2]Row = [[1, 2], [3, 4]];

// a function that takes a function and a ref
const applyTo: (Op, int, int)null = (const f: Op, ref target: int, const by: int){
    target = f(target, by);
};

applyTo(ops[1], table[0][1], 10);   // table[0][1] is now 12
applyTo(ops[2], table[1][0], 2);    // table[1][0] is now 6

// a big value on the heap, reached through a pointer
heap big: [3]int = [0, 0, 0];
var pb: *[3]int = &big;
pb[2] = table[0][1];                // big is now [0, 0, 12]
```

---

## 13. Not decided yet

These are left out on purpose until they're decided:

- **Basic types:** which ones there are and their sizes (`char` included), and what type a
  plain number like `3` or `3.0` has.
- **Conversions:** which types can be converted to which, how `int(3.7)` rounds, and what
  happens when a value doesn't fit (`int8(300)`).
- **Generic function types:** in `(type)([]T)T` the return type uses `T`, the type passed
  in, without naming it first. How a function's type refers to its own parameter ties in
  with names inside types (above).
- **Generics and returned functions:** `first` returns a function that uses `T`, one of
  `first`'s parameters, which is the open question about returned functions and local
  values (below).
- **Pointing at part of an array:** a fat pointer to only some elements, e.g. `&a[1:3]`.
- **Names inside types:** whether struct and function types include the names, e.g.
  `{value: int, next: *Node}` and `(x: int, y: int)int`, or only the types, as now
  (`{int, *Node}`, `(int, int)int`).
  - *Advantage of names:* a later part of a type can refer to an earlier one by name. That
    is exactly what generics and `any` need, so they come with no extra rule:
    ```go
    const show: (T: type, v: T)null = (const T: type, const v: T){ ... };
    ```
    Here the type of `v` is "whatever was passed as `T`", which a type without names can't
    say. The same works for a struct holding a type and a value: `{T: type, value: T}`.
  - *Advantage of names:* field names are part of the type, so `n.value` can be checked from
    the type alone. Without them, two values of type `Node` could name their fields
    differently.
  - *Cost:* every type gets longer, e.g. `(x: int, y: int)int` instead of `(int, int)int`.
  - *Middle option:* names only where a later part of the type needs them, e.g.
    `(T: type, T)null` and `{T: type, T}`, with every other type left without names.
- **`{…}` as a map or a struct:** `{int}bool` is a map and `{int}` would be a struct with one
  field. A possible rule: a `{…}` followed by a type is a map, otherwise a struct.
- **Control flow:** `if`, loops and the like.
- **Refs and `const`:** whether a ref can refer to a `const`.
- **`const` pointers:** whether `*p = …` is allowed when `p` is a `const`.
- **`hand` in function types:** with first-class functions the caller only sees the type,
  so the type may need to show that a function hands, e.g. `(int)hand *Node`.
- **Copies of an owning pointer:** `var q: *Node = list.next;` makes a pointer that doesn't
  own the node; if the owner goes first, `q` points at nothing (see 14.2).
- **Using a `null` pointer:** what `*p` does when `p` is `null`.
- **Pointer lifetimes:** what `&` may point at, and what happens to a pointer whose
  target's scope has ended.
- **Returned functions and local values:** whether a function returned from another
  function can use that function's local values after it has ended. One idea (7.7): the
  function value carries a copy of the values it uses.
- **A ref into heap data** that a pointer stops reaching.
- **Reassigning a whole map:** whether the new value needs the same keys.

---

## 14. Problems to look at later

Problems the current rules run into. Each shows an example that the rules allow today and
some possible fixes. None of the fixes is decided.

### 14.1 Refs don't show in types, so first-class functions can't be checked

```go
const bump: (int)null = (ref x: int){ x = x + 1; };
const show: (int)null = (const x: int){ ... };
const fs: [2](int)null = [bump, show];     // same type, so allowed
fs[i](n);       // does n change? it depends on i, which is only known when the program runs
fs[i](3);       // fine for show, an ERROR for bump, and nothing can tell which before running
```

Possible fixes: mark ref arguments at the call (`bump(ref n)`), or don't allow a function
with ref parameters to be stored in an array or a map.

### 14.2 A pointer can outlive what it points at

```go
heap a: int = 1;
var p: *int = &a;
{
    heap n: int = 5;
    p = &n;
}               // n dies here
*p = 6;         // p points at nothing
```

Pointers are first-class, so they can be stored, returned and copied anywhere. Possible
fixes: a pointer can't be stored anywhere that lives longer than what it points at (needs a
checker, like Rust), or the target stays alive while a pointer reaches it (turned down
earlier).

### 14.3 Heap data leaving the function that made it (settled)

Settled by `hand` (10.6): a function hands its heap value to a pointer or a `ref` in the
caller, which becomes the owner.

### 14.4 Values whose type is a `var` (settled)

Settled (4.1, 4.2): named types are `const` or `struct`, so a type can never change, so there is no `var` type and every
value's type is known before the program runs.

### 14.5 `(Op)` has two meanings when the value is a type

```go
const Op: type = (int, int)int;
const T: type = (Op);        // a function that takes an Op, or just Op in brackets?
```

Named types are ordinary names, so brackets around one are ambiguous. Since every function
type now ends with its return type (`(Op)null`, `(Op)int`), `(Op)` on its own can't be a
function type, which mostly settles this. Possible fix for the rest: in a type, `( … )`
always starts a function and is never used for grouping.

### 14.6 Arrays or maps of unsized arrays

```go
const rows: {int}[]int = {1: [1, 2], 2: [3, 4, 5]};   // values of different lengths
```

Values of different sizes can't sit side by side on the stack, so finding `rows[k]` isn't a
simple position. Possible fixes: unsized arrays inside an array or a map must all have the
same length, or only fixed-size `[n]T` is allowed there.

### 14.7 Which types can be map keys

`{(int)int}bool` or `{[]int}bool`: when are two functions, or two arrays, the same key?
Possible fix: only types that can be compared can be keys (to decide together with the
basic types).

### 14.8 A ref into a map entry, then the whole map is reassigned

```go
var seen: {int}bool = {1: true, 4: false};
ref four: bool = seen[4];
seen = {1: true, 2: false};    // if allowed, four's entry is gone
```

Depends on how reassigning a whole map works (section 13).

### 14.9 A ref to a `const`

```go
const limit: int = 10;
bump(limit);    // would change a constant
```

Because of 14.1 this can't always be caught before the program runs.

### 14.10 A function made inside another function that uses its local values

The returned-function case from section 13, but it also happens when the inner function is
stored somewhere that lives longer, such as a heap array, without being returned.

### 14.11 Array sizes from names

```go
const N: int = 3;
var a: [N]int = [1, 2, 3];     // allowed?
```

### 14.12 Structs (notes from 2024, not decided)

The decided struct rules are in section 11. These are the earlier notes, kept for the parts
still open.

**A struct is a value of type `type`**, declared like everything else (rule 1):

```go
const LinkedList: type = {
    const Node: type = {
        var x: int = 3;
        var next: *Node = null;
    }
    var head: Item = Item();

    const append: (int) = (const value: int){
        const new_node: *Node = new(Node(value))
        if (head == null) {
            head = new_node;
        } else {
            var current_node: *Node = head;
            while (current_node != null) {
                current_node = current_node.next
            } else {
                &current_node.next = new_node;
            }
        }
    }

    const pop: (int) = (const index: int){
        var current_node: *Node = head;
        while (current_node.next.next != null) {
            current_node = current_node.next;
        } else {
            delete(current_node.next);
            &current_node.next = null;
        }
    }
}
```

What the sketch shows:

- **Fields** are declared with `var` or `const`, each with a starting value (`var x: int = 3;`).
- **Types can be nested:** `Node` is declared inside `LinkedList`.
- **Methods** are just function values inside the struct (`append`, `pop`).
- **Making one** is calling the type: `Node(value)`, `Item()`.
- **Fields** are reached with `.`: `current_node.next`.
- `null` is used for "no next node" (`var next: *Node = null;`).

Later notes (May 2024):

- "&foo.leafs gives pointer to leafs and if i have pointer to foo if i get pointer_foo.leafs
  is give pointer": `&` of a field gives a pointer to that field, and reading a field through
  a pointer also gives a pointer.
- "const declared during run time are added to stack, so having var and const inside of a
  struct works"
- "look into struct constructor though"

A Go-style version was tried and dropped (29–31 Mar 2024) in favour of the form above:

```go
type Foo struct { x int; y int }
var f Foo = Foo{x: 1, y: 2}
var fp *Foo = &f
fp.x            // gives a pointer to x
```

**Clashes with today's rules:**

- `new(...)` and `delete(...)` aren't in the rules.
- `&current_node.next = new_node;` puts `&` on the left of an assignment, which the pointer
  rules don't cover.
- `var head: Item = Item();`: `Item` is never defined (probably meant `*Node = null`).
- A linked list needs nodes that outlive the function that made them: now possible with
  `hand` (10.6). 14.2 is still open for copies of a pointer.
- `while … else` and `if` are control flow, which isn't decided yet.
