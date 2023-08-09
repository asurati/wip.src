---
title: "CPP: Nested Replacement Ambiguity"
date: '2023-08-07'
categories:
  - Programming
tags:
  - C
  - CPP
  - preprocessor
---

The [C2X](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3096.pdf)
standard gives an example of a case where . . .
> . . . it is not clear whether a replacement is nested or not.

Below is the example, in some detail, to expose the ambiguity.

---

### **6.10.4.4 Rescanning and further replacement**

The example source code is:

``` c
    #define f(a) a*g
    #define g(a) f(a)
    f(2)(9)
```

The tokens subjected to macro replacement are those that constitute the
invocation line `f(2)(9)`. The `active-macro-stack` is initially empty.

Once the scanner reads the first identifier `f` on the line, and searches the
list of defined macros for a match with that identifier, it finds a hit for a
function-like macro named `f`. On further scanning the line, it recognizes an
invocation, of the form `f(2)`, of that macro. The tokens forming the `rest` of
the line, `(9)`, are, at this point, external to the recognized
macro-invocation and to its replacement-list.

To prepare for processing the macro-invocation `f(2)`, the preprocessor (`CPP`)
pushes, on the `active-macro-stack`, an entry corresponding to the
macro-invocation. The name of the macro, `f`, is the most important property of
the entry.

The stack at this point is:

```
    // active-macro-stack


    f(2)    <---- bottom of the stack
    ---------------------------------
```

Now towards expanding the `f(2)` macro-invocation.

The full expansion (i.e. complete macro replacement) of the argument `2` is `2`
itself (the symbol `2` isn't subject to such replacement in the first
place, for it is not an identifier in the language of CPP).
After substituting the argument into the replacement-list of the macro
`f`, the situation is as shown below:

```
    f(2)     rest
    +---     +---
    |        |
    v---     v---
    2*g      rest
```

The CPP now rescans the token-sequence `2*g`. The tokens for the symbols `2`
and `*` are not subject to replacement (they aren't `identifiers` in the
language of the CPP), hence they are output as they are, without any change.
The identifier `g` is part of the replacement-list of the macro `f` currently
being expanded, but it doesn't match with the name of any active macro
(i.e. with the name of any macro that is being represented on the
 `active-macro-stack` at this point in time.)
Hence, this token for the identifier `g` is allowed replacement during
rescanning.

But `g` is the name of a function-like macro. Since the rescanning step has
access to the `rest` of the tokens, it can decide on the replacement of `g`
after looking ahead one token to see if it can possibly manage to construct a
complete macro-invocation.

In this example, the rescanning step can indeed manage to construct such an
invocation. The ambiguity arises exactly at this point.

---

### **Option 1**

One way to think about this situation is as described below:

```
    f(2)     (9)  <--- rest of the tokens
    +---     +--
    |        |
    v--      v--
    2*g  ->  (9)



             g(9)
             +---
```

The token for the identifier `g` can be thought of as moving **out**
(marked in the above diagram with the arrow <nobr>`->`</nobr>) of the
influence of the active macro `f` whose replacement-list generated it,
because it needs tokens that are external to the replacement-list of which it
is a part. As a result, the entry for the macro-invocation `f(2)` can be
popped off the `active-macro-stack` and the entry for the macro-invocation
`g(9)` can be pushed.

```
    // active-macro-stack

    f(2)    <---- bottom of the stack
    ---------------------------------
```

```
    // active-macro-stack

    (stack empty)
    ---------------------------------
```

```
    // active-macro-stack

    g(9)    <---- bottom of the stack
    ---------------------------------
```

The macro-invocation `f(2)` will no longer influence the macro
processing that lies ahead, including the immediate processing of the
macro-invocation `g(9)`, even though the token for the identifier `g` was
donated by the replacement-list of the very macro-invocation `f(2)` that was
popped off.

Now towards expanding the `g(9)` macro-invocation.

The full expansion of the argument `9` is `9` itself. After substituting the
argument into the replacement-list of the macro `g`, the situation is as
shown below:

```
    g(9)        rest is empty
    +---
    |
    v---
    f(9)
```

The CPP now rescans the token-sequence `f(9)`. The identifier `f` is part of
the replacement-list of the macro `g` currently being expanded, but it doesn't
match with the name of any active macro. Hence, this token for the identifier
`f` is allowed replacement during rescanning.

It so happens that the substitution of the replacement-list of the macro `g`
results in a complete invocation of the macro `f`, without even
looking ahead into the `rest` of the tokens (Note that `rest` here is empty for
this example.)

Since the rescan determined that the identifier `f` is subject to replacement,
it pushes an entry corresponding to the macro-invocation `f(9)` onto the
`active-macro-stack`.

```
    // active-macro-stack


    f(9)    <---- top of the stack
    g(9)    <---- bottom of the stack
    ---------------------------------
```

Now towards expanding the `f(9)` macro-invocation.

The full expansion of the argument `9` is `9` itself. After substituting the
argument into the replacement-list of the macro `f`, the situation is as
shown below:

```
    g(9)        (rest is empty)
    +---
    |
    v---
    f(9)
    +---
    |
    v--
    9*g
```

The CPP now rescans the token-sequence `9*g`. The tokens for `9` and `*` are
not subject to replacement, hence they are output as they are, without
any change. The identifier `g` is part of the replacement-list of the macro
`f` currently being expanded (and also indirectly of the replacement-list of
the macro `g`. See the `active-macro-stack` above), and it does match with the
name of an active macro, `g`; an invocation of the form `g(9)` is currently
active as evident from the `active-macro-stack`. Hence, this token for the
identifier `g` is marked to prevent its replacement.

Even if there were further tokens in the source file (there aren't in this
example) that produced a complete macro-invocation of the macro `g`, that
invocation would not be considered for replacement because the token `g` is
already marked for non-replacement.

The output, then, is `2*9*g`.

---

### **Option 2**

Another way to think about the same situation is as described below:

```
    f(2)     (9)  <--- rest of the tokens
    +---     +--
    |        |
    v---     v--
    2*g  <-  (9)



    g(9)
    +---
```

The token-sequence `(9)` can be thought of as moving **into** (marked in the
above diagram with the arrow <nobr>`<-`</nobr>) the influence of the
`active-macro-stack`, specifically of the active macro `f` with whose
replacement-list the sequence will be joined. This movement is required to
construct the macro-invocation `g(9)`. As a result, the entry for the
macro-invocation `f(2)` remains active, influencing this one macro-invocation
`g(9)`. But that entry does not influence any macro processing that may occur
on tokens lying after `(9)`. CPP pushes onto the `active-macro-stack` an entry
corresponding to the macro-invocation `g(9)`.

```
    // active-macro-stack


    g(9)    <---- top of the stack
    f(2)    <---- bottom of the stack
    ---------------------------------
```

Now towards expanding the `g(9)` macro-invocation.

The full expansion of the argument `9` is `9` itself. After substituting the
argument into the replacement-list of the macro `g`, the situation is as
shown below:

```
    f(2)     (9)  <--- rest of the tokens
    +---     +--
    |        |
    v---     v--
    2*g  <-  (9)



    g(9)       (rest is empty)
    +---
    |
    v---
    f(9)
```

The CPP now rescans the token-sequence `f(9)`. The identifier `f` is part of
the replacement-list of the macro `g` currently being expanded, and it does
match with the name of an active macro, `f`; an invocation of the form `f(2)`
is currently active as evident from the `active-macro-stack`. Hence, this token
for the identifier `f` is marked to prevent its replacement.

The output, then, is `2*f(9)`.

---

### **Conclusion**

The direction of the movement between the tokens under the influence of the
top macro on the `active-macro-stack` and the tokens external to the
replacement-list of that macro determines the behavior that is exhibited by the
implementation.

Note that the tokens, external to the replacement-list of the macro, say `m0`,
that sits at the top of the `active-macro-stack`, can still be a part of the
replacement-list of a macro, say `m1`, that sits directly beneath `m0` on the
stack. Thus, such tokens are not considered external to `m1` after the
processing pops `m0` off the stack.

Subjecting the example to the latest `GCC`, `Clang/LLVM` and `MSVC` compilers
for `x86-64` shows that they behave as described in
[Option 1](#option-1) - i.e., their behavior is consistent with that where the
tokens move *out* of the influence of the latest-expanded macro, instead of
that where the tokens external to the replacement-list are brought *into* the
influence.

---

*Section 6.10.4.4* of the document declares these options/behaviors as
`unspecified behavior`:

> behavior, that results from the use of an unspecified value, or other
> behavior upon which this document provides two or more possibilities and
> imposes no further requirements on which is chosen in any instance

The section also warns:

> Strictly conforming programs are not permitted to depend on such unspecified
behavior.

*Annex J, Section J.1, Item #24* says:

>When a fully expanded macro replacement list contains a function-like macro
> name as its last preprocessing token and the next preprocessing token from
> the source file is a (, and the fully expanded replacement of that macro ends
> with the name of the first macro and the next preprocessing token from the
> source file is again a (, whether that is considered a nested replacement
> (6.10.4). . . . . . . . . .[is unspecified behavior.]


