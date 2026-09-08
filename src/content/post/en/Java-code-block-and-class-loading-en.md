---

title: "Java Code Blocks, Class Loading, and Initialization Order"
publishDate: 2026-09-08
description: "An introduction to Java initialization blocks, class loading, constructors, and initialization order."
tags: [Java, OOP, JVM, Class Loading]
---

Java has two kinds of initialization blocks: **static initialization blocks** and **instance initialization blocks**.

A static initialization block is executed when the class is initialized, and it runs only once. Creating multiple objects of the same class does not cause the static block to run again.

An instance initialization block, on the other hand, is executed every time an object is created, so it can run multiple times.

When a constructor is called, the instance initialization block runs **before the code inside the constructor**. This also means that overloaded constructors do not change this behavior—the instance initialization block is still executed for every object creation.

```java
public class CodeBlock01 {
    public static void main(String[] args) {

        Movie movie = new Movie("a");
        System.out.println(Movie.id);
        Movie.Exhibition(); // The output shows that the static block runs only once
    }
}

class Movie {
    private String name;
    private String director;
    private int price;

    public static int id = 0;

    static {
        System.out.println("Static initialization block was called");
    }

    public Movie(String name) {
        this.name = name;
    }

    public Movie(String name, String director) {
        this.name = name;
        this.director = director;
    }

    public Movie(String name, String director, int price) {
        this.name = name;
        this.director = director;
        this.price = price;
    }

    public static void Exhibition() {
        System.out.println("The movie starts!");
    }
}
```

So, here comes the question:

**When exactly is a class loaded? What happens during class loading? And what is the exact order of execution?**

Common situations that can trigger class initialization include:

* Creating an instance of the class
* Creating an instance of a subclass, which requires its superclass to be initialized
* Accessing a static member of the class, such as a static field or method
* Using reflection
* Starting the class containing the `main` method

Let's take a look at the initialization order.

When creating an object, the initialization process within a class can be roughly understood as:

1. Static field initialization and static initialization blocks
   Static field initialization and static blocks have the same priority. When there are multiple static fields and static blocks, they are executed in the order in which they appear in the source code.

2. Instance field initialization and instance initialization blocks
   These also have the same priority and are executed in source-code order.

3. The constructor

```java
public class CodeBlock02 {
    public static void main(String[] args) {
        A a = new A();
    }
}

class A {

    public static int n1 = getN1(); // Step 1: call getN1()
    public int n2 = getN2();        // Step 3: call getN2()

    static {
        System.out.println("A static initialization block"); // Step 2
    }

    {
        System.out.println("A instance initialization block"); // Step 4
    }

    public A() {
        System.out.println("Constructor"); // Step 5
    }

    public static int getN1() {
        System.out.println("A getN1"); // Step 1
        return 10;
    }

    public int getN2() {
        System.out.println("A getN2"); // Step 3
        return 10;
    }
}
```

The output is:

```text
A getN1
A static initialization block
A getN2
A instance initialization block
Constructor
```

**Wait, why does it happen in this order?**

Because there is actually an implicit `super()` at the beginning of a constructor.

Instance field initialization and instance initialization blocks belong to the **instance initialization phase**, which is automatically completed before the constructor body begins.

That is why the instance initialization block runs before the constructor body.

Static members, meanwhile, are initialized when the class itself is initialized, so they naturally run before instance initialization and constructor execution.

At this point, you have probably noticed something important:

**`super()`**

That means when a subclass is instantiated, the superclass has to be initialized first.

The overall order becomes:

> Superclass static initialization → subclass static initialization → superclass instance initialization and constructor → subclass instance initialization and constructor

```java
public class CodeBlock03 {
    public static void main(String[] args) {
        BBB b = new BBB();
    }
}

class AAA {

    static {
        System.out.println("AAA static initialization block"); // STEP 1
    }

    {
        System.out.println("AAA instance initialization block"); // STEP 3
    }

    public AAA() {
        System.out.println("AAA constructor"); // STEP 4
    }
}

class BBB extends AAA {

    static {
        System.out.println("BBB static initialization block"); // STEP 2
    }

    {
        System.out.println("BBB instance initialization block"); // STEP 5
    }

    public BBB() {
        System.out.println("BBB constructor"); // STEP 6
    }
}
```

The output is:

```text
AAA static initialization block
BBB static initialization block
AAA instance initialization block
AAA constructor
BBB instance initialization block
BBB constructor
```

Finally, remember:

> A static initialization block can only directly access static members, while an instance initialization block can access both static and instance members.

![Java initialization order](../_assets/code-block.png)
