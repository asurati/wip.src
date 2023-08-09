---
title: "CPP: Macro Replacement: Page 182, EXAMPLE 3"
date: '2023-08-09'
categories:
  - Programming
tags:
  - C
  - CPP
  - preprocessor
---

This post explains the EXAMPLE 3 given on Page# 182
(page# that would be found printed on the page of a paper-copy, not the one
displayed on a pdf reader application) of the working draft,
dated April 1, 2023, of the
[C2X](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3096.pdf)
standard.

---

``` c
#define x 3
#define f(a) f(x * (a))
#undef x
#define x 2
#define g f
#define z z[0]
#define h g(\~{ }
#define m(a) a(w)
#define w 0,1
#define t(a) a
#define p() int
#define q(x) x
#define r(x,y) x ## y
#define str(x) # x
f(y+1) + f(f(z)) % t(t(g)(0) + t)(1);
g(x+(3,4)-w) | h 5) & m
(f)^m(m);
p() i[q()] = { q(1), r(2,3), r(4,), r(,5), r(,) };
char c[2][6] = { str(hello), str() };
```

---

### <ins>Expansion of `f(y+1)`</ins>

The CPP pushes on the currently empty `active-macro-stack-#0` an entry
corresponding to the invocation `f(y+1)`:

```
    // active-macro-stack-#0

    f(y+1)    <---- bottom of the stack
    -----------------------------------
```

The argument `y+1` is expanded to itself, since the identifier `y` isn't the
name of any macro, and the symbols `+` and `1` are not subject to expansion.

After argument-substitution, the state is:

```
    f(y+1)    (rest is: + f(f(z)) % . . .)
    +-----
    | after arg-substitution
    v-----------
    f(x * (y+1))
```

The CPP now rescans the token-sequence `f(x * (y+1))`. The identifier `f`
matches with the name of an active macro, `f` itself, as evident from the state
of the `active-macro-stack-#0`. Hence, this token for the
identifier `f` is marked for non-replacement. In this post, such marked
identifiers are written in `CAPITAL`. The current state of expansion is:

```
    f(y+1)    (rest is: + f(f(z)) % . . .)
    +-----
    | after arg-substitution
    v-----------
    f(x * (y+1))
    F(x * (y+1))    <--- mark f for non-replacement
```

Since `F` is marked for non-replacement, the token-sequence `(x * (y+1))` isn't
considered to be a part of an invocation of the macro `f`. The CPP moves ahead
and reaches `x`. Although a macro named `x` is defined, undefined and then
redefined, the definition that is active is the latest one lexically
before the tokens currently being expanded. Accordingly, `x` is expanded to
`2`, as shown below.

The identifier `x` is the name of an obj-like macro. It also doesn't match with
the name of any active macro, as evident from the state of the
`active-macro-stack-#0`. Hence, this token for the identifier `x` is allowed
replacement. The CPP pushes on to the `active-macro-stack-#0`
an entry for the macro-invocation `x`:

```
    // active-macro-stack-#0

    x         <---- top of the stack
    f(y+1)    <---- bottom of the stack
    -----------------------------------
```

The replacement-list of the macro `x` is just `2`:

```
    f(y+1)    (rest is: + f(f(z)) % . . .)
    +-----
    | after arg-substitution
    v-----------
    F(x * (y+1))    <--- mark f for non-replacement
      +
      |
      v
      2
```

The CPP now rescans the token `2`. The token isn't subject to replacement.
Thus, the expansion of the macro-invocation `x` is `2`. The CPP will move out
of the boundaries of the replacement-list of the macro-invocation `x`, with no
other processing related to that invocation pending. The corresponding entry
within the `active-macro-stack-#0` is popped off.

The state is:

```
    // active-macro-stack-#0

    f(y+1)    <---- bottom of the stack
    -----------------------------------
```

```
    f(y+1)    (rest is: + f(f(z)) % . . .)
    +-----
    | after arg-substitution
    v-----------
    f(x * (y+1))
    F(x * (y+1))    <--- mark f for non-replacement
    F(2 * (y+1))    <--- replace x by 2
```

The token-sequence `* (y+1))` remains as it is. With that, the CPP now moves
out of the boundaries of the replacement-list of the macro-invocation `f(y+1)`,
with no other processing related to that invocation pending. The corresponding
entry within the `active-macro-stack-#0` is popped off, and the stack is now
empty.

The output at this stage is: `F(2 * (y+1))`.

(Note that the capital letters only declare that the token was
marked for non-replacement. The actual output of a CPP implementation will be
`f(2 * (y+1))`.)

The `rest` begins with `+`. That symbol is output as it is. Then follows the
invocation `f(f(z))`.

---

### <ins>Expansion of `f(f(z))`</ins>

The CPP pushes on to the now empty `active-macro-stack-#0` an entry
corresponding to the invocation `f(f(z))`.

```
    // active-macro-stack-#0

    f(f(z))    <---- bottom of the stack
    ------------------------------------
```

```
    f(f(z))    (rest is: % t(t(g) . . .)
    +------
```

The argument is `f(z)`. It must be fully expanded, over a new
active-macro-stack, before proceeding.

---

#### Expansion of `f(z)`

The CPP pushes on to a new `active-macro-stack-#1` an entry
corresponding to the invocation `f(z)`.

```
    // active-macro-stack-#1

    f(z)    <---- bottom of the stack
    ---------------------------------
```

```
    f(z)    (rest is: % t(t(g) . . .)
    +---
```

The argument is `z`, which is the name of an object-like macro. It must
be fully expanded, over a new active-macro-stack, before proceeding.

---

#### Expansion of `z`

The CPP pushes on to a new `active-macro-stack-#2` an entry
corresponding to the invocation `z`.

```
    // active-macro-stack-#2

    z    <---- bottom of the stack
    ---------------------------------
```

Now towards expanding `z`:

```
    z   (rest is:  empty)
    +
    | after substitution
    v---
    z[0]
```
The CPP now rescans the token-sequence `z[0]`. The identifier `z` matches the
name of an active macro, `z` itself, as evident from the
`active-macro-stack-#2`. Hence, this token of the identifier `z` is marked for
non-replacement. Note that this token will be substituted in the expansions of
`f(z)` and of `f(f(z))`. These substitutions do not change the fact that this
token is marked for non-replacement. The rescanning that will occur once the
processing of macros `f(z)` and `f(f(z))` resumes will not expand this token.
The mark is persistent.

```
    z   (rest is:  empty)
    +
    | after substitution
    v---
    z[0]
    Z[0]    <--- mark z for non-replacement
```

The token-sequence `[0]` is output as it is, without change. With that, the CPP
moves out of the boundaries of the replacement-list of the macro-invocation
`z`, with no other processing for that invocation pending. The corresponding
entry within the `active-macro-stack-#2` is popped off; the stack is now empty.

---

#### Expansion of `f(z)` continues

```
    // active-macro-stack-#1

    f(z)    <---- bottom of the stack
    ---------------------------------
```

The fully expanded argument is substituted into the replacement-list of the
macro `f`:

```
    f(z)    (rest is:   % t(t(g) . . .)
    +---
    | after arg-substitution
    v--------------
    f(x * (Z[0]+1))
```

The CPP now rescans the token-sequence `f(x * (Z[0]+1))`. Since the identifier
`f` matches with the name of an active macro, `f` itself, as evident
from the `active-macro-stack-#1`, this token for the identifier `f` is marked
for non-replacement. Note that this token will make its way through to the
replacement-list of `f(f(z))`; even there this token will remain
non-replaceable.

```
    f(z)    (rest is:   % t(t(g) . . .)
    +---
    | after arg-substitution
    v--------------
    f(x * (Z[0]+1))
    F(x * (Z[0]+1))    <--- mark f for non-replacement
```

The token-sequence `(x * (Z[0]+1))` therefore doesn't form a macro-invocation
of the macro `f`.

Since the identifier `x` doesn't match with the name of any active
macro, as evident from the state of the `active-macro-stack-#1`, it is allowed
to be replaced. Its replacement is just `2`.

```
    f(z)    (rest is:   % t(t(g) . . .)
    +---
    | after arg-substitution
    v--------------
    f(x * (Z[0]+1))
    F(x * (Z[0]+1))    <--- mark f for non-replacement
    F(2 * (Z[0]+1))    <--- replace x by 2
```

Since the identifier `z` is marked for non-replacement, it is not considered
for replacement.

The CPP now moves out of the boundaries of the replacement-list of the
macro-invocation `f(z)`, with no other processing for that invocation pending.
The corresponding entry within the `active-macro-stack-#1` is popped off;
the stack is now empty.

The output at this stage is: `F(2 * (Z[0]+1))`.

---

### <ins>Expansion of `f(f(z))` continues</ins>

```
    // active-macro-stack-#0

    f(f(z))    <---- bottom of the stack
    ------------------------------------
```

```
    f(f(z))     (rest is:   % t(t(g) . . .)
    +------
```

The fully expanded argument is substituted into the replacement-list of the
macro `f`:

```
    f(f(z))     (rest is:   % t(t(g) . . .)
    +------
    | after arg-subsitution
    v---------------------
    f(x * (F(2 * (Z[0]+1))))
```

The CPP now rescans the token-sequence `f(x * F(2 * (Z[0]+1)))`. The first
identifier `f` matches the name of an active macro, `f` itself, as evident from
the `active-macro-stack-#0`. Hence, this token for the identifier `f` is marked
for non-replacement.

```
    f(f(z))     (rest is:   % t(t(g) . . .)
    +------
    | after arg-subsitution
    v---------------------
    f(x * (F(2 * (Z[0]+1))))
    F(x * (F(2 * (Z[0]+1))))    <--- mark f for non-replacement
```

Since the identifier `x` doesn't match with the name of any active
macro, as evident from the state of the `active-macro-stack-#1`, it is allowed
to be replaced. Its replacement is just `2`.

```
    f(f(z))     (rest is:   % t(t(g) . . .)
    +------
    | after arg-subsitution
    v---------------------
    f(x * (F(2 * (Z[0]+1))))
    F(x * (F(2 * (Z[0]+1))))    <--- mark f for non-replacement
    F(2 * (F(2 * (Z[0]+1))))    <--- replace x by 2
```

The CPP now moves out of the boundaries of the replacement-list of the
macro-invocation `f(f(z))`, with no other processing for that macro pending.
The corresponding entry within the `active-macro-stack-#0` is popped off;
the stack is now empty.

The cumulative output until now is:

`F(2 * (y+1)) +  F(2 * (F(2 * (Z[0]+1))))`.

The `rest` begins with `%`. That symbol is output as it is. Then follows the
invocation `t(t(g)(0) + t)`, with `(1); . . .` as the `rest`.

---

### <ins>Expansion of `t(t(g)(0) + t)`</ins>

The CPP pushes on to the now empty `active-macro-stack-#0` an entry
corresponding to the macro-invocation `t(t(g)(0) + t)`

```
    // active-macro-stack-#0

    t(t(g)(0) + t)    <---- bottom of the stack
    -------------------------------------------
```

```
    t(t(g)(0) + t)      (rest is: (1); . . .)
    +-------------
```

The argument is `t(g)(0) + t`. It must first be fully expanded,
over a new active-macro-stack, before proceeding.

---

#### Expansion of `t(g)(0) + t`

The CPP pushes on to a new `active-macro-stack-#3` an entry corresponding to
the invocation `t(g)`, with `(0) + t` as `rest`.

```
    // active-macro-stack-#3

    t(g)    <---- bottom of the stack
    ---------------------------------
```

```
    t(g)    (rest is:   (0) + t)
    +---
```

The argument is `g`, which is the name of an object-like macro.
It must be fully expanded, over a new active-macro-stack, before
proceeding.

---

#### Expansion of `g`

The CPP pushes on to a new `active-macro-stack-#4` an entry corresponding to
the invocation `g`

```
    // active-macro-stack-#4

    g    <---- bottom of the stack
    ---------------------------------
```

```
    g   (rest is:  empty)
    +
```
Now towards expanding `g`:

```
    g   (rest is:  empty)
    +
    | after substitution
    v
    f
```

The expansion of `g` is `f`. The CPP now rescans the token `f`. The identifier
`f` does not match with the name of any active macro, as evident from the
`active-macro-stack-#4`. Hence, this token for the identifier `f` is allowed
replacement. The identifier `f` matches with the name of a function-like macro,
but the `rest` here is empty. The CPP now moves out of the boundaries of the
replacement-list of the macro-invocation `g`. The corresponding entry within
the `active-macro-stack-#0` is popped off;the stack is now empty.

---

#### Expansion of `t(g)(0) + t` continues

```
    // active-macro-stack-#3

    t(g)    <---- bottom of the stack
    ---------------------------------
```

The fully expanded argument is substituted into the replacement-list of the
macro `t`:

```
    t(g)    (rest is: (0) + t)
    +---
    | after arg-substitution.
    v
    f
```

The CPP now rescans the token `f`. It arrived from the expansion of the
argument `g`, with its ability to be replaced intact. The identifier
`f` doesn't match with the name of any active macro, as evident from the
state of the `active-macro-stack-#3`.

The identifier `f` matches the name of a function-like macro, and looking ahead
into the `rest`, the CPP can construct a complete invocation of
that macro in the form `f(0)`. Here the point of ambiguity, as described in the
[post](http://localhost:1313/wip/post/2023/08/07/cpp-nested-replacement-ambiguity),
arises. Choosing [Option 1](http://localhost:1313/wip/post/2023/08/07/cpp-nested-replacement-ambiguity/#option-1),
the identifier `f` is donated out
of the influence of the `t(g)` macro-invocation. As a result, the entry
corresponding to the invocation `t(g)` is popped off the
`active-macro-stack-#3` and an entry corresponding to the invocation `f(0)` is
pushed:

```
    // active-macro-stack-#3

    f(0)    <---- bottom of the stack
    ---------------------------------
```

```
    f(0)    (rest is: + t)
    +---
```

After argument-substitution:

```
    f(0)    (rest is: + t)
    +---
    | after arg-substitution
    v-----------
    f(x * (0))
```

The process now follows similar to other expansions of the macro `f` described
above:

```
    f(0)    (rest is: + t)
    +---
    | after arg-substitution
    v-----------
    f(x * (0))
    F(x * (0))    <--- mark f for non-replacement
    F(2 * (0))    <--- replace x by 2
```

With this, the CPP moves out of the boundaries of the replacement-list of the
macro-invocation `f(0)`. The corresponding entry within
the `active-macro-stack-#3` is popped off; the stack is now empty.

The `rest` consists of token-sequence `+ t`. That sequence is expanded as is.
The identifier `t` does not match with the name of any active macro,
as evident by the `active-macro-stack-#3`; the stack is empty. It does match
with the name of the macro `t`, but the macro is function-like, and here
there's nothing beyond `t` in the invocation. Hence, the token for the
identifier `t` is allowed replacement during further rescanning.

The output is: `F(2 * (0)) + t`.

The cumulative output until now is:

`F(2 * (y+1)) +  F(2 * (F(2 * (Z[0]+1)))) % F(2 * (0)) + t`.

---

### <ins>Expansion of `t(t(g)(0) + t)` continues</ins>

```
    // active-macro-stack-#0

    t(t(g)(0) + t)    <---- bottom of the stack
    -------------------------------------------
```

The fully expanded argument is substituted into the replacement-list of the
macro `t`:

```
    t(t(g)(0) + t)  (rest is: (1); . . .)
    +-------------
    | after arg-substitution
    v---------------
    F(2 * (0)) + t
```

The CPP now rescans the token-sequences `F(2 * (0)) + t`. The identifier `f`
is already marked for no-replacement. The identifier `t` matches the name of an
active macro, `t` itself, as evident from the state of the
`active-macro-stack-#0`. Hence this token for the identifier `t` is marked for
non-replacement.

```
    t(t(g)(0) + t)  (rest is: (1); . . .)
    +-------------
    | after arg-substitution
    v---------------
    F(2 * (0)) + t
    F(2 * (0)) + T    <--- mark t for non-replacement
```

The CPP now moves out of the boundaries of the replacement-list of the
macro-invocation `t(t(g)(0) + t)`. The corresponding entry within the
`active-macro-stack-#0` is popped off; the stack becomes empty.

The `rest` is `(1);<new-line>g(x+ . . . `. The next macro-invocation is for
the macro `g`.

The cumulative output until now is:
`F(2 * (y+1)) +  F(2 * (F(2 * (Z[0]+1)))) % F(2 * (0)) + T(1);<new-line>`.

---

### <ins>Expansion of `g(x+(3,4)-w)`</ins>

The CPP pushes on to the now empty `active-macro-stack-#0` an entry
corresponding to the invocation `g(x+(3,4)-w)`.

```
    // active-macro-stack-#0

    g(x+(3,4)-w)    <---- bottom of the stack
    -------------------------------------------
```

```
    g(x+(3,4)-w)    (rest is:  | h 5 . . .)
    +-----------
```

After argument-substitution:

```
    g(x+(3,4)-w)    (rest is:  | h 5 . . .)
    +-----------
    | after arg-substitution
    v-------------
    f(2+(3,4)-0,1)
```

The CPP now rescans the token-sequence `f(2+(3,4)-0,1)`. The identifier `f`
does not match with the name of any active macro, as evident by the state of
the `active-macro-stack-#0`. Hence, this token for the identifier `f` is
eligible for replacement.

The token-sequence corresponds to a complete invocation of the macro `f`,
without looking ahead into the `rest` of the source file.

The CPP therefore pushes an entry on to the `active-macro-stack-#0`
corresponding to the invocation `f(2+(3,4)-0,1)`.

```
    // active-macro-stack-#0

    f(2+(3,4)-0,1)    <--- top of the stack
    g(x+(3,4)-w)      <--- bottom of the stack
    -------------------------------------------
```

After argument expansion and substitution:

```
    f(2+(3,4)-0,1)    (rest is: | h 5 . . .)
    +-------------
    | after arg-substitution
    v-------------------
    f(x * (2+(3,4)-0,1))
```

After rescanning:

```
    f(2+(3,4)-0,1)    (rest is: | h 5 . . .)
    +-------------
    | after arg-substitution
    v-------------------
    f(x * (2+(3,4)-0,1))
    F(x * (2+(3,4)-0,1))    <--- mark f for non-replacement
    F(2 * (2+(3,4)-0,1))    <--- replace x by 2
```

With this, the CPP moves out of the boundaries of the replacement-list of the
macro-invocation `f(2+(3,4)-0,1)`. The corresponding entry within the
`active-macro-stack-#0` is popped off.

```
    // active-macro-stack-#0

    g(x+(3,4)-w)      <--- bottom of the stack
    -------------------------------------------
```

But with that, the CPP also moves out of the boundaries of the
replacement-list of the macro-invocation `g(x+(3,4)-w)`.
The corresponding entry on the `active-macro-stack-#0` is popped off;
the stack is now empty.

The cumulative output until now is:

`F(2 * (y+1)) +  F(2 * (F(2 * (Z[0]+1)))) % F(2 * (0)) + T(1);`<br>
`F(2 * (2+(3,4)-0,1))`.

The `rest` begins with `|`. That symbol is output as is. The `rest` is now
`h 5) & m . . .`.

---

### <ins>Expansion of `h`</ins>

In short:

```
    // active-macro-stack-#0

    h    <--- bottom of the stack
    -----------------------------
```

```
    h    (rest is: 5) & m . . .)
    +
    | after substitution
    v------
    g(\~{ }
```

After rescanning, and choosing
[Option 1](http://localhost:1313/wip/post/2023/08/07/cpp-nested-replacement-ambiguity/#option-1):

```
    // active-macro-stack-#0

    g(\~{ } 5)    <--- bottom of the stack
    -----------------------------
```

```
    g(\~{ } 5)    (rest is: & m (f) . . .)
    +---------
    | after arg-substitution
    v---------
    f(\~{ } 5)
```

After rescanning,

```
    // active-macro-stack-#0

    f(\~{ } 5)    <--- top of the stack
    g(\~{ } 5)    <--- bottom of the stack
    -----------------------------
```

```
    g(\~{ } 5)    (rest is: & m (f) . . .)
    +---------
    | after arg-substitution
    v---------
    f(\~{ } 5)
    +---------
    | after arg-substitution
    v---------------
    f(x * (\~{ } 5))
```

After rescanning,

```
    g(\~{ } 5)    (rest is: & m (f) . . .)
    +---------
    | after arg-substitution
    v---------
    f(\~{ } 5)
    +---------
    | after arg-substitution
    v---------------
    f(x * (\~{ } 5))
    F(x * (\~{ } 5))    <--- marking f for non-replacement
    F(2 * (\~{ } 5))    <--- replacing x by 2
```

The CPP moves out of the boundaries of both the macros `f` and `g`.
The `active-macro-stack-#0` becomes empty.

The cumulative output until now is:

`F(2 * (y+1)) +  F(2 * (F(2 * (Z[0]+1)))) % F(2 * (0)) + T(1);`<br>
`F(2 * (2+(3,4)-0,1)) | F(2 * (\~{ } 5))`.

The `rest` consists of `& m (f)^m(m); . . .`.

---

### <ins>Expansion of `m(f)`</ins>

In short:

```
    // active-macro-stack-#0

    m(f)    <--- bottom of the stack
    -----------------------------
```

```
    m(f)    (rest is: ^m(m); . . .)
    +---
    | after arg-substitution
    v---
    f(w)
```

The identifier `f` is allowed replacement. The argument `w` is expanded to
`0,1`.

```
    // active-macro-stack-#0

    f(w)    <--- top of the stack
    m(f)    <--- bottom of the stack
    -----------------------------
```

```
    m(f)    (rest is: ^m(m); . . .)
    +---
    | after arg-substitution
    v---
    f(w)
    +---
    | after arg-substitution
    v-----------
    f(x * (0,1))
```

After rescanning,

```
    m(f)    (rest is: ^m(m); . . .)
    +---
    | after arg-substitution
    v---
    f(w)
    +---
    | after arg-substitution
    v-----------
    f(x * (0,1))
    F(x * (0,1))    <--- mark f for non-replacement
    F(2 * (0,1))    <--- replace x by 2
```

The CPP now moves out of the boundaries of the replacement-list of both the
macro-invocations `f(w)` and `m(f)`. The `active-macro-stack-#0` is now empty.

The cumulative output until now is:

`F(2 * (y+1)) +  F(2 * (F(2 * (Z[0]+1)))) % F(2 * (0)) + T(1);`<br>
`F(2 * (2+(3,4)-0,1)) | F(2 * (\~{ } 5)) & F(2 * (0,1))`.

The `rest` consists of `^m(m); . . .`

---

### <ins>Expansion of `m(m)`</ins>

In short:

```
    // active-macro-stack-#0

    m(m)    <--- bottom of the stack
    -----------------------------
```

```
    m(m)    (rest is: ^m(m); . . .)
    +---
    | after arg-substitution
    v---
    m(w)
    +---
    | after arg-substitution
    v---
    m(0,1)
    M(0,1)    <--- mark m for non-replacement.
```

The argument `w` is expanded to `0,1`. The identifier `m` is marked to prevent
replacement.

The CPP now moves out of the boundaries of the replacement-list of the
macro-invocation `m(m)`. The `active-macro-stack-#0` is now empty.

The cumulative output until now is:

`F(2 * (y+1)) +  F(2 * (F(2 * (Z[0]+1)))) % F(2 * (0)) + T(1);`<br>
`F(2 * (2+(3,4)-0,1)) | F(2 * (\~{ } 5)) & F(2 * (0,1))^M(0,1);`.

The `rest` still has two source lines worth of macro invocations, but they are
simple enough.

---

### <ins>Conclusion</ins>

The argument expansions run on separate active-macro-stacks. Any token that is
marked for non-replacement within the tokens of the argument expansion, retains
its mark, even if substituted into parent-constructs' replacement-lists and
then rescanned.
