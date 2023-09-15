---
title: "cc: Generalized LR Parsing"
date: '2023-08-26'
categories:
  - Programming
tags:
  - Compilers
  - C
---

*[Update #1](#update1) on: 28th August, 2023.*

*[Update #2](#update2) on: 13th September, 2023.*

---

This post demonstrates the [GLR](https://en.wikipedia.org/wiki/GLR_parser)
parsing of a tiny C program, with the reader assuming the role of an
[LR(1)](https://en.wikipedia.org/wiki/Canonical_LR_parser)
automaton/finite-state-machine.

The grammar constructs utilized here are adopted from the informal grammar
description in the C2x standard, augmented with a production

`TranslationObject -> TranslationUnit`,

where `TranslationObject` is a
non-terminal that does not occur on the RHS of any production.

The look-aheads (`las`) of an item are either not shown, or shown only
in brevity, as most items contain several tens of look-aheads.

A program
(Source file `lr.c` within the [x24](https://github.com/asurati/x24) project)
generates the
[collection of canonical-LR(1)-item-sets](/wip/data/lr1.item.sets.txt.gz),
each of which is referred to here with its index. Each item-set also represents
the corresponding state in the finite-state-machine.

For convenience, the `lr.c` program treats certain non-terminals, such as
`Identifier`, `StringLiteral`, and `IntegerConstant` as terminals.

---

The sample program:

``` c
int main()
{
    return 0;
}
```

---

### Step #0:

The stack initially contains the item-set #0.

```
    #0
    -------------------------------------------------- bottom of the stack
```

The machine is in the state #0, with no current token. The machine fetches the
next available token in the input-stream, the key-word `int`, as the
current token.

After the fetch, the parse forest is:

```
   int
```

The choice of items in the item-set-#0 is the shift-item

 `[TypeSpecifier -> . int] jump=3998`.

After the shift, the stack is:

```
    #3998
    int
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next available token in the input, and also the current look-ahead for the
machine, is the identifier `main`, known within the grammar as the token
`Identifier`.

The choice of items in the item-set-#3998 is the reduce-item

`[TypeSpecifier -> int .] las: ... Identifier ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `TypeSpecifier`, with the
stack as shown below:

```
    #0
    --------------------------------------------------
```

The parse forest is:

```
     TypeSpecifier
     |
     v
     int
```

---

### Step #1:

The machine is in the state #0, with `TypeSpecifier` as the current token.

The choice of items in the item-set-#0 is the shift-item

`[TypeSpecifierQualifier -> . TypeSpecifier] jump=3990`.

After the shift, the stack is:

```
    #3990
    TypeSpecifier
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next available token in the input, and also the current look-head for the
machine, is still the identifier `main`.

The choice of items in the item-set-#3990 is the reduce-item

`[TypeSpecifierQualifier -> TypeSpecifier .] las: ... Identifier ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `TypeSpecifierQualifier`, with the
stack as shown below:

```
    #0
    --------------------------------------------------
```

The parse forest is:

```
    TypeSpecifierQualifier
    |
    v
    TypeSpecifier
    |
    v
    int
```

---

### Step #2:

The machine is in the state #0, with `TypeSpecifierQualifier`
as the current token.

The choice of items in the item-set-#0 is the shift-item

`[DeclarationSpecifier -> . TypeSpecifierQualifier] jump=3981`.

After the shift, the stack is:

```
    #3981
    TypeSpecifierQualifier
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next available token in the input, and also the current look-head for the
machine, is still the identifier `main`.

The choice of items in the item-set-#3981 is the reduce-item

`[DeclarationSpecifier -> TypeSpecifierQualifier .] las: ... Identifier ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `DeclarationSpecifier`, with the
stack as shown below:

```
    #0
    --------------------------------------------------
```

The parse forest is:

```
    DeclarationSpecifier
    |
    v
    TypeSpecifierQualifier
    |
    v
    TypeSpecifier
    |
    v
    int
```


---

### Step #3:

The machine is in the state #0, with `DeclarationSpecifier` as the
current token.

The choice of items in the item-set-#0 are the shift-items

```
[DeclarationSpecifiers -> . DeclarationSpecifier] jump=3968
[DeclarationSpecifiers -> . DeclarationSpecifier AttributeSpecifierSequence] jump=3968
[DeclarationSpecifiers -> . DeclarationSpecifier DeclarationSpecifiers] jump=3968
```

Since they are concerned with shifting the same token, they all must
(and indeed they do) go to the same state.

After the shift, the stack is:

```
    #3968
    DeclarationSpecifier
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next available token in the input, and also the current look-head for the
machine, is still the identifier `main`.

The choice of items in the item-set-#3968 are these items, one of which is a
shift-item and the other is a reduce-item.

```
[DeclarationSpecifiers -> DeclarationSpecifier .] las: ... Identifier ...
[TypedefName -> . Identifier] jump=4221
```

The machine encounters a shift/reduce conflict. It forks the stack, and copies
the input pointer (or the input itself.)

The processing of the shift-item is described in the steps #3.x below, while
that of the reduce-item is described in the steps #4 and beyond, below.

---

### Step #3.1:

The machine is in the state #3968, with no current token. The machine fetches
the next available token in the input-stream, the identifier `main`,
as the current token.

After the fetch, the parse forest is:

```
    DeclarationSpecifier
    |
    v
    TypeSpecifierQualifier
    |
    v
    TypeSpecifier
    |
    v
    int                     main
```

After the shift, the stack is

```
    #4221
    Identifier(main)
    --------------------------------------------------
    #3968
    DeclarationSpecifier
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next available token in the input, and also the current look-head for the
machine, is the punctuator `(`.

The choice of items in the item-set-#4221 is the reduce-item

`[TypedefName -> Identifier .] las: ... ( ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `TypedefName`, with the
stack as shown below:

```
    #3968
    DeclarationSpecifier
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
    DeclarationSpecifier
    |
    v
    TypeSpecifierQualifier
    |
    v
    TypeSpecifier           TypedefName
    |                       |
    v                       v
    int                     main
```

---

### Step #3.2:

The machine is in the state #3968, with `TypedefName` as the
current token.

The choice of items in the item-set-#3968 is the shift-item

`[TypeSpecifier -> . TypedefName] jump=4016`.

After the shift, the stack is:

```
    #4016
    TypedefName
    --------------------------------------------------
    #3968
    DeclarationSpecifier
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next available token in the input, and also the current look-head for the
machine, is still the punctuator `(`.

The choice of items in the item-set-#4016 is the reduce-item

`[TypeSpecifier -> TypedefName .] las: ... ( ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `TypeSpecifier`, with the
stack as shown below:

```
    #3968
    DeclarationSpecifier
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
    DeclarationSpecifier
    |
    v
    TypeSpecifierQualifier  TypeSpecifier
    |                       |
    v                       v
    TypeSpecifier           TypedefName
    |                       |
    v                       v
    int                     main
```

---

### Step #3.3:

The machine is in the state #3968, with `TypeSpecifier` as the
current token.

The choice of items in the item-set-#3968 is the shift-item

`[TypeSpecifierQualifier -> . TypeSpecifier] jump=3990`.

After the shift, the stack is:

```
    #3990
    TypeSpecifier
    --------------------------------------------------
    #3978
    DeclarationSpecifier
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next available token in the input, and also the current look-head for the
machine, is still the punctuator `(`.

The choice of items in the item-set-#3990 is the reduce-item

`[TypeSpecifierQualifier -> TypeSpecifier .] las: ... ( ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `TypeSpecifierQualifier`, with the
stack as shown below:

```
    #3968
    DeclarationSpecifier
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
    DeclarationSpecifier    TypeSpecifierQualifier
    |                       |
    v                       v
    TypeSpecifierQualifier  TypeSpecifier
    |                       |
    v                       v
    TypeSpecifier           TypedefName
    |                       |
    v                       v
    int                     main
```

---

### Step #3.4:

The machine is in the state #3968, with `TypeSpecifierQualifier` as the
current token.

The choice of items in the item-set-#3968 is the shift-item

`[DeclarationSpecifier -> . TypeSpecifierQualifier] jump=3981`.

After the shift, the stack is:

```
    #3981
    TypeSpecifierQualifier
    --------------------------------------------------
    #3968
    DeclarationSpecifier
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
still the punctuator `(`.

The choice of items in the item-set-#3981 is the reduce-item

`[DeclarationSpecifier -> TypeSpecifierQualifier .] las: ... ( ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `DeclarationSpecifier`, with the
stack as shown below:

```
    #3968
    DeclarationSpecifier
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
                            DeclarationSpecifier
                            |
                            v
    DeclarationSpecifier    TypeSpecifierQualifier
    |                       |
    v                       v
    TypeSpecifierQualifier  TypeSpecifier
    |                       |
    v                       v
    TypeSpecifier           TypedefName
    |                       |
    v                       v
    int                     main
```

---

### Step #3.5:

The machine is in the state #3968, with `DeclarationSpecifier` as the
current token.

The choice of items in the item-set-#3732 are the shift-items

```
[DeclarationSpecifiers -> . DeclarationSpecifier] jump=3968
[DeclarationSpecifiers -> . DeclarationSpecifier AttributeSpecifierSequence] jump=3968
[DeclarationSpecifiers -> . DeclarationSpecifier DeclarationSpecifiers] jump=3968
```

After the shift, the stack is:

```
    #3968
    DeclarationSpecifier
    --------------------------------------------------
    #3968
    DeclarationSpecifier
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
still the punctuator `(`.

The choice of items in the item-set-#3968 is the reduce-item

`[DeclarationSpecifiers -> DeclarationSpecifier .] las: ... ( ...`.

with its look-ahead constraint satisfied.

After the reduction, the current token is `DeclarationSpecifiers`, with the
stack as shown below:

```
    #3968
    DeclarationSpecifier
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
                            DeclarationSpecifiers
                            |
                            v
                            DeclarationSpecifier
                            |
                            v
    DeclarationSpecifier    TypeSpecifierQualifier
    |                       |
    v                       v
    TypeSpecifierQualifier  TypeSpecifier
    |                       |
    v                       v
    TypeSpecifier           TypedefName
    |                       |
    v                       v
    int                     main
```

---

### Step #3.6:

The machine is in the state #3968, with `DeclarationSpecifiers` as the
current token.

The choice of items in the item-set-#3968 is the shift-item

`[DeclarationSpecifiers -> DeclarationSpecifier . DeclarationSpecifiers] jump=3978`

(Notice the right-associativity of `DeclarationSpecifiers`.)

After the shift, the stack is:

```
    #3978
    DeclarationSpecifiers
    --------------------------------------------------
    #3968
    DeclarationSpecifier
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
still the punctuator `(`.

The choice of items in the item-set-#3978 is the reduce-item

`[DeclarationSpecifiers -> DeclarationSpecifier DeclarationSpecifiers .] las: ... ( ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `DeclarationSpecifiers`, with the
stack as shown below:

```
    #0
    --------------------------------------------------
```

The parse forest is:

```
    DeclarationSpecifiers (note the right-associativity)
    |
    +-----------------------+
    |                       |
    |                       v
    |                       DeclarationSpecifiers
    |                       |
    |                       v
    |                       DeclarationSpecifier
    |                       |
    v                       v
    DeclarationSpecifier    TypeSpecifierQualifier
    |                       |
    v                       v
    TypeSpecifierQualifier  TypeSpecifier
    |                       |
    v                       v
    TypeSpecifier           TypedefName
    |                       |
    v                       v
    int                     main
```


---

### Step #3.7:

The machine is in the state #0, with `DeclarationSpecifiers` as the
current token.

The choice of items in the item-set-#0 is the set of these shift-items

```
[FunctionDefinition -> . DeclarationSpecifiers Declarator FunctionBody] jump=5
[Declaration -> . DeclarationSpecifiers ;] jump=5
[Declaration -> . DeclarationSpecifiers InitDeclaratorList ;] jump=5
```

After the shift, the stack is:

```
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next available token in the input, and also the current look-head for the
machine, is still the punctuator `(`.

There is no suitable reduce-item in the item-set-#5.
Hence, the machine attempts a shift.

---

### Step #3.8:

The machine is in the state #5, with no current token. The machine fetches the
next available token in the input-stream, `(`, as the current token.

After the fetch, the parse forest is:

```
    DeclarationSpecifiers (note the right-associativity)
    |
    +-----------------------+
    |                       |
    |                       v
    |                       DeclarationSpecifiers
    |                       |
    |                       v
    |                       DeclarationSpecifier
    |                       |
    v                       v
    DeclarationSpecifier    TypeSpecifierQualifier
    |                       |
    v                       v
    TypeSpecifierQualifier  TypeSpecifier
    |                       |
    v                       v
    TypeSpecifier           TypedefName
    |                       |
    v                       v
    int                     main                    (
```

The choice of items in the item-set-#5 is the shift-item

`[DirectDeclarator -> . ( Declarator )] jump=4747`.

After the shift, the stack is:

```
    #4747
    (
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
now the punctuator `)`.

There is no suitable reduce-item in the item-set-#4747.
Nor is there a shift-item that can shift the token `)`. The machine cannot move
any more.

This causes the termination of the #3.x fork of the step-#3 stack.

---

### Step #4:

(continuing from the end of Step #3, but jumping here instead of going through
 steps #3.x)

After the reduction, the current token is `DeclarationSpecifiers`, with the
stack as shown below:

```
    #0
    --------------------------------------------------
```

The parse forest is:

```
    DeclarationSpecifiers
    |
    v
    DeclarationSpecifier
    |
    v
    TypeSpecifierQualifier
    |
    v
    TypeSpecifier
    |
    v
    int
```

---

### Step #5:

The machine is in the state #0, with `DeclarationSpecifiers` as the
current token.

The choice of items in the item-set-#0 is the set of these shift-items

```
[FunctionDefinition -> . DeclarationSpecifiers Declarator FunctionBody] jump=5
[Declaration -> . DeclarationSpecifiers ;] jump=5
[Declaration -> . DeclarationSpecifiers InitDeclaratorList ;] jump=5
```

After the shift, the stack is:

```
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
still the identifier `main`. (Note that input consumed by fork #3.x has no
effect on the input-stream of the current fork.)

There is no suitable reduce-item in the item-set-#5.
Hence, the machine attempts a shift.

---

### Step #6:

The machine is in the state #5, with no current token. The machine fetches the
next available token in the input-stream, the identifier `main`,
as the current token.

After the fetch, the parse forest is:

```
    DeclarationSpecifiers
    |
    v
    DeclarationSpecifier
    |
    v
    TypeSpecifierQualifier
    |
    v
    TypeSpecifier
    |
    v
    int                     main
```

The choice of items in the item-set-#5 is the set of these shift-items

```
[DirectDeclarator -> . Identifier] jump=4736
[DirectDeclarator -> . Identifier AttributeSpecifierSequence]c$ jump=4736
```

After the shift, the stack is:

```
    #4736
    Identifier(main)
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
now the punctuator `(`.

The choice of items in the item-set-#4736 is the reduce-item

`[DirectDeclarator -> Identifier .] las: ... ( ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `DirectDeclarator`, with the
stack as shown below:

```
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
    DeclarationSpecifiers
    |
    v
    DeclarationSpecifier
    |
    v
    TypeSpecifierQualifier
    |
    v
    TypeSpecifier           DirectDeclarator
    |                       |
    v                       v
    int                     main
```

---

### Step #7:

The machine is in the state #5, with `DirectDeclarator` as the
current token.

The choice of items in the item-set-#5 is the set of these shift-items

```
[Declarator -> . DirectDeclarator] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ TypeQualifierList ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ AssignmentExpression ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ TypeQualifierList AssignmentExpression ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ static AssignmentExpression ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ static TypeQualifierList AssignmentExpression ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ TypeQualifierList static AssignmentExpression ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ * ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ TypeQualifierList * ]] jump=4708
[FunctionDeclarator -> . DirectDeclarator ( )] jump=4708
[FunctionDeclarator -> . DirectDeclarator ( ParameterTypeList )] jump=4708
```

After the shift, the stack is:

```
    #4708
    DirectDeclarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
still the punctuator `(`.

There are no suitable reduce-items in the item-set-#4708.
Hence, the machine attempts a shift.

---

### Step #8:

The machine is in the state #4708, with no current token. The machine fetches
the next available token in the input-stream, `(`, as the current token.

After the fetch, the parse forest is:

```
    DeclarationSpecifiers
    |
    v
    DeclarationSpecifier
    |
    v
    TypeSpecifierQualifier
    |
    v
    TypeSpecifier           DirectDeclarator
    |                       |
    v                       v
    int                     main              (
```

The choice of items in the item-set-#4708 is the set of these shift-items

```
[FunctionDeclarator -> DirectDeclarator . ( )] jump=4730
[FunctionDeclarator -> DirectDeclarator . ( ParameterTypeList )] jump=4730
```

After the shift, the stack is:

```
    #4730
    (
    --------------------------------------------------
    #4708
    DirectDeclarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
now the punctuator `)`.

There is no suitable reduce-item in the item-set-#4730.
Hence, the machine attempts a shift.

---

### Step #9:

The machine is in the state #4730, with no current token. The machine fetches
the next available token in the input-stream, the punctuator `)`.

After the fetch, the parse forest is:

```
    DeclarationSpecifiers
    |
    v
    DeclarationSpecifier
    |
    v
    TypeSpecifierQualifier
    |
    v
    TypeSpecifier           DirectDeclarator
    |                       |
    v                       v
    int                     main              (  )
```

The choice of item in the item-set-#4730 is the shift-item

`[FunctionDeclarator -> DirectDeclarator ( . )] jump=4731`.

After the shift, the stack is:

```
    #4731
    )
    --------------------------------------------------
    #4730
    (
    --------------------------------------------------
    #4708
    DirectDeclarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
now the punctuator `{`.

The choice of items in the item-set-#4731 is the reduce-item

`[FunctionDeclarator -> DirectDeclarator ( ) .] las: ... { ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `FunctionDeclarator`, with the
stack as shown below:

```
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
    DeclarationSpecifiers
    |
    v
    DeclarationSpecifier
    |
    v                       FunctionDeclarator
    TypeSpecifierQualifier  |
    |                       +-----------------+--+
    |                       |                 |  |
    v                       v                 |  |
    TypeSpecifier           DirectDeclarator  |  |
    |                       |                 |  |
    v                       v                 v  v
    int                     main              (  )
```

---

### Step #10:

The machine is in the state #5, with `FunctionDeclarator` as the
current token.

The choice of items in the item-set-#5 is the set of these shift-items

```
[DirectDeclarator -> . FunctionDeclarator] jump=4752
[DirectDeclarator -> . FunctionDeclarator AttributeSpecifierSequence] jump=4752
```

After the shift, the stack is:
```
    #4752
    FunctionDeclarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
still the punctuator `{`.

The choice of items in the item-set-#4752 is the reduce-item

`[DirectDeclarator -> FunctionDeclarator .] las: ... { ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `DirectDeclarator`, with the
stack as shown below:

```
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
    DeclarationSpecifiers
    |
    v                       DirectDeclarator
    DeclarationSpecifier    |
    |                       v
    v                       FunctionDeclarator
    TypeSpecifierQualifier  |
    |                       +-----------------+--+
    |                       |                 |  |
    v                       v                 |  |
    TypeSpecifier           DirectDeclarator  |  |
    |                       |                 |  |
    v                       v                 v  v
    int                     main              (  )
```

---

### Step #11:

The machine is in the state #5, with `DirectDeclarator` as the
current token.

The choice of items in the item-set-#5 is the set of these shift-items

```
[Declarator -> . DirectDeclarator] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ TypeQualifierList ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ AssignmentExpression ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ TypeQualifierList AssignmentExpression ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ static AssignmentExpression ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ static TypeQualifierList AssignmentExpression ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ TypeQualifierList static AssignmentExpression ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ * ]] jump=4708
[ArrayDeclarator -> . DirectDeclarator [ TypeQualifierList * ]] jump=4708
[FunctionDeclarator -> . DirectDeclarator ( )] jump=4708
[FunctionDeclarator -> . DirectDeclarator ( ParameterTypeList )] jump=4708
```

After the shift, the stack is:

```
    #4708
    DirectDeclarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
still the punctuator `{`.

The choice of items in the item-set-#4708 is the reduce-item

`[Declarator -> DirectDeclarator .] las: ... { ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `Declarator`, with the
stack as shown below:

```
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
                            Declarator
    DeclarationSpecifiers   |
    |                       v
    v                       DirectDeclarator
    DeclarationSpecifier    |
    |                       v
    v                       FunctionDeclarator
    TypeSpecifierQualifier  |
    |                       +-----------------+--+
    |                       |                 |  |
    v                       v                 |  |
    TypeSpecifier           DirectDeclarator  |  |
    |                       |                 |  |
    v                       v                 v  v
    int                     main              (  )
```

---

### Step #12:

The machine is in the state #5, with `Declarator` as the
current token.

The choice of items in the item-set-#5 is the set of these shift-items

```
[FunctionDefinition -> DeclarationSpecifiers . Declarator FunctionBody] jump=6
[InitDeclarator -> . Declarator] jump=6
[InitDeclarator -> . Declarator = Initializer] jump=6
```

After the shift, the stack is:
```
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
still the punctuator `{`.

There is no suitable reduce-item in the item-set-#6.
Hence, the machine attempts a shift.

---

### Step #13:

The machine is in the state #6, with no current token. The machine fetches the
next available token in the input-stream, `{`, as the current token.

After the fetch, the parse forest is:

```
                            Declarator
    DeclarationSpecifiers   |
    |                       v
    v                       DirectDeclarator
    DeclarationSpecifier    |
    |                       v
    v                       FunctionDeclarator
    TypeSpecifierQualifier  |
    |                       +-----------------+--+
    |                       |                 |  |
    v                       v                 |  |
    TypeSpecifier           DirectDeclarator  |  |
    |                       |                 |  |
    v                       v                 v  v
    int                     main              (  )  {
```

The choice of items in the item-set-#6 is the set of these shift-items

```
[CompoundStatement -> . { }] jump=3459
[CompoundStatement -> . { BlockItemList }] jump=3459
```

After the shift, the stack is:

```
    #3459
    {
    --------------------------------------------------
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
now the key-word `return`.

There is no suitable reduce-item in the item-set-#3459.
Hence, the machine attempts a shift.

---

### Step #14:

The machine is in the state #3459, with no current token. The machine fetches
the next available token in the input-stream, the key-word `return`, as the
current token.

After the fetch, the parse forest is:

```
                            Declarator
    DeclarationSpecifiers   |
    |                       v
    v                       DirectDeclarator
    DeclarationSpecifier    |
    |                       v
    v                       FunctionDeclarator
    TypeSpecifierQualifier  |
    |                       +-----------------+--+
    |                       |                 |  |
    v                       v                 |  |
    TypeSpecifier           DirectDeclarator  |  |
    |                       |                 |  |
    v                       v                 v  v
    int                     main              (  )  {  return
```

The choice of items in the item-set-#3459 is the set of these shift-items

```
[JumpStatement -> . return ;] jump=3940
[JumpStatement -> . return Expression ;] jump=3940
```

After the shift, the stack is:

```
    #3940
    return
    --------------------------------------------------
    #3459
    {
    --------------------------------------------------
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
now `0`, known to the grammar as an `IntegerConstant`.

There is no suitable reduce-item in the item-set-#3940.
Hence, the machine attempts a shift.

---

### Step #15:

The machine is in the state #3940, with no current token. The machine fetches
the next available token in the input-stream, `0` (or, `IntegerConstant`), as
the current token.

After the fetch, the parse forest is:

```
                            Declarator
    DeclarationSpecifiers   |
    |                       v
    v                       DirectDeclarator
    DeclarationSpecifier    |
    |                       v
    v                       FunctionDeclarator
    TypeSpecifierQualifier  |
    |                       +-----------------+--+
    |                       |                 |  |
    v                       v                 |  |
    TypeSpecifier           DirectDeclarator  |  |
    |                       |                 |  |
    v                       v                 v  v
    int                     main              (  )  {  return  0
```

The choice of items in the item-set-#3940 is the shift-item

`[PrimaryExpression -> . IntegerConstant] jump=3423`.

After the shift, the stack is:

```
    #3423
    IntegerConstant(0)
    --------------------------------------------------
    #3940
    return
    --------------------------------------------------
    #3459
    {
    --------------------------------------------------
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next token in the input, and also the current look-head for the machine, is
now the punctuator `;`.

The choice of items in the item-set-#3423 is the reduce-item

`[PrimaryExpression -> IntegerConstant .] las: ... ; ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `PrimaryExpression`, with the
stack as shown below:

```
    #3940
    return
    --------------------------------------------------
    #3459
    {
    --------------------------------------------------
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
                            Declarator
    DeclarationSpecifiers   |
    |                       v
    v                       DirectDeclarator
    DeclarationSpecifier    |
    |                       v
    v                       FunctionDeclarator
    TypeSpecifierQualifier  |
    |                       +-----------------+--+
    |                       |                 |  |
    v                       v                 |  |
    TypeSpecifier           DirectDeclarator  |  |             PrimaryExpression
    |                       |                 |  |             |
    v                       v                 v  v             v
    int                     main              (  )  {  return  0
```

---

### Step #16:

The machine is in the state #3940, with `PrimaryExpression` as the
current token.

The choice of items in the item-set-#3940 is the shift-item

`[PostfixExpression -> . PrimaryExpression] jump=3420`.

After the shift, the stack is:

```
    #3420
    PrimaryExpression
    --------------------------------------------------
    #3940
    return
    --------------------------------------------------
    #3459
    {
    --------------------------------------------------
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next available token in the input, and also the current look-head for the
machine, is still the punctuator `;`.

The choice of items in the item-set-#3420 is the reduce-item

`[PostfixExpression -> PrimaryExpression .] las: ... ; ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `PostfixExpression`, with the
stack as shown below:

```
    #3940
    return
    --------------------------------------------------
    #3459
    {
    --------------------------------------------------
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
                            Declarator
    DeclarationSpecifiers   |
    |                       v
    v                       DirectDeclarator
    DeclarationSpecifier    |
    |                       v
    v                       FunctionDeclarator
    TypeSpecifierQualifier  |
    |                       +-----------------+--+             PostfixExpression
    |                       |                 |  |             |
    v                       v                 |  |             v
    TypeSpecifier           DirectDeclarator  |  |             PrimaryExpression
    |                       |                 |  |             |
    v                       v                 v  v             v
    int                     main              (  )  {  return  0
```

---

### Step #17:

After some number of similar steps, the machine arrives at the state described
below, with `Expression` as the current token. The next available token in the
input, and also the current look-head for the machine, is still the punctuator
`;`.

The stack is:

```
    #3940
    return
    --------------------------------------------------
    #3459
    {
    --------------------------------------------------
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
             Expression
             |
             v
             AssignmentExpression
             |
             v
             ConditionalExpression
             |
             v
             LogicalOrExpression
             |
             v
             LogicalAndExpression
             |
             v
             InclusiveOrExpression
             |
             v
             ExclusiveOrExpression
             |
             v
             AndExpression
             |
             v
             EqualityExpression
             |
             v
             RelationalExpression
             |
             v
             ShiftExpression
             |
             v
             AdditiveExpression
             |
             v
             MultiplicativeExpression
             |
             v
             CastExpression
             |
             v
             UnaryExpression
             |
             v
             PostfixExpression
             |
             v
             PrimaryExpression
             |
             +-------------------------------------------------+
                                                               |
                            Declarator                         |
    DeclarationSpecifiers   |                                  |
    |                       v                                  |
    v                       DirectDeclarator                   |
    DeclarationSpecifier    |                                  |
    |                       v                                  |
    v                       FunctionDeclarator                 |
    TypeSpecifierQualifier  |                                  |
    |                       +-----------------+--+             |
    |                       |                 |  |             |
    v                       v                 |  |             |
    TypeSpecifier           DirectDeclarator  |  |             |
    |                       |                 |  |             |
    v                       v                 v  v             v
    int                     main              (  )  {  return  0
```

---

### Step #18:

The machine is in the state #3940, with `Expression` as the
current token.

The choice of items in the item-set-#3940 is the set of these shift-items

```
[JumpStatement -> return . Expression ;] jump=3942
[Expression -> . Expression , AssignmentExpression] jump=3942
```

After the shift, the stack is:

```
    #3942
    Expression
    --------------------------------------------------
    #3940
    return
    --------------------------------------------------
    #3459
    {
    --------------------------------------------------
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next available token in the input, and also the current look-head for the
machine, is still the punctuator `;`.

There is no suitable reduce-item in the item-set-#3942.
Hence, the machine attempts a shift.

---

### Step #19:

The machine is in the state #3942, with no current token. The machine fetches
the next available token in the input-stream, the punctuator `;`, as the
current token.

After the fetch, the parse forest is:

```
             Expression
             |
             v
             .
             .
             .
             |
             v
             PrimaryExpression
             |
             +-------------------------------------------------+
                                                               |
                            Declarator                         |
    DeclarationSpecifiers   |                                  |
    |                       v                                  |
    v                       DirectDeclarator                   |
    DeclarationSpecifier    |                                  |
    |                       v                                  |
    v                       FunctionDeclarator                 |
    TypeSpecifierQualifier  |                                  |
    |                       +-----------------+--+             |
    |                       |                 |  |             |
    v                       v                 |  |             |
    TypeSpecifier           DirectDeclarator  |  |             |
    |                       |                 |  |             |
    v                       v                 v  v             v
    int                     main              (  )  {  return  0  ;
```

The choice of items in the item-set-#3942 is the shift-item

`[JumpStatement -> return Expression . ;] jump=3943`.

After the shift, the stack is:

```
    #3943
    ;
    --------------------------------------------------
    #3942
    Expression
    --------------------------------------------------
    #3940
    return
    --------------------------------------------------
    #3459
    {
    --------------------------------------------------
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The next available token in the input, and also the current look-head for the
machine, is now the punctuator `}`.

The choice of items in the item-set-#3943 is the reduce-item

`[JumpStatement -> return Expression ; .] las: ... } ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `JumpStatement`, with the
stack as shown below:

```
    #3459
    {
    --------------------------------------------------
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
          JumpStatement
          |
          +--+------------------+
          |  |                  |
          |  v                  |
          |  Expression         |
          |  |                  |
          |  v                  |
          |  .                  |
          |  .                  |
          |  .                  |
          |  |                  |
          |  v                  |
          |  PrimaryExpression  |
          |  |                  +---------------------------------+
          |  +-------------------------------------------------+  |
          +--------------------------------------------+       |  |
                                                       |       |  |
                            Declarator                 |       |  |
    DeclarationSpecifiers   |                          |       |  |
    |                       v                          |       |  |
    v                       DirectDeclarator           |       |  |
    DeclarationSpecifier    |                          |       |  |
    |                       v                          |       |  |
    v                       FunctionDeclarator         |       |  |
    TypeSpecifierQualifier  |                          |       |  |
    |                       +-----------------+--+     |       |  |
    |                       |                 |  |     |       |  |
    v                       v                 |  |     |       |  |
    TypeSpecifier           DirectDeclarator  |  |     |       |  |
    |                       |                 |  |     |       |  |
    v                       v                 v  v     v       v  v
    int                     main              (  )  {  return  0  ;
```

---


### Step #20:

After consuming the last input token `}`, the machine arrives at the state
described below, with `CompoundStatement` as the current token.
The input has been exhausted; hence the look-ahead for this and the subsequent
steps is always `$`, or `eof`.

The stack is:

```
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
       CompoundStatement
       |
       +--+-------------------------------+
       |  |                               |
       |  v                               |
       |  BlockItemList                   |
       |  |                               |
       |  v                               |
       |  BlockItem                       |
       |  |                               |
       |  v                               |
       |  UnlabeledStatement              |
       |  |                               |
       |  v                               |
       |  JumpStatement                   |
       |  |                               |
       |  +--+-------------------------+  |
       |  |  |                         |  |
       |  |  v                         |  |
       |  |  Expression                |  |
       |  |  |                         |  |
       |  |  v                         |  |
       |  |  .                         |  |
       |  |  .                         |  |
       |  |  .                         |  |
       |  |  |                         |  |
       |  |  v                         |  |
       |  |  PrimaryExpression         |  +--------------------------+
       |  |  |                         +--------------------------+  |
       |  |  +-------------------------------------------------+  |  |
       |  |                                                    |  |  |
       |  +--------------------------------------------+       |  |  |
       +--------------------------------------------+  |       |  |  |
                                                    |  |       |  |  |
                                                    |  |       |  |  |
                                                    |  |       |  |  |
                            Declarator              |  |       |  |  |
    DeclarationSpecifiers   |                       |  |       |  |  |
    |                       v                       |  |       |  |  |
    v                       DirectDeclarator        |  |       |  |  |
    DeclarationSpecifier    |                       |  |       |  |  |
    |                       v                       |  |       |  |  |
    v                       FunctionDeclarator      |  |       |  |  |
    TypeSpecifierQualifier  |                       |  |       |  |  |
    |                       +-----------------+--+  |  |       |  |  |
    |                       |                 |  |  |  |       |  |  |
    v                       v                 |  |  |  |       |  |  |
    TypeSpecifier           DirectDeclarator  |  |  |  |       |  |  |
    |                       |                 |  |  |  |       |  |  |
    v                       v                 v  v  v  v       v  v  v
    int                     main              (  )  {  return  0  ;  }
```

---

### Step #21:

The machine is in the state #6, with `CompoundStatement` as the
current token.

The choice of items in the item-set-#6 is the shift-item

`[FunctionBody -> . CompoundStatement] jump=3458`.

After the shift, the stack is:

```
    #3458
    CompoundStatement
    --------------------------------------------------
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The choice of items in the item-set-#3458 is the reduce-item

`[FunctionBody -> CompoundStatement .] las: ... eof ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `FunctionBody`, with the
stack as shown below:

```
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The parse forest is:

```
       FunctionBody
       |
       v
       CompoundStatement
       |
       +--+-------------------------------+
       |  |                               |
       |  v                               |
       |  BlockItemList                   |
       .  .                               .
       .  .                               .
       .  .                               .
       .  .                               .
```

---

### Step #22:

The machine is in the state #6, with `FunctionBody` as the
current token.

The choice of items in the item-set-#6 is the shift-item

`[FunctionDefinition -> DeclarationSpecifiers Declarator . FunctionBody] jump=7`.

After the shift:

```
    #7
    FunctionBody
    --------------------------------------------------
    #6
    Declarator
    --------------------------------------------------
    #5
    DeclarationSpecifiers
    --------------------------------------------------
    #0
    --------------------------------------------------
```

The choice of items in the item-set-#7 is the reduce-item

`[FunctionDefinition -> DeclarationSpecifiers Declarator FunctionBody .] las: ... eof ...`,

with its look-ahead constraint satisfied.

After the reduction, the current token is `FunctionDefinition`, with the
stack as shown below:

```
    #0
    --------------------------------------------------
```

---

### Step #23:

After a few more steps, the machine reaches its final state, with a successful
parse, with the current token set to `TranslationObject`.

The stack is:

```
    #0
    --------------------------------------------------
```

The (complete) parse tree is:

```
    TranslationObject
    |
    v
    TranslationUnit
    |
    v
    ExternalDeclaration
    |
    v
    FunctionDefinition
    |
    +--+
    |  |
    |  v
    |  FunctionBody
    |  |
    |  v
    |  CompoundStatement
    |  |
    |  +--+-------------------------------+
    |  |  |                               |
    |  |  v                               |
    |  |  BlockItemList                   |
    |  |  |                               |
    |  |  v                               |
    |  |  BlockItem                       |
    |  |  |                               |
    |  |  v                               |
    |  |  UnlabeledStatement              |
    |  |  |                               |
    |  |  v                               |
    |  |  JumpStatement                   |
    |  |  |                               |
    |  |  +--+-------------------------+  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  Expression                |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  AssignmentExpression      |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  ConditionalExpression     |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  LogicalOrExpression       |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  LogicalAndExpression      |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  InclusiveOrExpression     |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  ExclusiveOrExpression     |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  AndExpression             |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  EqualityExpression        |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  RelationalExpression      |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  ShiftExpression           |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  AdditiveExpression        |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  MultiplicativeExpression  |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  CastExpression            |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  UnaryExpression           |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  PostfixExpression         |  |
    |  |  |  |                         |  |
    |  |  |  v                         |  |
    |  |  |  PrimaryExpression         |  +--------------------------+
    |  |  |  |                         +--------------------------+  |
    |  |  |  +-------------------------------------------------+  |  |
    |  |  |                                                    |  |  |
    |  |  +--------------------------------------------+       |  |  |
    |  +--------------------------------------------+  |       |  |  |
    +-----------------------+                       |  |       |  |  |
    |                       |                       |  |       |  |  |
    |                       v                       |  |       |  |  |
    v                       Declarator              |  |       |  |  |
    DeclarationSpecifiers   |                       |  |       |  |  |
    |                       v                       |  |       |  |  |
    v                       DirectDeclarator        |  |       |  |  |
    DeclarationSpecifier    |                       |  |       |  |  |
    |                       v                       |  |       |  |  |
    v                       FunctionDeclarator      |  |       |  |  |
    TypeSpecifierQualifier  |                       |  |       |  |  |
    |                       +-----------------+--+  |  |       |  |  |
    |                       |                 |  |  |  |       |  |  |
    v                       v                 |  |  |  |       |  |  |
    TypeSpecifier           DirectDeclarator  |  |  |  |       |  |  |
    |                       |                 |  |  |  |       |  |  |
    v                       v                 v  v  v  v       v  v  v
    int                     main              (  )  {  return  0  ;  }
```

---

### **Update #1:** <a name="update1"></a>

Regenerated after fixing missing tokens in the
[grammar.txt](https://github.com/asurati/x24)

---

### **Update #2:** <a name="update2"></a>

Regenerated after fixing the first-set for the non-terminals.
