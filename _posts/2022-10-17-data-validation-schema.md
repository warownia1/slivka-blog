---
layout: post
title: Data validation with Schema
date: 2022-10-17
---

[Schema] is one of those libraries I hated the moment I looked into
the documentation. Don't get me wrong, the documentation is concise
clear and covers the usage in detail, but there is not much usage to
be covered. Let's start from the beginning, Schema is advertised as a
"library for validating Python data structures, such as those obtained
from config files, forms, external services or command-line parsing".
The entire library fits into a single 780-line module file which
already gives us some idea of how tiny it is. But, there are plenty of
really good and popular micro-frameworks, so why is this one so bad?
Because there is almost nothing in it.

[Schema]: https://github.com/keleshev/schema

In Schema, schemas are constructed by connecting functions into
logical statements with "or" and "and" operators with some syntactic
sugar sprinkled on top to allow making nested structures. And that's
all it has to offer, there are no validators, no input processing, and
no error messaging. All validation work is done with functions that
you have to build yourself, Schema only gives you a skeleton where you
can plug these functions. Here is an example from the documentation
with a simple use case that I'm going to break down:

```python
schema = Schema(
  [
    {
      "name": And(str, len),
      "age": And(Use(int), lambda n: 18 <= n <= 99),
      Optional("gender"): And(
        str, Use(str.lower), lambda s: s in ("squid", "kid")
      )
    }
  ]
)
```

The operators that Schema uses to declare schemas are: `Schema`
enclosing the whole schema, `And` and `Or` combining rules into
logical statements, `Optional` marking a value as optional and `Use`
converting the value using a supplied function. Additional operators
not included in this example are `Forbidden`, `Const`, `Literal` and
`Regex` (for some reason the regex validator has a special place in
the library). There are no validators included in the library,
everything else consists of Python callables which those operators
link together. This particular schema translates to we expect a list
of dictionaries containing "name' which is a string and has length,
"age" which is convertible to int and has a value between 18 and 99,
optional "gender" key having a string value that, after being
converted to lowercase, is one of "squid" or "kid".

The approach this library adopts forces imperative schemas rather than
declarative ones. It's not possible to load schemas from e.g. JSON
because the library does not provide any validators at all. Here is my
attempt at creating a declarative length validator:

```python
def length(min=None, max=None):
  def length_validator(value):
    if min is not None and value < min:
      return False
    if max is not None and value > max:
      return False
    return True
  return length_validator

schema = Schema(And(str, length(min=3)))
```

However, if I were to implement all validators trying to fit them into
this framework, I'd rather create my own framework, more suited to my
needs.

One more consequence of using plain functions and lambda expressions
as validators is the absolute lack of meaningful error messages. The
framework cannot acquire any error details from a simple true/false
return value. A typical error message you receive from Schema is
"&lt;function name&gt; should evaluate to True" (gets even worse when
using lambdas) with no information about what happened. If you are
looking for a library that checks your data but gives you no feedback
then here it is.

I didn't do my regular tests for this framework, because it offers so
little there is nothing to be tested.

Summary:

{% assign star = "&#9733;" %}
{% assign no_star = "&#9734;" %}

- *extensibility*: it's hard to talk about extensibility if there is
  no existing functionality to extend; adding new types and validators
  comes down to creating Python classes and functions which is kind-of
  an advantage {{star-}} {{-no_star-}} {{-no_star}}
- *capability*: you have *And*, *Or* and Python collections for your
  disposal, and that's it; it's up to you to implement everything else
  {{no_star-}} {{-no_star-}} {{no_star}}
- *error reporting*: almost non existent {{no_star-}} {{-no_star-}}
  {{-no_star}}
- *friendliness*: I think being simple one good thing about this
  library; on the other hand, making schemas with long chains of ands
  and ors gets ugly quickly {{star-}} {{-no_star-}} {{-no_star}}
- *JSON support*: schemas are not constructed declaratively; it would
  be possible to make it declarative with lots of effort; they claim
  schemas can be converted to JSON Schemas, but this feature is
  largely unimplemented and not needed {{no_star-}} {{-no_star-}}
  {{-no_star}}
- *normalisation*: the library actually allows you to coerce values to
  target types with `Use` or define defaults with `Optional` operator,
  and it's actually well-done {{star-}} {{-star-}} {{-star}}
- *code style*: linking lambda functions with logical operators is not
  particularly pretty, but if you are okay with writing Python code
  and don't want to spend time learning other more advanced schema
  languages, then it's a good choice; documentation is also quite good
  and explanatory {{star-}} {{-star-}} {{-no_star}}

Overall, this library does not add any value on top what you can
already do using plain Python script.
