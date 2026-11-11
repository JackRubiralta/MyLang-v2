# Idea 5: keeping a value in a function with `@`

**Status:** taken out of `DOCS.md` (Jack, 11 Nov 2026: "undo these changes to docs about the
ways around it, because they kind of break some of the rules already I said"). Kept here for
later.

What stayed in `DOCS.md` 7.8: a function borrows the variables it uses from around it, and
a variable can't die before a function that uses it. A function given back keeps the
parameters it uses (Jack: `makeAdder` should work, because `step` is a parameter), and it
can't outlive the arguments the call was given (Jack: if the argument dies first, the
compiler raises an error). A local
declared inside the function is never kept: returning a function that uses one is an error. To keep
a local within those rules (Jack, 11 Nov 2026): declare it on the heap and hand it with `@`
to a parameter of a function that gives back the function using it (DOCS 7.8, `countFrom`).

## Why it was taken out

These rules, which I filled in, clash with rules Jack had set:

- **`@x` is an `@T`, not a `T`** (DOCS 8.9; Jack, 10 Nov 2026: an `@T` is not used directly
  as a `T`). Here `@kept` in a body stood for the function's own `kept`, used as its value.
- **Reading order** (DOCS 8.2): `@list^.next` is "the ownership of that field", so `@count^`
  would be `@(count^)`. Here it had to mean "the function's own `count`, followed".
- **A take written inside a body that runs at every call** happened once, when the function
  was made.

## The text that was in DOCS.md 7.9

A function that has to outlive the variables it uses keeps a value on the heap. It takes the
value over with `@` in front of the name, the same mark as every other move (8.9):

```go
const makeCounter: ()()int = (){
    var count: ^int = ^0;
    return (){
        @count^ = @count^ + 1;
        return @count^;
    };                         // the function takes count's heap int over, and count is left null
};

var next: ()int = makeCounter();
next()                         // 1
next()                         // 2: the function keeps its value from one call to the next
```

- **The function takes the value when it's made,** which is when the program reaches its
  `(…){…}`, not at each call. The value doesn't move: the function becomes its owner, and the
  old variable is left `null`, as in every move.
- **Only a value on the heap can be taken,** so the value always stays where the code put
  it. The function keeps its owner, an address (7.7). `@` on a plain value from outside is an
  error: put a copy on the heap first (below).
- **Every use of that name in the body carries the `@`,** so each one shows it's the
  function's own. `@` marks the name here, as in `var @q: ^int = @p;` (2.1), so it's read
  before what is behind it: `@count^` is the function's own `count`, followed.
- **The function owns the value from then on.** It is freed when the function value ends or
  is given another function (8.11). Copying the function value copies it too (8.8), so two
  copies of a counter count separately. A value has one owner, so two functions can't take
  the same variable.

There are three ways to get the value onto the heap, depending on where it starts.

**A plain local or parameter: put a copy on the heap.**

```go
const makeAdder: (int)(int)int = (const step: int){
    var kept: ^int = ^step;                          // a copy of step, on the heap
    return (const n: int){ return n + @kept^; };     // the function takes kept over
};

makeAdder(5)(2)                // 7
```

**A local that is already on the heap: take it as it is,** as `makeCounter` does with
`count`.

**A parameter on the heap: take it the same way.** A parameter declared `var data: ^[3]int`
is the function's own copy, and one declared `var @data: ^[3]int` is the caller's own value,
taken over at the call (8.13). Either way it is a heap value of the outer function, so the
function it gives back can take it:

```go
const reader: (@^[3]int)(int)int = (var @data: ^[3]int){
    return (const i: int){ return @data^[i]; };      // the function takes data over
};

var values: ^[3]int = ^[10, 20, 30];
var read: (int)int = reader(@values);                // values is left null at the call
read(1)                        // 20: read owns the array now
```

A function can take some names and borrow others. The ones it borrows still can't die before
it.
