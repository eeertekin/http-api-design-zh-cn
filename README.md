# HTTP API 设计指南

## 概述

该指南讲解了一系列 TTP+JSON API 设计经验。这些经验最初来自 
[Heroku 平台 API](https://devcenter.heroku.com/articles/platform-api-reference) 
的实践。

该指南对此 API 进行了补充，并且对 Heroku 的新的内部 API 起到了指导作用。
我们希望在 Heroku 之外的 API 设计者也会对此感兴趣。

本文的目标是在保持一致性，且关注业务逻辑的同时，避免设计歧义。我们一直在寻找
_一种良好的、一致的、文档化的方法_来设计 API，但没必要是_唯一的/理想化的方法_。

本文假设读者已经对 HTTP+JSON API 的基本知识有所了解，
因此不会在指南中涵盖所有的基础概念。

欢迎对该指南给与[贡献](CONTRIBUTING.md)。

## 目录

* [基础](#基础)
  *  [必须使用 TLS](#必须使用-tls)
  *  [用 Accept 头指定版本](#用-accept-头指定版本)
  *  [利用 Etag 支持缓存](#利用-etag-支持缓存)
  *  [通过 Request-Id 跟踪请求](#通过-request-id-跟踪请求)
  *  [使用 Content-Range 进行分页](#使用-content-range-进行分页)
* [请求](#请求)
  *  [返回适当的状态码](#返回适当的状态码)
  *  [尽可能提供完整的资源](#尽可能提供完整的资源)
  *  [允许 JSON 编码的请求体](#允许-json-码的请求体)
  *  [使用一致的路径格式](#使用一致的路径格式)
  *  [小写的路径和属性](#小写的路径和属性)
  *  [为了方便支持非 id 的引用](#为了方便支持非-id-的引用)
  *  [最少的路径嵌套](#最少的路径嵌套)
* [Responses](#responses)
  *  [Provide resource (UU)IDs](#provide-resource-uuids)
  *  [Provide standard timestamps](#provide-standard-timestamps)
  *  [Use UTC times formatted in ISO8601](#use-utc-times-formatted-in-iso8601)
  *  [Nest foreign key relations](#nest-foreign-key-relations)
  *  [Generate structured errors](#generate-structured-errors)
  *  [Show rate limit status](#show-rate-limit-status)
  *  [Keep JSON minified in all responses](#keep-json-minified-in-all-responses)
* [Artifacts](#artifacts)
  *  [Provide machine-readable JSON schema](#provide-machine-readable-json-schema)
  *  [Provide human-readable docs](#provide-human-readable-docs)
  *  [Provide executable examples](#provide-executable-examples)
  *  [Describe stability](#describe-stability)

### 基础

#### 必须使用 TLS

必须使用 TLS 来访问 API，没有例外。任何试图阐明或解释什么时候用它合适，
什么时候用它不合适都是徒劳。让任何请求都需要使用 TLS。

对于非 TLS 请求都只响应 `403 Forbidden`。由于马虎的/恶意的客户端行为无法提供任何明确的保障，所以不建议使用重定向。
重定向的客户端使得服务器的流量成倍增长，并且会在第一次调用的时候让敏感的数据暴露出来，使得 TLS 不起作用。

#### 用 Accept 头指定版本

从一开始就对 API 添加版本。使用 `Accept` 头和自定义的内容类型来指定版本，例如：

```
Accept: application/vnd.heroku+json; version=3
```

最好不要用默认的版本，让客户端明确指出它们需要使用的版本。

#### 利用 Etag 支持缓存

在所有响应中包含 `ETag` 头，用以标识返回资源的特定版本。
用户应当可以在随后的请求中，通过在 `If-None-Match` 头中指定该值来检查过期。

#### 通过 Request-Id 跟踪请求

在每个 API 响应中包含 `Request-Id` 头，并附加一个 UUID 值。
如果服务器和客户端都对该值进行了记录，那么在跟踪和调试请求的时候会非常有用。

#### 使用 Content-Range 进行分页

对任何响应都进行分页，使得大量数据容易被处理。
使用 `Content-Range` 头来传递分页请求。参阅 [Heroku Platform API on Ranges](https://devcenter.heroku.com/articles/platform-api-reference#ranges) 中的例子来了解请求和响应的头、状态码、上限、排序和跳转的细节。

### 请求

#### 返回适当的状态码

对每一个请求都返回适当的 HTTP 状态码。根据本指南，成功的响应当使用以下代码：

* `200`: 对于 `GET` 以及完全同步的 `DELETE` 或 `PATCH` 的请求成功时
* `201`: 对于完全同步的 `POST` 请求成功时
* `202`: 对于异步的 `POST`、`DELETE` 或 `PATCH` 请求被接受
* `206`: `GET` 请求成功，不过只有部分内容被返回：参阅[前面关于分页的内容](#使用-content-range-进行分页)

在使用身份验证与身份验证错误码时务必当心：

* `401 Unauthorized`: 由于用户未进行身份验证，所以请求失败
* `403 Forbidden`: 由于用户无权对特定资源进行访问，所以请求失败

当遇到错误的时候，需要返回合适的代码里提供附加的信息：

* `422 Unprocessable Entity`: 请求可以被解析，但包含了错误的参数
* `429 Too Many Requests`: 请求达到频度限制，稍候再试
* `500 Internal Server Error`: 服务器发生了一些错误，检查状态站点或提交一个 issue

参阅 [HTTP response code spec](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)
了解用户错误与服务器错误的情况下的状态码。

#### 尽可能提供完整的资源

在可能的情况下，在响应中提供完整的资源（例如对象和其所有属性）。
在 200 和 201 响应中提供完整的资源，包括 `PUT`/`PATCH` 和 `DELETE` 请求，例如：

```
$ curl -X DELETE \  
  https://service.com/apps/1f9b/domains/0fd4

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
...
{
  "created_at": "2012-01-01T12:00:00Z",
  "hostname": "subdomain.example.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

202 响应将不会包含完整的资源，例如：

```
$ curl -X DELETE \  
  https://service.com/apps/1f9b/dynos/05bd

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

#### 允许 JSON 编码的请求体

对于 `PUT`/`PATCH`/`POST` 允许使用 JSON 编码的请求体，可以看作是对表单数据的替换或补充。
这与 JSON 编码的响应体对称，例如：

```
$ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'

{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "name": "demoapp",
  "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
  },
  ...
}
```

#### 使用一致的路径格式

##### 资源名

使用附带版本的资源名称，除非该资源在系统中仅有一个实例（例如，在大多数系统里，一个给定的用户只能有一个账户）。
这与引用特定资源的方法一致。

##### 操作

对于个别无须特定操作的资源，宁可使用直接的布局。而需要特定操作的情况下，
将其放置在标准的 `actions` 前缀后，来描述它们：

```
/resources/:resource/actions/:action
```
例如：

```
/runs/{run_id}/actions/stop
```

#### 小写的路径和属性

使用小写的、横线分隔的路径名称，与主机名一致，例如：

```
service-api.com/users
service-api.com/app-setups
```
属性也小写，但是使用下划线分隔，这样属性名在 JavaScript 里无须转义，例如：

```
service_class: "first"
```

#### 为了方便支持非 id 的引用

在某些情况下，让最终用户提供 ID 来标识一个资源可能不是那么方便。
例如，用户可能想的是 HeroKu 的应用名称，但是那个应用可能是用 UUID 标识的。
在这种情况里，可能需要同时接受 ID 和名称，例如：

```
$ curl https://service.com/apps/{app_id_or_name}
$ curl https://service.com/apps/97addcf0-c182
$ curl https://service.com/apps/www-prod
```
不要仅接受名字，而将 ID 排除在外。

#### 最少的路径嵌套

在数据模型中有着父子嵌套关系的资源，路径可能会深层嵌套，例如：

```
/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}
```
限制嵌套的深度，让资源相对于根路径来定位。使用嵌套来表示域集合。
例如，上面的例子中 dyno 属于一个 app，app 属于一个 org：

```
/orgs/{org_id}
/orgs/{org_id}/apps
/apps/{app_id}
/apps/{app_id}/dynos
/dynos/{dyno_id}
```

### 响应

#### 提供资源的 (UU)ID

Give each resource an `id` attribute by default. Use UUIDs unless you
have a very good reason not to. Don’t use IDs that won’t be globally
unique across instances of the service or other resources in the
service, especially auto-incrementing IDs.

Render UUIDs in downcased `8-4-4-4-12` format, e.g.:

```
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

#### 提供标准的时间戳

Provide `created_at` and `updated_at` timestamps for resources by default,
e.g:

```json
{
  ...
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T13:00:00Z",
  ...
}
```

These timestamps may not make sense for some resources, in which case
they can be omitted.

#### 使用 ISO8601 格式化的 UTC 时间

Accept and return times in UTC only. Render times in ISO8601 format,
e.g.:

```
"finished_at": "2012-01-01T12:00:00Z"
```

#### 嵌套的键关系

Serialize foreign key references with a nested object, e.g.:

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  ...
}
```

Instead of e.g:

```json
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  ...
}
```

This approach makes it possible to inline more information about the
related resource without having to change the structure of the response
or introduce more top-level response fields, e.g.:

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0...",
    "name": "Alice",
    "email": "alice@heroku.com"
  },
  ...
}
```

#### 生成结构化的错误

Generate consistent, structured response bodies on errors. Include a
machine-readable error `id`, a human-readable error `message`, and
optionally a `url` pointing the client to further information about the
error and how to resolve it, e.g.:

```
HTTP/1.1 429 Too Many Requests
```

```json
{
  "id":      "rate_limit",
  "message": "Account reached its API rate limit.",
  "url":     "https://docs.service.com/rate-limits"
}
```

Document your error format and the possible error `id`s that clients may
encounter.

#### 显示请求频度限制的状态

Rate limit requests from clients to protect the health of the service
and maintain high service quality for other clients. You can use a
[token bucket algorithm](http://en.wikipedia.org/wiki/Token_bucket) to
quantify request limits.

Return the remaining number of request tokens with each request in the
`RateLimit-Remaining` response header.

#### 在所有请求中都保持 JSON 简洁

Extra whitespace adds needless response size to requests, and many
clients for human consumption will automatically "prettify" JSON
output. It is best to keep JSON responses minified e.g.:

```json
{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z", "created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}
```

Instead of e.g.:

```json
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "last_login": "2012-01-01T12:00:00Z",
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

You may consider optionally providing a way for clients to retreive 
more verbose response, either via a query parameter (e.g. `?pretty=true`)
or via an `Accept` header param (e.g.
`Accept: application/vnd.heroku+json; version=3; indent=4;`).

### 辅助

#### 提供机器可识别的 JSON schema

Provide a machine-readable schema to exactly specify your API. Use
[prmd](https://github.com/interagent/prmd) to manage your schema, and ensure
it validates with `prmd verify`.

#### 提供可读的文档

Provide human-readable documentation that client developers can use to
understand your API.

If you create a schema with prmd as described above, you can easily
generate Markdown docs for all endpoints with with `prmd doc`.

In addition to endpoint details, provide an API overview with
information about:

* Authentication, including acquiring and using authentication tokens.
* API stability and versioning, including how to select the desired API
  version.
* Common request and response headers.
* Error serialization format.
* Examples of using the API with clients in different languages.

#### 提供可执行的例子

Provide executable examples that users can type directly into their
terminals to see working API calls. To the greatest extent possible,
these examples should be usable verbatim, to minimize the amount of
work a user needs to do to try the API, e.g.:

```
$ export TOKEN=... # acquire from dashboard
$ curl -is https://$TOKEN@service.com/users
```

If you use [prmd](https://github.com/interagent/prmd) to generate Markdown
docs, you will get examples for each endpoint for free.

#### 对稳定度进行描述

Describe the stability of your API or its various endpoints according to
its maturity and stability, e.g. with prototype/development/production
flags.

See the [Heroku API compatibility policy](https://devcenter.heroku.com/articles/api-compatibility-policy)
for a possible stability and change management approach.

Once your API is declared production-ready and stable, do not make
backwards incompatible changes within that API version. If you need to
make backwards-incompatible changes, create a new API with an
incremented version number.

