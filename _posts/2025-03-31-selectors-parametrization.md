---
layout: post
title: Selector function parametrization
---

Selectors are python callables used to determine the suitable runner for the given set of inputs.
They can be either callables or classes extending the `slivka.scheduler.BaseSelector`.
Previously, the only data that selectors received were command input parameters. A separate callable was needed whenever the containts changed, even if the logic was essentially identical.
Such approach required a lot of code repetition. Many selectors dealt with the same kind of inputs, but slightly differing boundary values.
The reusability can be encouraged by making selector functions more gerneric and take additional data from the configuration file.

## API

A new dataclass was added to store the context data for the selector function. An instance of this class, populated with data from the service configuration file, is passed as a `context` keyword argument if the function declares one. The old selector functions are unaffected.

```python
class SelectorContext:
    service: str
    runners: list[str]
    runner_options: dict[str, dict[str, Any]]
```

service
: Name of the service the context was created for

runners
: List of runner names in order they are declared in the config file.
  Typically, you want to test the runners in that order.

runner_options
: A dictionary mapping runner names to a dictionary of options.
  The keys of the dictionary match the names of the runners.
  The options dict store any values specified in the service configuration file.

Here is a template for the new selector function:

```python
def example_selector(inputs, context):
    # perform setup actions here ...
    for runner in context.runners:
        options = context.runner_options[runner]
        if condition_is_met_for_given_inputs(inputs, options):
            return runner
```

Here is an example which filters out runners based on the length of the json array provided in the *json_input* file. The code expects that the boundaries are defined using *min-json-array-length* and *max-json-array-length* options.

```python
def json_array_selector(inputs, context):
    with open(inputs['json_input']) as f:
        json_array = json.load(f)
    array_length = len(json_array)
    for runner in context.runnners:
        options = context.runner_options[runner]
        min_len = options.get('min_json_array_length', 0)
        max_len = options.get('max_json_array_length', sys.maxsize)
        if min_len <= array_length <= max_len:
            return runner 
```

This selector function can be now parametrized from the service configuration file by adding selector options to each runner definition.

```yaml
execution:
    runners:
        runnerA:
            type: SlivkaQueueRunner
            selector-options:
                max_json_array_length: 10
        runnerB:
            type: LSFRunner
            parameters:
                bsubargs: [-q, medium, -m, 1M]
            selector-options:
                min_json_array_length: 11
                max_json_array_length: 100
        runnerC:
            type: LSFRunner
            parameters:
                bsubargs: [-q, long, -m, 10M]
            selector-options:
                max_json_array_length: 1000
    selector: myscripts.myselectors.json_array_selector
```

If you opt for subclassing the `BaseSelector` class instead, the context options are passed to each `limit_*` method as keyword arguments. Here is the example from above rewritten using class-based syntax:

```python
class JSONArraySelector(BaseSelector):
    def setup(self, inputs):
        with open(inputs['json_input']) as f:
            json_array = json.load(f)
        self.array_length = len(json_array)

    def limit_runnerA(self, inputs, *, min_json_array_length=0, max_json_array_length=sys.maxsize):
        return min_json_array_length <= self.array_length <= max_json_array_length

    def limit_runnerB(self, inputs, *, min_json_array_length=0, max_json_array_length=sys.maxsize):
        return min_json_array_length <= self.array_length <= max_json_array_length

    def limit_runnerC(self, inputs, *, min_json_array_length=0, max_json_array_length=sys.maxsize):
        return min_json_array_length <= self.array_length <= max_json_array_length
```

Notice, that the classes do not dynamically adjust to the variable number and names of runners unlike a for loop does.

## Areas for improvement

### Input types

The `inputs` argument contains parameters "as passed to the command line". That means that all values are strings or arrays of strings. Writing selectors could be more intuitive and require less type convertions if the values were properly typed Python objects.

### Option names

The parametrization of selectors is still quite limited. The function can only test against inputs as long as some naming conventions are in place; e.g. the selector can count the number of fasta sequences provided in `input-fasta` parameter, but may not be able to recognize differently named input.

### Limited reusability

Parametrization allow some level of generalization of selector functions, but it's still very limited.
Similarly to the issue with option names, the selector cannot be reused if one service provides a different set of inputs that need to be tested.
A more generic approach would be to reuse the input validation logic performed when the job is submitted. This would be a major change that first requires inputs values to be stored in their respective type instead of string.
