---
layout: post
date: 2021-06-24
title: "Types of Internal Representation of Kotlin code"
author: akuleshov7
description: |
  Understanding of Internal Representation types generated by Kotlin compiler
keywords:
  - kotlin
  - serialization
---

In this post we will briefly describe different Internal Representations provided by Kotlin compiler.
Kotlin compiler requires several stages to parse and preprocess the code. And the main thing that should be done by compiler to process the code is to build proper intermediate representation.
<!--more-->
Kotlin Compiler builds a simple `Abstract Syntax Tree (AST)` from the code and enriches it to a special extended AST, called `Program Structure Interface` [PSI](https://github.com/JetBrains/kotlin/tree/37813d9d82c5a5ff246dd479dd34f754e44d3305/compiler/psi/src/org/jetbrains/kotlin/psi).
PSI contains functionality capable of working not only with high-level syntactical features, but with particular language specifics.
Put simply, in Kotlin, AST is a high-level interface used to work with nodes of the tree, whereas PSI is a more specific implementation. 

AST nodes are represented with the special interface [ASTNode](https://github.com/JetBrains/intellij-community/blob/master/platform/core-api/src/com/intellij/lang/ASTNode.java). This is an obvious implementation of a syntax tree written in Java and containing familiar methods, such as:
```kotlin
ASTNode getTreeNext();
ASTNode getTreePrev();
ASTNode getTreeParent();
@NotNull
ASTNode[] getChildren(
@Nullable TokenSet var1
);
```

In its turn, AstNode interface is implemented in the forms of Composite and Leaf elements of the Tree. As suggested by their names, they differ in one main characteristic: Leaf elements are nodes which do not have children.

<img src="/static/img/ast.png" width="400em">

Each node has its own type, represented by the common class `IElementType`, that has a special implementation for Kotlin in [KtNodeType](https://github.com/JetBrains/kotlin/blob/37813d9d82c5a5ff246dd479dd34f754e44d3305/compiler/psi/src/org/jetbrains/kotlin/KtNodeType.java). AST can be easily mutated, allowing for lightweight and simple automated code fixing. 

As with Kotlin AST, it is possible to work with a PSI directly.
This is the lowest layer in the compiler, responsible for parsing and internal storage of original source code.
We can treat PSI as a sort of extended AST.
PSI nodes are represented as classes that are implementing `PsiElement`. For example:
```kotlin
"hello" + "world"
```

The above code will be represented with the [KtBinaryExpression](https://github.com/JetBrains/kotlin/blob/37813d9d82c5a5ff246dd479dd34f754e44d3305/compiler/psi/src/org/jetbrains/kotlin/psi/KtBinaryExpression.java) type.
Where `KtBinaryExpression = KtStringTemplateExpression("hello") + KtStringTemplateExpression("world")` with a flag `operationToken` set to PLUS.
As can be seen from the example, these classes are naturally linked to the real original code blocks.
While working with nodes of the code it is always possible to switch to PSI from AST and vice versa, as this information is stored in the node.

As an example you can look at the following code:
```kotlin
if  (a!= null   ) {}
```

It will be represented by the following PSI tree:

<img src="/static/img/ast-example.png" width="400em">

Starting from version 1.4, Kotlin now has a new representation layer - a new Intermediate Representation (IR).
This new intermediate representation (also in a tree format) is designed to serve as a common representation for different compiler backends (JVM, JS or Native),
which are used to create platform specific builds. While reducing the complexity of the compiler infrastructure and splitting language-specific operations from platform-specific,
IR also allows the creation of the common compiler plugin API.
These plugins can help to analyze the code while possessing all the required contexts (such as imported methods, classes, and so on).
Usage of IR and compiler plugins can significantly reduce the complexity of static analysis compared to that using AST or PSI, and is especially important for analysis rules that require type resolution.

Future versions of Kotlin will also utilize a new frontend for IR, called Frontend IR (FIR).
While currently FIR is still in the early stages of development, it will aid in the development of new analysis tools, that are tightly integrated into the compiler infrastructure.

In the end we should also mention the `BindingContext` - it is the extra data collected by Kotlin compiler.
This information contains all the required data relating to inferred types, type nullability, usages, calls, and so on.
For example, using this information we can understand that the variable is nullable and null-checks are redundant.
The following picture illustrates how BindingContext works:

<img src="/static/img/binding-context.png" width="400em">

The New IR introduced in Kotlin 1.4 also contains this information about usages and types.
And completel new FIR (Frontend IR) will extend these ideas and make Internal Representation of Kotlin code more user-friendly.
Let's wait for it!

