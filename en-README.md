# Translated with Google Translate (not bad,ha ?)

# HTTP API Design Guide

## Overview

The guide explains the series of HTTP + JSON design experience API. Originally from these experiences
[Heroku platform API] (https://devcenter.heroku.com/articles/platform-api-reference) 
Practice.

The guide has been added to this API, and on Heroku's new internal API has played a guiding role.
We hope that in addition to Heroku's API designers will be interested.

Goal of this paper is to maintain consistency and focus on business logic, to avoid ambiguity design. We are always looking
_ Kind of good, consistent and documented approach _ to design API, but not necessarily the only _ / _ idealized way.

This article assumes that the reader has some knowledge of the basics of HTTP + JSON API's,
It will not cover all of the basic concepts in the Guide.

Welcome to the guidelines given to [contribute] (CONTRIBUTING.md).

## Table of Contents

* [Base] (# basis)
  * [Must use TLS] (# must use -tls)
  * [With the Accept header specifies version] (# specified version with -accept- head)
  * [Use Etag support caching] (# use -etag- supports caching)
  * (# Request by -request-id- tracking) [By Request-Id tracking request]
  * [Use Content-Range Paging] (# Use -content-range- paging)
* [Request] (# request)
  * [Return the appropriate status code] (# return the appropriate status code)
  * [Provide complete resources as] (# possible to provide a complete resource)
  * [Allow JSON-encoded request body] (# allow -json- request body code)
  * [Use a consistent path format] (# using a consistent path format)
  * [Lowercase path and attributes] (# lowercase path and attributes)
  * [In order to facilitate support non-id references] (# In order to facilitate support non -id- references)
  * [Minimal path nested] (# minimal path nested)
* [Response] (# Response)
  * [For resources (UU) ID] (# of resources -uuid)
  * [Provide standard timestamp] (# provides a standard timestamp)
  * [ISO8601 formatted using UTC time] (# Use -iso8601- formatted -utc- time)
  * [Nested foreign key relationships] (# nested foreign key relationships)
  * [Generate structured error] (# generate structured error)
  * [Display request frequency limit state] (# Display status request frequency limits)
  * [In all requests remain JSON concise] (# in all requests are maintained -json- concise)
* [Auxiliary] (# Auxiliary)
  * [Provide machine-readable JSON schema] (# provide machine-readable -json-schema)
  * [Provide readable documentation] (# provides a readable document)
  * [Provide executable examples] (# provides executable examples)
  * [Description of the degree of stability] (# of stability described)

### Basis

#### Must use TLS

TLS must be used to access the API, without exception. Any attempt to clarify or explain when to use it appropriate,
When to use it inappropriate futile. Make any request to use TLS.

Only respond to a request for non-TLS `403 Forbidden`. Because of sloppy / Client malicious behavior can not provide any explicit guarantee, so I do not recommend the use of redirection.
Redirect the client makes server traffic grow exponentially, and will at the time of the first call to make sensitive data exposed, making TLS does not work.

#### With the Accept header specifies version

From the outset, the API to add version. Use `Accept` head and custom content types to specify the version, for example:

`` `
Accept: application / vnd.heroku + json; version = 3
`` `

Best not to use the default version, so clear that they need the client version used.

#### Supports caching using Etag

Contains `ETag` head in all responses to identify specific version return resources.
Users should be able to in a subsequent request by specifying the value in the `If-None-Match` head to check the date.

#### By Request-Id tracking requests

Contains `Request-Id` head in each API response, and attach a UUID value.
If the server and client are the values ​​were recorded, then the tracing and debugging can be very useful when requested.

#### Use Content-Range Paging

Any response will be paged, making large amounts of data easily be processed.
Use `Content-Range` head pass paging request. See [Heroku Platform API on Ranges] (https://devcenter.heroku.com/articles/platform-api-reference#ranges) examples to understand the request and response headers, status code, the ceiling, sort and jump details.

### Request

#### Return the appropriate status code

For each request to return the appropriate HTTP status code. According to the guidelines, a successful response when using the following code:

* `200 ': a request for` GET` and `DELETE` fully synchronized when the success or` PATCH`
When fully synchronized for `POST` request success: *` 201`
* `202`: For asynchronous` POST`, `DELETE` or` PATCH` request is accepted
* `206`:` GET` request was successful, but only in part be returned: see [content on the front page of] (# Use -content-range- paging)

When using the identity verification and authentication error code must be careful:

* `401 Unauthorized`: Because the user is not authenticated, so the request fails
* `403 Forbidden`: Because users do not have access to specific resources, so the request fails

When an error is encountered when the need to return the appropriate code to provide additional information:

* `422 Unprocessable Entity`: requests can be resolved, but contains the wrong parameter
* `429 Too Many Requests`: request reaches the frequency limit, try again later
* `500 Internal Server Error`: Some server error occurred, check the status of the site or to submit an issue

See [HTTP response code spec] (http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)
Understand user error and the server status code in case of error.

#### Provides complete resources as

Where possible, and to provide a complete resource (such as an object and all of its properties) in the response.
Resources to provide a complete 200 and 201 responses, including `PUT` /` PATCH` and `DELETE` request, for example:

`` `
$ Curl -X DELETE \  
  https://service.com/apps/1f9b/domains/0fd4

HTTP / 1.1 200 OK
Content-Type: application / json; charset = utf-8
...
{
  "Created_at": "2012-01-01T12: 00: 00Z",
  "Hostname": "subdomain.example.com",
  "Id": "01234567-89ab-cdef-0123-456789abcdef",
  "Updated_at": "2012-01-01T12: 00: 00Z"
}
`` `

202 does not contain a complete response resources, such as:

`` `
$ Curl -X DELETE \  
  https://service.com/apps/1f9b/dynos/05bd

HTTP / 1.1 202 Accepted
Content-Type: application / json; charset = utf-8
...
{}
`` `

#### Allows JSON-encoded request body

For `PUT` /` PATCH` / `POST` allowed to use JSON-encoded request body, can be seen to replace or supplement the data on the form.
This JSON-encoded response body symmetry, for example:

`` `
$ Curl -X POST https://service.com/apps \
    -H "Content-Type: application / json" \
    -d '{"name": "demoapp"}'

{
  "Id": "01234567-89ab-cdef-0123-456789abcdef",
  "Name": "demoapp",
  "Owner": {
    "Email": "username@example.com",
    "Id": "01234567-89ab-cdef-0123-456789abcdef"
  }
  ...
}
`` `

#### Using the same path format

##### Resource name

Use the supplied version of the resource name, unless the resource is only one instance in the system (for example, in most systems, a given user can only have one account).
This is consistent with the methods refer to a specific resource.

##### Operation

For individual resources without specific operations, rather use direct layout. The situation requires a specific operation, the
After placing it in a standard prefix `actions` to describe them:

`` `
/ Resources /: resource / actions /: action
`` `
For example:

`` `
/ Runs / {run_id} / actions / stop
`` `

#### Lowercase path and attributes

Use lowercase, dash-separated path name, consistent with the host name, for example:

`` `
service-api.com/users
service-api.com/app-setups
`` `
Property is also lower, but the use underscores to separate, so the property name in JavaScript without escape, for example:

`` `
service_class: "first"
`` `

#### Non-id references in order to facilitate support

In some cases, allow the end user to provide ID to identify a resource may not be so easy.
For example, a user might want is HeroKu application name, but that application may be identified with a UUID.
In this case, there may need to accept the ID and name, for example:

`` `
$ Curl https://service.com/apps/{app_id_or_name}
$ Curl https://service.com/apps/97addcf0-c182
$ Curl https://service.com/apps/www-prod
`` `
Do not accept only the name, and the ID is excluded.

#### Nesting path with the least

In the data model has a father and son relationship nested resources, the path may be deeply nested, for example:

`` `
/ Orgs / {org_id} / apps / {app_id} / dynos / {dyno_id}
`` `
Limit nesting, so resources relative to the root path to locate. Use nested to represent the domain collection.
For example, the above example dyno belong to an app, app belong to an org:

`` `
/ Orgs / {org_id}
/ Orgs / {org_id} / apps
/ Apps / {app_id}
/ Apps / {app_id} / dynos
/ Dynos / {dyno_id}
`` `

### Response

#### For resources (UU) ID

To each resource a default `id` property. Unless there is a good reason, or otherwise use the UUID it.
Do not use those resources in other cross-server instances or services are not globally unique ID, in particular, do not use auto-incremented ID.

The UUID is defined as lowercase `8-4-4-4-12` format, for example:

`` `
"Id": "01234567-89ab-cdef-0123-456789abcdef"
`` `

#### Provides a standard timestamp

For resources provided by default `created_at` and` updated_at` timestamp, for example:

`` `Json
{
  ...
  "Created_at": "2012-01-01T12: 00: 00Z",
  "Updated_at": "2012-01-01T13: 00: 00Z",
  ...
}
`` `
These times, it may be said for certain resources no practical significance in these cases they can be omitted.

#### ISO8601 formatted using UTC time

Only use UTC time received or returned. Expression of time with ISO8601 format, for example:

`` `
"Finished_at": "2012-01-01T12: 00: 00Z"
`` `

#### Nested foreign key relationships

With nested objects to express foreign key relationships, such as:

`` `Json
{
  "Name": "service-production",
  "Owner": {
    "Id": "5d8201b0 ..."
  }
  ...
}
`` `

Instead of:

`` `Json
{
  "Name": "service-production",
  "Owner_id": "5d8201b0 ...",
  ...
}
`` `

This mechanism allows embedded information more relevant resources without having to modify the data structure response, or introduce more top fields, such as:

`` `Json
{
  "Name": "service-production",
  "Owner": {
    "Id": "5d8201b0 ...",
    "Name": "Alice",
    "Email": "alice@heroku.com"
  }
  ...
}
`` `

#### Generates structured error

Generate consistent, structured error response. Including machine-readable error `id`, human-readable error` message `
And an optional `url` guide customers to understand information and solutions about errors even further, for example:

`` `
HTTP / 1.1 429 Too Many Requests
`` `

`` `Json
{
  "Id": "rate_limit",
  "Message": "Account reached its API rate limit.",
  "Url": "https://docs.service.com/rate-limits"
}
`` `
Error `id` malformed and client may encounter written documentation.

#### Displays the status of the request frequency limit

Limit the frequency of the client's request to protect services and keep other clients a higher quality of service. You can use
[Token bucket algorithm] (http://en.wikipedia.org/wiki/Token_bucket) to verify the frequency of requests.

Each request will respond with a `RateLimit-Remaining` head returns the requested number of remaining requests token.

#### In all requests are simple to maintain JSON

Additional whitespace will increase the size of the response, which is unnecessary, and many artificial clients will automatically "beautification" JSON output.
So it is best kept to a minimum so that JSON responses, such as:

`` `Json
{"Beta": false, "email": "alice@heroku.com", "id": "01234567-89ab-cdef-0123-456789abcdef", "last_login": "2012-01-01T12: 00: 00Z" , "created_at": "2012-01-01T12: 00: 00Z", "updated_at": "2012-01-01T12: 00: 00Z"}
`` `

Instead of:

`` `Json
{
  "Beta": false,
  "Email": "alice@heroku.com",
  "Id": "01234567-89ab-cdef-0123-456789abcdef",
  "Last_login": "2012-01-01T12: 00: 00Z",
  "Created_at": "2012-01-01T12: 00: 00Z",
  "Updated_at": "2012-01-01T12: 00: 00Z"
}
`` `
Clients may also be considered for alternative ways to increase the output of a more detailed response, whether by request parameters (such as `? Pretty = true`)
Or by `Accept` head parameters (such as` Accept: application / vnd.heroku + json; version = 3; indent = 4; `).

### Auxiliary

#### To provide machine-readable JSON schema

Provide machine-readable schema to clear your API. Use [prmd] (https://github.com/interagent/prmd)
To manage these patterns, and use `prmd verify` to verify.

#### Provides a readable document

Readable documentation provided to allow developers to understand your client's API.

If prmd mentioned above create a schema, you can easily through the 
`Prmd doc` for all interfaces to create Markdown documents.

As an additional interface details, provide the following information for the API Overview:

* Authentication, including access to and use of authentication token;
* API degree of stability and release conditions, including how to select the target version of API;
* GM's request and response headers;
* Wrong format;
* Example using a different client languages.

#### Provides an executable examples

The user can provide input to understand the API calls the executable directly in the case of the example terminal.
In order to maximize scalability, these examples should be used for each row,
These users try to reduce the workload of the API, for example:

`` `
$ Export TOKEN = ... # acquire from dashboard
$ Curl -is https: //$TOKEN@service.com/users
`` `

If you use [prmd] (https://github.com/interagent/prmd) to generate Markdown documents,
You can easily get examples for each interface.

#### Describe the stability of the

On the degree of stability of your API will be described, including the maturity and stability of different interfaces.
For example, the use of prototype / development / production logo.

See [Heroku API compatibility policy] (https://devcenter.heroku.com/articles/api-compatibility-policy)
Possible way to understand the stability and change management.

Once the API is defined as suitable for production environments and is stable, do not make any changes would undermine the backward compatibility of the version of the API.
If you need to be backwards incompatible changes, create a new API has a higher version number.
