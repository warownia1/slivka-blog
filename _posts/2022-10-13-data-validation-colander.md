---
layout: post
title: Data validation with Colander
date: 2022-10-13
---

Colander is a system for deserialising and validating data coming from
text-based serialisation formats such as JSON, XML or HTTP form posts.
The library is relatively small having its entire code crammed into a
single Python module file. It's a component of a bigger Pylons Project
which gathers web-application-related tools and technologies. The
library primarily focuses on the serialisation and deserialization of
string values organised into structures with dictionaries, lists and
tuples. This solution seems rather odd because it requires you to
deserialise incoming data into python collections before deserialising
it again with Colander schemas. Built-in serialisation and
deserialisation features can be seen both as an advantage and
disadvantages. On the one side, users needing to verify incoming data,
and wanting to handle transport and formats themselves, may find the
forced deserialisation step troublesome; on the other side, users
feeding raw data into the schema for processing, may find this
feature desirable.

In Colander, schemas are organised into a tree where each `SchemaNode`
corresponds to an input parameter. Collection nodes such as mappings
or lists have children nodes attached to them which validate elements
of the collections. The hierarchical arrangement makes writing schemas
for complex data structures effortless and intuitive, but it comes
with several disadvantages that I'll explain further. The schemas are
defined either declaratively, using classes and their attributes to
create nodes (which I'm not a big fan of), or imperatively, building
schemas dynamically with code. However, there is no built-in support
for loading schemas directly from JSON objects. Both type
converters-checkers and validators are first-class citizens
implementing *Type* and *Validator* interfaces respectively allowing
for adding new rules to the existing schema classes without the hassle
of subclassing.

The biggest advantage of Colander over other validation libraries
discovered so far is error reporting. The validation errors are
organised into a hierarchical tree of `Invalid` exceptions reflecting
the structure of nodes in the schema. Each exception stores the path
to the node where the error occurred and an internationalisable error
message provided by a *translationstring* library. You can use the
auto-generated error message (which is often sufficient) but the great
power comes from the parsable tree structure and the lack of imposed
message format (and even language) allowing further processing before
the message is sent back to the front-end user.

Default values are provided using a `missing` argument for
deserialisation or a `default` argument for serialisation. The values
which are missing take a special `colander.null`
value which, depending on the defined *missing* behaviour, can be
returned as *null*, replaced with other value, or can be dropped
entirely. If no behaviour is specified, the field is required. It sets
a limitation where a value cannot be required and allow null as an
input at the same time when the deserializer interprets `None` as
a missing value (which most of them do).

The hierarchical design of the schema nodes and attached validators
makes writing nested schemas easier, but comes at a cost. Each
validator can only access the node it's attached to (and its children)
but has no ability to introspect the parent or sibling nodes making it
impossible for one parameter to require or exclude other parameters
existing on the same, or lower level. Such relations must be resolved
on the parent node level. This approach, despite being very logical,
is less intuitive. The developers also indicated that they do not
intend to add multi-field validation ([issue 77] on GitHub).

[issue 77]: https://github.com/Pylons/colander/issues/77

A random issue I encountered involves broken nested *Any* and *All*
validators preventing building composed validation rules ([issue 194]
on GitHub).

[issue 194]: https://github.com/Pylons/colander/issues/194

- [Test 01] - basic functionality
- [Test 02] - new constraint
- [Test 03] - new type

{% assign nbpath = "/notebooks/input-validation/colander" %}
[Test 01]: {% link {{nbpath}}/colander-01-basic-functionality.html %}
[Test 02]: {% link {{nbpath}}/colander-02-new-constraint.html %}
[Test 03]: {% link {{nbpath}}/colander-03-new-type.html %}

Summary:

{% assign star = "&#9733;" %}
{% assign no_star = "&#9734;" %}

- *extensibility*: Colander is easily extensible which was achieved by
  making type checkers and validators first-class objects
  {{star}}{{star}}{{star}}
- *capability*: hierarchical structure of the schema makes writing
  deeply nested schemas easy, but restricts validation to the current
  node only, preventing cross-field rules {{star}}{{no_star}}{{no_star}}
- *error reporting*: default representation of the errors is clear but
  the real power comes from the parseable error tree which allows
  rewriting or translating error messages {{star}}{{star}}{{star}}
- *friendliness*: schemas are coherent, and organised into a nested
  tree using Python classes; validators are attached to nodes they
  affect resulting in unambiguous relations; {{star}}{{star}}{{star}}
- *JSON support*: Colander is intended for the deserialisation of
  JSON-like structures but does not leverage JSON's strong typing;
  schemas are built declaratively with Python classes or imperatively,
  using JSON objects as schemas directly is not supported
  {{no_star}}{{no_star}}{{no_star}}
- *normalisation*: data normalisation is at the heart of the Colander
  library, its main focus is on conversion between strings and
  application types; default behaviour for missing fields is
  customisable {{star}}{{star}}{{star}}
- *code style*: the library documentation is excellent; API is
  extensible and straightforward to use; {{star}}{{star}}{{star}}

Overall, Colander seems to be a useful tool for deserialising input
from string-based data formats which doesn't involve complex
validation rules. Its de/serialisation capabilities, rich error
reporting and internationalisation are impressive, but the offered
range of data validators is rather lacking. I have an impression the
Colander library does not present a clear intention of what data
formats it is supposed to process. On the one hand, it requires
incoming data to be structured into Python dictionaries, lists and
tuples (which are very close to JSON structures); on the other hand,
it completely ignores the strong typing which Python and JSON provide,
converting all values to strings as in XML or HTTP post forms.
