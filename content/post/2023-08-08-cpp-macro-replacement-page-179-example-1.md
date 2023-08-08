---
title: "CPP: Macro Replacement: Page 179, EXAMPLE 1"
date: '2023-08-08'
categories:
  - Programming
tags:
  - C
  - CPP
  - preprocessor
---

This post explains the EXAMPLE 1 given on Page# 179
(page# that would be found printed on the page of a paper-copy, not the one
displayed on a pdf reader application) of the working draft,
dated April 1, 2023, of the
[C2X](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3096.pdf)
standard.

---

### **Page 179, EXAMPLE 1**

``` c
#define LPAREN() (
#define G(Q) 42
#define F(R, X, ...) __VA_OPT__(G R X) )
int x = F(LPAREN(), 0, <:-); // replaced by int x = 42;
```

The tokens for the identifiers `int` and `x` do not correspond to the names of
any macro. Hence, they are output as they are, without any change.

The token for the symbol `=` is not subject to replacement; it is output as it
is, without any change.

The macro-invocation is `F(LPAREN(), 0, <:-)`. The macro `F` is variadic, and
its invocation here does pass a non-empty argument-list corresponding to the
trailing `...` of its parameter-list. Thus, `__VA_ARGS__` consists
of a single argument made up of the non-empty token-sequence `<:-`;
`__VA_OPT__` is therefore obliged to expand the pp-tokens contained within it.

To begin the macro expansion, the CPP pushes on to the `active-macro-stack-#0`
an entry corresponding to this invocation of the macro `F`:

```
    // active-macro-stack-#0

    F(LPAREN(), 0, <:-)    <---- bottom of the stack
    ------------------------------------------------
```

The argument `LPAREN()` corresponds to the parameter `R`, the argument `0`
corresponds to the parameter `X`, and the argument `<:-` corresponds to the
parameter `...`.

---

#### Expansion of the argument `LPAREN()`

Since the parameter `R` occurs in the replacement-list of the
macro `F`, without being preceded by a `#` or a `##` and without being
followed by a `##`, the corresponding argument `LPAREN()` is subject to
argument-expansion. As it turns out, `LPAREN` is the name of a function-like
macro, and its invocation exists as the first argument to the macro `F`.

A new active-macro-stack, `active-macro-stack-#1` is initialized with the entry
corresponding to the invocation `LPAREN()`.

```
    // active-macro-stack-#1

    LPAREN()    <---- bottom of the stack
    -------------------------------------
```

The macro takes no parameters, and is provided with no arguments. Its
replacement-list is just `(`.

```
    LPAREN()    (rest is empty)
    +-------
    |
    v
    (
```

The CPP rescans the token-sequence `(`. The token for the symbol `(` is not
subject to replacement, hence it is output as it is, without any change.
This also means that the parameter `R` of the macro `F` is now bound to the
fully-expanded argument `(`.

At this stage, the CPP has completed the processing of the `LPAREN()`
invocation. That is, the processing now will move out of the boundaries of the
replacement-list of the invocation, without any previous processing
pending. The invocation no longer influences the macro processing that lies
ahead. Its entry from the `active-macro-stack-#1` is removed, and with that
removal, the stack becomes empty.

The CPP then attempts to process the `rest` of the tokens, but it is empty.

With that, the full expansion of the argument `LPAREN()` is complete.

---

#### Expansion of the argument `0`

The argument `0`, corresponding to the parameter `X`, is not an identifier;
it is not subject to the argument-expansion (IOW, its full expansion is the
same as its current form.)

---

#### Expansion of the argument for `__VA_ARGS__`

The argument corresponding to the parameter `__VA_ARGS__` is not subject to
argument-expansion.

---

#### Expansion of the argument for `__VA_OPT__`

Since the argument corresponding to `...` is non-empty, the parameter
`__VA_OPT__` that occurs in the replacement-list of the macro must be replaced
with the expansion of its pp-tokens. This expansion is carried out with respect
to the `active-macro-stack-#0` instead of creating a new stack as is usually
done for argument-expansion.

The identifier `G` matches the name of a function-like macro `G`, but within
`__VA_OPT__` here, `G` is not followed by a `(` token. Moreover, the name `G`
does not match with the name of any active macro (i.e. with the name of any
macro that has an entry in the `active-macro-stack-#0`.) Hence, this token for
the identifier `G` is allowed replacement during rescanning.

The identifier `R` represents the parameter `R` of the macro `F`. Its expansion
is `(`. (Note: if `R` were additionally a macro name, the parameter name seems
to bind more strongly than the macro name. That is, a macro parameter named `R`
hides a macro named `R`.)

The identifier `X` represents the parameter `X` of the macro `F`. Its expansion
is `0`.

---

#### Argument Substitution

After the parameters are substituted with the corresponding, expanded
arguments, the state is:

```
    F(LPAREN(), 0, <:-)     (rest is ;)
    +------------------
    | after arg-substitution
    v---
    G(0)
```

The state of the `active-macro-stack-#0` is:

```
    // active-macro-stack-#0

    F(LPAREN(), 0, <:-)    <---- bottom of the stack
    ------------------------------------------------
```

The CPP now rescans the token-sequence `G(0)`. The identifier `G` is part of the
replacement-list of the macro `F` currently being expanded. Its token was also
not marked for non-replacement during the `__VA_OPT__` processing. The
identifier also doesn't match with the name of any active macro. Hence, this
token for the identifier `G` is allowed replacement during rescanning.

But `G` is the name of a function-like macro, and a complete invocation of that
macro exists, even without reading from the `rest` of the tokens, as shown
above. Hence, the invocation is processed.

The CPP pushes on to the `active-macro-stack-#0` an entry corresponding to the
invocation `G(0)`:

```
    // active-macro-stack-#0

    G(0)                   <---- top of the stack
    F(LPAREN(), 0, <:-)    <---- bottom of the stack
    ------------------------------------------------
```

#### Expansion of the argument `0`

The argument `0`, corresponding to the parameter `Q`, is not an identifier;
it is not subject to the argument-expansion (IOW, its full expansion is the
same as its current form.)

---

#### Argument Substitution

After the parameters are substituted with the corresponding, expanded
arguments, the state is:

```
    F(LPAREN(), 0, <:-)     (rest is ;)
    +------------------
    | after arg-substitution
    v---
    G(0)
    +---
    | after arg-substitution
    v-
    42
```

The state of the `active-macro-stack-#0` is:

```
    // active-macro-stack-#0

    G(0)                   <---- top of the stack
    F(LPAREN(), 0, <:-)    <---- bottom of the stack
    ------------------------------------------------
```

The CPP now rescans the token-sequence `42`. The token for the symbol `42` is
not subject to replacement, hence it is output as it is, without any change.

At this stage, the CPP has completed the processing of the `G(0)` invocation.
That is, the processing now will move out of the boundaries of the
replacement-list of the invocation, without any previous processing
pending. The invocation no longer influences the macro processing that lies
ahead. Its entry from the `active-macro-stack-#0` is removed.

With that removal, the CPP has also completed the processing of the
`F(LPAREN(), 0, <:-)` invocation. The invocation no longer influences the
macro processing that lies ahead. Its entry from the `active-macro-stack-#0` is
removed, too.

The `active-macro-stack-#0` is now empty:

```
    // active-macro-stack-#0

    (stack is empty)
    ------------------------------------------------
```

The CPP next moves into the `rest` of the tokens from the source file. The only
token left is `;`, and it is not subject to replacement. It is output as it is,
without any change.

---

### **Result**

The invocation `F(LPAREN(), 0, <:-)` expanded to a single token, `42`.
Hence, the CPP outputs:

```c
int x = 42;
```

### **Notes**

The expansion of the `__VA_OPT__` parameter is carried out using the same
active-macro-stack as the one on which the `__VA_OPT__`-containing
macro-invocation resides.
