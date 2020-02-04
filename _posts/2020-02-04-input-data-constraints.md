---
layout: post
title: Specifying additional input data constraints
date: "2020-02-04"
---

Having a generic file field is not particularly useful since that's the content of the file that is important and needs to have the constraints to be put on rather than the file itself. Additionally, external clients needs to be able to properly interpret the input file fields, their content, type, format etc. In case of bioinformatic tools, a fasta file format might have individual validation rules such as the number or length of the sequences, presence of gaps, sequence names. These information need to be available on the client-side, so that the client can properly format and prepare the content, and on the server side, so the proper validation mechanisms can be applied.
In this post I'm going to discuss several different approaches to the problem, their advantages and disadvantages.


## Parametrising the media type (mimetype)

Accorfing to [RFC 2045](https://tools.ietf.org/html/rfc2045), the Content-Type header is `<type>/<subtype>[[; attribute=value] ...]`.
We can use the type and subtype to indicate the general type of the file e.g. text/plain, text/fasta and additional parameter to store extra information about this type which can be optionally interpreted by the client.

### Example configuration

```yaml
label: Input file
value:
  type: file
  required: yes
  media-type: application/fasta; seq-count=10; seq-len=150; gaps=no
```

### Advantages

- Client applications doesn't need to be modified. This method uses already existing elements.
- Interpreting the additional parameters is optional and up to the client software. No need to write separate clients or client extensions. Attributes which are not understood can be ignored.
- Fields can contain any arbitrary information according to the media type.
- Server side media type validators are already there, extra attributes can be passed as additional validation parameters. Adding custom validators is supported out of the box.

### Disadvantages

- Media type becomes a lengthy parameter containing more information than it really should.
- There is no validation whether the provided arguments are valid. Any typos would silently pass.
- Parameter names are arbitrary, system administrators may use different conventions making client applications incompatible with any but one specific Slivka instance.


## Providing additional data in the description

Additional data type and format parameters can be included in the field description e.g. in the yaml format at the end of description text.

### Advantages

- The same as with media-type parameters

### Disadvantages

- The same as with media-type parameters plus...
- Mixing machine-readable data with human-readable text.
- File type information ends up split into two different places.


## Extending the existing form fields

A typical object-oriented approach is to extend the base class/prototype whenever a new functionality is needed. This is how the input fields are currently modelled. A generic `BaseField` is extended by more specialised classes adding additional validation layers and parameters.

### Example configuration

```yaml
label: Input file
value:
  type: file.fasta
  required: yes
  media-type: application/fasta
  seq-count: 10
  seq-length: 150
  gaps-allowed: no
```

This configuration would map to a corresponding field class:

```python
class FastaFileField(FileField):
  def __init__(self, name, seq_count=None,
               seq_length=None, gaps_allowed=True, **kwargs):
    super().__init__(name, **kwargs)
    self.seq_len = seq_len
    self.seq_count = seq_count
    ...
    self.validators.append(partial(
      seqence_length_validator, seq_len
    ))
    ...

  def to_python(self, value):
    value = super().to_python(value)
    return Bio.SeqIO(value.stream)

  def __json__(self):
    j = super().__json__()
    j['type'] = 'file.fasta'
    j['seq-length'] = self.seq_len
    ...

  def to_cmd_parameter(self, value: Bio.Seq):
    # ???
```

### Advantages

- The idea fits nicely into the existing codebase. Little refactoring needs to be done on the server side.
- Service administrators may add their own fields to the application by extending the existing fields.
- Fields can easily validate any data provided to them by adding validators.
- Very strict rules and validation of configuration parameters. Errors and typos will be immediately reported and parametres definitions will be uniform among all slivka instances.

### Disadvantages

- Validation and type conversion (`validate`, `to_python`, `to_cmd_parameter`) will not work when the derived class operates on different data types. E.g. file field operates on `FileWrapper` while fasta file field uses `Bio.Seq` whose validators are incompatible with one another.
- Additional work needs to be put to create and maintain slivka extensions for additional input types and validators.
- Client application will not be able to properly parse unknown fields. Clients would either need to be built for a specific slivka project or allow modularity (additional modules to maintain for each client).
- Including most popular file types and allowing community to add to slivka project would alleviate some of the maintenance effort.
