---
layout: post
title: Inputs and Outputs of the Services
date: "2019-12-05 12:59 +0000"
---

Bioinformatic tools can accept variuos data formats and media-types.
Often the same media type may contain very different data. Applications
using Slivka system such as Jalview must be able to recognise the
input and output formats in order to properly classify the services
and format input data correctly.
For example, FASTA format may contain both amino acid and protein
sequences. Additionally, some programs may require only one
sequence to be present while the other may need multiple or aligned sequences.
Therefore it's necessary to provide additional metadata about the input/output
data of the services.

Service classifiers
-------------------
Some information about the input and output data can be contained in the
list of service classifiers. It would help other programs to group the
services according to the data they process. Applications like Jalview
may enable or disable the web services depending on the sequences the
user is currently working with. For bioinformatic tools EDAM ontology can
be used to specify data types.

Examples of data types specification: 

- `Input :: Data :: Alignment :: Sequence alignment :: Sequence alignment (nucleic acid)`
- `Input :: Data :: Alignment :: Protein alignment :: Sequence alignment (protein)`
- `Input :: Data :: Sequence :: Nucleic acid sequence :: RNA sequence`
- `Output :: Data :: Sequence features :: Protein features`

Examples of data format specification: 

- `Input :: Format :: Textual format :: Fasta-like (text) :: FASTA`
- `Input :: Format :: Textual format :: HMMER-aln`
- `Output :: Format :: JSON`

Alternatively, instead of shoving all inputs and output metadata to the service
classifiers list, they can be defined separately.   

Recognising input fields
------------------------
Currently, all tools in the slivka-bio repository take only one fasta file as
and input and produce either one fasta or clustal file or jalview annotations
and features files. Jalview can recognise them by investigating their media types.
However, it's far from reliable as some services may take and produce multiple
files with the same media type.

We can create a naming convention which all services need to comply with in order
to be picked up by Jalview properly. Currently all log files must be named "log"
and error log must be named "error-log" and both have "text/plain" media type.

File names used currently:

- alignment -- multiple sequence alignment output/input
- sequence(s) -- sequence file in the type specified in media type
- jalview-features -- jalview features file
- jalview-annotations -- jalview annotations file

