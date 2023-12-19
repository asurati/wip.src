---
title: "cc: Postfix Conditional Expressions"
date: '2023-12-19'
categories:
  - Programming
tags:
  - Compilers
  - C
---

This post demonstrates the outline of a method to convert a conditional
expression (possibly containing other such expressions within itself) into a
postfix expression. It may be possible to optimize the method further.

The conversion helps with evaluating the `constant-expression` of the `#if`
directive.

An expression such as `a ? b : c` is thought to be ending with a delimiter
`¿` (the upside-down question mark), so that, in its postfix representation,
the delimiter serves as the end-marker for the corresponding conditional
expression.

The operators are `*`, `+`, `?`, `¿` and `:`, while the variables are
lowercase alphabets (excluding `e`, to avoid confusion with `¿`), or constants.

---

### Rules of Precedence:<a name="precedence_rules"></a>
The operators `?` and `¿` have the same precedence and are right-associative
with respect each other. Each of them is also right-associative with respect to
itself. The right associativity allows processing the conditional expressions
from inside-out while maintaining the boundaries for each.

The precedence of the operators, from highest to lowest, is:

```
    *
    +
    ? ¿
    :
```

---

### Example #1:<a name="example1"></a>

The conditional expression is: `a ? b ? c : d + 2 * f : g`. Grouping them using
parentheses, `(a ? (b ? c : d + 2 * f) : g)`.

#### Step #1.00:

Maintain two stacks, `output` and `operator`, both of which are initially empty.

```
       input: a ? b ? c : d + 2 * f : g
      output:
    operator:
```
---

#### Step #1.01:

The element in the front of the input is the variable `a`. Push it on the top
of the `output` stack.

```
       input: ? b ? c : d + 2 * f : g
      output: a
    operator:
```
---

#### Step #1.02:

The next element in the input is the operator `?`. The `operator` is empty;
there is no need for precedence comparison yet. Push the `?` operator on the top
of both the stacks.

```
       input: b ? c : d + 2 * f : g
      output: a ?
    operator: ?
              a <-- marking to identify the controlling condition.
```
---

#### Step #1.03:

The next element in the input is the variable `b`. Push it on the top of the
`output` stack.

```
       input: ? c : d + 2 * f : g
      output: a ? b
    operator: ?
              a
```
---

#### Step #1.04:

The next element in the input is the operator `?`. The top of the `operator`
stack has an operator `?`. Since `?` is considered to be right-associative with
respect to itself, it gets pushed on the `operator` stack, as well as on the
`output` stack.

```
       input: c : d + 2 * f : g
      output: a ? b ?
    operator: ? ?
              a b
```
---

#### Step #1.05:

The next element in the input is the variable `c`. Push it on the top of the
`output` stack.

```
       input: : d + 2 * f : g
      output: a ? b ? c
    operator: ? ?
              a b
```
---

#### Step #1.06:

The next element in the input is the operator `:`. It corresponds to the `?`
operator (marked with `b`) on the top of the `operator` stack. Since the
precedence of `:` is lower than that of `?`, it causes the `?` on the top of the
`operator` stack to *change* to `¿`, marking the beginning of the last clause
of the conditional operator. Additionally, `:` gets pushed over the `output`
stack too.

```
       input: d + 2 * f : g
      output: a ? b ? c :
    operator: ? ¿
              a b
```
---

#### Step #1.07:

The next element in the input is the variable `d`. Push it on the top of the
`output` stack.

```
       input: + 2 * f : g
      output: a ? b ? c : d
    operator: ? ¿
              a b
```
---

#### Step #1.08:

The next element in the input is the operator `+`. Its precedence is higher than
the that of the operator on the top of the `operator` stack, `¿`. Push it on to
the stack.

```
       input: 2 * f : g
      output: a ? b ? c : d
    operator: ? ¿ +
              a b
```
---

#### Step #1.09:

The next element in the input is the constant `2`. Push it on the top of the
`output` stack.

```
       input: * f : g
      output: a ? b ? c : d 2
    operator: ? ¿ +
              a b
```
---

#### Step #1.10:

The next element in the input is the operator `*`. Its precedence is higher than
the that of the operator on the top of the `operator` stack, `+`. Push it on to
the stack.

```
       input: f : g
      output: a ? b ? c : d 2
    operator: ? ¿ + *
              a b
```
---

#### Step #1.11:

The next element in the input is the variable `f`. Push it on the top of the
`output` stack.

```
       input: : g
      output: a ? b ? c : d 2 f
    operator: ? ¿ + *
              a b
```
---

#### Step #1.12:

The next element in the input is the operator `:`.

Its precedence is lower than that of the operator on the top of the `operator`
stack, `*`. Pop `*` and push it on to the `output` stack.

```
       input: : g
      output: a ? b ? c : d 2 f *
    operator: ? ¿ +
              a b
```

The precedence of `:` is lower than that of the operator on the top of the
`operator` stack, `+`. Pop `+` and push it on to the `output` stack.

```
       input: : g
      output: a ? b ? c : d 2 f * +
    operator: ? ¿
              a b
```

The precedence of `:` is lower than that of the operator on the top of the
`operator` stack, `¿`. Pop `¿` and push it on to the `output` stack.

```
       input: : g
      output: a ? b ? c : d 2 f * + ¿
    operator: ?
              a
```

The precedence of `:` is lower than that of the operator on the top of the
`operator` stack, `?`. But, instead of popping it, *change* `?` to `¿`,
and then push `:` on to the `output` stack.


```
       input: g
      output: a ? b ? c : d 2 f * + ¿ :
    operator: ¿
              a
```
---

#### Step #1.13:

The last element in the input is the variable `g`. Push it on the top of the
`output` stack.

```
       input:
      output: a ? b ? c : d 2 f * + ¿ : g
    operator: ¿
              a
```

With the input empty, pop the `operator` stack and push the elements on to the
`output` stack in the order they are popped.

```
       input:
      output: a ? b ? c : d 2 f * + ¿ : g ¿
    operator:
```

The extent of each conditional expression can be seen on the stack, and it
matches with the grouping produced by adding parentheses.

With the help of the markers `?`, `:`, and `¿`, the evaluator can easily
skip/evaluate the sub-expressions, as commanded by their controlling condition.

```
      output: a ? b ? c : d 2 f * + ¿ : g ¿
              ^   ^     ^           ^ ^   ^
              |   |     |           | |   |
              |   +-----+-----------+ |   |
              +-----------------------+---+

              (a? (b? c : d + 2 *  f) :  g)
```
---
