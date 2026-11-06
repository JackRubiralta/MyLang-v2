# MyLang

A small language where **everything is a first-class value**: numbers, arrays, maps,
functions and even types are all declared, stored, passed and returned the same way.

This file holds the rules decided so far, each with the reason behind it. Anything not
decided yet is left out on purpose and listed at the end.

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
11. [Putting it together](#11-putting-it-together)
12. [Not decided yet](#12-not-decided-yet)
13. [Problems to look at later](#13-problems-to-look-at-later)

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
| function, no return value | `(P1, P2)` | `(int)` | takes an int, returns nothing |
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
          | "*" Type                          (pointer)
          | "(" [ TypeList ] ")" [ Type ]     (function: parameter types, then the return type)
TypeList  = Type { "," Type }
```

After a function's parameter list, whatever type follows is its return type. If no type
follows, it returns nothing.

### 3.5 Why

- **Why in front:** the four options (`int[]` or `[]int`, `int(int)` or `(int)int`) were
  compared on hard nested types and on 3,000 random ones. Only the all-in-front form gave
  every type exactly one meaning without brackets. Mixing front and back (like `int[]`
  with `(int)int`) makes types such as `(int)int[3]` mean two different things.
- **Why no brackets at all:** `(int)` already means "a function that takes an int", so a
  bracket used for grouping would silently change the meaning.
- **Why it reads well:** left to right is plain English, and it's the order you use the
  value in (3.3).
- **Why no `fn` keyword:** it isn't needed, so types stay short and clean.

---

## 4. Types are values too

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

**Why:** it follows from rule 1, so types get names the same way everything else does.
A long type can be named once and reused.

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

Leave out the return type:

```go
const show: (int) = (const value: int){
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
const bump: (int) = (ref x: int){
    x = x + 1;
};

var n: int = 3;
bump(n);               // n is now 4
bump(scores[1]);       // scores[1] goes up by 1
bump(3);               // ERROR: 3 isn't a variable

const swap: (int, int) = (ref a: int, ref c: int){
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
(ref int)              // ERROR: not a type; write (int)
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
const work: (int) = (const x: int){
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

### 9.4 Why

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

A pointer can be `null`: it points at nothing.

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

### 10.5 Why

- **Why `*T`:** it goes in front, like every other modifier (section 3).
- **Why indexing goes straight through:** otherwise you'd need brackets: `(*r)[0]`.

---

## 11. Putting it together

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

## 12. Not decided yet

These are left out on purpose until they're decided:

- **Basic types:** which ones there are and their sizes; strings; converting between types.
- **Structs.**
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

## 13. Problems to look at later

Problems the current rules run into. Each shows an example that the rules allow today and
some possible fixes. None of the fixes is decided.

### 13.1 Refs don't show in types, so first-class functions can't be checked

```go
const bump: (int) = (ref x: int){ x = x + 1; };
const show: (int) = (const x: int){ ... };
const fs: [2](int) = [bump, show];     // same type, so allowed
fs[i](n);       // does n change? it depends on i, which is only known when the program runs
fs[i](3);       // fine for show, an ERROR for bump, and nothing can tell which before running
```

Possible fixes: mark ref arguments at the call (`bump(ref n)`), or don't allow a function
with ref parameters to be stored in an array or a map.

### 13.2 A pointer can outlive what it points at

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

### 13.3 Heap data can never leave the function that made it

A heap variable dies with its scope, and a pointer to it would point at nothing once the
function ends (13.2). So a function can only hand back a copy: a big array is copied in
full, and linked structures like lists and trees can't be built.

### 13.4 A type stored in a `var` can change while the program runs

```go
var T: type = int;
var x: T = 3;
T = bool;       // what type is x now?
```

Types have to be known before the program runs. Possible fix: a type can only be a `const`.

### 13.5 `(Op)` has two meanings when the value is a type

```go
const Op: type = (int, int)int;
const T: type = (Op);    // a function that takes an Op, or just Op in brackets?
```

Named types are ordinary names, so brackets around one are ambiguous. Possible fix: in a
type, `( … )` always means a function and is never used for grouping.

### 13.6 Arrays or maps of unsized arrays

```go
const rows: {int}[]int = {1: [1, 2], 2: [3, 4, 5]};   // values of different lengths
```

Values of different sizes can't sit side by side on the stack, so finding `rows[k]` isn't a
simple position. Possible fixes: unsized arrays inside an array or a map must all have the
same length, or only fixed-size `[n]T` is allowed there.

### 13.7 Which types can be map keys

`{(int)int}bool` or `{[]int}bool`: when are two functions, or two arrays, the same key?
Possible fix: only types that can be compared can be keys (to decide together with the
basic types).

### 13.8 A ref into a map entry, then the whole map is reassigned

```go
var seen: {int}bool = {1: true, 4: false};
ref four: bool = seen[4];
seen = {1: true, 2: false};    // if allowed, four's entry is gone
```

Depends on how reassigning a whole map works (section 12).

### 13.9 A ref to a `const`

```go
const limit: int = 10;
bump(limit);    // would change a constant
```

Because of 13.1 this can't always be caught before the program runs.

### 13.10 A function made inside another function that uses its local values

The returned-function case from section 12, but it also happens when the inner function is
stored somewhere that lives longer, such as a heap array, without being returned.

### 13.11 Array sizes from names

```go
const N: int = 3;
var a: [N]int = [1, 2, 3];     // allowed?
```
