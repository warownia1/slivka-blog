---
layout: post
title: Beginnings of org.testj2s
date: 2022-08-23
---

In the previous post I talked about issues and ideas I encountered
trying to figure out how to test transpiled applications.
Now, it's time to focus on
the implementation of the testing framework.

## Structure of testj2s

The core of the framework consists of two classes: a ``JSTestRunner``
that runs tests in javascript, and a ``JSTestFacade`` that
collects test data and sends it to the former for execution.

A ``JSTestRunner`` class consists of
setters for a test class, a data provider method, setup and teardown
methods and a method under test.
It also provides a dummy ``main(String[] args)`` method that do nothing
and can be used as a java2script entry point for the tests.
The brain of the class is contained in a ``runTest()`` method.
A simplified implementation of this method is shown below:

```java
public TestResult[] runTest() {
  Object[][] dataSets = dataProviderMethod.invoke(null);
  TestResult[] results = new TestResult[dataSets.length];
  for (int i = 0; i < dataSets.length; i++) {
    TestResult result = results[i] = new TestResult();
    Object context = testClass.getDeclaredConstructor().newInstance();
    Object[] parameters = dataSets[i];
    for (Method setup : setupMethods)
      setup.invoke(context);
    try {
      testMethod.invoke(context, parameters);
      result.status = SUCCESS;
    }
    catch (Throwable th) {
      result.status = ERROR;
      result.exception = th;
    }
    for (Method teardown : teardownMethods)
      teardown.invoke(context);
  }
  return results;
}
```

In principle, it first generates input data for the test by invoking
supplied data provider method. The method should be static or make
no references of ``this`` inside their body, similar to how JUnit's
data providers work.
For each set of input parameters, it creates a context object which will
store variables needed by a test, and runs setups on it. After that
it invokes test method with the parameters and stores the outcome.
Note that a new instance of the context object is created for each
test run so all setups are invoked before each test, contrary to
testng, context objects are not reused between tests.
Finally a teardown is called to clean up after the test and the process
is repeated until all input data sets are exhausted.

The definition of the facade class looks very similar, it identical
setters for a test class, a data provider, setups, teardowns and a test method.
The ``runTest()`` method takes a single argument ``JavascriptExecutor``
which is being used to start scripts with that run actual tests instead.
A pseudo-java implementation of this method is shown below:

```java
public List<TestResult> runTest(JavascriptExecutor driver) {
  var jsonResults = driver.executeScript(
    script,  // javascript to execute
    testClass.getName(), 
    dataProviderMethod.getName(),
    setupMethods.map(m -> m.getName()),
    teardownMethods.map(m -> m.getName()),
    testMethod.getName(),
    testMethod.getParameterTypes().map(t -> t.getName())
  )
  List<TestResult> results = new ArrayList<>();
  for (var json : jsonResults) {
    var result = new TestResult();
    result.status = json.get("status");
    result.exception = parseException(json.get("exception"));
    results.add(result);
  }
  return results;
}
```

This class is accompanied by a script file that contains
code needed to setup and run the ``JSTestRunner`` in javascript.
The facade object passes parameters to the script using additional
arguments to ``executeScript`` following the script itself.
The script, shown below, loads those arguments as strings and
uses java2script machinery to convert them back to the original types.

```javascript
var JSTestRunner = Clazz.load("org.testj2s.JSTestRunner");
var JsonUtils = Clazz.load("org.testj2s.JsonUtils");

// load arguments in the same order as given to executeScript
var testClassName = arguments[0];
var dataProviderName = arguments[1];
var setupNames = arguments[2] || [];
var teardownNames = arguments[3] || [];
var testName = arguments[4];
var testArgTypeNames = arguments[5];

var testClazz = Clazz.forName(testClassName);
var getTestMethod = testClazz.getDeclaredMethod$S$ClassA;

var suite = Clazz.new_(JSTestRunner.c$, []);
suite.setTestClass$Class(testClazz);
suite.setDataProvider$java_lang_reflect_Method(
  getTestMethod(dataProviderName, [])
);
setupNames.forEach(name => {
  suite.addSetup$java_lang_reflect_Method(getTestMethod(name, []));
});
teardownNames.forEach(name => {
  suite.addTeardown$java_lang_reflect_Method(getTestMethod(name, []));
});
suite.setTest$java_lang_reflect_Method(
  getTestMethod(testName, testArgTypeNames.map(JsonUtils.parseType$S))
);

return suite.runTest$().map(r => {
  return {
    "status": r.status,
    "exceptions": JsonUtils.dumpException$Throwable(r.exception)
  };
});
```

This code may look intimidating, but it is basically using java
reflections to retrieve methods for their names and pass them to
the ``JSTestRunner`` object.
Once the tests are run, the results are returned back to the
code that executed the script becoming a return value of the
``executeScript`` method.

## Writing testj2s tests

In order to use the framework we need to define actual tests
that test our code and facades that run them in javascript.
This procedure can be automated with annotation processor
so that no extra code has to be written by the users.

In my examples I use [jetty](https://www.eclipse.org/jetty/) to
serve transpiled files and [testng](https://testng.org/doc/) as
a target testing framework. However, users can use any frameworks
of their choice.
Consider following two test classes that we want to run in javascript,
one having a test method relying on setups and teardowns, the other
having a test method that uses data sourced from other method.
I'll mark these methods using testng annotations, although the
annotations are not needed and are, in fact, erased by java2script
during transpilation.

{% raw %}
```java
public class TestCase1 {
  @BeforeMethod
  public void setUp() { /* prepare testing environment */ }

  @AfterMethod
  public void tearDown() { /* tidy up after tests */ }

  @Test
  public void test0() { /* run test */ }
}

public class TestCase2 {
  @DataProvider
  public Object[][] testData() { 
    /* generate input data */ 
    return new Object[][] {{1}, {2}, {3}}
  }

  @Test
  public void test1(int arg0) { /* run test */ }
}
```
{% endraw %}

Parallel to those tests, we need to create facades that can be
run by build tools, thus they need to be actual testng or JUnit
tests. The facades set up a server and a selenium driver and
delegate test execution to their javascript counterparts. We can move
the setup methods to an abstract base class and extend from that
to reduce boilerplate.

```java
public class AbstractTestFacade {
  protected static RemoteWebDriver driver;
  protected static Server server;
  protected static final String mainPage = "org_testj2s_JSTestRunner.html";

  @BeforeTest(alwaysRun = true)
  public void setupJetty() throws Exception {
    server = new Server(0);
    ResourceHandler handler = new ResourceHandler();
    handler.setResourceBase("site");
    handler.setDirectoriesListed(true);
    server.setHandler(handler);
    server.start();
  }

  @AfterTest
  public void teardownJetty() throws Exception {
    server.stop();
    server = null;
  }

  @BeforeTest(alwaysRun = true, dependsOnMethods = "setupJetty")
  public void setupDriver() throws Exception {
    // you may change to other driver installed on your system
    driver = new FirefoxDriver();
    driver.get(server.getURI().toString() + mainPage);
    Thread.sleep(1000); // wait for java2script to load
  }

  @AfterTest
  public void teardownDriver() {
    driver.quit();
    driver = null;
  }
}
```

The common setup methods defined in this base class are responsible
for serving transpiled files form *site* directory and starting up
a remote browser session pointing to an empty test page.
These methods are run before all of the facade tests start.

Moving on to the facades, we create classes with matching signatures
to the test classes (this convention is not exactly required,
but helps identifying which methods are being run).

```java
public class TestCase1_Facade extends AbstractTestFacade {
  @Test
  public void test0() {
    Class<?> testClass = TestCase1.class;
    var facade = new JSTestFacade(testClass);
    facade.addSetup(testClass.getMethod("setUp"));
    facade.addTeardown(testClass.getMethod("tearDown"));
    facade.setTest(testClass.getMethod("test0"));
    var results = facade.runTest(driver);
    // process results e.g. rethrow exceptions
    for (var result : results ) {
      if (!result.isSuccess())
        throw result.getException();
    }
  }
}

public class TestCase2_Facade extends AbstractTestFacade {
  @Test
  public void test1() {
    Class<?> testClass = TestCase2.class;
    var facade = new JSTestFacade(testClass);
    facade.setDataProvider(testClass.getMethod("testData"));
    facade.setTest(testClass.getMethod("test1", int.class));
    var results = facade.runTest(driver);
    // process results e.g. rethrow exceptions
    for (var result : results ) {
      if (!result.isSuccess())
        throw result.getException();
    }
  }
}
```

Done! You can now run these facades with your favourite build tool.
You'll see a browser window pop up for a moment
when the tests are run in the browser. The outcome of the
facade tests should reflect the outcome of the underlying tests
run in javascript. There is plenty of boilerplate happening here
which I'll take care of later.

## Multiple tests packaged as one

For now, let's focus on a different problem. As you may have noticed,
the ``runTest`` method doesn't just run a single test, it runs the test
method once for every set of parameters provided by the data provider.
This usually is one if the test takes no input data, but in every
other scenario one call to ``runTest`` executes multiple tests under
a single, non-parametrised facade test.
Even though the implementation above runs ``test1`` three times
(once for every input) only the first failure is reported and the
external tools have no way of telling which one of the multiple runs
failed.

For that, the facade test must also be run multiple times, once
for each time the underlying test is run. A naive approach would
be to run a data provider on the java end and pass the produced
parameters to the script. However, the parameters can be complex
objects and only maps/json objects are allowed to go
across the java-javascript boundary. It's advisable to construct
objects that participate in tests in the same environment
which those tests are run.

A viable alternative is to instantiate input parameters on both
java and js ends simultaneously, assuming that the data providers
work the same in both java and js.
This solution would produce particularly
nice output reports, because the facade tests would be provided
with actual parameters that can be included in the final test report.
This approach poses certain difficulties, however, that I discuss
in the following section.

I propose the following criteria for facade tests that ensure
readable and informative output reports:

- facade class name should include the original test class name, identical
  names are not allowed in the same package, but ``<test class>_Facade``
  would clearly indicate which test suite is being run;
- facade method names should match those of the original tests, there is no
  reason to name them differently;
- facade method parameters should match those of the underlying test,
  this ensures matching signatures in the report and no conflict
  when (however unlikely) a test method is overloaded;
- a facade method should be run once for every time an actual
  test method is run so that the report contains the correct number
  of tests being run;
- facade methods should be provided the same parameters as the underlying
  test, even if those parameters won't be used, this ensures they
  are present in the final report.

### Running tests in data providers

Sticking to the current design of the framework, we are confined to
running all parametrised variants of the test within a single call
to ``executeScript``. That means that the script must be executed
before the facade test methods are executed and the number of tests
is determined by the size of the results list returned by the script.
A simple way to execute this idea is to place the script execution
inside a data provider and return each test result as an input
parameter to the facade test.

```java
public class TestCase2_Facade extends AbstractTestFacade {
  @DataProvider
  public Object[][] test1_dataProvider() {
    Class<?> testClass = TestCase2.class;
    var facade = new JSTestFacade(testClass);
    facade.setDataProvider(testClass.getMethod("testData"));
    facade.setTest(testClass.getMethod("test1", int.class));
    return facade.runTest(driver).stream()
        .map(r -> new Object[]{r}).toArray(Object[][]::new);
  }

  @Test(dataProvider = "test1_dataProvider")
  public void test1(TestResult result) throws Throwable {
    if (!result.isSuccess())
      throw result.getException();
  }
}
```

The main advantage of this solution is that it's simple to implement
and ensures the most critical conditions. Facade class is named after
the original test class, test method names match and, most importantly,
the facade test method is executed as many time as the corresponding
test was run in javascript and exceptions can be thrown individually
for every run. The biggest downside is lack of proper method
parameters. Those tests produce no meaningful information of
what parameters were used for each test, which may be troublesome
when debugging failing tests. Additionally, it doesn't allow
overloading test methods (which shouldn't be practised anyway) 
with different inputs, resulting in name clash.

### Test factories

Another, albeit less common, feature that testng provides is
test factories -- methods that create tests dynamically from
input data. They can be used to produce tests with fake
input parameters that will be propagated to the output report.
Here is a prototype of such test factory:

```java
public class TestCase2_Factory extends AbstractTestFacade {
  @Factory
  public Object[] createInstances() {
    Class<?> testClass = TestCase2.class;
    var facade = new JSTestFacade(testClass);
    facade.setDataProvider(testClass.getMethod("testData"));
    facade.setTest(testClass.getMethod("test1", int.class));
    var results = facade.runTest(driver);
    var data = asList(new TestCase2().testData());
    assert results.size() == data.size();
    return new Object[] { new TestCase2_Facade(data, results) };
  }
}

public class TestCase2_Facade {
  public List<Object[]> data;
  public List<TestResult> results;

  public TestCase2_Facade(List<Object[]> data, List<TestResult> results) {
    this.data = data;
    this.results = results;
  }

  @DataProvider
  public Iterator<Object[]> dataProvider() {
    return data.listIterator();
  }

  @Test(dataProvider = "dataProvider")
  public void test1(int arg0) {
    var myArgs = new Object[] {arg0};
    var index = data.indexOf(myArgs);
    if (index < 0) throw new IllegalArgumentException();
    if (!results.get(index).isSuccess())
      throw results.get(index).getException();
  }
}
```

The actual tests execution is moved to the factory that instantiates
a new ``TestCase2_Facade`` object giving it input data and results of
running tests in javascript. Since I wanted test methods signatures
to match, I don't pass the result in an argument directly;
instead, the result can be obtained by mapping results to input parameters
using list indices.

Apart from class and method name, there is nothing specific to a
particular test in the ``TestCase2_Facade`` class. If I were to write
such facade class for any other test, its implementation would be
identical (with only difference in class and method signatures).
It hints that most of that code could be moved to a common class.

A fundamental flaw of this approach is that a single facade class
can only serve one test method. Adding more test methods would
significantly complicate its implementation.
Tools such as reflection and Testng's
[dependency injection](https://testng.org/doc/documentation-main.html#dependency-injection)
may be particularly useful for implementing test factories.

## Generating code automatically

I'll not go into much detail of how annotation processors
generate test sources, but a brief overview should be
sufficient to get a general idea how it's done.
I decided to stick to the data provider approach due to its
simplicity. I also focused on generating j2s tests form testng
tests because it's the framework I'm familiar with and use in my other
projects.

First, I introduced a new annotation ``@J2STest``. This
annotation has no parameters by itself, but acts as a marker for
the annotation processor indicating that "this method is of interest
to you". Once all methods marked with ``@J2STest`` are collected, the processor
verifies that they are valid testng tests and groups them by classes.
Then, it scans each class containing at least one j2s test searching
for setup, teardown and data provider methods.
Finally it generates facade test classes, one for each discovered
test class. It creates a test method and a data provider for
each annotated test where ``JSTestFacade`` object in instantiated
and executed.
