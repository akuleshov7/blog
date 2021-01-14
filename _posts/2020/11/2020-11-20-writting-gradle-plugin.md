---
layout: post
date: 2020-11-20
title: "Gradle plugin: tips and tricks."
author: petertrr
description: |
  How to create your own gradle plugin. Our experience in diktat project.
image: /static/img/logo-1024.png
keywords:
  - kotlin
  - static analyzers
---


This article is describing our experience and problems that we have faced when we were creating Gradle plugin for our [diktat](https://github.com/cqfn/diKTat) project.
<!--more-->

### Introduction
Gradle is a build automation tool. It originated in Java ecosystem as an alternative to maven, but from the very beginning supported another approach.
While maven has declarative xml-based configuration with a strictly defined lifecycle, gradle uses code-based configuration for creating a *task graph*.
Gradle is very configurable and can be extended by using *plugins*.

Gradle isn't bound only to Java builds. Moreover, due to its plugin-based approach, gradle can be used to build applications in different languages, including 
other JVM languages or even C++ and Swift.

Gradle build is configured using a *build configuration script*: either `build.gradle` written in Groovy or `build.gradle.kts` written in Kotlin. When gradle is
initialized, it calls all declared plugins, which register *tasks*. A task is the main unit of work in gradle, and all tasks can have dependencies on other tasks,
e.g. packaging of jar depends on java files compilation. User calls a specific task when invoking gradle from CLI, like `gradle jar`, and all dependent tasks are
invoked as well.

We are developing [diktat](https://confluence-msc.rnd.huawei.com/display/DKT/diKTat+Readme) - an automatic kotlin code style checker and formatter, which is intended
to be used as CI/CD tool that constantly checks quality of code that developers are adding to their projects. Gradle became popular in Kotlin ecosystem,
so we are releasing a dedicated gradle plugin to run diktat as a gradle task.

### Implementing a plugin
The only essential class for a gradle plugin is an implementation of the interface `Plugin<Project>`. Gradle supports other generic parameters, depending on where
the plugin is applied, but for the majority of cases you want a plugin applied to a project - these are all plugins that are applied in `build.gradle`. When gradle is
initialized, the `apply` method is called, where a plugin can do some useful work having access to a gradle project and register some tasks.

```kotlin
import org.gradle.api.Plugin
import org.gradle.api.Project

class DiktatGradlePlugin : Plugin<Project> {
    /**
     * @param project a gradle [Project] that the plugin is applied to
     */
    override fun apply(project: Project) {
        // inside this method you can interact with project and register tasks
    }
}
```

There are a couple of not mandatory, but useful things that can be added. The one which is really useful in an *extension*.

An extension is a class which fields and methods can be accesses in a build.gradle and look like a special configuration block.
Running
```kotlin
project.extensions.create("diktat", DiktatExtension::class.java)
```
inside plugin's `apply` method allows you to write
```kotlin
diktat {
    // is this scope `this` object is an instance of `DiktatExtension`
    debug = true  // same as `this.debug = true`
}
```

After all necessary configuration is loaded (from project, which includes properties or CLI arguments, and from the extension), the plugin can register tasks
depending on configuration parameters. This is done inside the `apply` method using `project.tasks.register(taskName)` method. Tasks can be configured by passing a
lambda to `register` method, or tasks can be created as separate classes containing the configuration logic.

For us, the main configuration option is `input` - a set of files which will be analyzed by diktat. When `diktatCheck` task is called, diktat plugin 
runs analysis on these files. For interactions with file system gradle has a lot of useful APIs, we are using a FileCollection. If `diktatExtension.inputs` is a 
`FileCollection`, you can specify a set of files, for example, like this inside your `build.gradle.kts`:
```kotlin
diktat {
    inputs = files("src/**/*.kt")
}
```

### Developing gradle plugin inside a maven-based project
We use maven to build our project, but the natural way to develop gradle plugins is from gradle. Otherwise, you'd have to manually fill in all metadata,
and there may be complications in using gradle-specific dependencies in maven. On the other hand, we don't want to have two sets of commands for building
and releasing different parts of the project, so we decided to wrap gradle build inside a maven module. It can be done by `maven-exec-plugin`, which accepts
the path to executable and CLI arguments. Artifacts produced by gradle are then attached to maven project using `build-helper-maven-plugin`.

Of course, there are some complications incorporating gradle inside maven.

* Gradle plugin needs two deployed artifacts. The first one is a regular jar file with special metadata in META-INF, and the second one is a so called 'gradle plugin marker'.
  The plugin marker is an artifact containing only a single pom file with a single dependency, and it is used to associate plugin id with actual artifact coordinates.
  Normally, plugin marker jar is created by `java-gradle-plugin` plugin, but maven requires that single module has single pom.xml. So the plugin marker pom has to be
  added explicitly as a maven module.
  
* Another problem is that maven uses its pom.xml when deploying artifacts, and dependency info from build.gradle is lost. To avoid dependency version
  duplication (dependency declaration duplication can't be avoided easily) we pass dependency versions via CLI gradle properties (e.g. calling gradle with
  `-PdiktatVersion=0.1.4` allows using property like `val diktatVersion by project` inside a build script).

### Last steps
Now you can install a plugin to a local or remote repository and use it in your build! If you are already using diktat or want to try it in gradle, you can add it like this:
```kotlin
plugins {
    id("org.cqfn.diktat.diktat-gradle-plugin") version "0.1.5"
}

diktat {
    inputs = files("src/**/*.kt")
}
```
