---
layout: post
title: Lexical analyser for parameter constraints
date: 2021-03-23
---

Following the previous post I decided to use the logical expressions
for conditional parameters as it would be relatively convenient
to write configuration file and the parser was not too difficult
to make.

# Step 1 -- tokenisation

Before any processing can be started, the expression needs to be
tokenised and each token needs to be identified with the type
and value. It can be easily done with regular expressions checked
in the following order:

parentheses
: left and right parentheses `\(`, `\)`

number
: an integer or decimal optionally preceded by a minus sign or followed
  by `e` and an exponent `-?\d+(\.\d+)?([eE]-?\d+(\.\d+)?)?`

string
: a quoted string literal `"([^"\\]|\\.)*"`

keyword
: a special word which is not immediately followed by a letter,
  number or dash character which would make it an identifier;
  the keywords are: `and`, `or`, `xor`, `not` and `null`;
  for `true` and `false` we follow the C and Python convention of 
  them being equal to 0 and 1 respectively
  (a better approach might be to have no separate regex for
  keywords and filter them out later from the identifiers)
  `(and|or|xor|not|null)(?:$|(?=[^\w\-]))`

identifier
: a variable that references another field of the form; it can
  contain letter, number and dash character and must start with
  a letter `[A-Za-z_][\w\-]*`

operator
: a mathematical or comparison operators such as +, -, *, <=, > etc.
  and a special operator # meaning "the length of"
  `[#+*\/\-]|[<>]=?|[=!]=`

skip
: a white character that would be skipped `[ \t]`

invalid
: if the matching reaches this point it means that the token 
  is invalid, so it captures everything `.`

# Step 2 -- syntax tree

Before the tokens can be evaluated they need to be reorganised
into a tree preserving operator precedence. For that I used the
[shunting-yard algorithm](https://en.wikipedia.org/wiki/Shunting-yard_algorithm)
that can convert an expression from an infix notation to a postfix
notation or an abstract syntax tree. 

# Step 3 -- evaluation

From that point a postfix notation can be easily evaluated using
the variables provided from the form fields. It starts with an empty
stack an iterates over the tokens in the postfix expression.
If the token is a number, string or null, it's value is pushed to the
stack as is; if it's an identifier, the value is looked up in the
variables dictionary first.
The unary and binary operators pop one or two items from the stack
respectively, perform the operation and push the result to the stack.
In the end, if the expression was valid, we should end up with
a single item on top of the stack which is the result of the
expression.

# Caveats

This approach should work well in most simple cases, however its
completely unsuitable when the analysed parameter is a file or an
array. In those cases a more sophisticated approach is needed.
Additionally, since we want users to extend the existing form
fields and provide more variable types, it might be desirable to
allow advanced users to make custom forms and validators with Python
code.

Further, even though the parameters can be validated on the
server-side, we also need to communicate additional constraints to
the clients so the validation can also happen one the client-side.
This would be impossible with an arbitrary Python methods executing
on the server-side.