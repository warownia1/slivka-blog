---
layout: post
title: Adding expressions to form validation
date: 2021-03-25
---

Adding custom expressions involving other parameters to the
form validation routine is not as straightforward as adding
additional field constraint.
First, the constraint is not longer acting on the field
value only and may involve other fields as well, hence it needs
to be performed after the form is loaded.
Second, the default values of some fields might cause collisions
with the values of other fields provided by the users.
It also creates the need to distinguish whether the value originates
from the user form or from the default during validation process.

The new form validation process may consist of the following steps

1. Fetch values from the input form for each field and prepare a dict
   of default values.
2. Validate each provided value and convert to the field's
   corresponding type. Do not substitute default for None
3. Combine input values with default values i.e. using ChainMap
4. Run conditional checks for each field using the combined map.
   Mark fields that failed the test as disabled if default was used,
   raise validation error otherwise.
5. Change default values for disabled fields to None.
6. Run conditional checks again, this time raising errors for all 
   unmet conditions.

This will require moving default value substitution out of the
``Field#validate()`` method and performing the substitution in the
``Form`` class.
