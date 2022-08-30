---
layout: post
title: "java2script: exceptions from recursive methods"
date: 2022-08-26
---

During testing of the testj2s library I encountered an unusual problem:
whenever an exception is instantiated from a recursive call, the entire
browser tab completely freezes (to the point it's even impossible to
reload the page). Everything works just fine if no recursion is present,
but having just one level of recursion breaks everything.

It started with the need to test exception dumping capability
on the javascript side. I made a simple script that is executed by
selenium that constructs an exception from provided arguments and
converts it into a json object.

```javascript
var JsonUtils = Clazz.load("org.testj2s.JsonUtils");

// construct exception object from input data
function newException(className, message, cause) {
  if (cause) {
    cause = newException(cause.className, cause.message, cause.cause);
  }
  var cls = Clazz.load(className);
  var initName = "c$";
  var args = [];
  if (message) {
    initName += "$S";
    args.push(message);
  }
  if (cause) {
    initName += "$Throwable";
    args.push(cause);
  }
  return Clazz.new_(cls[initName], args);
}
var exception = newException(arguments[0].className,
    arguments[0].message, arguments[0].cause);

// dump exception - method under test
return JsonUtils.dumpException$Throwable(exception);
```

Content with such a concise and neat script, I run tests to realise that
they do not complete and run indefinitely. The tests involving
exceptions with causes were those that couldn't finish indicating that
there is something wrong with recursion. After further investigation
I discovered that it's java2script's implementation of
``Throwable.fillInStackTrace`` that causes freezing. But why?

## Stack trace in javascript

In order to understand the cause of the problem we need to take a
look at how javascript handles stack traces or rather lack of thereof.
From withing the scope of every javascript function, there is a special
variable named `arguments` available. It contains a list of positional
arguments passed to the function as well as a reference to the
function itself - ``callee``. The function reference has multiple
properties, the most relevant to us is the ``caller`` property
that points to the last caller of that function. 
This can be observed on the following example:

```javascript
function callerFunction() {
  calleeFunction();
}

function calleeFunction() {
  console.log(arguments.callee.caller);
}

callerFunction();
```

Running this code prints the ``callerFunction()`` reference object to the console.
There is one important thing to note here: javascript doesn't maintain a
traceback as a stack, instead, it stores the caller in the ``caller``
property of the called function reference. It creates a bizarre
result when a function calls itself.
Consider the following example:

```javascript
function recursiveFunction(recurse) {
  if (recurse) {
    recursiveFunction(false);
  }
}

recursiveFunction(true);
```

When the ``recursiveFunction`` is called by itself its ``caller``
property of that function is set to itself. Therefore, even though
there is only one level of recursion present here, trying to
traverse the stack by visiting the ``caller`` of the current function
returns the same function with the same value of the ``caller``.
It creates a seemingly infinite stack trace.

Since java2script's implementation of ``Throwable.fillInStackTrace``
creates the stack by looping the chain of functions pointer to by the
``caller`` property until ``null`` is encountered, if there is
a loop in this call graph, the visiting loop never finishes.

## Does it affect transpiled code?

Apparently not (I'm not certain though). To see why,
we need to dive into a swingjs' implementation of the *fillInStackTrace*
method. As at version 3.3.1v1, it can be located in a *swingjs2.js*
file, line 20674. Here are the first few lines of the method:

```javascript
var caller = arguments.callee.caller;
var i = 0;
while (caller.caller) {
	caller = caller.caller;
	if (++i > 3 && caller.exClazz || caller == Clazz.load)
		break;
}
```
This piece is responsible for retrieving an actual caller method from the stack,
as top four elements on the stack are ``fillInStackTrace$``,
``Throwable``'s constructor, ``setEx`` function, and ``Clazz.new_``
and need to be skipped.
Non-transpiled functions do not set a ``caller.exClazz`` property
thus the condition under the *if* statement is never true, and, 
if there is any recursion occurring, the
caller of the function is the same function resulting in the
condition under the *while* statement to be always true.
Transpiled functions do, however, set the ``exClazz`` property
causing the loop to exit after the third element.
