# Kept names after a `;` (taken out of DOCS.md)

**Status:** taken out of `DOCS.md` on 12 November 2026 (Jack: get rid of the `;` syntax "for
that one use case"). Kept here as a record.

`DOCS.md` 7.8 keeps a local with the rules that already exist: the local goes on the heap, and
a function whose parameter asks for an ownership takes it with `@` and gives back the function
that uses it (`countFrom(@total)` in `newCounter`).

What this file holds is a piece of syntax that did the same job without the helper: names the
function keeps, declared after a `;` in its brackets and set once, when the function is made.
It is close to C++ lambda init-capture (`[mine = std::move(table)]`) and Swift capture lists.

## The text that was in DOCS.md 7.8

**To keep a local, declare what the function keeps after a `;` in its brackets.** After the
parameters, a `;` starts the names the function keeps. Each is an ordinary declaration with a
value (2.1). It runs once, when the function is made, not at each call:

```go
const makeLookup: ()(int)int = (){
    var table: ^[3]int = ^[10, 20, 30];               // a local, on the heap (8.3)
    return (const i: int; var @mine: ^[3]int = @table){
        return mine^[i];                              // mine took table's array as the
    };                                                // function was made; table is left null
};

var look: (int)int = makeLookup();
look(0)                // 10
look(2)                // 30
```

- **A kept name is like a parameter filled in when the function is made.** The `= value` is
  what tells them apart: a parameter never has one, because the call gives it.
- **The type doesn't show it.** The function above is still a `(int)int`, because a call
  doesn't give the kept names. A function value is still two addresses (7.7).
- **The declaration says how the value is kept.** `var k: int = step` keeps a copy, `const b:
  &int = &x` keeps a borrow, and `var @mine: ^[3]int = @table` takes the value over. A kept
  borrow ties the function to what it borrows (8.5): it can't outlive `x`.
- **A kept name lives as long as the function.** A `var` one can change, and the change stays
  for the next call. Copying the function copies its kept names (8.8), and a kept heap value is
  freed when the function ends (8.11).
- **The body still runs at each call, from the top.** Nothing in it runs early.

A helper whose parameter takes the value does the same job without the `;`:

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
