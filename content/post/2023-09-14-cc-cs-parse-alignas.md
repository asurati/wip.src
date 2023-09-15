---
title: "cc: Context-Sensitive Parsing of alignas"
date: '2023-09-14'
categories:
  - Programming
tags:
  - Compilers
  - C
---

This post demonstrates the ambiguity that arises when parsing the C23
`TypeSpecifier` [`alignas`](https://en.cppreference.com/w/c/language/_Alignas),
and the context-sensitive nature of the parsing required to correctly
parse the specifier.

---

### The ambiguity:

The `TypeSpecifier` is represented by the non-terminal `AlignmentSpecifier`
in the C2x [grammar](https://github.com/asurati/x24/blob/main/grammar.txt):

```
    AlignmentSpecifier: alignas ( TypeName )
    AlignmentSpecifier: alignas ( ConstantExpression )
```

The ambiguity arises from the fact that both `TypeName` and
`ConstantExpression` can derive an `Identifier`.

For e.g., in the declaration

```c
    alignas(a) int b;
```

the variable `b` is declared to be of type `int`, with its address aligned at
the result of the expression `alignas(a)`.

The identifier `a` can be a `TypedefName` (`TypeName` derives `TypedefName`):

```c
    typedef struct {
        long double ld;
    } a;
```

Or, it can be an identifier that represents a `ConstantExpression`
(`ConstantExpression` derives `PrimaryExpression` derives `Identifier`):

```c
    constexpr int a = 32;
```

Or, it can be an identifier that represents a `ConstantExpression`
(`ConstantExpression` derives `PrimaryExpression` derives
 `EnumerationConstant` derives `Identifier`):

```c
    enum {
        a = 32;
    };
```

---

### The ambiguity in an LR(1) parser:

The collection of [canonical-LR(1)-item-sets](/wip/data/lr1.item.sets.txt.gz)
shows the reduce/reduce conflicts as described below.

```
print_item_set: item-set[   0]:k----------------------
[TranslationObject -> . TranslationUnit] jump=1
print_item_set: item-set[   0]:-----------------------
. . .
[AlignmentSpecifier -> . alignas ( TypeName )] jump=4025
[AlignmentSpecifier -> . alignas ( ConstantExpression )] jump=4025
. . .
print_item_set: item-set[   0]:done-------------------
```

```
print_item_set: item-set[4025]:k----------------------
[AlignmentSpecifier -> alignas . ( TypeName )] jump=4026
[AlignmentSpecifier -> alignas . ( ConstantExpression )] jump=4026
print_item_set: item-set[4025]:done-------------------
```

```
print_item_set: item-set[4026]:k----------------------
[AlignmentSpecifier -> alignas ( . TypeName )] jump=4027
[AlignmentSpecifier -> alignas ( . ConstantExpression )] jump=4029
print_item_set: item-set[4026]:-----------------------
. . .
[TypedefName -> . Identifier] jump=983
[PrimaryExpression -> . Identifier] jump=983
[EnumerationConstant -> . Identifier] jump=983
. . .
print_item_set: item-set[4026]:done-------------------
```

```
print_item_set: item-set[ 983]:k----------------------
[TypedefName -> Identifier .] las: ... ) ...
[PrimaryExpression -> Identifier .] las: ... ) ...
[EnumerationConstant -> Identifier .] las: ... ) ...
print_item_set: item-set[ 983]:done-------------------
```

The parser experiences reduce/reduce conflicts, as it lands into the
`item-set-#983` immediately after shifting the identifier `a`. The next
available token in the input, and also the current look-head for the parser, is
the punctuator `)`.

---

### The ambiguity in an Earley parser:

In an Earley parser, the ambiguity manifests as two back-edges pointing to the
completion of the non-terminal `AlignmentSpecifier`, corresponding to the two
different ways the reduction to `AlignmentSpecifier` can be performed.

```
cc_token_print: CC_TOKEN_RIGHT_PAREN ')'
item-set[   4]:r[   5]----------------------
[   0]:[ALIGNMENT_SPECIFIER -> alignas ( TYPE_NAME ) .] (0)
[   1]:[ALIGNMENT_SPECIFIER -> alignas ( CONSTANT_EXPRESSION ) .] (0)
[   2]:[TYPE_SPECIFIER_QUALIFIER -> ALIGNMENT_SPECIFIER(4,0)(4,1) .] (0)
. . .
```

---

### The context-sensitivity:

To determine exactly what the identifier `a` represents, the parser must
consult the symbol-table as soon as it parses the identifier `a`.
This also implies that the parser, as it parses the constructs, must keep the
symbol-table up-to-date, so that relevant information is available to resolve
any ambiguities.
