# MyLang

The rules decided so far, each with the reason behind it. Anything not decided yet is
left out (see the list at the end).

Code blocks are marked `go` only for colouring; the code is MyLang. The examples use `int`
and `bool` as stand-ins, because the basic types aren't decided yet.

## 1. Everything is a first-class value

Functions, arrays, maps and types are all values, just like numbers. Any value can be
stored in a name, put inside an array or a map, passed to a function, and returned from one.

**Why:** with no special cases, everything combines with everything else. An array of
functions or a function that returns a map needs no extra rules.

## 2. Declaring a name

Every declaration has the same shape, whatever the value is:

```go
const name: Type = value;
var   name: Type = value;
```

- `const`: the name can't be given a new value.
- `var`: the name can be given a new value: `name = value;`

Two more kinds of name, `ref` and `heap`, come in sections 8 and 9.

**Why:** since functions, arrays, maps and types are all values, they are all declared the
same way. A function is declared exactly like a number.

## 3. Types: every modifier goes in front

| Kind | Form | Example | Read as |
|---|---|---|---|
| fixed array | `[n]T` | `[3]int` | array of 3 ints |
| unsized array | `[]T` | `[]int` | array of ints |
| map | `{K}V` | `{int}bool` | map from int to bool |
| function | `(P1, P2)R` | `(int, int)int` | takes two ints, returns an int |
| function, no return value | `(P1, P2)` | `(int)` | takes an int, returns nothing |
| pointer | `*T` | `*int` | pointer to an int |

The modifiers stack in any combination:

```go
[3](int)int          // array of 3 functions, each takes an int and returns an int
(int)[]int           // function that takes an int and returns an array of ints
(int)(int)int        // function that takes an int and returns a function (int)int
{int}(int)int        // map from int to functions
[3]{int}bool         // array of 3 maps
[2][3]int            // array of 2 arrays of 3 ints
[3]*int              // array of 3 pointers
*[3]int              // pointer to an array of 3 ints
```

**Why in front:** the four options (`int[]` or `[]int`, `int(int)` or `(int)int`) were
compared on hard nested types and on 3,000 random ones. Only the all-in-front form gave
every type exactly one meaning without brackets, and brackets aren't possible here anyway,
because `(int)` already means a function. A type also reads left to right in plain English,
and in the order you use the value: `(int)(int)[3]int` is used as `f(1)(2)[0]`.

**Why no `fn` keyword:** it isn't needed, so types stay short and clean.

## 4. Types are values too

A type is a value of type `type`, so it is declared like anything else:

```go
const Pair: type = [2]int;
const Op: type = (int, int)int;

var p: Pair = [1, 2];
```

**Why:** it follows from rule 1. Types get names the same way everything else does.

## 5. Arrays

```go
const primes: [3]int = [2, 3, 5];       // fixed size
var scores: []int = [10, 20, 30];       // unsized: the length comes from the first value

primes[0]          // 2
scores[0] = 15;    // an element can be changed
```

The length of an unsized array is set by its first value and never changes, so a function
can take or return an unsized array of any length.

**Why:** values live on the stack (section 9), and a stack value has a fixed size.

## 6. Maps

```go
var seen: {int}bool = {1: true, 4: false};

seen[4]            // false
seen[4] = true;    // an entry can be changed
```

A map has a fixed size, like an array: its keys are set by its first value and never
change.

**Why:** like arrays, maps live on the stack, so their size can't change.

## 7. Functions

The type gives the parameter types, then the return type. The value gives the parameters,
each declared like a variable, then the body:

```go
const add: (int, int)int = (const x: int, const y: int){
    return x + y;
};

add(1, 2)          // 3
```

A function that returns nothing has no return type, e.g. `(int)`. A function in a `var`
can be swapped for another function of the same type.

**Why:** parameters are declared exactly like names (rule 2), and the function's type
reads the same way as every other type (rule 3).

## 8. `ref`: another name for a variable

```go
var b: int = 3;
ref r: int = b;
r = 5;             // b is now 5
```

- A ref can refer to a variable, an array element or a map entry.
- A ref is **not a type**: it can't go in an array or a map, or be returned.
- A function can take a `ref` parameter to change the caller's variable. Its type shows
  it, `(ref int)`, and `ref` is allowed only there in a type.

```go
const bump: (ref int) = (ref x: int){
    x = x + 1;
};
var n: int = 3;
bump(n);           // n is now 4
```

**Why a way of declaring, not a type:** a reference type couldn't go everywhere other
types can (an array of references or a returned reference breaks), so it would break
rule 1. As a kind of name, every type stays first-class.

**Why it's safe:** a ref can't be stored or returned, so it can never outlive the place it
refers to. That place is always somewhere on the stack, including an array element or a
map entry.

**Why `ref` shows in the function type:** the caller has to know whether to pass the
variable itself or a copy, and with first-class functions the caller only has the type.

## 9. The stack and the heap

For now, every value lives on the stack, unless it's declared `heap`.

### `heap`

```go
heap grid: [3][3]int = [[0, 0, 0], [0, 0, 0], [0, 0, 0]];
```

- A heap variable can change, like a `var`.
- It dies when its scope ends (the `{ }` it was declared in), just like a stack variable,
  unless a pointer to it is still around (below).
- Declaring always makes a new value: `heap copy: [3][3]int = grid;` is a second heap value.

**Why a way of declaring:** where a value lives is about the name, like `ref`. A `heap int`
is still an `int`, so every function still works with it.

**Why it changes:** a heap value that can't change has little point.

**Why it dies with its scope:** it behaves exactly like everything else, and nothing is
handed from one name to another without you seeing it.

### Pointers

To keep heap data alive after a function ends, the function returns a pointer to it:

```go
const make: (int)*int = (const x: int){
    heap n: int = x;
    return &n;         // n's scope ends, but the pointer keeps its value alive
};

var p: *int = make(5);
*p                     // 5
*p = 6;                // change it through the pointer
```

- `&` gives a pointer, and only to something on the heap. `*p` reads or changes what `p`
  points at.
- A heap value is freed when its scope has ended **and** no pointer to it is left.
- Indexing and calling go straight through a pointer: if `q` is a `*[3]int`, `q[0]` is its
  first element.

**Why `*T`:** it goes in front, like every other modifier (rule 3).

**Why `&` only on the heap:** a stack value is freed when its scope ends, so a pointer to it
could be left pointing at nothing. Refs already share stack values safely.

**Why freed only when nothing reaches it:** a pointer can then never point at freed memory,
and there's no `delete` to forget.

**Why indexing goes straight through:** `(*q)[0]` would need brackets.

## Not decided yet

- the basic types (which ones, sizes), strings, converting between types
- structs
- whether a ref can refer to a `const`
- what a `const` pointer allows
- whether a returned function keeps the values it uses
- a ref into heap data that a pointer stops reaching
- reassigning a whole map
