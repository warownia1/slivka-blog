---
layout: post
title: Data validation libraries - JSON Schema
date: 2022-09-28
---

This post is a follow-up to a discussion in a previous post:
[Defining service parameters using composition][previous post].
The input parameters definitions that slivka admins write in a config
file to specify web service inputs are essentially schemas used for
input validation. Defining schemas and sanitising untrusted input is
such a common task that it's extremely unlikely that there are no
libraries doing it already. A quick search on the internet revealed at
least a couple of them. Here is a list of popular ones that frequently
appear in "python libraries for validating data" listings (links lead
to posts dedicated to library analysis):

[previous post]: {% post_url 2022-06-10-parameters-using-composition %}

- [JsonSchema](#jsonschema) - GitHub: [python-jsonschema/jsonschema]
- [Cerberus] - GitHub: [pyeve/cerberus]
- [Colander] - GitHub: [Pylons/colander]
- [Pydantic] - GitHub: [pydantic/pydantic]
- [Schema] - GitHub: [keleshev/schema]
- [Schematics] - GitHub: [schematics/schematics]
- [Valideer] - GitHub: [podio/valideer]
- [Voluptuous] - GitHub: [alecthomas/vouluptuous]

[python-jsonschema/jsonschema]: https://github.com/python-jsonschema/jsonschema
[Cerberus]: {% post_url 2022-10-10-data-validation-cerberus %}
[pyeve/cerberus]: https://github.com/pyeve/cerberus
[Pylons/colander]: https://github.com/Pylons/colander
[pydantic/pydantic]: https://github.com/pydantic/pydantic
[keleshev/schema]: https://github.com/keleshev/schema
[schematics/schematics]: https://github.com/schematics/schematics
[podio/valideer]: https://github.com/podio/valideer
[alecthomas/vouluptuous]: https://github.com/alecthomas/voluptuous


If any of those libraries can be easily
adjusted to run job parameters validation for slivka, we may not need
to write our own. I judged those libraries by the following criteria
ordered from the most to the least important:

- *extensibility*: is it possible and easy enough to add custom types
  and rules?
- *capability*: do they offer rich functionality e.g. nested structures,
  exclusive pairs of parameters, conditional validation?
- *error reporting*: are validation error message human friendly and
  informative of what went wrong?
- *friendliness*: does it feel natural to write schemas or are they
  closer to writing code?
- *JSON support*: is it possible to convert JSON structures to
  schemas or even use them directly?
- *normalisation*: do they offer type coercion and default values
  insertion prior to validation?
- *code style*: my personal and very subjective opinion on how the
  library feels to the coder.

Here are a few tests that I want to perform with each validation
library to compare their syntaxes and capabilities:

- (trivial) basic functionality test - validate primitive input
  parameters and collections of primitives where fields can be
  optional or nullable i.e.

  ```json
  {
    "title": {"type": "string"},
    "subtitle": {"type": "string", "optional": true},
    "authors": {"type": "string[]"},
    "in store": {"type": "boolean"},
    "number of pages": {"type": "integer", "min": 1},
    "special edition": {"type": "string", "nullable": true}
  }
  ```

  ```json
  {
    "title": "Design Patterns",
    "subtitle": "Elements of Reusable Object-Oriented Software",
    "authors": [
      "Erich Gamma", "Richard Helm", "Ralph Johnson", "John Vlissides"
    ],
    "in store": true,
    "number of pages": 138,
    "special edition": null
  }
  ```

- (easy) new constraint test - implement "is even" constraint on an
  integer parameter

  ```json
  {
    "type": "integer",
    "isEven": true
  }
  ```


- (easy) new type test - add new "dimensions" type that accepts
  3-items tuple and add validators acting on that tuple i.e.
  
  ```json
  {
    "package size": {
      "type": "dimensions",
      "maxLength": 10,
      "maxVolume": 250
    }
  }
  ```

- (intermediate) simple dynamic validation test - make two parameters
  required but mutually exclusive i.e.

  ```json
  {
    "input data": {"type": "string"},
    "depth": {"type": "integer"},
    "accurate method": {
      "type": "number",
      "required": true,
      "exclude": "fast method"
    },
    "fast method": {
      "type": "number",
      "required": true,
      "exclude": "accurate method"
    }
  }
  ```

- (hard) advanced dynamic validation test - select "payment method"
  from enum and require cardholder name and card number if the payment
  method is "credit card" i.e.

  ```json
  {
    "payment method": {
      "type": "string",
      "enum": ["transfer", "PyPal", "credit card"]
    },
    "card number": {
      "type": "string",
      "required": {
        "if": {"payment method": {"const": "credit card"}}
      }
    },
    "cardholder name": {
      "type": "string",
      "required": {
        "if": {"payment method": {"const": "credit card"}}
      }
    }
  }
  ```

## JsonSchema

I start with the library, that I have already been using and I am well
familiar with. The [JSON Schema] is "a vocabulary that allows you to
annotate and validate JSON documents". It provides means to describe
data formats and automatically verify their correctness. It lets you
define and verify arbitrarily complex JSON documents, however, at the
cost of even more complex and verbose schema language. This is great
if you need strict and powerful document definitions, but writing such
schemas can be daunting, especially for non-programmers as the syntax
is close to writing ASTs that visit and validate document nodes. The
schema itself does not offer any extensibility, however, the Python
library implementing the schema validation allows writing custom types
and validators, but it requires some code gymnastics and it seems this
feature was not meant to be used.

- [Test 01] - basic functionality
- [Test 02] - new constraint
- [Test 03] - new type
- [Test 04] - dynamic validation (exclusion)
- [Test 05] - dynamic validation (condition)

[Test 01]: {% link /notebooks/input-validation/jsonschema/jsonschema-01-basic-functionality.html %}
[Test 02]: {% link /notebooks/input-validation/jsonschema/jsonschema-02-new-constraint.html %}
[Test 03]: {% link /notebooks/input-validation/jsonschema/jsonschema-03-new-type.html %}
[Test 04]: {% link /notebooks/input-validation/jsonschema/jsonschema-04-simple-dynamic-validation.html %}
[Test 05]: {% link /notebooks/input-validation/jsonschema/jsonschema-05-advanced-dynamic-validation.html %}

Summary:

- *extensibility*: original JSON schema works with native JSON types
  only: primitives, objects and arrays; extending validators with
  custom type checkers is not quite supported &#9733;&#9734;&#9734;
- *capability*: JSON schema is very rich, allows for nested mappings
  and dynamic validation of fields based on other fields.
  &#9733;&#9733;&#9733;
- *error reporting*: in simple cases, errors produced by jsonschema
  are clear and readable; however, when complex schemas are used,
  especially involving conditions, the error messages are cryptic and
  uninformative &#9733;&#9733;&#9734;
- *friendliness*: schemas are difficult to write and can get really
  complex; they are very logical, but not easy to follow.
  &#9733;&#9734;&#9734;
- *JSON support*: schemas are written in JSON for JSON ergo full JSON
  support. &#9733;&#9733;&#9733;
- *normalisation*: no automatic coercion, normalisation or default
  values, what you put is what you get. &#9734;&#9734;&#9734;
- *code style*: the library usage is very minimal, extending
  validators requires some code gymnastics and hacking, documentation
  is somewhat lacking. &#9733;&#9734;&#9734;

[JSON Schema]: https://json-schema.org/
