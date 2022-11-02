---
layout: post
title: "Model-view-controller in the presentation of service parameter"
date: 2022-06-09
---

In this post, I'll continue the discussion from the [previous post]({%
post_url 2022-01-12-input-forms-splitting-data-and-presentation %})
focusing on isolating application logic that handles service
parameters. I'm aiming to design an architecture that meets the
following goals:

1. single responsibility - parameter classes should only do a bare
   minimum needed to perform application logic
2. independent of transport - parameter objects and validators should
   be oblivious to the context they are used in and to where the data
   comes from. they should be usable programmatically as well as
   through JSON API and HTML forms.
3. extendable via plugins - it should be easy to add validators and
  new types via plugins.

Given the criteria, let's find out where the current design is lacking
and how it could be improved. The analysis will focus on two topics:
how the form is interacting with the "outside world", and how the
application logic is realised within the form.

## Input processing

Let's start with a broader picture of how the data is handled by forms.
The outline below shows the classes
involved in parameter processing and their relations. 

![Design diagram]({{ site.baseurl }}{%link /assets/2022/06/09/current-design-diagram.svg %})

Starting from the bottom, there is a form factory class which creates
input forms from configuration files. This introduces a nice
abstraction layer, dividing the configuration loader from the actual
object instantiation. This separation should be preserved as it allows
to make modifications to the loader (being config file loader or mock
form provider used for testing) without changing the core parts.

Next up is the ``FormDeclaration`` which contains the (meta)data of
the form. Most importantly, it's an aggregation of fields which define
available parameters and their constraints. The form declaration
contains no user input data. Those can be provided to the ``bind(data,
files)`` method that returns a bound form i.e. form + user input.

Each field (parameter) contains information such as parameter name,
description, default value and a set of constraints. The ``Field``
class also defines methods to display the field as JSON
(``as_json()``) or as HTML widget (``as_html()``). Those two methods
violate the first rule - single responsibility principle. The logic
(which should be universal and not tied to any particular use case) is
responsible for displaying the field and parsing HTTP request values
(more on that later), which is certainly out of the scope of the
application logic.

The input data is provided to the fields directly from the HTTP
request, so they can pick relevant values from the request. After
validation, each field performs value conversion to the the format
that can be consumed by the database and (if applicable) saves the
files to the disk. This violates our second rule, that the logic
should be independent of the transport. Forms and fields should not be
aware of nor care where and how the data is delivered and stored. It
creates unnecessary coupling and makes forms unusable in any context
other than an HTTP request.

The last step involves saving the job parameters to the database. The
action is performed directly by the form which requires ``Form`` class
to have access to the database. This is, again, in violation of the
second point which states that logic should be independent of the
transport. One approach to maintain the separation is to move all
database-related calls to the external "framework" code. Another
option is to implement a repository pattern which is an abstraction
layer between objects in the application code and entries in the
database. Both solutions can be used simultaneously for the best
results. Here is an [article from Joe Petrakovich][repository pattern]
explaining why you should use the repository pattern in your software.

[repository pattern]: https://makingloops.com/why-should-you-use-the-repository-pattern/

## MVC to the rescue

The [MVC] pattern divides the software into three components
responsible for modelling data internally, displaying it to users and
processing user input respectively. Following this separation, the
internal logic of forms and fields should be stripped off of anything
related to presentation and underlying data transport. A well-isolated
data model should bear no hints of what framework, transport or
storage it's been designed for such that it becomes an external
"implementation detail". The views are provided by separate
classes/functions that "know" how the data structures are presented in
a particular context e.g. HTTP request/response, SQL query, JSON
object. Controllers are operating along with views and process user
input in the context of a particular view.

In MVC terms, the fields would make the model part, and will be
accompanied by renderers that produce their HTML/JSON representation.
A request parsing would be performed by a controller associated with a
particular view. This way, the controller that exists along the
framework "knows" how to process and store incoming data and the
underlying fields are only responsible for processing the model data
and logic. This change is relevant in making slivka usable
programmatically as an embedded application, in contrast to the
current use as a standalone server.

[MVC]: https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller

The following diagram shows the flow of data that starts from an HTTP
request and ends on the job being executed. A few pieces of the
process are not entirely clear to me at the moment and require more
thought to be put into them.

![Data flow diagram]({{ site.baseurl }}{%link /assets/2022/06/09/data-flow.svg %}){:.centered}

When an HTTP request arrives at the system, it's received by a web
service's view/controller whose purpose is to translate the language
of the HTTP world to the application model objects. The controller is
a part of a server framework and has access to outer resources
(networking, databases, file system) but performs no application
logic. Then, it sends repacked data to the inner layers of the
application for processing. Once the processed data is returned to the
controller, it decides on what to send back to the user. The
subsequent part is responsible for marshalling the data into a format
suitable for exchange with a database. I think it's a bit of a grey
zone where the data model mixes with its appearance in the database.
On one hand, the data model should be oblivious to a database layer,
but certain types i.e. files need extra details coming from the model
to be able to be (re)stored in the database. The last stage involves
retrieving the input parameters from the database, converting them to
the data model followed by converting to command line arguments.

As you can see, maintaining the MVC pattern requires more classes to
be created, each fulfilling its specific role. But, I believe it's
much easier to maintain and test multiple smaller well-defined and
round classes than having multi-purpose monolithic code with no clear
boundaries and roles.
