---
layout: post
title: Validation schemas - conclusions with examples
---


Describing the structure of input parameters is an important part of
defining web services. A single schema plays multiple roles in the
process of creating web services and enabling them for the users.
First, the front-end users need to be informed of what the inputs of
the service they want to use are. Requesting the service data yields
the list of input parameters with their roles and constraints. It is
similar to HTML forms that websites display to collect input from
users and send it back to servers. Next, once the user submits the
data to the server, the schema is used to translate text input
received from the web to application objects, check if the inputs are
valid and report any potential errors back to the user. Finally, it
maps the application objects to the command line parameters that can
be sent to the shell.

Those requirements impose several criteria which the schema language
needs to meet to be useful as a web service input definition language:

- The schema itself must be parsable so that the back-end and the
  front-end applications can convert it to other formats such as HTML
  forms, JSON structures or GUI. Any validation library which uses
  raw Python functions as validators does not meet that criterion.
- The structure of the schema must allow views and controllers to
  convert between their respective data types (e.g. strings) and
  application objects.
- The schema needs to be accompanied by validators which can check
  data structures against it.
- Converters should be able to use the schema to convert application
  objects to strings usable as command line arguments.
- It should be relatively easy to add new rules and types which follow
  all the requirements above.

During slivka development, I created a simple schema that can describe
input parameters of services as primitive data types, files and arrays
accompanied by simple constraints rules. This solution quickly proved
insufficient, new rules couldn't be added without creating new types,
and adding new types wasn't well-thought-out either. Validators were
strongly coupled to the web framework data structures and the
database. The tipping point was adding dependency and exclusion rules
working across multiple fields that broke the mechanism of falling
back to default for missing values. The input processing code needed a
redesign, but I did not know how to properly solve those problems.
Finally looking into other Python schema libraries gave me a better
insight into those problems and possible solutions.

## Exclude rule with a default

Let's start with a trivial example of a single parameter with a
default value

```yaml
param-A:
  default: 0
  max: 9
```

A typical course of action for a validator is:

1. get the value of a parameter
2. if the value is not present substitute the default value
3. pass the value to validators

Passing an empty mapping as input to the schema above results in a
following output:

```yaml
param-A: 0
```

Even such a trivial example already brings the first question. Should
the default value be put through the validation process? In some
cases, the validation can be quite an expensive operation and in most
cases, the default can be safely assumed to be valid under the schema.
We could pass the default value through the validation process once
and skip any future validations involving the default, or could we?
Let's complicate the schema a bit

```yaml
param-A:
  default: 0
  excludes: param-B
param-B:
  required: false
```

Following the rules stated above and skipping validation of default
values, feeding the schema the following input data:

```yaml
param-B: 1
```

results in a successful validation and the following output:

```yaml
param-A: 0
param-B: 1
```

The outcome is definitely not valid under the schema.
Since the _param-A_ is missing, it's given a default value of 0 and the
validation is skipped for that field. Then, since the _param-B_ does
not have any validation rules assigned, it passes the validation as
is. The problem is that even though the default value for the
_param-A_ is valid by itself, it is not when other parameters are
involved. There is a problem with our validation logic which adding
the new _excludes_ rule exposed.

A naive solution is to abandon optimisation and always run all
validators even for default values, but it would be just covering up
an underlying problem with the schema which caused that issue in the
first place. We should re-think how the _excludes_ rule works. For you
see, each "regular" validation rule can be applied to the value of the
field (node) it's assigned to. For example, rules such as _min_,
_max_, _is upper_, _min length_ etc. attached to the fields, can
consume their value. If you think of a validator as a function acting
on a value you can immediately see that all of those functions can
take an isolated value of the parameter and output a validation result
e.g.

```python
Min(4).validate(8.23)
Max(10).validate(7)
IsPrime().validate(11)
IsUpper().validate("HELLO WORLD")
MaxLength(4).validate("Hi!")
```

What would _exclude_ validator do when given e.g. a number?

```python
Exclude(["param-B"]).validate(0)
```

It can't do anything with that value, because it doesn't know the
context in which the value is used. _Exclude_ rule needs information
about a higher-level structure which contains the parameters. That
brings us to the conclusion that _exclude_ is not a validator that can
be applied to primitive parameters (leaf nodes) but it checks mappings
for the presence (or absence) of keys and values. A properly
implemented _excludes_ rule should introspect the keys of a mapping,
not the individual values.

```python
Excludes('param-A', ['param-B']).validate({'param-A': 0,})
```

Using pseudo-schema language, the exclusion should be declared at the
top level of the schema, not under any of the values:

```yaml
type: object
properties:
  param-A:
    default: 0
  param-B:
    required: false
exclusions:
  param-A: [param-B]
```

Providing the same input as previously to the new schema

```yaml
param-B: 1
```

now results in a failed validation. Starting from the leaf nodes, the
missing value for the _param-A_ is assigned _0_ (doesn't need to be
validated), and the value of the _param-B_ goes through validation and
passes. Once all leaf nodes are processed, the validators of the
higher-level mapping run and report a failure because the illegal
_param-B_ field is present when the _param-A_ field is present.

It solved the problem of inconsistency, but exposed a different one,
it's not possible to provide a value for a parameter which is excluded
by other parameters that were given default values. I could argue it's
a logical consequence of such constructed schema, and most machines
would totally agree with me, but many human users would be unhappy
that they can't use _default_ and _exclude_ on the same field at the
same time.

How about using the default and if it results in an invalid schema,
roll it back and try again? I can say it straight away, it's a pretty
terrible idea and here is an example illustrating why.

What would happen when empty input is given to this schema?

```yaml
properties:
  param-A:
    default: 0
  param-B:
    default: 1
exclusions:
  param-A: ["param-B"]
  param-B: ["param-A"]
```

1. validator initially sets both values of _param-A_ and _param-B_ to
   their respective defaults then it detects that the validation of
   _param-A_ fails so it tries setting it back to nothing. Result:
   ```yaml
   param-B: 1
   ```

2. validator sets both values of _param-A_ and _param-B_ to their
   respective defaults, then it detects that _param-A_ excludes
   _param-B_ so it tries disabling _param-B_. Result:
   ```yaml
   param-A: 0
   ```

3. validator sets both values of _param-A_ and _param-B_ to their
   respective defaults, then it detects that both caused errors so it
   tries disabling both and re-runs the validation. Result:
   ```yaml
   <empty mapping>
   ```

Rolling back default values create inconclusive results that depend on
the internal implementation of the validation algorithm and, even
worse, on the order of appearance of the parameters which may differ
depending on the JSON deserialization library used. And we didn't even
get to user-defined validation rules or schema combinations. Having
inconsistent results is significantly worse than the inconvenience of
not having _default_ and _exclude_ on the same parameter.

## Are default values bad?

Having the ability to define default fallback values for the
parameters seemed so natural that I have't even considered not adding
it. Later, when introducing conditional parameters I thought about how
unnecessarily complicated and ugly their implementation was. I
couldn't accept it and wanted to cancel this new feature. I didn't
think for a moment, that the problem is with the _default_ value
substitution and conditional _require/exclude_ only exposed that
issue. 

It wasn't until I dived into the source code of the existing
validation libraries when I started seeing a certain pattern. They
either offered default values but didn't support higher-level
validators involving multiple fields, or they offered inter-field
validators but not defaults, or they offered both and produced
ambiguous results.

It doesn't immediately mean that default values are inherently bad or
completely incompatible with conditional rules. As I demonstrated
above, if you design the schema carefully not allowing validators to
validate the parent or sibling nodes, you can have both defaults and
conditions without consistency issues. Such schemas may not be
particularly human-friendly but they work.

This is not the only issue with defaults. Changing user input may
alter the behaviour of the underlying command line tool subverting
user expectations. Some programs behave differently when provided with
no parameter or provided the default parameter explicitly.
Substituting the default for missing parameters takes away the
possibility of providing no value for the parameter from the user. I
believe that input values supplied by the front-end user should
reflect parameters supplied to the command line program as closely as
possible. We could follow the JSON Schema approach where default
values are hints rather than fallback values.

## Which solution should we adopt?

There are multiple ways the schemas can be done each having its
benefits and drawbacks. The most strict approach is that adopted by
JSON schema where validators are attached exactly to the types they
affect and collections such as mappings and lists are defined
explicitly.


```yaml
properties:
  param-A:
    type: int
    min: 0
    max: 10
    default: 0
  param-B:
    type: string
    max-length: 999
  param-dependent:
    type: boolean
  param-excluding:
    type: boolean
  param-list:
    type: list
    items:
      type: float
      min: 0
      max-exclusive: 1
    min-length: 1
dependencies:
  param-dependent: [param-B]
exclusions:
  param-excluding: [param-A]
```

This schema style is the easiest to process due to its accuracy, but
it's also quite verbose. Each validation rule is attached to the node
it can process. It adds one extra level on top of the schema for
dependencies and exclusions rules that process the dataset as a whole.
Additionally, collection types have one additional depth level, one
for validating the list, and the other for validating items of that
list. It also allows specifying default values for the list and the
items separately.

```yaml
param-list:
  type: list
  items:
    type: float
    default: 0
  default: [0.1, 0.2]
```

To flatten the schema a bit, we can introduce more lax rules regarding
lists assuming that all validation rules under the list type refer to
its items, not the list itself and using a pair of brackets to
indicate that the type is an array.

```yaml
param-list:
  type: float[]
  min: 0
  max-exclusive: 1
```

This more concise notation takes away the possibility to parametrize
the list itself with min and max length requirements and the default
value. We could play with the notation including the constraints
inside the brackets i.e. `float[0-3]`, `float[0-*]` but it would make
processing schemas programmatically more difficult and APIs are
primarily consumed by other software and not humans.

The schema can also be simplified by removing the top-level object
making it implicit and flattening the schema by one level. Dependency
rules that apply to the whole object could then be attached to the
parameters that those rules affect.

```yaml
param-A:
param-B:
param-dependent:
  depends on: [param-B]
param-excluding:
  excludes: [param-A]
```

Moving dependency rules to the parameters they apply to looks a bit
more intuitive, but breaks the natural data processing order from the
innermost types (nodes) to the outer levels. The dependency validator
attached to the node must also receive broader information about the
data structure holding that parameter to work properly. It may be
troublesome with validators nested inside lists and is completely
incompatible with default values.

In the end, I think that schema syntax should not be made fancier and
more human-friendly at the cost of being more difficult to parse and
slightly ambiguous. Quoting the Zen of Python: "Explicit is better
than implicit".
