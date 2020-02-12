---
layout: post
title: Value type conversions
date: "2020-02-07"
---

In order to validate the value provided by the user the program takes the following steps:

- Fetch the value from the request data
- Convert the value to the format it can work with (e.g. int, float)
- Run validation process, check if the input is correct
- Convert the value to the format which can be passed directly as a command line argument

Until now, the conversion and validation processes was kept separate. Field classes were adding their own validators to the list and all of them were applied in the validation stage.
This solution, however, imposes a certain limitation: the inheriting class may require modified convertion method which may not be compatible with superclass' conversion and validators.

Let's take a look at the example. `DatetimeField` extends `IntegerField` as it takes the input data as POSIX timestamp and converts it to the datetime object.

```python
class DatetimeField(IntegerField):
  def to_python(self, value):
    value = super().to_python()
    return datetime.fromtimestamp(value)
  
  def to_cmd_parameter(self, value):
    return datetime.strftime("%d/%m/%Y")
```

We can immediately see an issue here. `DatetimeField`'s `to_python` returns `datetime` object which will be passed to the validators originating from the `IntegerField` which take integers as input. 

## Ideas

### Combine conversion and validation

Conversion and validation processes can be combined into a single method. The value will be both converted and validated in the super class before being converted further in the sub classes. Hovewer, this introduces some code repetitions as each class needs to include a super call and None check and we lose the benefits of validators composition.

### Strict inheritance requirements

We may forbid inheritance when overridden `to_python` would produce a different result so that values produced in sub-classes are compatible with parent's validators.
However, in that case, specialised file fields which operate on the file content would not be able to inherit from the `FileField` as the file content is not the file object.
That defeats the purpose of having inheritance if it cannot be used.

### Use composition for both converters and validators

Each fields will be a composition of converters and validators arranged in the sequence. The value will consecutively pass through converters and validators which dynamically change and check the value.
Albeit flexible and powerful, this method seems complex and very error prone. If one of the elements changes, the entire chain of conversion will be broken.
Additionally, it works just like combining the converters and validators into a single method.

## Command line parameter conversion problem

The final issue I'd like to discuss is the conversion of the parameter to the command line argument. In case of primitives, most of the time, it's as simple as converting them to strings. The problem appears with file fields where streams and file content needs to be converted to the path.

The simplest solution would be to write the stream to the file every time it's converted to the command line parameter. However, it would create duplicates of every file.
