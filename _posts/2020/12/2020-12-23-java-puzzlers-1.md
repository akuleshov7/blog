---
layout: post
date: 2020-12-23
title: "Tricky Java puzzlers (part1)"
author: akuleshov7
description: |
  These Java puzzlers can be found everywhere: on interviews, Java certifications and even in your daily work.
keywords:
  - java
  - puzzlers
---

### Puzzlers: why do you need them?
In a modern world the developer spends 80% of his time on Code Reading. So it is extremely important for him to
read the code from the "whiteboard". Understanding of the code written by some other author can be more important than
to write your own code. That's why on interviews to "good" companies or on Java certification exam you could be asked 
to read simple snippets of code and tell what is the code doing. Solving such puzzlers is also useful for your daily routine as
this helps you to understand the code more quickly and refreshes your knowledge of core language specifics.
And the last but not the least - finally it is fun and trains your brain! So I have collected such puzzlers for you: some of them were created by me,
some of them were taken from interviews, and some were taken from the Java OCA exam. Enjoy it! Read the code below and 
answer what is this code doing. 
<!--more-->

PS there is an outstanding book called "Java Puzzlers: Traps, Pitfalls, and Corner Cases" - I suggest to read it. 

All examples below can be found in [git: akuleshov7](https://github.com/akuleshov7/small-interviews)

### Puzzler #0
Let's start with something easy. All of us know principles of OOP, right? Let's remember how inheritance works in Java.
What will be printed by this code snippet? The answer is easy!
```java
class Parent {
    int a = 0;
    void foo() {
        System.out.println(a);
    }
}

class Child extends Parent {
    int a = 1;
    public static void main(String[] args) { new Child().foo(); }
}
```
 
<details> 
<summary>Answer</summary>
<b>The right answer is: 0.</b> The only trick here is that in Java you are not able to inherit class (or so-called instance) properties.
You can only hide them. So when you call `package-private` method `foo()` it will take the property from same scope where the method is declared.
In our case it is the property from the instance of Parent class.
</details>

### Puzzler #1
Let's move to more complex examples and refresh your knowledge of both Inheritance and Polymorphism in Java.
What will be printed? You can also answer to the side-questions that are asked in the comments for this code. 

```java
interface A {
    default void foo() { System.out.print("A"); } // what is the default modifier used for?
}

class C {
    public void foo() { System.out.print("C"); }
}

class B extends C implements A  {
    public void foo() { System.out.print("B"); }

    public static void main(String[] args) {
        C c = new B();
        c.foo();
        A a = new B();
        a.foo();
    }
}
```
<details> 
  <summary>Answer </summary>
  <b>The right answer is: BB.</b>. This one can confuse you a little bit, because we are creating objects of type `B`, but referencing them with types `C` and `A`.
  Anyway this is like Polymorphism works in Java. Object is stored in one type and can be referenced to with the other type. And all methods by the default in Java are "virtual".
  This means that the overridden method `foo()` that is called for the object will be resolved using the sub-class and the overridden version of this method will be used.  
  for this object  
</details>

### Puzzler #2
Let's imagine that we are a compiler (sometimes it can be useful, especially when you are on white-board interviews). Will this code compile? What will be printed if it compiles?
Child class implements two interfaces. Both of them have methods with the default implementation (imagine that you are a compiler for Java8). This code tests your knowledge of interfaces and type casting.
```java
interface Foo { default int foo() {return 1;} }
interface Bar { default int foo() {return 2;} }
class Child implements Foo, Bar {
    // Will this compile? Both interfaces have this method with implementation.
    public int foo() { return 0; }
    public static void main(String[] args) {
        // Creating a Child instance, but assigning to Object
        Object childObject = new Child();
        // Casting it to the interface Foo
        Foo f = (Foo) childObject;
        // It looks like we are casting one independent interface to another
        Bar b = (Bar) f;
        // What will be printed? Which method is called?
        System.out.print(b.foo());
        System.out.print(f.foo());
    }
}
```
<details> 
  <summary>Answer </summary>
  <b>The right answer is: it compiles and prints 00.</b>. Actually this case was very tricky for me when I got this on my OCA exam. 
  The chain of castings is very confusing here. Let's see how it works. First, there is absolutely no problem that we implement two interfaces with the same default method, because 
  in the sub-class we define the implementation of this method. Second, the casting works fine and even is not throwing aby exceptions. Even when we think that we
  are casting non-related types `Foo` and `Bar`. This happens because Java sees that the instance of class `Child` can be referenced by both `Foo` and `Bar` types as it implements both interfaces.
  And finally, 00 will be printed because the overridden version of method `foo()` is used (otherwise Java would not be able to resolve naming conflict).
</details>

### Puzzler #3
And again we will think as a compiler. Will this code be compiled with Java8+? Also try to answer questions in the comments.

```java
public abstract interface TestInterface {
    public abstract void foo();
    public int a = 0;
}

class Test implements TestInterface {
    public void foo() {}
}

class Main {
    public static void main(String ... args) {
        // there is no constructor Test1() - how will it work?
        Test a = new Test();
        Test b = new Test();
	
       // we are trying to access public property from interface - will it work?
       System.out.println(a.a++); // println1
       System.out.println(a.a++); // println2
    }
}
```

<details> 
  <summary>Answer </summary>
  <b>The right answer is: it will not be compiled.</b>. All these `abstract` keywords are ok for interface (in real life when you do not set them explicitly they are added by the compiler).
   The problem appears on the line println1, because all properties in the interface are static final. And you will not be able to increment final variable.
</details>

### Puzzler #4

```java
/**
 * Imagine that class Parent and class Child stay in different packages. Will this compile?
 * If yes - what will be printed?
 */

package a;
public class Parent {
    protected void foo() {  System.out.println("parent"); }
}

package b;
class Child extends a.Parent {
    protected void foo() { System.out.println("child"); }
    public static void main(String[] args) {
        a.Parent test = new Child();
        test.foo();
    }
}
```

<details> 
  <summary>Answer </summary>
  <b>Suddenly you will get a compiler error: foo() has protected access in a.Parent</b>. This will happen because even when you use `protected`
   keyword you are not able to access this protected method in static methods of a sub-class, if this sub-class stays in the other package.
   Remember that if everything stays in the same package `protected` modifier will work in the same way as default (package-private) modifier.
</details>

### Puzzler #5
Let's remember how references work in Java. Is Java "pass-by-value" or "pass-by-reference" programming language? That will be not the only one trick here.
```java
class A {
    private String value;

    public String getValue() {
        return value;
    }

    public void setValue(String value) {
        this.value = value;
    }

    A(String val) {
        this.value = val;
    }

    public static void main(String[] args) {
        A a = new A("a");
        A b = new A(a.getValue());
        a.setValue("b");

        System.out.println(b.getValue());
    }
}
```

<details> 
  <summary>Answer </summary>
  <b>The right answer is: a. And yes, Java is "pass-by-value" language (don't argue, just google)</b> Even if all objects are used with references, when you pass those objects to a method you will pass a copy of a reference.
  For example, in following method the change to the argument `a` will not be visible outside of the method context: void foo(A a) { a = new A(); }. The second trick here is that String is a special type.
  Strings are immutable objects in Java. So when you change the object `a` - you do not affect the object `b`, because they have different references to string objects.
</details>
