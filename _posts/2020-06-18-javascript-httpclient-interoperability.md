---
layout: post
title: HttpClient and javascript interoperability
date: "2020-06-18"
---

Slivka client will be soon transpiled to javascript. This raises several
issues with third-party dependencies, especially org.apache.http.client.
The Apache http library is quite large and unnecessary as javascript
and web browsers have their own, native methods for http connections.
On the other hand, it is currently an industry stardard for Java applications.


Why get rid of Apache HttpClient?
---------------------------------
 - large package of which only small bit is being used


Why keep Apache HttpClient?
---------------------------
 - well establised and ready-to-use http client
 - supporting https
 - maintanance is someone else's responsibility


Compromise
----------

We can create our own minimal HttpClient interface supporting GET and POST
which will be implemented by Apache HttpClient wrapper on JVM and jQuery
wrapper on JS
