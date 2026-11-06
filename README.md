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
- The basic types aren't decided yet, so the examples use `int` and `bool` as stand-ins,
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
| `kind` | what sort of name it is: `const`, `var`, `ref` or `heap` |
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

### 2.2 The four kinds of name

| Kind | Can it be given a new value? | Where its value lives | Section |
|---|---|---|---|
| `const` | no | stack | 2.3 |
| `var` | yes | stack | 2.3 |
| `ref` | it *is* another variable: assigning changes that variable | wherever that variable is | 8 |
| `heap` | yes | heap | 9 |

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

---

## 4. Types are values too

### 4.1 Declaring a type

A type is a value of type `type`, so it's declared like anything else and then used
wherever a type goes:

```go
const Pair: type = [2]int;
const Op: type = (int, int)int;
const Table: type = {int}[]int;

var p: Pair = [1, 2];
const add: Op = (const x: int, const y: int){ return x + y; };
const ops: [2]Op = [add, add];       // an array of Op
```

### 4.2 A type in a `var` can change

Like any `var`, a type in a `var` can be given a new value. A value keeps the type it was
declared with; only what you declare afterwards gets the new type:

```go
var T: type = int;
var x: T = 3;          // x is an int
T = int8;
var y: T = 3;          // y is an int8
x                      // still an int
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
```

`first(int)` gives back a function made for `int`, so what each call works with is
written right there in the code.

### 4.4 Why

- **Why types are values:** it follows from rule 1, so types get names the same way
  everything else does, and a long type can be named once and reused.
- **Why a `var` type is allowed:** a type is a value like any other, so it can be a `var`
  like any other.
- **Why generics are just functions:** types are values, and functions already take
  values, so nothing new is needed. It follows from rule 1.
- **Why a value keeps its type:** a value's type is fixed when it's declared, just as
  declaring makes a new value (2.4). Changing `T` later doesn't reach back and change `x`.

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
const first: ([]int)int = (const values: []int){
    return values[0];
};
first([7, 8])          // 7
first([1, 2, 3, 4])    // 1
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

### 8.5 Why

- **Why a kind of name, not a type:** a reference type couldn't go everywhere other types
  can (an array of references or a returned reference breaks), so it would break rule 1.
  As a kind of name, every type stays first-class.
- **Why it's safe:** a ref can't be stored or returned, so it can never outlive the place
  it refers to. That place is always a spot in memory, including an array element or a
  map entry.
- **Why `ref` stays out of types:** a ref is a kind of name, not a type, so it belongs
  with the name (`ref x: int`), not in the type.

---

## 9. The stack and the heap

For now, every value lives on the stack, unless it's declared `heap`.

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

### 9.3 Declaring makes a new heap value

Like every declaration (2.4), declaring a heap variable makes a new value:

```go
heap grid: [3]int = [1, 2, 3];
heap copy: [3]int = grid;      // a second heap value
copy[0] = 0;                   // grid is still [1, 2, 3]
```

### 9.4 Underneath: a heap variable is an unofficial pointer

A heap variable's own slot is on the stack, like every name. What that slot holds is the
address of the value on the heap, and every use of the name goes through it automatically,
so you never write `*` or `&` for it:

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

A pointer to a fixed array, `*[5]int`, doesn't need the length part, because its type
already says 5.

`str` is a fat pointer to characters:

```go
const str: type = *[]char;
```

### 10.6 Why

- **Why `*T`:** it goes in front, like every other modifier (section 3).
- **Why a fat pointer is a struct:** it's two values kept together, the start and the
  length, which is exactly what a struct is (section 11). Nothing about it is hidden: `*`
  shows it's a pointer, and `[]` shows the length comes with it.
- **Why `.length`:** the length is part of the value, so it's read like a field. Fixed
  arrays, unsized arrays, maps and fat pointers all read it the same way.
- **Why indexing goes straight through:** otherwise you'd need brackets: `(*r)[0]`.

---

## 11. Structs

### 11.1 A struct type is its shape

A struct's type lists its field types in braces, `{int, *Node}`. It's a type like any
other, so it's named the same way (section 4):

```go
const Node: type = {int, *Node};
const Settings: type = {bool, int};
```

### 11.2 A struct value is a block of declarations

A struct value is a block of declarations, one per field and in the same order as the
type, each with `const` or `var`, a name, a type and a starting value:

```go
var n: Node = {
    var value: int = 0;
    var next: *Node = null;
};
```

This mirrors functions: the type lists only the types, and the value gives the names.

| | Type (after `:`) | Value (after `=`) |
|---|---|---|
| function | `(int, int)int` | `(const x: int, const y: int){ … }` |
| struct | `{int, *Node}` | `{ var value: int = 0; var next: *Node = null; }` |

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
var n: Node = { var value: int = 0; var next: *Node = null; };
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

- **Why a struct type is declared like any type:** types are values (section 4), so a
  struct type needs no special keyword.
- **Why the type lists only the field types:** it works like a function's type, which
  lists only the parameter types; the names belong in the value.
- **Why singletons need nothing new:** a struct value can be declared directly with `var`
  or `const`, like any other value.
- **Why no `this`:** a function inside a struct is already in the struct's scope, so the
  fields are just names it can see.
- **Why a struct is copied, not pointed at:** every other value is copied when it's declared
  or assigned (2.4), and structs behave the same. Because the variable holds its own block,
  it can never be left pointing at a block that has been removed.

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
const applyTo: (Op, int, int) = (const f: Op, ref target: int, const by: int){
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

- **Basic types:** which ones there are and their sizes (`char` included); converting between types.
- **Generic function types:** in `(type)([]T)T` the return type uses `T`, the type passed
  in, without naming it first. How a function's type refers to its own parameter ties in
  with names inside types (above).
- **Generics and returned functions:** `first` returns a function that uses `T`, one of
  `first`'s parameters, which is the open question about returned functions and local
  values (below).
- **Pointing at part of an array:** a fat pointer to only some elements, e.g. `&a[1:3]`.
- **Names inside types:** whether struct and function types include the names, e.g.
  `{value: int, next: *Node}` and `(x: int, y: int)int`, or only the types, as now
  (`{int, *Node}`, `(int, int)int`). Without names in the type, two values of type `Node`
  can name their fields differently, so `n.value` can't be checked from the type alone.
- **`{…}` as a map or a struct:** `{int}bool` is a map and `{int}` would be a struct with one
  field. A possible rule: a `{…}` followed by a type is a map, otherwise a struct.
- **Nodes that outlive the function that made them** (for linked lists; see 14.2, 14.3).
- **Control flow:** `if`, loops and the like.
- **Refs and `const`:** whether a ref can refer to a `const`.
- **`const` pointers:** whether `*p = …` is allowed when `p` is a `const`.
- **Heap data outliving a function:** a heap variable dies with its scope, so returning
  `&` of one would point at nothing. How a function hands heap data back is open.
- **Using a `null` pointer:** what `*p` does when `p` is `null`.
- **Pointer lifetimes:** what `&` may point at, and what happens to a pointer whose
  target's scope has ended.
- **Returned functions and local values:** whether a function returned from another
  function can use that function's local values after it has ended.
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

### 14.3 Heap data can never leave the function that made it

A heap variable dies with its scope, and a pointer to it would point at nothing once the
function ends (14.2). So a function can only hand back a copy: a big array is copied in
full, and linked structures like lists and trees can't be built.

### 14.4 Checking values whose type is a `var`

Decided (4.2): a type in a `var` can change, and each value keeps the type it was declared
with. Still open: if `T` changes inside something like an `if`, the type of a later
declaration `var y: T = …` can't be known before the program runs, so that check has to
happen while it runs.

### 14.5 `(Op)` has two meanings when the value is a type

```go
const Op: type = (int, int)int;
const T: type = (Op);    // a function that takes an Op, or just Op in brackets?
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
- A linked list needs nodes that outlive the function that made them, which 14.2 and 14.3
  don't allow yet.
- `while … else` and `if` are control flow, which isn't decided yet.
