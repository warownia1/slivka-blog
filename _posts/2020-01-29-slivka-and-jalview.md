---
layout: post
title: Why Slivka and Jalview are not getting along
date: "2020-01-29"
---
Slivka is a middleware providing a configurable and generic way of executing command line programs through its REST API.
The interoperability with various programs on the server side as well as on the client side comes at the cost of being less specialised.
Software like Jalview require more specific interface and, as a result, those two designs doesn't fit into one another.

A brief recall of slivka operation:
> First, Slivka sends a list of available parameters and inputs to the client.
> They include flags, parameters, input data and files mixed together.
> Then, the client sends back values for each of these parameters.

In the simplest case of programmatic access via Jupyter, Python script or raw HTTP a person writing the code is responsible for identifying each parameter and providing it with a proper value.
Things get more difficult when the client is not a human.

First of all, Jalview needs to make a decision which parameters should be filled automatically and which should be passed down to the user, i.e. we can't expect the user to type in the sequences manually, Jalview should do it automatically based on the displayed view.
The main difficulty is that Slivka does not give any hints as of which fields are suitable for automatic completion. At the moment, Jalview assumes all the _file_ type fields to be filled automatically with fasta sequences and any other field to be left for the user to complete.  
**How can Jalview identify input parameters that can be inserted automatically?**

Once we find the parameters which need to be filled in automatically, the data needs to be provided in the correct format. In case of numbers or flags the choice is very limited. On the other hand, generic *file* type doesn't say anything about what data should be contained in that file and how the data should be formatted.
E.g. protein sequences can be respresented in multiple formats such as fasta or pfam.
Similarly, fasta format doesn't indicate the content of the file as it can be one or more aligned or unaligned nucleotide or protein sequences.  
**How indicate data types and format of the input fileds such that they're unambiguous for Jalview, but they're not too Jalview-specific at the same time?**
