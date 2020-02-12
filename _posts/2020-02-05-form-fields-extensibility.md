---
layout: post
title: Improving fields extensibility
date: "2020-02-05"
---

Current implementation of form fields assumes that every `BaseField` subclass provides its own `to_python` implementation and appends it's own validators to the validators list.
It creates the problem when two inheriting fields want to operate on different data types.

The list of validators should, therefore, be bound to the specific class for compatibility of the field value. In this approach, each class can call superclass' `run_validation` method which does both conversion and validation and re-use or alter the value for further processing. 

```python

class BaseField:
  def __init__(self, required=False, multiple=False, default=None):
    self.multiple = multiple
    self.default = default
    self.required = required

  def run_validation(self, value):
    """
    Performs conversion and validation of the input value.

    :returns: Converted value
    :raises ValidationError: The value did not pass validation
    """
    if value is None and self.required:
      raise ValidationError("Field is required", "requires")
    else:
      return value

  def validate(self, value):
    if self.multiple:
      if value is None:
        value = ()
      if not value:
        if self.default is not None:
          return [self.default]
        else:
          return [self.run_validation(None)]
      else:
        return [self.run_validation(val) for val in value]
    else:
      if valie is None and self.default is not None:
        return self.default
      else:
        return self.run_validation(value)


class IntegerField(BaseField):
  def __init__(self, min=None, max=None, **kwargs):
    super().__init__(**kwargs)
    self.__validators = []  # list of validators is now private
    self.min = min
    self.max = max
    if max is not None:
      self.__validators.append(partial(_max_value_validator, max))
    if min is not None:
      self.__validators.append(partial(_min_value_validator, min))
    self.run_validation(self.default)

  def __to_python(self, value):
    return int(value)

  def run_validation(self, value):
    value = super().run_validation(value)
    # run super validation, optionally capture it's converted value
    if value is None:
      return None
    value = self.__to_python(value)
    for validator in self.__validators:
      validator(value)
    return value
```