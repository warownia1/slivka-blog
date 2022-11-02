---
layout: post
title: Data validation with Valideer
date: 2022-10-20
---

Valideer is a lightweight library for data validation and coercion.
They use own declarative schema language but using Python API to
build schemas is also supported (and required for anything but very
basic rules). The library is well-tested and mature. They include
validators for several common types as well as collections such as
lists and mappings.

The library makes no distinction between validators, types and
adaptors. The types are, actually, validators too that check if the
value is of the specified type. The validators can also act as
adaptors altering the incoming data or converting them to the correct
type. It requires extra care to apply validators in the correct order,
otherwise, the value may not be processed correctly.

The library is inconsistent in what it considers a validator and what
a validator's attribute is. For example, a `String` object is a
validator that checks the value type. It accepts two attributes:
`min_length` and `max_length` which add two more validation rules.
Following that logic, an `Integer` object should accept _min_ and
_max_ boundaries as the arguments too. Instead, it does not take any
parameters and such validation is performed by a `Range` validator
which takes a schema of a type it performs validation on as the first
argument followed by _min_ and _max_ value ranges. Here is a
comparison of those two validators:

```python
str_validator = String(min_length=1, max_length=99)
int_validator = Range(Integer(), min_value=0, max_value=99)
```

An additional limitation imposed by the library is that each field can
only have a single validator assigned to it. Multiple validators must
be joined together using composite validators such as `Nullable`,
`AnyOf`, `AllOf` or `ChainOf`; the last one can be used to apply
adaptors in sequence. It makes converting JSON documents to schemas
and reading schemas more difficult because instead of a flat mapping
of rules, you have to deal with nested lists joined by _and_ or _or_.

Errors produced by Valideer are clear and informative, they are not
hard to parse and contain the path to the error. However, validators
only report the first encountered error which can badly affect user
experience.

- [Test 01] - basic functionality
- [Test 02] - new constraint
- [Test 03] - new type

[Test 01]: {{ site.baseurl }}{% link /notebooks/input-validation/valideer/valideer-01-basic-functionality.html %}
[Test 02]: {{ site.baseurl }}{% link /notebooks/input-validation/valideer/valideer-02-new-constraint.html %}
[Test 03]: {{ site.baseurl }}{% link /notebooks/input-validation/valideer/valideer-03-new-type.html %}

{% assign star = "&#9733;" %}
{% assign nostar = "&#9734;" %}

Summary:

- *extensibility*: adding new validators is really straightforward;
  you can register validators by names and re-use them in schemas
  easily {{star-}} {{-star-}} {{-star}}
- *capability*: validators provided by the library are not very
  capable, more advanced rules must be added with extensions;
  cross-field validation rules are not present {{star-}} {{-nostar-}}
  {{-nostar}}
- *error reporting*: error objects provide good amount of detail about
  the error including the schema path; having only the first
  encountered error reported is a serious limitation {{star-}}
  {{-nostar-}} {{-nostar}}
- *friendliness*: Valideer uses own succinct schema languages which is
  easy-to-use but insufficient for anything but very trivial cases;
  Python API is a bit inconsistent with how the validation rules are
  defined {{star-}} {{-nostar-}} {{-nostar}}
- *JSON support*: very simple schemas can be defined with JSON only,
  but it's insufficient in most cases; the validators leverage strong
  typing to verify data which goes on par with JSON types {{star-}}
  {{-star-}} {{-nostar}}
- *normalisation*: values can be converted to correct types using
  `AdaptTo` and `AdaptBy` validators; however, conversion to the
  correct type does not happen by type validators such as `String` or
  `Integer` {{star-}} {{-nostar-}} {{-nostar}}
- *code style*: Valideer does not adopt flat mappings of validators,
  instead schemas are composed of sub-schemas connected by logic
  operators or chained together to form a "pipeline"; lack of
  consistency in how the schemas are organised is a big downside
  {{star-}} {{-nostar-}} {{-nostar}}


Overall, Valideer is a very simple and lightweight library which
offers an acceptable range of features out-of-the-box. The lack of
complex validation rules is compensated by the ease of writing custom
rules. It does not offer any advanced validation rules and error
reporting is quite poor.