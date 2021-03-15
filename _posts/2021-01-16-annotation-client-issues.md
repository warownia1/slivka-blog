---
layout: post
title: My issues with annotation clients design
date: 2021-01-16
---

A bit of my frustration
-----------------------

When building web services menu, I expected that using
`PrefferedServiceRegistry#populateWSMenuEntry()` should work just fine.
After all, the method was designed to build menu for the
`ServiceWithParameters` -- an abstract base class for any other web
services.

... I couldn't be more wrong about it.

First of all, the `PreferredServiceRegistry` does not build the menu
by itself, it uses `attachWSMenuEntry` of the `ServicesWithParameters`
flag. It means that this class plays a dual role here, it represents
a generic service and is responsible for building the UI. It's already
a red flag when the class fulfills two completely different roles,
even worse when it's an abstract class.

It got even more terrified when I realised how the menu items are
being created there. Depending on the service type, it creates new
instances of either `MsaWSClient` or `SequenceAnnotationWSClient`.
Those classes, extending `Jws2Client`, add menu items that construct
even more of themselves when clicked.

Those `Jws2Client` objects are not treated nor working like objects but
functions. They provide multiple overloaded constructors that perform
operations instead of initializing the object and **their purpose
change depending on which constructor was used**!

The actual problem
------------------

Because a new worker is created every time the job is started or the
sequences are updated, the approach of registering and deregistering
workers for interactive calculations doesn't work properly. Starting
workers doesn't work because they are not the same workers that were
registered previously.

The easy solution
-----------------

The least effort solution is to allow one-shot workers to be started
without being registered first. 

The proper solution
-------------------

First of all, we need to completely get rid of the `Jws2Client` in its
current form and split it to several smaller classes, one for every
purpose the class serves.  Then, there needs to be exactly one
`AlignCalcWorker` instance for each repeatable job instead of creating
a new instance for each run.  Finally, the `ServiceWithParameters`
should not contain code creating menu entries. Instead, it may provide
an implementation of `WSMenuEntryProviderI`, i.e. a pre-exising
classes or static object that will be responsible for menu.
This approach will definitely improve code readibility and remove the
need for extra hacks that currently make AACon work.
