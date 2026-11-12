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
