---
layout: post
title: REST API changes
date: 2021-05-17
---

Along with the update to the configuration files, some changes
will be introduced to the REST API as well. They aim to improve
the structure of resources under the endpoints, provide more
information to the clients, change property names to conform to the
[Google JSON style guide](https://google.github.io/styleguide/jsoncstyleguide.xml)
and properly use HTTP headers to pass the information about resource
locations.

# Resources

First, we are going to specify the resources that the REST API will
operate with; they are: *service*, *job* and *file*.

## Services

Service represents an available service on the server that jobs
can be submitted for. The changes include improved parameter names,
extra details such as description, author, version and license,
including input parameters and health status in the resource instead
of having a separate endpoint.
The structure of the service resource is following:

~~~json
{
  "@url": "<resource location>",
  "id": "<identifier (previously name)>",
  "name": "<human-readable name (previously label)>",
  "description": "<description>",
  "author": "<author of the tool>",
  "version": "<software version>",
  "license": "<software license>",
  "classifiers": [
    "<classifier/tag>"
  ],
  "parameters": [
     "<ParameterObject>"
  ],
  "status":{
    "status": "OK | WARNING | DOWN",
    "errorMessage": "<details>",
    "timestamp": "<report time>"
  }
}
~~~

The `ParameterObject` properties depend on the parameter type and
are dictated by its corresponding field class.
Every parameter has the following properties:

~~~json
{
  "id": "<parameter id>",
  "name": "<human readable name>",
  "description": "<description>",
  "type": "<type>",
  "required": "<is required>",
  "array": "<is array or single value>",
  "default": "<default value>"
}
~~~

For the built-in types, there are additional parameters specific
to that type.

~~~json
{
  "type": "integer",
  "min": "<lower bound>",
  "max": "<upper bound>",
}
~~~

~~~json
{
  "type": "float",
  "min": "<lower bound>",
  "max": "<upper bound>",
  "minExclusive": "<bool>",
  "maxExclusive": "<bool>",
}
~~~

~~~json
{
  "type": "text",
  "minLength": "<minimum length>",
  "maxLength": "<maximum length>",
}
~~~

~~~json
{
  "type": "flag",
}
~~~

~~~json
{
  "type": "choice",
  "choices": {
    "<choice>": "<mapped value>"
  },
}
~~~

~~~json
{
  "type": "file",
  "mediaType": "<accepted media-type>",
  "mediaTypeParameters": [
    "<additional type hints>"
  ],
  "extensions": [
    "<extension hints>"
  ],
  "maxSize": "<file size>"
}
~~~

## Jobs

Job resource is now extended with extra information about the
parameters the job was submitted with and submission and completion
time. Similar to the *ServiceResource*, we added a *@url* property
pointing to the resource location.
The *JobResource* structure is following:

~~~json
{
  "@url": "<resource location>",
  "id": "<job identifier>",
  "service": "<service the job was submitted to>",
  "parameters": {
    "<parameter id>": "<parameter value>"
  },
  "submissionTime": "<ISO 8601 datetime>",
  "completionTime": "<ISO 8601 datetime>",
  "status": "<current job status>"
}
~~~

## Files

File resource remains mostly unchanged. Files are sub-resources of
jobs (with exception of the uploaded files which are sub-resources
of a phony "upload" job) and will be accessed as such. The file 
content is accessible at a different location provided by a proxy
server, not slivka.

~~~json
{
  "@url": "<resource location>",
  "@content": "<file content url>",
  "jobId": "<parent job id>",
  "path": "<file path within the job>",
  "label": "<human-readable label>",
  "mediaType": "<media type>"
}
~~~

# Endpoints

As the resources now contain more information about the server-side
object the endpoint are less fragmented and their number is reduced.
We updated some endpoints to better fit to the RESTful architecture.
The main changes are:

- */service/{service}*, */service/{service}/presets*,
  */service/{service}/presets/{preset}* combined into one resource
  under */service/{service}* endpoint.
- POST */service/{service}* changed to */service/{service}/jobs*
  since it creates a job resource. However, the created job is also
  not a service's sub-resource so some changes might be needed.
- */tasks* renamed to */jobs*
- File resources are accessed under */job/{jid}/files* and
  */job/{jid}/files/{path}*. However, uploaded
  files do not fit into this schema. Additionally, using paths is
  discouraged in REST and some alternative identifier might need to be
  implemented for example blake2 hashes of file paths or
  paths with slashes replaced.
