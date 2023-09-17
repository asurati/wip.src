---
title: "cc: Decoding Types"
date: '2023-09-17'
categories:
  - Programming
tags:
  - Compilers
  - C
---

This post demonstrates a method to decode the `Declaration`s and
`TypeName`s found in the C language, into a list of types, or a `type-list`.
The method is similar to that
used in converting expressions in infix notation into a corresponding
[postfix](https://en.wikipedia.org/wiki/Reverse_Polish_notation) notation.

---

### Rules of Precedence:<a name="precedence_rules"></a>

The precedence of `*` is lower than that of `[]` or `()`. The latter two have
the same precedence, but only `[]` can occur more than once at the same level
without an intervening pair of parentheses, and they are scanned in a
left-to-right manner (i.e. `[]` is left-associative in relation to itself).

Constructs such as `()[]`, `()()`, or `[]()`, though grammatically allowed,
violate the semantic rules of the language.

- `()[]` denotes a function that returns an array,
- `()()` denotes a function that returns a function, and
- `[]()` denotes an array of functions.

All these three possibilities are semantically prohibited by the C standard.

Note that the punctuator `*` is right-associative in
relation to itself, and that the precedence-ordering punctuator `(` is
also right-associative in relation to itself.

---

### Rules for Extraneous Parentheses:<a name="extra_paren_rules"></a>

A pair of parentheses that is used for precedence-ordering (and therefore is
not considered an enclosure for the parameter-list of a function-type) is
necessary only if both of the following conditions are satisfied:

1. A punctuator `*` follows the opening punctuator `(`.

2. A punctuator `[` or `(` follows the closing punctuator `)`.

If at least one of the conditions is violated, the corresponding pair of
precedence-ordering parentheses is extraneous and therefore not required.

The stack-based method described here doesn't need to perform any special steps
to handle extraneous pairs; the precedence rules and the algorithm handle them
automatically.

---

### Rules for Typedef:<a name="typedef_rules"></a>

Handling `typedef`s is a straightforward splicing of type-lists. The
`DeclarationSpecifiers` will house the `TypedefName` instead of a
builtin-type. The type-list of final target (after recursively resolving
more `TypedefName`s) of the `TypedefName` is appended to the type-list of the
`Declarator` to generate a complete type-list.

---

### Example #1:<a name="example1"></a>

This example is from the section *5.12 Complicated Declarations* in the
2<sup>nd</sup> edition of the text
*The C Programming Language* by Kernighan and Ritchie.

```c
    char (*(*x[3])())[5]
```

The `TypeSpecifier` `char` belongs to `DeclarationSpecifiers`. The rest of the
declaration belongs to `Declarator`. The scanning of the `Declarator` is as
described below.

Create an `operand-stack` that is initially empty. Create an `output-list`,
also initially empty, that will store the `Identifier` and punctuators that
make up the `Declarator`. The `Identifier` of a `Declarator` will also be the
first element in the output-list.

When scanning a `Declarator`, any `(`, found in the input after the
`Identifier` has been moved into the output-list, is considered as
representing the beginning of the declaration of a function-type.
The declarations of the parameters of a function-type are scanned recursively,
in the same manner, on per-parameter private stacks.


The starting state for this example:

```
            input: char (*(*x[3])())[5]
      output-list:
    operand-stack: (Note: The top of the stack is at its right-most end.)
```
---

#### Step #1.01:

Move `char` into `DeclarationSpecifiers`. The rest of the input is a
part of the `Declarator`.

```
            input: (*(*x[3])())[5]
      output-list:
    operand-stack:
```
---

#### Step #1.02:

A punctuator `(` is found next. It is a precedence-ordering left-parenthesis,
and not the one that begins the parameter-list of a function-type. The latter
is found only after the `Identifier` has been moved into the output-list.
Push the punctuator `(` on to the operand-stack.

```
            input: *(*x[3])())[5]
      output-list:
    operand-stack: (
```

---

#### Step #1.03:

A punctuator `*` is found next. Since the top of the operand-stack contains
a precedence-ordering punctuator `(`, the punctuator `*` must be directly
pushed on to the operand-stack.

```
            input: (*x[3])())[5]
      output-list:
    operand-stack: ( *
```

---

#### Step #1.04:

A punctuator `(` is found next. It too is a precedence-ordering punctuator `(`;
therefore, push it on to the operand-stack.

```
            input: *x[3])())[5]
      output-list:
    operand-stack: ( * (
```

---

#### Step #1.05:

A punctuator `*` is found next. Similar process as shown in Step #1.03.

```
            input: x[3])())[5]
      output-list:
    operand-stack: ( * ( *
```

---

#### Step #1.06:

An `Identifier` `x` is found next. Place it at the beginning of the currently
empty output-list. Any punctuator `(` found later will be treated
as the beginning of the parameter-list of a function-type, and not as a
precedence-ordering punctuator `(`.

```
            input: [3])())[5]
      output-list: x
    operand-stack: ( * ( *
```

---

#### Step #1.07:

A punctuator `[` is found next. It's precedence is higher than that of the
punctuator `*` found on the top of the operand-stack.
Since the punctuator `[` begins an array-type, scan the constructs between it
and the corresponding closing punctuator `]`; in this example it is just a
number `3`. Hence, this declares an `array[3]`.

```
            input: )())[5]
      output-list: x array[3]
    operand-stack: ( * ( *
```

---

#### Step #1.08:

A punctuator `)` is found next. This signals that one must pop the
operand-stack and move the elements found on it into the output-list, until a
`(` is popped. Then, both the punctuators, `(` and `)`, are discarded.

```
            input: ())[5]
      output-list: x array[3] *
    operand-stack: ( *
```

---

#### Step #1.09:

A punctuator `(` is found next. Since the `Identifier` of the `Declarator`
has already been moved into the output-list (i.e. the output-list is not empty;
the `Identifier` will always be the first entry on it), the punctuator
`(` found is treated as the beginning of the parameter-list of a function-type.
Scan the parameters recursively, and consume the closing `)`.

```
            input: )[5]
      output-list: x array[3] * func()
    operand-stack: ( *
```

---

#### Step #1.10:

A punctuator `)` is found next. This signals that one must pop the
operand-stack and move the elements found on it into the output-list, until a
`(` is popped. Then, both the punctuators, `(` and `)`, are discarded.

```
            input: [5]
      output-list: x array[3] * func() *
    operand-stack:
```

---

#### Step #1.11:

A punctuator `[` is found next. The operand-stack is empty; there is no need
for precedence comparisons. Since the punctuator `[` begins an array-type,
scan the constructs between it and the corresponding closing punctuator `]`;
in this example it is just a number `5`. Hence, this declares an `array[5]`.

```
            input:
      output-list: x array[3] * func() * array[5]
    operand-stack:
```

---

#### Step #1.12:

The input is now exhausted. The operand-stack too is empty. Thus, the scan is
grammatically correct, though the output-list may have to be semantically
verified for correctness, to flag
[semantically invalid constructs](#precedence_rules).

The given declaration can be described by moving into the output-list from its
left-end to its right-end, and then appending the type (`char` in
this example) collected in the `DeclarationSpecifiers`.

Note that
`DeclarationSpecifiers` also collect `TypeQualifiers` (this example has none),
such as `const`, `restrict`, `volatile`, and `_Atomic`, that further qualifies
the `TypeQualifier` (`char` in this example).

The output-list is:

```
   output-list: x array[3] * func() * array[5]
```

The description is

```
    x is ...
    an array[3] of ...
    a pointer to ...
    a function that takes no params and returns ...
    a pointer to ...
    an array[5] of ...
    a char.
```

---
### Example #2:

This example demonstrates the fact that the method can deal with the extraneous
pair of parentheses.

The rules referred to in the descriptions below are the rules `(1)` and `(2)`
found in the
[Rule for Extraneous Parentheses](#extra_paren_rules) section.

Each pair found extraneous is removed, without changing the meaning of the
`Declaration`. The below steps demonstrate a manual method of removing
extraneous pairs, and arriving at a minimal representation of a type.

```
    int (*((((x)[3]))));
             ^ ^
             | |
             +-+-------- violates rule (1), so not necessary

    int (*(((x[3]))));
```

```
    int (*(((x[3]))));
            ^    ^
            |    |
            +----+---- violates rules (1) and (2), so not necessary

    int (*((x[3])));
```


```
    int (*((x[3])));
           ^    ^
           |    |
           +----+--- violates rules (1) and (2), so not necessary

    int (*(x[3]));
```

```
    int (*(x[3]));
          ^    ^
          |    |
          +----+--- violates rules (1) and (2), so not necessary

    int (*x[3]);
```

```
    int (*x[3]);
        ^     ^
        |     |
        +-----+- violates rule (2), so not necessary

    int *x[3];
```

The `Declaration` `int (*((((x)[3]))));` therefore is:

```
    x is ...
    an array[3] of ...
    a pointer to ...
    an int.
```

---

One can also confirm that the description is correct, by consulting the AST
produced by `clang`:

```bash
    $ # On bash shell

    $ cat 1.c
    $ int (*((((x)[3]))));

    $ clang -E -Xclang -ast-dump -fno-color-diagnostics 1.c
    TranslationUnitDecl 0x56313f5e0938 <<invalid sloc>> <invalid sloc>
    | . . .
    | . . .
    `-VarDecl 0x56313f63f738 <<stdin>:1:1, col:19> col:11 x 'int (*[3])':'int (*[3])'
```

The `TypeName` `int (*[3])` has the same type as that in the `Declaration`
`int (*x[3])`. From the rules about extraneous parentheses, the pair isn't
required, and so the type is the same as `int *[3]`.

For some reason, `clang` doesn't remove the pair. The type shown by `clang`,
although correct, isn't in its minimal possible representation.

As demonstrated below, `gcc` doesn't suffer from this problem, as it displays
the type not in the form of a string, as `clang` does, but in the form of an
hierarchy of the AST nodes that mirrors the type-list that will be built later.

---

One can also confirm that the description is correct, by consulting the AST
produced by `gcc`:

```bash
    $ # On bash shell

    $ cat 1.c
    int main()
    {
            int (*((((x)[3]))));
    }

    $ cc -fdump-tree-original-raw 1.c -c
    $ cat 1.c.005t.original

    @5      var_decl         name: @9       type: @10      scpe: @11
                             srcp: 1.c:3                   size: @12
                             algn: 64       used: 0
    @9      identifier_node  strg: x        lngt: 1

    # The type of the variable "x" is given in the tree-node @10
    @10     array_type       size: @12      algn: 64       elts: @17
                             domn: @18

    # Thus, "x" is of an array-type. The total size of the array is given
    # by the tree-node @12, which is an integer-constant 192 (bits). The
    # total size of the array is therefore 24 bytes.
    @12     integer_cst      type: @21     int: 192

    # Each element of array "x" is of type given by tree-node @17.
    @17     pointer_type     size: @26      algn: 64       ptd : @13

    # Thus, each element of array "x" is a pointer to some type. The size
    # of the pointer is given by the tree-node @26; it is 64 bits = 8 bytes.
    # Thus the array "x" has dimensions 24/8 = 3, or array[3].
    @26     integer_cst      type: @21     int: 64

    # The pointed-to type is given by tree-node @13.
    @13     integer_type     name: @22      size: @23      algn: 32
                             prec: 32       sign: signed   min : @24
                             max : @25
    @22     type_decl        name: @35      type: @13
    @35     identifier_node  strg: int      lngt: 3

    # Thus, "x" is an array[3] of a pointer to an int, or as can be declared,
    # 'int *x[3]'.
```



---

The stack-based method, described in [Example #1](#example1), does not have to
carry out any special steps to handle extraneous parentheses, as
demonstrated below.

The initial state for this example is:

```
            input: int (*((((x)[3]))))
      output-list:
    operand-stack:
```

The `TypeSpecifier` `int` is a part of the `DeclarationSpecifiers` and not
of the `Declarator` that follows. Scanning the `Declarator` as shown below.

```
            input: (*((((x)[3]))))
      output-list:
    operand-stack:
```

```
            input: *((((x)[3]))))
      output-list:
    operand-stack: (
```

```
            input: ((((x)[3]))))
      output-list:
    operand-stack: ( *
```

```
            input: (((x)[3]))))
      output-list:
    operand-stack: ( * (
```

```
            input: ((x)[3]))))
      output-list:
    operand-stack: ( * ( (
```
```
            input: (x)[3]))))
      output-list:
    operand-stack: ( * ( ( (
```

```
            input: x)[3]))))
      output-list:
    operand-stack: ( * ( ( ( (
```

```
            input: )[3]))))
      output-list: x
    operand-stack: ( * ( ( ( (
```

```
            input: [3]))))
      output-list: x
    operand-stack: ( * ( ( (
```

```
            input: ))))
      output-list: x array[3]
    operand-stack: ( * ( ( (
```

```
            input: )))
      output-list: x array[3]
    operand-stack: ( * ( (
```

```
            input: ))
      output-list: x array[3]
    operand-stack: ( * (
```

```
            input: )
      output-list: x array[3]
    operand-stack: ( *
```

```
            input:
      output-list: x array[3] *
    operand-stack:
```

Thus, the description is

```
    x is ...
    an array[3] of ...
    a pointer to ...
    an int.
```

which is the same as that for `int *x[3]`.

---

### Example #3:

This example demonstrate how the precedence-ordering parentheses, that are
not extraneous, are handled by the method.

The initial state for the example is:


```
            input: int (*x)[3]
      output-list:
    operand-stack:
```

The `TypeSpecifier` `int` is a part of the `DeclarationSpecifiers` and not
of the `Declarator` that follows. Scanning the `Declarator` as shown below.


```
            input: (*x)[3]
      output-list:
    operand-stack:
```

```
            input: *x)[3]
      output-list:
    operand-stack: (
```

```
            input: x)[3]
      output-list:
    operand-stack: ( *
```

```
            input: )[3]
      output-list: x
    operand-stack: ( *
```

```
            input: [3]
      output-list: x *
    operand-stack:
```

```
            input:
      output-list: x * array[3]
    operand-stack:
```

Thus, the description is

```
    x is ...
    a pointer to ...
    an array[3] of ...
    an int.
```

---

### Handling TypeNames:

A `TypeName` is similar to a `Declaration` that is devoid of its `Identifier`.

Since an `Identifier` is not present in a `TypeName`, the rules for building a
type-list for it have to be adjusted to handle its absence.

Simply removing an `Identifier` from a `Declaration` doesn't always create
a `TypeName` that corresponds to the type found in that `Declaration`. That is,
`AbstractDeclarator` and `Declarator` are not exactly the same constructs.

---

### Rules for TypeNames:<a name="typename_rules"></a>

An empty-pair of parentheses is always assumed to represent an empty
parameter-list; it never represents a (supposed) `Identifier` that is
surrounded by the pair.

The location of the `Identifier` can be decided by following these rules
for an `AbstractDeclarator`.

1. If an identifier has not yet been found (i.e. the output-list is
   empty), and if an array-type (i.e. `ArrayAbstractDeclarator`) is found,
   then the location of the `Identifier` is to the immediate left of the
   corresponding opening punctuator `[`.
2. If an identifier has not yet been found (i.e. the output-list is
   empty), and if a punctuator ')' is found, then there are two separate
   situations, depending on whether the pair of parentheses is empty or
   not:
    1. If the pair of parentheses is empty, then the location of the
       `Identfier` is to the immediate left of the corresponding opening
       punctuator `(`.
    2. If the pair of parentheses is not empty, then the location of the
       `Identfier` is to the immediate left of the corresponding closing
       punctuator `)`.

---

### Example #4:<a name="example4"></a>

From [Example #3](#example3), the `Declaration` `int (*((((x)[3]))))` is
equivalent to `int *x[3]`.

Removing `x` from the `Declaration` `int *x[3]` results in a `TypeName`
`int *[3]` that is exactly the same type as that of `x`.

The initial state for the example is:

```
            input: int *[3]
      output-list:
    operand-stack:
```

The `TypeSpecifier` `int` is a part of the `SpecifierQualifierList` and not
of the `AbstractDeclarator` that follows. Scanning the `AbstractDeclarator`
as shown below.

```
            input: *[3]
      output-list:
    operand-stack:
```

```
            input: [3]
      output-list:
    operand-stack: *
```

A punctuator `[` is found next in the input. Its precedence is higher than that
of the punctuator `*` found on the top of the operand-stack. Hence, the
processing of the punctuator `[` takes priority of that of `*`.

The output-list is empty; hence, the rule `(1)` of the
[Rules for TypeNames](#typename_rules) applies.

The position of the `Identifier` is shown here in `int *x[3]` through
the use of the name `x`.

An `Identifier` `x` is placed at the start of the currently empty output-list,
as a place-holder.

Since the location of the `Identifier` is now available, the rules now are the
same as they were when scanning a `Declaration`.

```
            input: [3]
      output-list: x
    operand-stack: *
```

```
            input:
      output-list: x array[3]
    operand-stack: *
```

Now empty the operand-stack:

```
            input:
      output-list: x array[3] *
    operand-stack:
```

As can be seen, this `Declaration` is the same as that of a variable that is
`an array[3] of a pointer to an int`.

---

### Example #5:

From [Example #3](#example3), and [Example #4](#example4),
the `Declaration` `int (*((((x)[3]))))` is equivalent to `int *x[3]`.

But, removing `x` from the `Declaration` `int (*((((x)[3]))))` results in a
`TypeName` `int (*(((()[3]))))` that is semantically invalid, and therefore,
not equivalent to `int *x[3]`, as shown below.

The initial state for the example is:

```
            input: int (*(((()[3]))))
      output-list:
    operand-stack:
```

The `TypeSpecifier` `int` is a part of the `SpecifierQualifierList` and not
of the `AbstractDeclarator` that follows. Scanning the `AbstractDeclarator`
as shown below.

```
            input: (*(((()[3]))))
      output-list:
    operand-stack:
```

```
            input: *(((()[3]))))
      output-list:
    operand-stack: (
```

```
            input: (((()[3]))))
      output-list:
    operand-stack: ( *
```

```
            input: ((()[3]))))
      output-list:
    operand-stack: ( * (
```

```
            input: (()[3]))))
      output-list:
    operand-stack: ( * ( (
```

```
            input: ()[3]))))
      output-list:
    operand-stack: ( * ( ( (
```

```
            input: )[3]))))
      output-list:
    operand-stack: ( * ( ( ( (
```

A punctuator `)` is found next in the input.
Before the punctuator is removed from the input-stream, one can see that
the output-list is empty, the corresponding opening punctuator `(` is on the
top of the operand-stack (i.e., this pair of parentheses is empty.)

Hence, the rule `(2.1)` of the [Rules for TypeNames](#typename_rules) applies.

The position of the `Identifier` is shown here in `int (*(((x()[3]))))` through
the use of the name `x`. As can be seen, this `Declaration` is not same as
`int (*((((x)[3]))))`.

The rule says that the position of the `Identifier` is to the immediate left
of the opening punctuator `(`. To apply this rule, pop the punctuator `(` from
the top of the operand-stack, and move it back into the start of the input.
Then, place `x` at the start of the currently empty output-list as a
place-holder for the actual `Identifier`.
The current state then is as shown below:

```
            input: ()[3]))))
      output-list: x
    operand-stack: ( * ( ( (
```

Since the location of the `Identifier` is now available, the rules now are the
same as they were when scanning a `Declaration`.

```
            input: [3]))))
      output-list: x func()
    operand-stack: ( * ( ( (
```

```
            input: ))))
      output-list: x func() array[3]
    operand-stack: ( * ( ( (
```

Here, the partial type can be described as:

```
   x is a function that takes no parameters and returns ...
   an array[3] of ...
```

The fact, that the given `TypeName` turns out (partially) to be of a
function that returns an array, violates one of the semantic rules mentioned in
[Rules of Precedence](#precedence_rules). A function can never return an
array. The scanning of the `TypeName` can stop here, but let us continue,
since `clang`, as shown below, also informs about the type of the array that
the function returns.

```
            input: )))
      output-list: x func() array[3]
    operand-stack: ( * ( (
```

```
            input: ))
      output-list: x func() array[3]
    operand-stack: ( * (
```

```
            input: )
      output-list: x func() array[3]
    operand-stack: ( *
```

```
            input:
      output-list: x func() array[3] *
    operand-stack:
```

The description of the type is:

```
    x is ...
    a function that takes no parameters and returns ...
    an array[3] of ...
    a pointer to ...
    an int
```

The output from `clang`, shown below, notes that the function returns
`an array[3] of a pointer to an int`. The `gcc` compiler decides to not show
the exact type the function returns, perhaps because a function, returning an
array (of *any* legal/illegal type), is in violation of the standard.

---

The outputs from `gcc` and `clang`:

```bash
    $ cat 1.c
    int main(void)
    {
            return sizeof(int (*(((()[3])))));
    }


    $ cc 1.c
    1.c: In function ‘main’:
    1.c:3:9: error: type name declared as function returning an array
        3 |         return sizeof(int (*(((()[3])))));
          |         ^~~~~~


    $ clang 1.c
    1.c:3:25: error: function cannot return array type 'int (*[3])'
        3 |         return sizeof(int (*(((()[3])))));
          |                                ^
    1 error generated.
```

---

### Example #6:

The type of the `Declaration` `char (*(*x[3])())[5]` is same as the `TypeName`
`char (*(*[3])()))[5]` obtained by removing the `Identifier` `x`.

Within the `TypeName`, the location of the `Identifier` is provided by the
application of the rule `(1)` of the [Rules for TypeNames](#typename_rules).
It is the same as the location of `x` in the `Declaration`.

---

### Example #7:

The type of the `Declaration` `int (*x)[3]` is same as the `TypeName`
`int (*)[3]` obtained by removing the `Identifier` `x`.

Within the `TypeName`, the location of the `Identifier` is provided by the
application of the rule `(2.2)` of the [Rules for TypeNames](#typename_rules).
It is the same as the location of `x` in the `Declaration`.

---
