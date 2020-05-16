# Java Interview Questions

[https://docs.oracle.com/javase/tutorial/java/TOC.html](https://docs.oracle.com/javase/tutorial/java/TOC.html)
[https://docs.oracle.com/javase/tutorial/java/concepts/index.html](https://docs.oracle.com/javase/tutorial/java/concepts/index.html)

[https://www.tutorialspoint.com/java/java_interview_questions.htm](https://www.tutorialspoint.com/java/java_interview_questions.htm)

[https://www.javacodegeeks.com/2014/04/java-interview-questions-and-answers.html](https://www.javacodegeeks.com/2014/04/java-interview-questions-and-answers.html)

**Level 1 – General Questions:**

1.  Why is Java a “Platform Independent Programming Language”?  What gives Java its 'write once and run anywhere' nature?
    *   The bytecode.  Java is compiled to be a byte code which is the intermediate language between source code and machine code.
    *   Each Java source file is compiled into a bytecode file, which is executed by the JVM.
    *   The JVM provides the platform specific translation from bytecode to specific platform instructions.
    *   This byte code is not platform specific and hence can be fed to any platform.

2.  What is a class?
    *   A class is a blue print from which individual objects are created.
    *   A class can contain fields and methods to describe the behavior of an object.

3.  What is an interface?
    *   An interface is a contract between a class and the outside world
    *   An interface is a collection of abstract methods.
    *   A class implements an interface, thereby inheriting the abstract methods of the interface.

4.  What is the advantage of using interfaces?
    *   Interfaces are mainly used to provide polymorphic behavior
    *   Implementing an interface allows a class to become more formal about the behavior it promises to provide
    *   Interfaces function to break up the complex designs and clear the dependencies between objects.

5.  Describe 2 major API packages/frameworks in the Java (SE) core library:

    *   IO / NIO
    *   Collections API
    *   Concurrency
    *   UI Frameworks (not relevant to us):  AWT, Swing
    *   [http://www.moreprocess.com/programming/java/list-of-api-packages-in-core-java](http://www.moreprocess.com/programming/java/list-of-api-packages-in-core-java)

6.  Describe the minimum requirements for a simple Java program that can be executed.  How do you run it?
    *   Define a Java class with a “main” method.
    *   Compile with javac (or IDE), run with java/jre (or IDE)

7.  What is the difference between JRE and JDK?
    *   JRE: Java Runtime Environment. It is basically the Java Virtual Machine where your Java programs run on. It also includes browser plugins for Applet execution.
    *   JDK: It's the full featured Software Development Kit for Java, including JRE, and the compilers and tools (like JavaDoc, and Java Debugger) to create and compile programs.

8.  What are “generics”?  And what are they used for?
    *   [http://www.javainterview.in/p/generics-interview-questions.html](http://www.javainterview.in/p/generics-interview-questions.html)
    *   Can developers create their own classes to accept generics?  Yes.   Can you explain how?
    *   Where do you see them used most often?   Collections API

9.  What are “annotations”?   Give example.
    *   Annotations, a form of metadata, provide data about a program that is not part of the program itself.
    *   Uses:
        *   Information for the compiler  (e.g. @Override)
        *   Compile-time and deployment-time processing
        *   Runtime processing --- they can be examined at runtime

10.  What are some IDE's available for Java development?

    *   Eclipse, NetBeans, IntelliJ

**Level 2 – Best Practices, Coding and Design**

1.  Describe some best practices around using the Java Collections framework - for best results, we should ask about specific Collections. i.e. Map, Set, List, etc
    *   “Choosing the right type of the collection to use, based on the application’s needs, is very crucial for its performance”
        For example, don’t choose ArrayList if frequent removal/inserts are done to middle of list
    *   “Some collection classes allow us to specify their initial capacity. Thus, if we have an estimation on the number of elements that will be stored, we can use it to avoid rehashing or resizing”
    *   “Always use Generics for type-safety, readability, and robustness”
    *   “Program in terms of interface not implementation”

2.  In software design, what is meant by separation of concerns?  Give an example to illustrate.
    *   [https://effectivesoftwaredesign.com/2012/02/05/separation-of-concerns/](https://effectivesoftwaredesign.com/2012/02/05/separation-of-concerns/)
        *   The idea that a software system must be decomposed into parts that overlap in functionality as little as possible
    *   [https://en.wikipedia.org/wiki/Separation_of_concerns](https://en.wikipedia.org/wiki/Separation_of_concerns)

3.  Describe 2 design patterns (e.g. from GOF)
    *   Creation:  Singleton, Factory, Abstract Factory, Builder
    *   Structural: Adapter, Composite, Flyweight, Proxy
    *   Behavioral Patterns:  Iterator, Mediator, Observer, Template, Visitor
    *   [http://geekswithblogs.net/subodhnpushpak/archive/2009/09/18/the-23-gang-of-four-design-patterns-.-revisited.aspx](http://geekswithblogs.net/subodhnpushpak/archive/2009/09/18/the-23-gang-of-four-design-patterns-.-revisited.aspx)
