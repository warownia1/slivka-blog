---
layout: post
title: Data validation with Cerberus
date: 2022-10-10
---

[Cerberus] is a lightweight library for data validation functionality
designed to be easily extensible with custom types and validations. It
tries to accomplish advanced input validation with a relatively simple
schema language. Cerberus uses plain dictionaries to define schemas
such that they can be directly loaded from JSON or YAML files. The
schema language is straightforward, human-friendly and mainly focused
on flat key-value mappings; although, nested multi-level data
structures are well-supported too. Extra features that Cerberus offers
include automatic data coercion and computation of default values.

The main strength of the library is the ability to add arbitrary new
types and validators by subclassing the base validator and adding new
methods. However, the requirement for creating subclasses to add new
features makes the code more static and difficult to add new
validators dynamically without resorting to metaclasses. There is a
pending proposal to make validation functions "first-class" citizens
that would allow composing validators from smaller pieces.

Additionally, the library lacks validation rules that can be applied
conditionally across different fields. Inspecting other fields is
limited to checking for their presence or matching their values with
constants using `excludes` and `dependencies` rules. Advanced
conditional rules can be built by combining rules with logical
operators using `oneof`, `allof`, `anyof` and `noneof` rules. However,
doing so sacrifices the clarity of the validation output and the
schema, and is even discouraged by the authors.

The validator objects, which are containers for the schemas, are not
stateless. They store transient variables used during validation in
own object properties as well as the status of the last processed
document. This raises the question of reusability of such schemas for
multiple documents and thread safety.

There are a few reports of inconsistent behaviour involving `nullable`
rule. The `nullable` validation rule is always present on the field
and takes precedence before any other rule, allowing null values by
default. If the validated value is null, this rule erases some other
rules from processing queue leading to unexpected and sometimes
inconsistent results.


[Cerberus]: https://docs.python-cerberus.org/en/stable/

- [Test 01] - basic functionality
- [Test 02] - new constraint
- [Test 03] - new type
- [Test 04] - dynamic validation (exclusion)
- [Test 05] - dynamic validation (condition)

[Test 01]: {{ site.baseurl }}{% link /notebooks/input-validation/cerberus/cerberus-01-basic-functionality.html %}
[Test 02]: {{ site.baseurl }}{% link /notebooks/input-validation/cerberus/cerberus-02-new-constraint.html %}
[Test 03]: {{ site.baseurl }}{% link /notebooks/input-validation/cerberus/cerberus-03-new-type.html %}
[Test 04]: {{ site.baseurl }}{% link /notebooks/input-validation/cerberus/cerberus-04-simple-dynamic-validation.html %}
[Test 05]: {{ site.baseurl }}{% link /notebooks/input-validation/cerberus/cerberus-05-advanced-dynamic-validation.html %}

Summary:

{% assign star = "&#9733;" %}
{% assign no_star = "&#9734;" %}
- *extensibility*: Cerberus was made with extensibility in mind;
  new types and validators can be added with ease {{star}}{{star}}{{star}}
- *capability*: the library offers rich validation of simple types and
  collections, however, schemas involving multiple values are not
  supported {{star}}{{star}}{{no_star}}
- *error reporting*: the errors produced by validators are readable,
  but errors in composite rules are not well structured
  {{star}}{{star}}{{no_star}}
- *friendliness*: schemas are coherent and well-organised, rules are
  defined under the properties they affect and there are no global
  document rules {{star}}{{star}}{{star}}
- *JSON support*: schemas are plain Python dictionaries, all rules can
  be defined using JSON objects only {{star}}{{star}}{{star}}
- *normalisation*: Cerberus offers automatic coercion through function
  hooks or normalisation methods provided by the validator; default
  values can be either constants or can be computed on demand by
  factory methods {{star}}{{star}}{{star}}
- *code style*: API for extending validators is user friendly;
  documentation is good, there are some inconsistencies with null
  validation; schema objects are stateful storing last validated
  object {{star}}{{no_star}}{{no_star}}
