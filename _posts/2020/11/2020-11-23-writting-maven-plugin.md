---
layout: post
date: 2020-11-23
title: "Writting maven plugin using Kotlin."
author: petertrr
description: |
  How to create your own maven plugin when you are using Kotlin.
image: /static/img/logo-1024.png
keywords:
  - kotlin
  - static analyzers
---

If you are a Java developer you must be familiar with [maven](https://maven.apache.org) - a widely used Java build system.
Maven has simple and reliable execution lifecycle: it runs several consecutive phases (like compile, test, package or deploy),
and during each phase a set of plugins can be run, for example javac via maven-compiler-plugin during phase 'compile'.

For a particular plugin one declares a `<plugin>` section in `pom.xml`, and each plugin can have several *executions*. Each 
execution is bound to a particular phase of build lifecycle, and it may include several *goals*. Goal is a particular action
the plugin can execute, for example compile source code or run tests or check code style.

We are developing [diktat](https://confluence-msc.rnd.huawei.com/display/DKT/diKTat+Readme) - an automatic kotlin code style checker
and formatter, which is intended to be used as CI/CD tool that constantly checks quality of code that developers are adding
to their projects. To provide a convenient way to run it for all developers we support run from CLI and are preparing to release
a dedicated maven plugin to run diktat directly from maven.
<!--more-->

###  Designing a plugin
###  MOJO class
A main structural entity in maven plugin is a *MOJO*. MOJO stands for 'Maven POJO' or 'Maven plain old Java object', and it is
a class that defines logic for a particular plugin goal. MOJO is a Java class annotated with `@Mojo` annotation, but since we
are developing kotlin code style checker, we are going to implement our plugin in kotlin too. Kotlin code can be compiled in
Java bytecode, so it won't be a problem.

###  Desired usage
When the plugin is ready, we want to invoke it like this:
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.cqfn.diktat</groupId>
            <artifactId>diktat-maven-plugin</artifactId>
            <version>${diktat.version}</version>
            <executions>
                <execution>
                    <phase>verify</phase>
                    <goals>
                        <goal>check</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```
With this config in `pom.xml` during `mvn verify` the goal `check` will be run.

###  Implementation
Implementation of the plugin seems straightforward:
```kotlin
@Mojo("check")
class DiktatCheckMojo: AbstractMojo() {
    /**
     * Paths that will be scanned for .kt(s) files
     */
    @Parameter(property = "diktat.inputs")
    var inputs = listOf("\${project.basedir}/src")

    // other properties...

    override fun execute() {
        // all logic goes here
    }
}
```

The main point is the `execute` method which will be called during our MOJO invocation. Also, maven lets us capture parameters passed
via configuration in pom.xml by simply annotating properties with `@Parameter`.

###  Sharing common logic among plugin goals
Note, that only classes annotated with `@Mojo` will be considered as mojos, not the classes extending `AbstractMojo`. It means
that we can create a base abstract class extending `AbstractMojo` with common logic and configuration parameters and then
create implementations of that base class for every particular plugin goal that we want.

For example, in `diktat-maven-plugin` we want to support two operation modes: checking code style and fixing it. The following code
will help us achieve this:

```kotlin
class DiktatBaseMojo : AbstractMojo() {
    /**
     * Path to diktat yml config file. Can be either absolute or relative to project's root directory.
     */
    @Parameter(property = "diktat.config", defaultValue = "diktat-analysis.yml")
    lateinit var diktatConfigFile: String

    abstract fun runAction(params: KtLint.Params)

    override fun execute() {
        // common logic
        runAction(params)
    }
}

@Mojo("check")
class DiktatCheckMojo : DiktatBaseMojo() {
    override fun runAction(params: KtLint.Params) {
        // logic of check goal
    }
}

@Mojo("fix")
class DiktatFixMojo : DiktatBaseMojo() {
    override fun runAction(params: KtLint.Params) {
        // logic of fix goal
    }
}
```

###  Instrumentation
What is less straightforward is how to package our code as a maven plugin.

You should create a maven plugin as a separate maven module in (possibly) multi-module maven project.
The first step is to change the module's packaging to a special value:
```xml
    <packaging>maven-plugin</packaging>
``` 

Then we need to add a `maven-plugin-plugin` to `pom.xml` of our plugin module. This plugin will create our plugin's *descriptor* -
a file called `plugin.xml`, which will be included in resulting jar and will store all metadata that maven needs to run our plugin.

`plugin.xml` pretty much acts as a `pom.xml` for a plugin: it contains groupId, artifactId and version of plugin as well as
some special fields, e.g. `goalPrefix` is a string which will be used when calling plugin goals from CLI (i.e. in `diktat:check` `diktat` is a goalPrefix).

This file also contains a list of mojos - xml descriptors for all plugin goals that we created. Mojo has a list of parameters, so that 
IDE can suggest you values when configuring your `pom.xml`, as well as goal and parameter *descriptions* - pieces of documentation
that IDE can show you when selecting plugin parameter.

Here appears the only real caveat of developing maven plugin entirely in kotlin: `maven-plugin-plugin` by default retrieves
descriptions from JavaDocs. Kotlin doesn't have JavaDocs, it has KDocs which are handled differently with special `dokka` tool.

Luckily, `maven-plugin-plugin` uses a set of `extractors` to retrieve descriptions, and extractors can be overridden. Luckily x2, 
there already exists an open-source maven plugin called [kotlin-maven-plugin-tools](https://github.com/gantsign/kotlin-maven-plugin-tools),
which has embedded dokka and provides `kotlin` extractor. After adding this plugin to our build and adding to `maven-plugin-plugin` configuration
```xml
<extractors>
    <!-- Extractor from kotlin-maven-plugin-tools -->
    <extractor>kotlin</extractor>
</extractors>
```
we get a nice `plugin.xml` with goals and properties descriptions.

###  Last steps
Now you can install a plugin to a local or remote maven repository and use it in your build!
