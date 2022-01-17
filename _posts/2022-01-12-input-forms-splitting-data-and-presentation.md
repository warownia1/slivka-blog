---
layout: post
title: Job request forms - splitting data and presentation
date: 2022-01-12
---

In this post I'll take a look at the job request input forms,
how they process and present the data. The forms are designed
after Django forms. A quick recap, a form is created declaratively
by subclassing a `BaseForm` which has `DeclarativeFormMetaclass` a its
metaclass. The metaclass inspects new class' attributes writing
those subclassing `BaseField` to the `fields` attribute.
When the form is instantiated with the values for input parameters
it passes each value to it's corresponding field's validation
method to check for correctness. The form gives then the possibility
to save a new job request entry to the database.

Too many responsibilities
=========================

The most fundamental responsibilities that field classes have are:
- containers for field properties loaded form config such as
  id, name, description, value requirements and constraints
- fetching value form `MultiDict`s that came from http request
- value conversion (from http request to python value) and validation
  against the constraints
- value conversion to command line argument

The thing that smells here is fetching value form request data and
converting it from string to python value. It suggests that our
domain logic must be used in the http request context. It can be argued,
however, that reading data from the multi-dicts (which just happen to come
from the http request) and creating values form strings are
web-framework-agnostic enough. The forms are still usable programmatically
even without the framework, so it's not a big deal.
But it gets worse.

Each field is shown to the REST API user using the JSON format whenever they
request the input form for the service. Since every field type has
its own unique properties i.e. value constraints, I added a `__json__`
method to each field with no hesitation. This is a violation of single
responsibility principle. Now, the field not only handles the logic
but the presentation as well.
If it weren't bad enough, once the need for web forms emerged,
I added yet another `widget` method which generated HTML input tag
for this particular field. This not only violates SRP even more,
but is completely out-of-scope of what slivka is supposed to provide.

The problems don't end on presentation. The file field, which is somewhat
special, allows values which are: `FileStorage` objects that `werkzeug`
library provides us with, ids of previously uploaded files, results
of other jobs. The first option tightly binds our logic to the type which
is specific to the framework. The second and third require access to
the file system and the database, which input validation logic should
not depend on. Adding to the mix the special ability to save the file with
`FileField#save_file` that `Form` objects must call for file fields
creates quite a mess.

"Pure" logic
============

If I were to design the field/parameter class which contains nothing but the
logic needed to perform its job it would be something like that:

~~~python
class Parameter:
  id: string
  name: string
  description: string
  validators: List[Validator]

  def __init__(self, id, name, description, validators):
    self.id = id
    self.name = name
    self.description = description
    self.validators = validators

  def validate(self, form, data):
    for validator in self.validators:
      validator.validate(form, self, data)
~~~

The field has only few properties shared between all fields and its behaviour
is defined by adding validators to it (composition). The field is
not responsible for parsing the value to the correct type, reading it
from http request, database or file system.
If needed, the type can be enforced using validators e.g.

~~~python
class IsIntValidator:
  def validate(self, form, field, data):
    if not isinstance(data, int):
      raise ValueError("Field value is not an integer")
~~~

An array parameter can be implemented as a wrapper around the `Parameter`

~~~python
class ArrayParameter:
  parameter: Parameter
  validators: List[Validator]  # array specific validators

  def __init__(self, parameter, validators):
    self.parameter = parameter
    self.validators = validators

  def validate(self, form, data):
    for value in data:
      parameter.validate(form, value)
    for validator in self.validators:
      validators.validate(form, self, data)
~~~

Benefits
--------
The biggest benefit of this approach is that `Parameter` contains
parameter information, performs value validation and nothing more
that would go beyond its responsibilities. The responsibility to prepare
the input data is moved to the caller (e.g. web framework) which knows
where the data comes from, in what format and has necessary means
(database, filesystem, configuration) to prepare it properly. Database
and file system handlers no longer need to be arbitrarily handed to
forms or fields. Consequently, code is testable (no more file system and
database mocks), usable programmatically (you can write your own interface
and use it right away).
The `Parameter` also doesn't care how the framework shows it to the user
as it is a responsibility of the framework to display data in the 
right format.

Let's take `FileField` as an example. The field doesn't
need to be aware of where the file comes from, whether it's an in-memory
stream, an existing file or if it was created from the id stored in the database.
What it does need to know is that if it gets a file-like object, it can
perform the validation on it. On the other hand, the framework knows
how the file was obtained and hence knows how to convert in into a
file-like object and store it afterwards.

Issues
------
This Utopian case would perform great doing just application logic
but is completely useless from the user's perspective.

### Parameter type

First of all, the type of the field/parameter cannot be inferred 
from the object itself. Possible solutions are
- determine the type by searching for the type validator -
  probably the worst option as there is nothing that guarantees
  that exactly one type validator is present
- add `type` argument to `Parameter`'s constructor that will be stored
  in object property - neat and easy solution that doesn't require
  creating subclasses for each type.
- create dummy subclass extending `Parameter` for each parameter type -
  classes can have `type` property which can be also inspected 
  by `__init_subclass__` that would add the class to a field factory
  as a recognised type.
  
### Parameter properties

Knowing the type of the parameter is not enough to render it properly,
users need to know additional constraints that the parameter imposes.
Using the code presented above, the only way would be to scan all the
validators, compare using `isinstance` and then introspect the validator
objects themselves. This would heavily rely on the validator implementations
as there is nothing in the `Validator` interface that would allow
seeing its internal parameters. It looks more like an ugly hack rather
than a clean implementation. Other possibility is for each `Validator`
to expose a dict of flags describing its function. Those flags
would be used to generate appropriate values in JSON response.
This, on the other hand, leaks some presentation details to the logic.

### Presenters

Regardless whether we stick to the current solution where constraints
are part of the field object or we define field's behaviour using
the composition of validators, Each field type needs an individual 
presenter class that would render the domain object as e.g.JSON or HTML widget.
This is not an issue but a consequence of using layered architecture.
It separates logic from presentation, though it adds some complexity
to the system.
For each parameter type, there would need to be a presenter class
for each supported user interface. Despite better code organisation,
the extra coding overhead may deter some people from writing slivka
extensions.

### Processing input

Just because we removed input processing from the `Parameter` class
it doesn't mean it disappeared. Now the framework (or other interface)
has to know what's the appropriate value type for the parameter.
The most logical place to do it is in the presenter which mediates
data exchange between the framework and the logic.

It is still unclear to me if converting the value to the appropriate
type should be responsibility of the `Parameter` and belongs to
the application logic layer or is the part of the presentation layer.

Saving requests
===============

After the input parameters has been processed they need to be saved
as a new job request. Currently, saving requests is performed by
the form object which uses each field's `to_arg` method to
convert the value to the command line argument. If the field is
a `FileField`, its `save_file` method is called with database handler
and target directory.

If we imagine that instead of serializing the data to bson and storing
it in the mongo database we want to push the request object to the
queue or send it with ZMQ it becomes apparent that building and
storing the request is not a responsibility of the form object but
an external job request builder and sender.
First, the request needs to be built from the form data. Simply creating
a new `JobRequest` object with inputs taken directly from the form
should work. Remember that now the `JobRequest` is a domain entity
which has nothing to do with a database. The values it store does not
need to conform to any database or transfer protocol requirements.
The request object is then given to job request repository/sender
for sending it to the scheduler. This is the sender which is aware of
the underlying transfer protocols and can properly serialise the
`JobRequest` object along with the input parameters and send it to
i.e. database.

## Choosing proper serialisation method

Now we get to the problem: "How does the sender object know how to
serialise each input parameter?". First, it needs to have an individual
serialisation method for each field type. What if a custom field is
added via plugin? How do we write and register a serialisation method
for that new type?
Second, since the form and its fields no longer participate in the
serialisation process, how could the sender determine a proper
serialisation method from just a parameter value?
For it to work, some extra metadata must be passed along the parameter
or the form must participate in the serialisation process.

## Trouble with files

For the sake of performing the validation logic, it doesn't matter if
the "file" is an open stream or a real file on the filesystem.
Saving the files before processing the parameters is also discouraged
as we don't want to store files of invalid jobs. Hence, the file must
be saved after a successful validation. Since it is the framework
that knows which files came from the request as streams, it should be
responsible for saving the files. Saving all files that came in the
request is trivial, communicating where the files were saved to the
other components, so they can serialise and assign an id, is not easy.
Remember that in the form data, the files are plain file-like objects
(streams) that carry no information whether and where they are stored
on a disk.

Possible solution is to create a thin wrapper around the file which
adds information where the file is stored and what's its id.
The *files* will be fetched or stored using `FileRepository`.
There also needs to be a factory method creating *files* from 
`werkzeug.datastructures.FileStorage` objects.
The framework still needs to be able to tell which values require saving
so it needs to keep a list of unsaved files or iterate over all
parameter searching for unsaved files. The first option seems preferable.

Reinventing the wheel?
======================

Django-forms-inspired job request forms were one of the first implemented
elements of slivka. Even though having a component written specifically
to work with slivka is tempting, sometimes it might be better to use
the existing generic library and adjust the system to work with it.
One of the notable mentions is `wtforms`, a flexible forms validation
and rendering library. In spite of `wtforms` being focused on HTML
forms, it has rich functionality in terms of value validation and could
be adopted to slivka with extra effort.
The adjustments:
- dedicated form factory that would construct forms from config files
  rather than using declarative style supported by `wtforms`
- having presenters that would render fields as JSONs
- adding support for files and adding media type validators which are
  not present in the library by default
