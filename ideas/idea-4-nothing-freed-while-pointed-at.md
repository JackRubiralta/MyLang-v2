# Idea 4: nothing is freed while something still points at it

**Status:** a sketch for three points `DOCS.md` (8.16) still lists as open, worked out together
because they are one problem. Nothing here is in `DOCS.md`.

Lines marked **(Jack)** are things Jack said. Everything else is a sketch.

---

## 1. Three problems, one cause

| # | The open point | How the value goes away while a pointer still leads to it |
|---|---|---|
| 1 | dangling pointers | its owner is given something new, or set to `null` |
| 6 | moving into an owner that ends sooner | it is moved, and its new owner ends first |
| 5 | returning a borrower from a function | either of the above, done to what the function was lent |

A **pointer** here means a borrower (`&T`) or an ownership (`@T`).

Rule 8.5 says a borrower can't outlive what it borrows, and checks it by comparing where
names end. That covers a value that ends at a `}`. It doesn't cover a value that is made to
end earlier. "Don't do this though." / "We have to think of solution." (Jack)

## 2. The rule

To reach a value on the heap, you follow its owner. So:

**While a pointer that was made by following an owner is still used below, that owner can't
be changed.**

"Changed" is everything that frees or moves what the owner holds:

| The line | What it does to what `p` owns |
|---|---|
| `p = …;` and `p = null;` | frees it |
| `@p = …;` | frees it |
| `@q = @p;` and `var @q: T = @p;` | moves it away |
| `take(@p)` | moves it into the function |
| `change(&p)`, where the function can change `p` | may do any of these |

"Still used below" means written anywhere further down the function. MyLang has no loops,
so later in the text is later in time, and the compiler only has to read down.

This is rule 8.5 carried through. 8.5 says a borrower can't outlive what it borrows. This
says what it borrows can't be made to end early either.

## 3. Which owners a pointer followed

The compiler reads it off the line that made the pointer:

| The pointer | Owners it followed | So these can't be changed while it is used below |
|---|---|---|
| `&p` | none: it points at `p` itself | nothing (8.5 covers `p`'s `}`) |
| `&p^` | `p` | `p` |
| `@list^.next` | `list` | `list` |
| `&list^.next^` | `list`, then `list^.next` | `list` and `list^.next` |
| `&b^.value`, where `b` is a borrower | the ones `b` followed | the same as for `b` |

An owner **beyond** where a pointer points was not followed, so it can still be changed.

## 4. Problem 1: dangling pointers

```go
var p: ^int = ^5;
var b: &int = &p^;               // b followed p
p = ^6;                          // ERROR: b is used below, and this frees what p owns
b^
```

To change the value where it is, so `b` sees it: `p^ = 6;`.

```go
var h: @^Node = @list^.next;     // h followed list
list = makeNode(9);              // ERROR: h is used below
var @n: ^Node = h;
```

Taking the node out first makes both lines fine, because `h` isn't used after that:

```go
var h: @^Node = @list^.next;
var @n: ^Node = h;
list = makeNode(9);              // fine
```

A cursor can change what lies beyond it:

```go
var cur: &Node = &list^.next^;   // cur followed list and list^.next
cur^.next = makeNode(9);         // fine: cur^.next is beyond cur
cur^.value
```

## 5. Problem 6: a value may be moved anywhere

A move changes the owner it empties, so the rule already covers it. Where the value goes
makes no difference:

```go
var data: ^[1000]int = ^[...];
var b: &[1000]int = &data^;          // b followed data
{
    var @inner: ^[1000]int = @data;  // ERROR: b is used below, and this moves what data owns
}
b^[0]
```

Without the last line it is fine: the array is freed at the `}`, and nothing leads to it any
more. A pointer made after the move is made from `inner`, so rule 8.5 ties it to `inner`'s
block.

So moving into an owner that ends sooner needs no rule of its own, and it is allowed. The
earlier rule, "ownership can't be handed into an inner scope" (Jack), was there to stop a
borrower of the old owner outliving the value. This rule stops exactly that, for every
move, and it lets a parameter that asks for an ownership take one (DOCS 8.12).

## 6. Problem 5: a function may return a borrower

**Inside the function nothing new is needed.** The returned borrower lives on after the
call, so by rule 8.5 it can't come from anything that ends with the call: a local, a copied
parameter, or a value the function took over. It can come from a borrowed parameter, because
what that points into lives in the caller.

```go
const firstValue: (&Node)&int = (var n: &Node){
    return &n^.value;                // fine: it comes from n
};

const bad: (&Node)&int = (var n: &Node){
    var local: int = n^.value;
    return &local;                   // ERROR (8.5): local ends with the call
};
```

**At the call, one thing is new: where the result comes from.** The caller only sees the
type, so the result counts as coming from everything that was lent to the function:

- it can't outlive any of the borrowed arguments (8.5);
- if any of them is a `const`, the result has to be a `const` borrower (8.6);
- it counts as having followed the owners on the way to what was lent **and every owner
  inside it**, because the compiler doesn't know how deep the function went.

```go
var v: &int = firstValue(&list^);    // v comes from list's data
list = makeNode(9);                  // ERROR: v is used below
v^
```

A returned borrower can still change what lies beyond itself, like any cursor:

```go
var cur: &Node = second(&list^);
cur^.next = makeNode(9);             // fine: reached through cur itself
```

With two borrowed arguments the type can't say which one the result came from, so it counts
as coming from both. That is safe, and sometimes stricter than needed.

## 7. Calls

While a call runs, its pointer arguments are in use. So the rule applies to the call line
itself, with the look, change, take levels (DOCS 8.9) saying what the function may do:

| The argument | The function may | So the call is an error when another pointer in use followed… |
|---|---|---|
| `f(x)`, a copy | nothing to the caller's value | never |
| `f(&x)` to a `const` parameter | read | never |
| `f(&x)` to a `var` parameter | change `x`, and every owner inside it | `x`, or an owner inside it |
| `f(@x)` | take `x`'s value | `x` |

"Another pointer in use" is one used below in the caller, or another argument of the same
call: `f(&list^.next^, @list)` hands the list over while the first argument points into it.

A function's type doesn't say whether a `&T` parameter is read or changed (Jack: "leave it").
For a function called by its own name the compiler looks at the function. For one called
through a variable, an array or a map, every `&` argument counts as "may change".

## 8. What stays with the programmer

- **A pointer kept inside a heap value,** like the `prev` in DOCS 8.14. It isn't a variable,
  so the compiler can't see when it is used. This is the same case as copying that list:
  the programmer writes it out.
- **Code that is safe but has to be reordered or split,** such as two borrowers returned
  from the same list, when one is used to change the list.

## 9. A function can change a variable from outside itself (Jack)

"It can change [a] variable defined outside of its scope." (Jack) It is in DOCS 7.8. So the
table in section 7 is not the whole story: a call can change an owner without being handed
it, and the call line doesn't show it.

```go
var list: ^Node = makeNode(1);

const reset: ()void = (){
    list = makeNode(9);              // changes a variable from outside: the old first node is freed
};

var b: &Node = &list^;               // b followed list
reset();                             // nothing on this line shows that list changes
b^.value                             // b points into freed memory
```

For the rule in section 2 to hold, the compiler has to know which outside variables a call
changes (sketch):

| The call | What the compiler knows |
|---|---|
| a function called by its own name | it reads the function, and the functions that one calls by name: the outside variables they change or take |
| a function called through a variable, an array or a map | not which function runs, so the call counts as changing every outside variable |

With that, `reset();` above is an error: `b` is used below, and the call changes `list`.

Two more things come with a function being able to use outside variables:

- **A function that outlives the variables it uses.** A function made inside another
  function can use that function's locals. If it is returned or stored, it is still there
  after they are gone. Using an outside variable is like borrowing it, so rule 8.5 would say
  such a function can't outlive the variables it uses. Top-level functions that use
  top-level variables are always fine. (open)
- **The call shows nothing.** "The mark at a call shows what a function may do" holds for a
  variable that is passed (DOCS 8.9).

## 10. Why not a check while running

Every heap value could carry a number that changes when it is freed, with every pointer
remembering the number it saw, and every follow comparing them. That would cover `prev` too.
But every pointer and every heap value gets bigger, and every follow does work the code
doesn't show.

## 11. Open

- Whether to take this.
- "Still used below", or "still in scope". The second is closer to how 8.5 is worded and
  rejects more: in section 4 the fixed example would need `h` in a block of its own.
- How the compiler learns what a call changes (section 9), and a function that outlives the
  variables it uses.
