---
layout: post
title: Making plugins in Python
date: 2022-06-21
---

Many times, when creating an application, you'll want the ability to
add additional features via plugins. This adds modularity to your
application and encourages third parties to write extensions to your
application. Typically, plugins are written as individual packages
that can be installed separately with pip or setuptools. Your
application then needs a way to automatically discover installed
packages and load them dynamically.

# Discovering plugin modules

There are three major approaches to automatic plugin discovery, each
achieving the same goal, finding and loading packages. They are
described in details in the [Python Packaging Guide]{:target="_blank"}.

[Python Packaging Guide]: https://packaging.python.org/en/latest/guides/creating-and-discovering-plugins/

## Using package names

If all plugins for your application follow the same naming convention,
the [``pkgutil.iter_modules()``][iter_modules]{:target="_blank"}
function combined with filters can be used to find and import plugin modules.

[iter_modules]: https://docs.python.org/3.10/library/pkgutil.html#pkgutil.iter_modules

```python
import importlib
import pkgutil

plugins = {
  name: importlib.import_module(name)
  for finder, name, ispkg
  in pkgutils.iter_modules()
  if name.startswith('myapp_')
}
```
## Using namespace packages

[Namespace packages] are packages that allow you to split the sub-packages
and modules within a single packages across multiple distributions.
They provide a convention for where to place plugins and a way to
perform discovery. For example, if you designate ``myapp.plugins``
sub-package as a namespace package, then other distributions can
provide their modules to that namespace. Once installed, you can use
[``pkgutil.iter_modules()``][iter_modules]{:target="_blank"} to discover
all modules placed under the namespace:

[Namespace packages]: https://packaging.python.org/en/latest/guides/packaging-namespace-packages/

```python
import importlib
import pkgutil

import myapp.plugins

def iter_ns(ns_pkg):
  return pkgutil.iter_modules(ns_pkg.__path__, ns_pkg.__name__ + '.')

plugins = {
  name: importlib.import_module(name)
  for finder, name, ispkg
  in iter_ns(myapp.plugins)
}
```

## Using package metadata

Setuptools provides support for plugins via ``entry_points`` argument
to ``setup()``. The package may register plugins as entry points:

```python
setup(
  entry_points={"myapp.plugins": "a = myapp_plugin_a"}
)
```

Then, the plugins can be discovered by using ``importlib.metadata.entry_points``
or ``importlib_metadata.entry_points()`` for Python 3.6-3.9:

```python
from importlib.metadata import entry_points

entry_points = entry_points(group='myapp.plugins')
plugins = {ep.name: ep.load() for ep in entry_points}
```

This approach, however, targets Python version 3.10+ and lacks
compatibility with older versions.


# Registering components

Once the plugin code is loaded and executed, it can do whatever it wants
to start working. Most often, however, plugins add new functions,
classes or components to an existing system.

## Subclass hooks

Defining ``__init_subclass__`` method in the class creates a hook
that will be called every time a new subclass of that class is created.
This is particularly useful to register subclasses of a plugin
base class and avoids writing cumbersome metaclasses e.g.:

```python
class TranslationPlugin:
  langs = {}

  def __init_subclass__(cls, **kwargs):
    super().__init_subclass__(**kwargs)
    TranslationBase.langs[cls.lang] = cls
  ...

class ESTranslation(TranslationPlugin):
  lang = "ES"
  ...
```

``__init_subclass__`` can also accept keyword arguments like that:

```python
class TranslationPlugin:
  langs = {}

  def __init_subclass__(cls, lang, **kwargs):
    super().__init_subclass__(**kwargs)
    TranslationBase.langs[lang] = cls

class ESTranslation(TranslationPlugin, lang="ES"):
  ...
```

It is crucial to call to ``super().__init_subclass__`` for proper
multiple inheritance and mixins support.

## Decorators

A nicer (in my opinion) and more flexible approach is to use decorators
to register plugin components. It has an advantage of being usable on
classes, methods and functions, not just classes. Additionally, it
doesn't enforce certain class hierarchy in favour of duck-typing.

One way of making plugin registering decorator is to by creating a custom
collection i.e. dict and adding a ``register`` method acting as a
decoration to it:

```python
class PluginCollection(dict):
  def register(self, key, item=None):
    if item is None:
      return partial(self.register, key)
    if key in self:
      raise KeyError(f"key {key} already used")
    self[key] = item
    return item

plugins = PluginCollection()
```

It can be then used in one of the following ways:

```python
@plugins.register("class-plugin")
class PluginClass:
  ...

@plugins.register("function-plugin")
def plugin_function():
  ...

plugins.register("object-plugin", PluginClass())
```

After the plugins are loaded, the main class may access the collection
to retrieve registered components.
