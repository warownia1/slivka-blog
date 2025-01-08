---
layout: post
title: To null or not to null
date: 2024-05-17
---

The previous discussion of default values prompted me to reconsider
the treatment of missing and null values. Currently, missing values
and literal null values are treated equivalently. If the default value
is provided, it's substituted for the missing parameter, otherwise
null is used. Once we stop using default values as fallbacks for
nulls, we need to decide how the `required` validator is supposed to
treat missing and null values. In the current implementation, the
behaviour of required fields is somewhat unintuitive when default is
supplied. Since null or missing values are replaces with default
values, the parameter value is never really empty. The table below
shows the validation results for different combinations of parameter
settings and provided value.

|       | _(value)_ | null | _(missing)_ |
| required w/o default | ✔️ | ✖️ | ✖️ |
| required w/ default  | ✔️ | ✔️ | ✔️ |
| optional w/o default | ✔️ | ✔️ | ✔️ |
| optional w/ default  | ✔️ | ✔️ | ✔️ |

Once default value substitutions are removed, it may be useful to
differentiate between missing and null values. Currently there is no
way to require literal null as a parameter value. The new
implementation could make _required/optional_ property control whether
the missing value is accepted while new _nullable_ property would
control whether nulls are accepted as valid values. By default, fields
would be required and non-nullable, replicating the current
implementation behaviour. The nullable property could be changed
whenever there is a need for accepting null as a valid value. All
possible validation results are presented in the following table.

|       | _(value)_ | null | _(missing)_ |
| required               | ✔️ | ✖️ | ✖️ |
| required and nullable  | ✔️ | ✔️ | ✖️ |
| optional               | ✔️ | ✖️ | ✔️ |
| optional and nullable  | ✔️ | ✔️ | ✔️ |

## Practicality

Having fine-grained control over parameter input sounds like a good
thing. However, I started considering if that feature is practical at
all. Command line parameters generally fall into one of those two
categories: they either contain a value or act as an on/off switch.

An example of an on/off switch is a `git merge`'s option
`--commit`/`--no-commit`. The option enables (default) or disables the
auto-commit when merging. In slivka, it would be represented as a _no
commit_ boolean parameter that would add `--no-commit` to the command
if set to _true_ and nothing if set to false, null or not set. In case
of flags, distinguishing null from missing value is not needed.

How about the case, where an actual value is supplied. We can use `git
log <path>` as an example of a optional parameter. If the parameter
value is not supplied then the only reasonable behaviour is the
argument not appear in the command line. What about the value being
null? Null should not become a value, so changing it to "null" literal
or even an empty string is not an option. A reasonable behaviour is to
skip the parameter just like in case of the missing value.

Distinguising between null and missing value is also not practical
from the REST API perspective. HTTP forms do not distinguish between
parameters with no value and missing parameters. Additionally, data
type is always text and there is no _null_ literal, making it
impossible to set the value to null using HTTP forms.

In conclusion, having nullable fields is not needed.
