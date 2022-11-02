---
layout: post
title: "Defining service parameters using composition."
date: 2022-06-10
---

This post is the continuation of the previous post:
[Job request forms - splitting data and presentation][previous post]
Today I focus on improving the
implementation of service parameters. Each input parameter of the
service is represented by a ``Field`` instance. It contains parameter
metadata such as its name, description, type etc. and performs the
validation of input values. Below, I present the stripped-down
declaration of the field class containing only parts relevant to the
value validation (application logic):

[previous post]: {{ site.baseurl }}{% post_url 2022-01-12-input-forms-splitting-data-and-presentation %}

```python
class BaseField:
   id: str
   name: str
   description: str
   default: Any
   required: bool

   def validate(self, value: Any) -> Any:
      if value is None and self.required:
         raise ValidationError("field is required")
      return value
```

The ``BaseField`` contains common attributes which are used by all of
its more specialised subclasses. The validation method of the base
class doesn't do much apart from checking for empty values against the
"required" constraint. The functionality is added to the field by
subclassing it and overriding the ``validate`` method. Below are
examples of how the ``BaseField`` can be extended to provide functions
to process different input types such as integers, decimals, text,
flags and files.

```python
class IntegerField(BaseField):
   min: int
   max: int

   def validate(self, value: Any) -> Any:
      ...  # test for min and max

class DecimalField(BaseField):
   min: int
   min_exclusive: bool
   max: int
   max_exclusive: bool

   def validate(self, value: Any) -> Any:
      ...  # test boundary conditions

class TextField(BaseField):
   min_length: int
   max_length: int

   def validate(self, value: Any) -> Any:
      ...  # test text length

class BooleanField(BaseField):
   def validate(self, value: Any) -> Any:
      ...  # test if valid bool

class ChoiceField(BaseField):
   choices: OrderedDict[str, str]

   def validate(self, value: Any) -> Any:
      ...  # test if a valid choice

class FileField(BaseField):
   media_type: str

   def validate(self, value: Any) -> Any:
      ...  # read file and test content

```

In addition to those types, there is also a special mixin class which
can turn the field into its array counterpart. 

```python
class ArrayFieldMixin(BaseField, ABC):
   def validate(self, value: Any) -> Any:
      ...  # run super.validate on each array element
```

It all looks ok at first but this solution has a significant flaw:
adding and/or combining features coming from third parties is not
straightforward and results in a deep inheritance hierarchy with
diamond problems all over the place. Consider this: you are asked to
add a prime integer parameter (e.g. for a cryptographic key
generator). The way to go is to subclass an ``IntegerField`` as
follows:

```python
class PrimeIntegerField(IntegerField):
   def validate(self, value: Any) -> Any:
      ...  # test if prime
```

The other time you are asked to include a parameter for an integer
which is divisible by a number "x", so you create another subclass:

```python
class DivisibleIntegerField(IntegerField):
   divisible_by: int

   def validate(self, value: Any) -> Any:
      ...  # test if divisible by x
```

Now, if you wanted to combine those two functionalities, there is no
way to do it other than multiple inheritance, messing with
superconstructors and combining ``validate`` methods from multiple
superclasses. As the number of features grows, so does the number of
possible combinations of these features making it quickly impossible
to manage. The issue comes from the fact, that we are using the wrong
pattern here.

Let's step back and ask a following question "what does it mean to add
validators to the field?". New validators are extending the
functionality of the field type, but they do it by adding to an
existing type rather than *extending* it. The field is, in fact,
*composed* of smaller parts that work together making the whole. If
the type of the field reflects the type of the value it is acting on,
which is int in both cases, then we are not dealing with new or
extended types as we include new validators. We do not utilise
polymorphism, which is one of the reasons to use inheritance, so
extending types brings no benefits, quite the opposite, it's harmful.
A more appropriate solution is to have one generic field (or one field
type per value type that we can handle) whose features are added by
composition. New properties can be included without building a
multi-level hierarchy of classes. Therefore, this pattern:

![Features as composition]({{ site.baseurl }}{%link /assets/2022/06/10/compositing-parameters.svg %}){:.centered}

is more suitable than this:

![Features as inheritance]({{ site.baseurl }}{%link /assets/2022/06/10/inheritance-parameters.svg %}){:.centered}

Using composition introduces more classes into play, but it has a huge
advantage of being scaleable, reusable and testable in isolation.
There is an initial overhead, but after that, adding a new field
constraint is as simple as creating a single class/function.
