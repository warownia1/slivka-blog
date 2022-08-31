---
layout: post
title: "Testing transpiled code: what lead to org.testj2s"
date: 2022-08-22
---

One of the challenges with transpiling java code to javascript
was verifying its reliability. This is typically done with a set
of unit tests written with your favourite testing framework
such as JUnit or Testng. Those tests can be automatically run by
a deploying software or by a programmer to effortlessly
catch some bugs.
Even though javascript comes with a plethora of testing frameworks,
they are not particularly suited for such obscure use case
that transpiling is. Not only they require re-writing existing tests
in javascript but also manually tapping into arcane machinery of
swingjs.

Goal: we need some kind of framework that can re-use existing Testng tests,
transpile and run them automatically in javascript and, as a bonus,
produce an output report that can be understood by existing testing
software plugins.

## First failure -- no annotations allowed

The first experiment I performed was creating a simple test framework
imitating JUnit that can be transpiled and run in javascript.
The test methods were annotated with a ``@J2STest`` marker annotation
indicating to the ``JSTestRunner`` which methods are to be used
as tests. In order for the test to be started, swingjs needs an
entry point i.e. *main* method called when the website is visited.
Overall, the tests classes would have looked like this:

```java
public class MyTestSuite {
  @J2STest
  public void myTest0() {
    // ...
  }

  @J2STest
  public void myTest1() {
    // ...
  }

  public static void main(String[] args) {
    new JSTestRunner(MyTestSuite.class).start();
  }
}
```

The test code was transpiled to javascript and, when the website
was opened, a new *test runner* was instantiated and started. The role
of the runner was to introspect the content of the provided class,
execute test methods and print a report.

```java
@Retention(RUNTIME)
@Target(METHOD)
public @interface J2STest {}
```


```java
public class JSTestRunner {
  Class<?> testSuite;

  public JSTestRunner(Class<?> testSuite) {
    this.testSuite = testSuite;
  }

  public void start() {
    for (var method: testSuite.getDeclaredMethods()) {
      if (method.isAnnotationPresent(J2STest.class)) {
        try {
          Object context = testSuite.getDeclaredConstructor().newInstance();
          method.invoke(context);
          /** @j2sNative console.info("PASS"); */
        }
        catch (AssertionError ae) {
          /** @j2sNative console.assert(false, ae.detailMessage); */
        }
      }
    }
  }
}
```

There is not much going on in the ``JSTestRunner``, as it was meant
to be minimalistic. The runner checks all the methods of the
provided test class and, if one has ``J2STest`` annotation, it's
executed and outcome is printed to the console. The annotation
does not declare any parameters either and targets methods only.
An important bit is it's retention policy set to *RUNTIME* so that
the annotation is preserved and cen be seen by the runner in runtime.

### All tests passed: 0 tests run

Once the foundation for my "framework" were set, it was time to
write a few test methods and classes to see it in action.
The compilation was successful, the transpiled code showed no signs of
issues either and tests executed properly in JVM.
I proceeded to run them in a browser to find out that no tests are being run.
The reason being, despite having dedicated property for storing annotations, transpiled
methods had no annotations attached to them.

**Issue #1**: swingjs does not support annotations (except some
"essential" ones). A solution, not involving runtime 
annotations, is needed.

### Using test* method names

A honourable mention goes to the test declaration style used in
Python's unittest framework (and also JUnit 3 which it is based on).
Due to the lack of annotations in the early versions of java,
JUnit 3 used special method naming conventions to identify test,
setup and teardown methods. Every test class had to extend
``UnitTest`` type, define setups and teardowns inside ``setUp``
and ``tearDown`` methods and perform tests in methods named
``test*``.

This method of discovering tests works quite well with java2script
as it doesn't rely on annotations. However, it also has very limited
functionality compared to those that modern testing frameworks provide.
Annotations add ability to provide extra details about the test 
such as input data, groups the test belongs to, and its dependencies.
We need to make at least few of those functions available if we want to
replicate the basic functionality of testng in javascript.

## Second attempt -- declarative tests

Even though no annotations, and therefore automatic test pickup, are
available, it should still be possible to provide test methods
to the runner programmatically. I modified the ``JSTestRunner``
so that the test methods are added with ``addTest`` method instead
of being automatically discovered.

```java
public class JSTestRunner {
  Class<?> testSuite;
  List<Method> testMethods = new ArrayList<>();

  public JSTestRunner(Class<?> testSuite) {
    this.testSuite = testSuite;
  }

  public void addTest(Method testMethod) {
    testMethods.add(testMethod);
  }

  public void start() {
    for (var method: testMethods) {
      try {
        Object context = testSuite.getDeclaredConstructor().newInstance();
        method.invoke(context);
        /** @j2sNative console.info("PASS"); */
      }
      catch (AssertionError ae) {
        /** @j2sNative console.assert(false, ae.detailMessage); */
      }
    }
  }
}
```

The test methods must be added programmatically to the runner inside
the ``main`` method body of the test suite class.
For that, we modify it as follows:

```java
public class MyTestSuite {
  // ... omitting declaration of tests

  public static void main(String[] args) {
    var runner = new JSTestRunner(MyTestSuite.class);
    runner.addTest(MyTestSuite.class.getDeclaredMethod("myTest0"));
    runner.addTest(MyTestSuite.class.getDeclaredMethod("myTest1"));
    runner.start();
  }
}
```

The bad news is that declaring test methods is now tedious and repetitive,
and we didn't even got to setups, teardowns and data providers.
Adding those features introduces even more (hopefully avoidable)
boilerplate code. The good news is that it works, which
likely outweighs all inconveniences of boilerplate.

Given the starting point I could follow by adding setup and
teardown methods as well as data providers to the runner increasing
the amount of boilerplate code needed to define the tests.
To put in into perspective, here is a sample of a test that involves
a data provider, setups and teardowns:

```java
runner.addTest(
  newTest(MyTestSuite.class.getDeclaredMethod("myTest0", int.class)
    .withDataProvider(MyTestSuite.class.getDeclaredMethod("dataProvider0"))
    .withSetup(MyTestSuite.class.getDeclaredMethod("setup0"))
    .withSetup(MyTestSuite.class.getDeclaredMethod("setup1"))
    .withTeardown(MyTestSuite.class.getDeclaredMethod("teardown0"))
  )
)
```

Yuck! It would be okay if it was used to build a few objects,
but not tens or hundreds; especially since all that information
is already defined inside the class using Testng annotations.

**Issue #2**: amount of boilerplate and repetition increased
dramatically and need to be reduced. This boilerplate can be
generated automatically from the annotations.

### Source generation to the rescue -- sort of

We already established that no annotations make it through to
the transpiled code. However, they are present during compilation
and can be used to automatically produce the boilerplate
needed to run tests in javascript. The transpiler will then pick
up that source file and transpile it to javascript.
The wolves are fed and the sheep are whole. Coders do not need to
repeat test declarations and write massive boilerplate by hand and
swingjs transpiler gets the code that does not rely on annotations
in runtime.

Generating new source files during compilation is as tricky as it
sounds. It requires diving into Java compiler API and language models
defined inside the ``java.compiler`` module.
The [Annotation Processing tutorial](https://www.baeldung.com/java-annotation-processing-builder)
from Baeldung has proved invaluable for understanding and writing
annotation processors. I made one that uses
existing testng annotations to generate test wrappers that are executed
in javascript just to realise that ... 

**Issue #3**: generated code is not being transpiled to javascript,
thus using compile-time annotations to generate javascript tests is off-limits.
This leaves me with option to use annotations in java code only.

## Selenium to the rescue

### Reporting test outcome

Reporting test outcome is a thing I haven't given much notice thus far.
Currently, the test report is printed to the console as either *PASS*
or assertion error messages. This is acceptable if you start tests
manually and then read the output form the browser console,
but nod good for automation.
The minimum I wanted to achieve was wrapping tests in a testing
framework so that they can be run by build tools i.e. gradle or eclipse
and report failures if any of the tests fail.

Two major candidates that run tests in javascript are
[cypress](https://www.cypress.io/) and [selenium](https://www.selenium.dev/);
the former being an all-in-one testing framework and IDE for
simulating and testing user interactions that runs natively in javascript;
the latter being a browser automation tool, a programmatic access to
a web-driver interface.

After going through their documentations and examples I came to the
following conclusions: the main characteristics of cypress are:

 - good at testing client-server interactions;
 - useful for emulating user interactions with the web page;
 - javascript ecosystem only;
 - comes with big overhead of the entire testing framework;

while selenium's are:

 - good at inspecting and manipulating DOM;
 - embeddable, works with any testing framework, allows to
  execute browser actions from java code/tests;
 - multiplatform -- no language constraint
 - minimalistic -- browser automation driver only;

The choice was on selenium mainly for it's ease of use (the documentation
could have been more extensive though) and integration with java.
The procedure for running tests would take the following steps:

1. Create a class containing test methods and a ``main`` method where
  the ``JSTestRunner`` is declared and started.
2. Transpile this test class to javascript, it creates
  an entry page *your_package_name_ClassName.html*.
3. Create a wrapper class which starts a jetty server serving files in
  the *site* directory and opens the entry page with selenium web driver.
4. Create a wrapper test method that reads the browser console output
  and throws any assertion error encountered in the output.

This naive approach had several problems and ultimately didn't work.
The biggest and most obvious one was that reading the console
output is not as straightforward as I thought. Few browsers
support it via an undocumented experimental API, while others
straight deny any programmatic access to the console.

**Issue #4**: reading console output with selenium is not doable.

The more subtle problem is that since all tests are run in a single "hit"
to the web page, the test method that opens the page can, in turn,
only report whether all tests passed or not.
The external build tool cannot determine the outcome of individual tests
as it only "sees" one test method being run.
Sure, those can be split into multiple test wrapper,
but it requires making separate classes with their own entry points
for every single test.
Now, we would need to repeat test declarations in three different
places and create lots of classes whose only purpose is wrapping
a test in a ``main`` method.

It looked really hopeless, I desperately started browsing web driver
API searching for anything that allows reading the console output
and found something much better: ``executeScript``.

### Execute script

At the first glance, the ability to execute scripts in a browser
from a java code doesn't solve any of the problems I had at that time.
However, it opened a much better opportunity: instead of cramming
test declarations inside a *main* method I could dynamically create
and execute scripts that run tests one at the time. Those scripts can also
return a value -- ideal for returning statuses, exceptions and
stack traces of every individual test. Moreover, the code that
generates scripts does not need to be transpiled, so I can utilise
annotations and processors for generating scripts and the java code starting
those scripts. All the boilerplate we saw earlier, written in java
and transpiled to javascript, can now be generated from annotations
and run with ``executeScript`` method omitting transpilation process
(which doesn't work for generated code).

The next step is to transfer all those ideas into code.
