---
layout: post
date: 2020-11-10
title: "Kotlin Linters: which should I choose?"
author: akuleshov7
description: |
  Kotlin Linters: which should I choose?
keywords:
  - kotlin
  - static analyzers
---
<img src="/static/img/logo-1024.png" width="200em">

This article will make a brief comparison of existing **opensource** linters and static analyzers for Kotlin and introduce a new Kotlin linter (checker&fixer) called [diKTat](https://github.com/cqfn/diKTat).

## **Static analyzers for Kotlin**

### Ktlint. 3.8k stars

[Ktlint](https://github.com/pinterest/ktlint) is a popular an anti-bikeshedding Kotlin linter with a built-in formatter created by [pinterest](https://github.com/pinterest).
It tries to capture (reflect) official code style from [kotlinlang.org](https://kotlinlang.org/docs/reference/coding-conventions.html) and [Android Kotlin Style Guide](https://developer.android.com/kotlin/style-guide) and then automatically apply these rules to your codebase.
Ktlint **checks** and can automatically **fix** code and it claims to be simple and easy to use. As it is focused more on checking code-style and code-smell related issues, ktlint inspections are working with Abstract Syntax Tree generated by [Kotlin parser](https://github.com/JetBrains/kotlin/tree/master/compiler).
Ktlint framework has some basic utilities to make the work with Kotlin AST easier, but anyway all inspections work with original [ASTNode](https://github.com/JetBrains/intellij-community/blob/master/platform/core-api/src/com/intellij/lang/ASTNode.java).  

Ktlint has been developing since 2016 and from then on it has 3.8k stars, 309 forks and 390 closed PRs (at least on the moment of writing this article). It looks to be the most popular and **mature** linter in the Kotlin community right now.
There have been written ~15k lines of code.

Ktlint has it’s own set of rules, which divides on standard and experimental rules. But unfortunately the number of fixers&checkers in the standard ruleset is very few (~20 rules) and inspections are very trivial. 

Ktlint can be used as a plugin via `Maven`, `Gradle` or command line app. To configure rules in Ktlint you should modify `.editorconfig` file - this is the only configuration that ktlint provides.
Actually you even **can’t configure specific rules** (for example to disable or suppress any of them), instead you can provide some common settings like the number of spaces for indenting.
In other words, ktlint has a **”fixed hardcoded” codestyle** that is not very configurable. Properties should be specified under `\[\*.kt,kts\]`.

If you want to implement your own rules you need to create a your own ruleset. Ktlint is very user-friendly for creation of custom rulesets.
In this case ktlint will parse the code using a Kotlin parser and will trigger your inspection (as visitor) for each and every node of AST.
Ktlint is using java’s `ServiceLoader` to discover all available ”RuleSets”. `ServiceLoader` is used to inject your own implementation of rules for the static analysis.
In this case ktlint becomes a third-party dependency and a framework.
Basically you should provide implementation of `RuleSetProvider`interface.

Ktlint refers to article on [medium](https://medium.com/@vanniktech/writing-your-first-ktlint-rule-5a1707f4ca5b) on how to create a ruleset and a rule.

**To summarize**: Ktlint is very mature and useful as a framework for creating your own checker&fixer of Kotlin code and doing AST-analysis.
It can be very useful if you need only simple inspections that check (and fix) code-style issues (like indents). 

### Detekt. 3.2k stars

Detekt is a static code analysis tool. It operates on an abstract syntax tree (AST) and meta-information provided by Kotlin compiler.
On then top of that info, it does a complex analysis of the code.
However, this project is more **focused on checking the code** rather than fixing.
Similarly, to ktlint, it has it’s own rules and inspections. 
Detekt uses wrapped ktlint to redefine rules as it’s formatting rules.

Detekt supports such features as code smell analysis, bugs searching and code-style checking. 
It has a highly configurable rule sets (can even make suppression of issues from the code). And the number of checkers is quite big: it has more than **100 inspections**.
Detekt has IntelliJ integration, third-party integrations for `Maven`, `Bazel` and Github actions, mechanism for suppression of their warnings with `@Suppress`annotation and many more. 
It is being developed since 2016 and today it has 3.2k stars, 411 forks and 1850 closed PRs. It has about 45k lines of code. And it's codebase is the biggest comparing to other analyzers.

**To summarize**: Detekt is very useful as a Kotlin static analyser for CI/CD.
It tries to find bugs in the code and is focused more on checking of the code.
Detekt has 100+ rules that check the code.

### Ktfmt. 200 stars

Ktfmt is a program that formats Kotlin code, suddenly based on `google-java-format`.
It’s development has started in Facebook in the end of 2019.
It can be added to your project through a `Maven dependency`, `Gradle dependency`, IntelliJ plugin or you can run it through a command line.
Ktfmt is not a configurable application at all, so to change any rule logic you need to download the project and redefine some constants. 
Ktfmt has 214 stars, 16 forks, 20 closed PRs and around 7500 lines of code.

**To summarize**: no one knows why Facebook has invested their money in this tool. Nothing new was not introduced.

### DiKTat. 150 stars 

Diktat as well as ktlint and detekt is a static code analysis tool.
But diktat is not only a tool, but also a [coding convention](https://github.com/cqfn/diKTat/blob/master/info/guide/diktat-coding-convention.md) that in details describes all the rules that you should follow when writing a code on Kotlin.
It’s development has started in 2020 and at the time of writing this article diKTat has 150 stars and 13 forks. DiKTat operates on AST provided by kotlin compiler.
So why diKTat is better?
 
First of all, it supports much more rules than ktlint. 
It's ruleset includes **more than 100 rules**, that can both check and fix your code.

Second, **diKTat is configurable**. A lot of rules have their own settings, and all of them can be easily understood.
For example, you can choose whether you need a copyright, choose a length of line or you can configure your indents.
 
Third, diKTat is very easy to configure. You don’t need to spend hours only to understand what each rule is doing.
Diktat's ruleset is a `.yml` file, where each rule is commented out with the description.
Also you can suppress error on the particular lines of code using `@Suppress` annotation in your code.
 
DiKTat can be used as a CI/CD tool in order to avoid merging errors in the code. Overall it can find code smells and code style issues.
Also it can find pretty not obvious bugs by complex AST analysis. Diktat works with `maven`, `gradle` and as command-line application powered by `ktlint`.

**To summarize**: diktat contains a strict coding convention that was not yet introduced by other linters. It works both as a checker and as a fixer.
Diktat has much more inspections (100+) and is very configurable (each inspection can be disabled/configured separately), so you can configure it for your particular project.

### A few words about Jetbrains

Jetbrains created one of the best IDEs for Java and Kotlin called IntelliJ. This IDE supports a built-in linter. However, it is not a well-configurable tool, you are not able to specify your own coding convention and it **is not useful for CI/CD** as it is highly coupled with UI. Unfortunately such static analysis is not so effective as it cannot prevent merging of the code with bugs into the repository. As experience shows - many developers simply ignorethose static analysis errors until they are blocked from merging their pull requests. So it is not so suitable for CI/CD, but very good for finding and fixing issues inside your IDE.

## Can you tell me more about this new linter?

Diktat - is a new linter for Kotlin, but is not only a set of inspections that can can be used for **detecting** and **autofixing** code smells in CI/CD process. Diktat also is a strict [**coding standard**](https://github.com/cqfn/diKTat/blob/master/info/guide/diktat-coding-convention.md) for Kotlin that has suggestions for Kotlin developers on how to write good, clear code.

Now diktat was already added to the lists of [static analysis tools](https://github.com/analysis-tools-dev/static-analysis) and to [kotlin-awesome](https://github.com/KotlinBy/awesome-kotlin) and we receive help from the community (testing/PRs), but anyway we need your help (and stars) to make Kotlin code in the world more clear.

**Why should I use diktat in my CI/CD when I have** [ktlint](https://github.com/pinterest/ktlint) **and** [detekt](https://github.com/detekt/detekt)**?**

First of all - actually you can combine diktat with any other static analyzers. And diktat is even using ktlint-framework for parsing the code into the AST.And we are trying to contribute to those projects. But the main features of diktat are the following:

1. it has more inspections. It has 100+ inspections that are tightly coupled with it's codestyle.
2. it has unique inspections that are missing in other linters
3. it is highly configurable: each and every inspection can be configured and suppressed from the code or from the configuration file
4. it has a strict detailed coding convention that you can use in your project

**I don't want to install Diktat, can I try it somehow?**

Yes, you can use online demo that we have created for both diktat and ktlint here: [https://ktlint-demo.herokuapp.com/demo](https://ktlint-demo.herokuapp.com/demo)

<img src="/static/img/demo.png">

You will not even need to install or download diktat.