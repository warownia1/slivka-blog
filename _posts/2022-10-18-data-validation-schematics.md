---
layout: post
title: Data validation with Schematics
date: 2022-10-18
---

Schematics is yet another library to organise types into structures,
validate them and convert between native Python objects and
language-agnostic primitives. The project seems to be pretty mature.
The structure of the code is clearly inspired by ORM systems such as
SQLAlchemy or Django ORM. Object structures are represented by models
having fields that store and validate data, similar to SQL tables.
Converting to and from primitive types leverages Python's strong
typing and is particularly useful when working with JSON, which is
also strongly typed. Contrary to Colander, Schematics does not force
you to de/serialize to and from strings. This feature can be extremely
useful for converting primitives from a configuration file or a
database to Python types and turning them back to primitives before
saving them to the database or sending them over the wire.

The elementary pieces that make schemas up are _Types_, which inspect
and mutate data. Each _Type_ defines methods to:

- coerce primitive data to appropriate Python objects
- convert Python types into serializable formats
- validate data

The _Types_ provided by the library out of the box are not limited to
primitives and collections. They also offer native support for date and
time, multilingual strings, hashes and internet addresses. Moreover, a
field type can also be another model which encourages reusability and
composite structures. For example, _DateTimeType_ defines a
`to_native` method which converts ISO-8601 date to Python's _datetime_
object:

```python
>>> dt_t = DateTimeType()
>>> dt_t.to_native('2022-10-17T18:13:24')
datetime.datetime(2022, 10, 17, 18, 13, 24)
```

A `to_primitive` method does the inverse work of converting _datetime_
objects to ISO-8601 formatted string:

```python
>>> dt_t = DateTimeType()
>>> dt_t.to_primitive(datetime.datetime(2022, 10, 17, 18, 13, 24))
'2022-10-17T18:13:24'
```

Finally, `validate` checks the value against provided constraints:

```python
>>> int_t = IntType(min_value=1)
>>> int_t.validate(2)
2
```

Schematics adopts the approach of adding new validators by extending
the base types. A reasonable solution for adding validators
dynamically to the existing types is creating separate mixin classes
defining validators and combining them in a single subclass. It is
possible to add custom validators to the existing types by passing the
list of validation functions to the _Type_ constructor, however, doing
so takes away the possibility to parse the schema and extract
validator parameters back from it. Adding new types is trivial and
requires subclassing a `BaseType` or one of its sub-types and
implementing `to_primitive`, `to_native` and optionally `convert`
methods.

The library also provides means to supply defaults for missing values
or make fields required or optional. The _Types_ do differentiate
between missing values and _None_, with missing values being replaced
by the default and the _None_ values left intact. At the same time,
the _None_ value does not pass the _required_ validation. This
produces a bit peculiar result where specifying _None_ for a required
field having a default is an error while leaving it empty is not. It's
because defaults are substituted before validation and participate in
the validation process instead. 

What Schematics does not provide are inter-field validation rules. The
validators that require or exclude dependent fields based on the value
or presence of a field are not implemented. The behaviour can be
defined in a model-level validation, but such a solution is too rigid
to be useful in schemas created dynamically from parameter
definitions.

The library documentation is mediocre at most. Basic functionality is
well explained, but advanced features and API reference is largely
missing. Additionally, the documentation is out-of-date, covers an old
version of the library still utilising Python 2.7 and hasn't been
updated since 2016. The library itself hasn't been updated for over
the year and, in its current state, does not work under Python 3.10 or
higher ([issue #628] on GitHub).

[issue #628]: https://github.com/schematics/schematics/issues/628

- [Test 01] - basic functionality
- [Test 02] - new constraint
- [Test 03] - new type

[Test 01]: {% link /notebooks/input-validation/schematics/schematics-01-basic-functionality.html %}
[Test 02]: {% link /notebooks/input-validation/schematics/schematics-02-new-constraint.html %}
[Test 03]: {% link /notebooks/input-validation/schematics/schematics-03-new-type.html %}

Summary:

{% assign star = "&#9733;" %}
{% assign no_star = "&#9734;" %}

- *extensibility*: creating new types is very straightforward in
  Schematics, however, validators are tightly a part of the _Type_ and
  cannot be freely added or removed {{star-}} {{-star-}} {{-no_star}}
- *capability*: adding schemas as fields in other schemas makes
  writing complex structures relatively easy; the range of built-in
  types and validators is satisfactory, but the schema lacks
  cross-field conditional rules {{star-}} {{-no_star-}} {{-no_star}}
- *error reporting*: error messages are clear and informative,
  returned as a parsable tree {{star-}} {{-star-}} {{-star}}
- *friendliness*: schemas are coherent, organised into nested
  structures using Python classes; validators are the integral part of
  _Types_ which minimises ambiguity, but makes then unable to reach
  outside the _Type_ they are attached to {{star-}} {{-star-}}
  {{-star}}
- *JSON support*: Schematics is very good at de/serialising JSON data,
  it leverages strong typing of JSON objects; schemas, however, are
  built declaratively with classes and cannot be loaded from JSON
  directly {{star-}} {{-star-}} {{-no_star}}
- *normalisation*: Schematics normalises input data before processing,
  converting primitive types to Python objects; normalisation is not
  obtrusive and converts to and from all available primitives not just
  strings; default values can be supplies as either constants or
  factory functions granting users great flexibility {{star-}}
  {{-star-}} {{-star}}
- *code style*: the library offers familiar API, but the documentation
  is missing important information and is largely outdated; most
  importantly, the library does not work in the latest Python version
  {{star-}} {{-no_star-}} {{-no_star}}

Overall, the Schematics library is a good choice for writing static schemas that reflect the data you store in a database or send or receive on the wire. It offers familiar API to anyone who already worked with ORM systems. It's not particularly useful for creating schemas dynamically as we need it in slivka. The biggest downside is lack of support for latest version of Python, poor documentation and poor maintenance.
