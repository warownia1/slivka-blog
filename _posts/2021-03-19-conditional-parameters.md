---
layout: post
title: Expressions for conditional parameters
date: 2021-03-19
---

Following the addition of hmmer tools and the need for validating
parameters that depend on each other I started thinking of how
the conditional constraints can be incorporated into the service
configuration file.

## Syntax


First, we need to define syntax that will be used to specify the 
parameter requirements. After that, we need to be able to evaluate
the condition and decide how to act upon it.

### Python's eval

Probably the most straightforward solution would be to use Python's
built-in ``eval`` function and evaluate the expression
substituting form values as variables. However, one problem is
that the field names allow "-" character which is not allowed in
Python identifiers. Additionally, a great care needs to be taken
to eliminate any potential code injection.

### Syntax tree

A promising alternative is to write a syntax tree using yaml.
e.g. ``other-value != null and other-value >= 5`` will be written
as

~~~yaml
and:
  - neq:
    - other-value
    - null
  - geq:
    - other-value
    - 5
~~~

This approach would be very easy to for the program to understand
and would require near to no pre-processing, but it's very unreadable
and not very convenient for humans.

### Logical expression

An alternative to the syntax tree, but relying on the same principle,
would be having a logical expression that will be then converted to
the syntax tree. It would combine the convenience of writing the
expression in plain text with the safety of the syntax tree
at the cost of much more complicated code. It requires lexical
analyser and defining each operation individually. This approach
can be very error prone as lexical analysis is not an easy task.

### Subclassing BaseForm

Another interesting option is to utilise the existing code for
creating forms ``BaseForm`` and ``DeclarativeFormMetaclass``.
The system administrator would create form subclasses and contain
more complicated field validation operation in the ``validate``
method. This is the most flexible of all solutions, but goes
beyond configuration files are requires writing Python code.
