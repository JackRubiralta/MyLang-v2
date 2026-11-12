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
        count^ = count^ + 1;
        return count^;
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
        count^ = count^ + 1;                  // is made; total is left null
        return count^;
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
if (p == null || p^ == 0) { ... }          // p^ is followed only when p isn't null
if (!ready) { ... }                        // runs when ready is false
```
