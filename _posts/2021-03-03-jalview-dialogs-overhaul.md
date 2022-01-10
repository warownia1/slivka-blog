---
layout: post
title: Jalview dialogs overhaul
date: 2021-03-03
---

The very nature of dialog window is to block access to the
underlying windows and wait for the user's response. Java
swing (and awt) blocks the current event dispatch thread when
the dialog window is shown and returns the selected option
when it's closed. This obviously doesn't play well with javascript's
single-threaded nature.

SwingJS works around this issue by assuming that the parent
component implements `PropertyChangeListener` and sends it
a `PropertyChangeEvent` when the dialog window is closed.
In Jalview, on the other hand, we use a `JvOptionPane` which
subclasses `JOptionPane` and implements `PropertyChangeListener`.
If javascript is detected as a platform, we pretend that the
parent window is the option pane object itself (even though the
purpose of the JOptionPane is to be the child of the dialog window).
Then, the code that would be normally executed after the dialog
is wrapped in a `Runnable` and added to the option pane to be
executed as a callback.

# Improvement suggestion

The current approach requires taking special care of two scenarios:
blocking dialogs in Java and callback-based in javascript.
However, the code making use of dialog windows should be platform
agnostic and work the same way in both cases. As the blocking
solution is pretty much impossible to implement in JS it's much
easier to use callbacks in Java.

Since the dialog window returns the value (the return is just deferred)
and we would like to preserve linearity of the code, it seems natural
to use `CompletionStage`s and/or `Future`s as the basis for the
asynchronous dialogs. The method displaying the dialog window
(either blocking or not) would return the `CompletionStage`
that would be completed with the selected option.
It allows methods to chain and return the completion stages
themselves resulting in a clean, easy to follow and unambiguous
code.

# Issues

Even though the implementation of the proposed solution is
relatively easy in pure Java, integrating it with swingjs is much
more challenging due to its current way of returning values
from dialog windows. Making quite significant changes to the
swingjs dialog mechanisms might be required.
