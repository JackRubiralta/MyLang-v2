# MyLang rules

The rules decided so far: declaring with `const`, `var`, `ref` and `heap`, first-class
values, the types for arrays, maps, functions and pointers, and the heap. Structs come later.
(Code blocks are marked `go` only for colouring; the code is MyLang.)

## 1. Everything is a first-class value

Numbers, arrays, maps, functions and types are all values. Any value can be:

- stored in a `const` or a `var`
- put inside an array or a map
- passed to a function
- returned from a function

Values live on the stack, unless they are declared `heap` or made with `new` (section 10).

## 2. Declaring

Every declaration has the same shape, whatever the value is:

```go
const name: Type = value;
var   name: Type = value;
```

- `const`: the name can't be assigned again.
- `var`: the name can be assigned again: `name = value;`
- `ref`: another name for an existing variable (section 9).
- `heap`: a variable whose value is kept on the heap (section 10).
- Every statement ends with `;`

```go
var count: int = 3;
count = 4;                 // fine: var
const limit: int = 10;
limit = 11;                // ERROR: limit is const
```

## 3. Basic types

```go
int  uint  float  char  bool  str
int8  int16  int32  int64  int128  int256
```

## 4. Types: every modifier goes in front

| Kind | Form | Example | Read as |
|---|---|---|---|
| fixed array | `[n]T` | `[3]int` | array of 3 ints |
| unsized array | `[]T` | `[]int` | array of ints |
| map | `{K}V` | `{int}bool` | map from int to bool |
| function | `(P1, P2)R` | `(int, int)int` | takes two ints, returns an int |
| function, no return value | `(P1, P2)` | `(int)` | takes an int, returns nothing |
| pointer | `*T` | `*int` | pointer to an int |

Because every modifier goes in front, a type reads left to right, the modifiers stack in any
combination, and a type never needs brackets:

```go
[3](int)int          // array of 3 functions, each takes an int and returns an int
(int)[]int           // function that takes an int and returns an array of ints
(int)(int)int        // function that takes an int and returns a function (int)int
{int}(int)int        // map from int to functions (int)int
[3]{int}bool         // array of 3 maps
{int}[]int           // map from int to arrays of ints
[2][3]int            // array of 2 arrays of 3 ints
[3]*int              // array of 3 pointers
*[3]int              // pointer to an array of 3 ints
(int)*int            // function that takes an int and returns a pointer
```

The grammar:

```
Type      = BasicType
          | "[" [ NUMBER ] "]" Type           (array)
          | "{" Type "}" Type                 (map: {key}value)
          | "*" Type                          (pointer)
          | "(" [ ParamList ] ")" [ Type ]    (function: parameters, then the return type)
ParamList = Param { "," Param }
Param     = [ "ref" ] Type                    (ref only here, see section 9)
```

## 5. Types are values too

A type is a value of type `type`, so it is declared like anything else:

```go
const str: type = []char;
const Pair: type = [2]int;
const Op: type = (int, int)int;

var p: Pair = [1, 2];
const add: Op = (const x: int, const y: int){
    return x + y;
};
```

## 6. Arrays

```go
const primes: [3]int = [2, 3, 5];       // fixed size
var scores: []int = [10, 20, 30];       // unsized: the length comes from the first value
const grid: [2][3]int = [[1, 2, 3], [4, 5, 6]];

primes[0]          // 2
grid[1][2]         // 6
```

The length of an unsized array is set by its first value and never changes:

```go
var a: []int = [1, 2, 3];
a = [1, 2, 4, 5];              // ERROR: a has length 3
var b: []int = [1, 2, 3, 4];   // fine: a new declaration
```

So a function can take or return an unsized array of any length.

## 7. Maps

```go
var seen: {int}bool = {1: true, 4: false};

seen[4]            // false
seen[4] = true;    // fine: 4 is already a key
seen[7] = true;    // ERROR: 7 isn't a key, and the map can't grow
```

A map has a fixed size, like an array, and lives on the stack. Its size is set by its
first value and never changes.

## 8. Functions

The type gives the parameter types, then the return type. The value gives the parameters,
each declared like a variable, then the body:

```go
const add: (int, int)int = (const x: int, const y: int){
    return x + y;
};

add(1, 2)          // 3
```

A function that returns nothing has no return type:

```go
const show: (int) = (const value: int){
    ...
};
```

A function in a `var` can be swapped for another function of the same type:

```go
var foo: (int, int)int = (const x: int, const y: int){
    return x + y;
};
foo = (const x: int, const y: int){
    return x * y;
};
```

## 9. References: `ref`

`ref` is a third way to declare a name, next to `const` and `var`. A ref is another name
for an existing variable: reading it reads that variable, and assigning to it changes that
variable.

```go
var b: int = 3;
ref r: int = b;
r = 5;             // b is now 5
```

- A ref refers to a variable, an array element or a map entry, never to a value like `3`.
- A ref is **not a type**. You can't put one in an array or a map, or return one, so
  `[3]ref int` and `(int)ref int` don't exist. Every type stays first-class.
- Because a ref can't be stored or returned, it can never outlive what it refers to.

### `ref` parameters

A function can take a parameter as a `ref`, so it can change the caller's variable. The
function's type shows it, because the caller has to pass a variable, not a copy:

```go
const bump: (ref int) = (ref x: int){
    x = x + 1;
};

var n: int = 3;
bump(n);           // n is now 4
bump(3);           // ERROR: 3 isn't a variable
```

`ref` is allowed inside a function type's parameter list, and nowhere else in a type.
(`const` and `var` on a parameter don't show in the type: they only affect the function's
own copy.)

## 10. The heap and pointers

### `heap`: a variable kept on the heap

`heap` is a fourth way to declare a name. A heap variable can change, like a `var`, but
its value is kept on the heap instead of the stack, for values too big for the stack.

```go
heap grid: [3][3]int = [[0, 0, 0], [0, 0, 0], [0, 0, 0]];
grid[1][1] = 5;
```

- A heap variable dies when its scope ends (the `{ }` it was declared in), just like a
  stack variable, and its value is freed then.
- Declaring always makes a new value: `heap copy: [3][3]int = grid;` is a second heap value.

### Pointers: `*T`

`new(value)` puts a value on the heap and gives back a pointer to it. That value isn't tied
to any scope: it lives until `delete` frees it.

```go
var p: *int = new(5);
*p                 // 5: the value p points at
*p = 6;            // change it
delete(p);         // free it
```

A pointer is an ordinary first-class value, so it can go in arrays and maps and be passed
and returned. To make heap data outlive a function, the function returns a pointer:

```go
const make: (int)*int = (const x: int){
    return new(x);
};

var q: *int = make(5);     // the 5 is still alive after make ends
delete(q);
```

## 11. First-class examples

Functions, arrays and maps go inside each other, into functions and out of them:

```go
const sub: (int, int)int = (const x: int, const y: int){
    return x - y;
};

// an array of functions
const ops: [2](int, int)int = [add, sub];
ops[1](5, 3)                   // 2

// a map of functions
const byKey: {int}(int, int)int = {1: add, 2: sub};
byKey[1](2, 2)                 // 4

// a function that takes a function
const twice: ((int)int, int)int = (const f: (int)int, const x: int){
    return f(f(x));
};

// a function that returns a function
const adder: (int)(int)int = (const n: int){
    return (const x: int){
        return x + n;
    };
};
adder(5)(1)                    // 6

// a function that returns an array
const three: (int)[3]int = (const x: int){
    return [x, x, x];
};
three(7)[0]                    // 7

// a function that takes an array
const first: ([]int)int = (const values: []int){
    return values[0];
};

// a map of arrays, and an array of maps
const rows: {int}[]int = {1: [1, 2], 2: [3, 4, 5]};
const flags: [2]{int}bool = [{1: true}, {2: false}];
```
