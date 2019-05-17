# Chapter 1 
## Modularity Matters

Goal of modularity: **managing and reducing complexity.**

**Modularization** is the act of decomposing a system into self-contained but interconnected modules. Modules are identifiable artifacts containing code, with metadata describing the module and its relation to other modules.

Modules must adhere to three core tenets:
* **Strong encapsulation**. A module must be able to conceal part of its code from other modules.
* **Well-defined interfaces**. Encapsulation is fine, but if modules are to work together, not everything can be encapsulated. Code that is not encapsulated is, by definition, part of the public API of a module.
* **Explicit dependencies**. Modules often need other modules to fulfill their obligations. Such dependencies must be part of the module definition, in order for modules to be self-contained.

## JARs as Modules?

JAR files seem to be the closest we can get to modules pre-Java 9.

## Classpath Hell

The classpath is used by the Java runtime to locate classes. In our example, we run Main, and all classes that are directly or indirectly referenced from this class need to be loaded at some point. You can **view the classpath as a list of all classes that may be loaded at runtime**.

A condensed view of the resulting classpath could looks like this:

    java.lang.Object
    java.lang.String
    ...
    sun.misc.BASE64Encoder
    sun.misc.Unsafe
    ...
    javax.crypto.Cypher
    javax.crypto.SecretKey
    ...
    com.myapp.Main
    ...
    com.google.common.base.Joiner
    ...
    com.google.common.base.internal.Joiner
    org.hibernate.validator.HibernateValidator
    org.hibernate.validator.constraints.NotEmpty
    ...
    org.hibernate.validator.internal.engine.ConfigurationImpl

There’s no notion of JARs or logical grouping anymore. **All classes are sequenced into a flat list**, in the order defined by the -classpath argument. When the JVM loads a class, it reads the classpath in sequential order to find the right one. As soon as the class is found, the search ends and the class is loaded.

**Classes are loaded lazily.**

More insidious problems arise when duplicate classes are on the classpath. Maven resolves dependencies transitively, it’s not uncommon for two versions of the same library (say, Guava 19 and Guava 18) to end up in this set, through no fault of your own. Now both library JARs are flattened into the classpath, in an undefined order. **Whichever version of the library classes comes first is loaded.**

## Java 9 Modules

Modules can either export or strongly encapsulate packages. Furthermore, they express dependencies on other modules explicitly.

The most essential platform module in the modular JDK is **java.base**. It exposes packages such as java.lang and java.util, which no other module can do without. Because you cannot avoid using types from these packages, **every module requires java.base implicitly**.

These are the most important benefits of the Java Platform Module System:

**Reliable configuration**.
The module system checks whether a given combination of modules satisfies all dependencies before compiling or running code. This leads to fewer run-time errors.

**Strong encapsulation**.
Modules explicitly choose what to expose to other modules. Accidental dependencies on internal implementation details are prevented.

**Scalable development**.
Explicit boundaries enable teams to work in parallel while still creating maintainable codebases. Only explicitly exported public types are shared, creating boundaries that are automatically enforced by the module system.

**Security**.
Strong encapsulation is enforced at the deepest layers inside the JVM. This limits the attack surface of the Java runtime. Gaining reflective access to sensitive internal classes is not possible anymore.

**Optimization**.
Because the module system knows which modules belong together, including platform modules, no other code needs to be considered during JVM startup. It also opens up the possibility to create a minimal configuration of modules for distribution.

# Chapter 2
## Modules and the Modular JDK

Prior to the Java module system, the runtime library of the JDK consisted of a hefty rt.jar, weighing in at more than 60 megabytes. It contains most of the runtime classes for Java: the ultimate monolith of the Java platform.

## The Modular JDK

The JDK now consists of about 90 platform modules, instead of a monolithic library. **The Java module system does not allow compile-time circular dependencies between modules**.

## Incubator Modules

All incubator modules have the jdk.incubator prefix. Like a new HttpClient API was shipped in the jdk.incubator.httpclient module.

## Module Descriptors

A module has a name, it groups related code and possibly other resources, and is described by a module descriptor. The module descriptor lives in a file called **module-info.java**.

    module java.prefs {
      requires java.xml;

      exports java.util.prefs;

    }

* The **requires** keyword indicates a dependency, in this case on module java.xml
* A single package from the java.prefs module is **exported** to other modules

Modules live in a global namespace; therefore, module names must be unique. A **module** descriptor always **starts with the module keyword**, followed by the name of the module.

## Is Public Still Public?

Until Java 9, things were quite straightforward. If you had a public class or interface, it could be used by every other class. As of Java 9, public means public only to all other packages inside that module. Only when the package containing the public type is exported can it be used by other modules.

## Implied Readability

    module java.sql {
      requires transitive java.logging;
      requires transitive java.xml;

      exports java.sql;
      exports javax.sql;
      exports javax.transaction.xa;
    }

The **requires** keyword is now followed by the **transitive** modifier. A normal **requires** allows a module to access types in exported packages from the required module only. **requires transitive** means that any module requiring java.sql will now automatically be requiring java.logging and java.xml.

A **nontransitive dependency** means the dependency is necessary to support the internal implementation of that module. **A transitive dependency means the dependency is necessary to support the API of the module.**

## Qualified Exports

In some cases, you’ll want to **expose a package only to certain other modules**. You can do this by using **qualified exports** in the module descriptor.

    module java.xml {
      exports com.sun.xml.internal.stream.writers to java.xml.ws
    }

The exported package is accessible only by the modules specified after to. Multiple module names, separated by a comma, can be provided as targets for a qualified export.
In general, avoid using qualified exports between modules in an application.

## Module Resolution and the Module Path

Modules are resolved from the module path, as opposed to the classpath. Whereas the classpath is a flat list of types (even when using JAR files), the module path contains only modules.

When you want to run an application packaged as a module, you need all of its dependencies as well. Module resolution is the process of computing a minimal required set of modules given a dependency graph and a root module chosen from that graph. Every module reachable from the root module ends up in the set of resolved modules. It contains these steps:

* Start with a single root module and add it to the resolved set.
* Add each required module ( requires or requires transitive in module-info.java) to the resolved set.
* Repeat previous step for each new module added to the resolved set.

### Example

    module app {
        requires java.sql;
    }

Steps of module resolution:
* Add app to the resolved set; observe that it requires java.sql.
* Add java.sql to the resolved set; observe that it requires java.xml and java.logging.
* Add java.xml to the resolved set; observe that it requires nothing else.
* Add java.logging to the resolved set; observe that it requires nothing else.
* No new modules have been added; resolution is complete.

## Using the Modular JDK Without Modules

Java 9 can be used like previous versions of Java, **without moving your code into modules**. Code compiled and loaded outside a module ends up in the unnamed module. In contrast, all modules you’ve seen so far are explicit modules, defining their name in module-info.java. The unnamed module is special: it reads all other modules, including the java.logging module in this case.

# Chapter 3

## Working with Modules

### Example:

    package com.modules.hello;

    public class HelloWorld {

        public static void main(String[] args){
            System.out.println("Hello world");
        }
    }


The layout of the sources on the filesystem looks as follows:

    \---src
        \---helloworld
            |   module-info.java
            |
            \---com
                \---modules
                    \---hello


Compared to the traditional layout of Java source files, there are two major differences:
* There is an extra level of indirection: below src we introduce another directory, helloworld. **The directory is named after the name of the module we’re creating.** 
* Inside this module directory we find both the source file (nested in its package structure as usual) and a module descriptor.

Our Modular Hello World example is quite minimalistic:

    module helloworld {
    }

**The name must match the name of the directory containing the module descriptor.**

## Naming Modules

In Java it’s customary to make package names globally unique by using reverse DNS notation.

## Compilation
































