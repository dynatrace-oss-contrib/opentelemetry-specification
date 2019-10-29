# Semantic Conventions

_TODO: Remove rule `exclude_rule 'MD025'` from `.mdlstyle.rb` once this document is split and we no longer need 2 top-level headers._

These documents define standard names and values of Resource labels and
Span attributes.

* [Resource Conventions](data-resource-semantic-conventions.md)
* [Span Conventions](#span-conventions)

# Span Conventions

In OpenTelemetry spans can be created freely and it’s up to the implementor to
annotate them with attributes specific to the represented operation. Spans
represent specific operations in and between systems. Some of these operations
represent calls that use well-known protocols like HTTP or database calls.
Depending on the protocol and the type of operation, additional information
is needed to represent and analyze a span correctly in monitoring systems. It is
also important to unify how this attribution is made in different languages.
This way, the operator will not need to learn specifics of a language and
telemetry collected from multi-language micro-service can still be easily
correlated and cross-analyzed.

**Table of contents**

- [HTTP](#http)
  * [Name](#name)
  * [Status](#status)
  * [Common Attributes](#common-attributes)
  * [HTTP client](#http-client)
  * [HTTP server](#http-server)
    + [Definitions](#definitions)
    + [Semantic conventions](#semantic-conventions)
  * [Example](#example)
- [Databases (SQL and NoSQL) client calls](#databases-sql-and-nosql-client-calls)
  * [Connection-level attributes](#connection-level-attributes)
  * [Call-level attributes](#call-level-attributes)
  * [Notes for specific database types](#notes-for-specific-database-types)
- [Remote procedure calls](#remote-procedure-calls)
  * [Attributes](#attributes)
  * [gRPC](#grpc)
    + [Status](#status-1)
    + [Events](#events)
- [Messaging systems](#messaging-systems)
  * [Definitions](#definitions-1)
  * [Conventions](#conventions)
  * [Messaging attributes](#messaging-attributes)
- [General attributes](#general-attributes)
  * [General miscellaneous attributes](#general-miscellaneous-attributes)
  * [General source code attributes](#general-source-code-attributes)
  * [General network connection attributes](#general-network-connection-attributes)
  
## HTTP

This section defines semantic conventions for HTTP client and server Spans.
They can be used for http and https schemes
and various HTTP versions like 1.1, 2 and SPDY.

### Name

Given an [RFC 3986](https://tools.ietf.org/html/rfc3986) compliant URI of the form `scheme:[//host[:port]]path[?query][#fragment]`,
the span name of the span SHOULD be set to the URI path value,
unless another value that represents the identity of the request and has a lower cardinality can be identified
(e.g. the route for server spans; see below).

### Status

Implementations MUST set status if the HTTP communication failed
or an HTTP error status code is returned (e.g. above 3xx).

In the case of an HTTP redirect, the request should normally be considered successful,
unless the client aborts following redirects due to hitting some limit (redirect loop).
If following a (chain of) redirect(s) successfully, the Status should be set according to the result of the final HTTP request.

Don't set a status message if the reason can be inferred from `http.status_code` and `http.status_text` already.

| HTTP code               | Span status code      |
|-------------------------|-----------------------|
| 100...299               | `Ok`                  |
| 3xx redirect codes      | `DeadlineExceeded` in case of loop (see above) [1], otherwise `Ok` |
| 401 Unauthorized ⚠      | `Unauthenticated` ⚠ (Unauthorized actually means unauthenticated according to [RFC 7235][rfc-unauthorized])  |
| 403 Forbidden           | `PermissionDenied`    |
| 404 Not Found           | `NotFound`            |
| 429 Too Many Requests   | `ResourceExhausted`   |
| Other 4xx code          | `InvalidArgument` [1] |
| 501 Not Implemented     | `Unimplemented`       |
| 503 Service Unavailable | `Unavailable`         |
| 504 Gateway Timeout     | `DeadlineExceeded`    |
| Other 5xx code          | `InternalError` [1]   |
| Any status code the client fails to interpret (e.g., 093 or 573) | `UnknownError` |

Note that the items marked with [1] are different from the mapping defined in the [OpenCensus semantic conventions][oc-http-status].

[oc-http-status]: https://github.com/census-instrumentation/opencensus-specs/blob/master/trace/HTTP.md#mapping-from-http-status-codes-to-trace-status-codes
[rfc-unauthorized]: https://tools.ietf.org/html/rfc7235#section-3.1

### Common Attributes

| Attribute name | Notes and examples                                           | Required? |
| :------------- | :----------------------------------------------------------- | --------- |
| `component`    | Denotes the type of the span and needs to be `"http"`. | Yes |
| `http.method` | HTTP request method. E.g. `"GET"`. | Yes |
| `http.url` | Full HTTP request URL in the form `scheme://host[:port]/path?query[#fragment]`. Usually the fragment is not transmitted over HTTP, but if it is known, it should be included nevertheless. | Defined later. |
| `http.target` | The full request target as passed in a [HTTP request line][] or equivalent, e.g. `/path/12314/?q=ddds#123"`. | Defined later. |
| `http.host` | The value of the [HTTP host header][]. When the header is empty or not present, this attribute should be the same. | Defined later. |
| `http.scheme` | The URI scheme identifying the used protocol: `"http"` or `"https"` | Defined later. |
| `http.status_code` | [HTTP response status code][]. E.g. `200` (integer) | If and only if one was received/sent. |
| `http.status_text` | [HTTP reason phrase][]. E.g. `"OK"` | No |
| `http.flavor` | Kind of HTTP protocol used: `"1.0"`, `"1.1"`, `"2"`, `"SPDY"` or `"QUIC"`. |  No |

It is recommended to also use the general [network attributes][], especially `net.peer.ip`. If `net.transport` is not specified, it can be assumed to be `IP.TCP` except if `http.flavor` is `QUIC`, in which case `IP.UDP` is assumed.


[HTTP response status code]: https://tools.ietf.org/html/rfc7231#section-6
[HTTP reason phrase]: https://tools.ietf.org/html/rfc7230#section-3.1.2

### HTTP client

This span type represents an outbound HTTP request.

For an HTTP client span, `SpanKind` MUST be `Client`.

If set, `http.url` must be the originally requested URL,
before any HTTP-redirects that may happen when executing the request.

One of the following sets of attributes is required (in order of usual preference unless for a particular web client/framework it is known that some other set is preferable for some reason; all strings must be non-empty):

* `http.url`
* `http.scheme`, `http.host`, `http.target`
* `http.scheme`, `peer.hostname`, `peer.port`, `http.target`
* `http.scheme`, `peer.ip`, `peer.port`, `http.target`

Note that in some cases `http.host` might be different
from the `peer.hostname`
used to look up the `peer.ip` that is actually connected to.
In that case it is strongly recommended to set the `peer.hostname` attribute in addition to `http.host`.

For status, the following special cases have canonical error codes assigned:

| Client error                | Trace status code  |
|-----------------------------|--------------------|
| DNS resolution failed       | `UnknownError`     |
| Request cancelled by caller | `Cancelled`        |
| URL cannot be parsed        | `InvalidArgument`  |
| Request timed out           | `DeadlineExceeded` |

This is not meant to be an exhaustive list
but if there is no clear mapping for some error conditions,
instrumentation developers are encouraged to use `UnknownError`
and open a PR or issue in the specification repository.

### HTTP server

To understand the attributes defined in this section, it is helpful to read the "Definitions" subsection.

#### Definitions

This section gives a short summary of some concepts
in web server configuration and web app deployment
that are relevant to tracing.

Usually, on a physical host, reachable by one or multiple IP addresses, a single HTTP listener process runs.
If multiple processes are running, they must listen on distinct TCP/UDP ports so that the OS can route incoming TCP/UDP packets to the right one.

Within a single server process, there can be multiple **virtual hosts**.
The [HTTP host header][] (in combination with a port number) is normally used to determine to which of them to route incoming HTTP requests.

The host header value that matches some virtual host is called the virtual hosts's **server name**. If there are multiple aliases for the virtual host, one of them (often the first one listed in the configuration) is called the **primary server name**. See for example, the Apache [`ServerName`][ap-sn] or NGINX [`server_name`][nx-sn] directive or the CGI specification on `SERVER_NAME` ([RFC 3875][rfc-servername]).
In practice the HTTP host header is often ignored when just a single virtual host is configured for the IP.

Within a single virtual host, some servers support the concepts of an **HTTP application**
(for example in Java, the Servlet JSR defines an application as
"a collection of servlets, HTML pages, classes, and other resources that make up a complete application on a Web server"
-- SRV.9 in [JSR 53][];
in a deployment of a Python application to Apache, the application would be the [PEP 3333][] conformant callable that is configured using the
[`WSGIScriptAlias` directive][modwsgisetup] of `mod_wsgi`).

An application can be "mounted" under some **application root**
(also know as *[context root][]* *[context prefix][]*, or *[document base][]*)
which is a fixed path prefix of the URL that determines to which application a request is routed
(e.g., the server could be configured to route all requests that go to an URL path starting with `/webshop/`
at a particular virtual host
to the `com.example.webshop` web application).

Some servers allow to bind the same HTTP application to multiple `(virtual host, application root)` pairs.

> TODO: Find way to trace HTTP application and application root ([opentelemetry/opentelementry-specification#335][])

[PEP 3333]: https://www.python.org/dev/peps/pep-3333/
[modwsgisetup]: https://modwsgi.readthedocs.io/en/develop/user-guides/quick-configuration-guide.html
[context root]: https://docs.jboss.org/jbossas/guides/webguide/r2/en/html/ch06.html
[context prefix]: https://marc.info/?l=apache-cvs&m=130928191414740
[document base]: http://tomcat.apache.org/tomcat-5.5-doc/config/context.html
[rfc-servername]: https://tools.ietf.org/html/rfc3875#section-4.1.14
[ap-sn]: https://httpd.apache.org/docs/2.4/mod/core.html#servername
[nx-sn]: http://nginx.org/en/docs/http/ngx_http_core_module.html#server_name
[JSR 53]: https://jcp.org/aboutJava/communityprocess/maintenance/jsr053/index2.html
[opentelemetry/opentelementry-specification#335]: https://github.com/open-telemetry/opentelemetry-specification/issues/335

#### Semantic conventions

This span type represents an inbound HTTP request.

For an HTTP server span, `SpanKind` MUST be `Server`.

Given an inbound request for a route (e.g. `"/users/:userID?"`) the `name` attribute of the span SHOULD be set to this route.
If the route does not include the application root, it SHOULD be prepended to the span name.

If the route cannot be determined, the `name` attribute MUST be set as defined in the general semantic conventions for HTTP.

| Attribute name | Notes and examples                                           | Required? |
| :------------- | :----------------------------------------------------------- | --------- |
| `http.server_name` | The primary server name of the matched virtual host. This should be obtained via configuration. If no such configuration can be obtained, this attribute MUST NOT be set ( `net.host.name` should be used instead). | [1] |
| `http.route` | The matched route (path template). E.g. `"/users/:userID?"`. | No |
| `http.client_ip` | The IP address of the original client behind all proxies, if known (e.g. from [X-Forwarded-For][]). Note that this is not necessarily the same as `net.peer.ip`, which would identify the network-level peer, which may be a proxy. | No |

[HTTP request line]: https://tools.ietf.org/html/rfc7230#section-3.1.1
[HTTP host header]: https://tools.ietf.org/html/rfc7230#section-5.4
[X-Forwarded-For]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For

**[1]**: `http.url` is usually not readily available on the server side but would have to be assembled in a cumbersome and sometimes lossy process from other information (see e.g. <https://github.com/open-telemetry/opentelemetry-python/pull/148>).
It is thus preferred to supply the raw data that *is* available.
Namely, one of the following sets is required (in order of usual preference unless for a particular web server/framework it is known that some other set is preferable for some reason; all strings must be non-empty):

* `http.scheme`, `http.host`, `http.target`
* `http.scheme`, `http.server_name`, `net.host.port`, `http.target`
* `http.scheme`, `net.host.name`, `net.host.port`, `http.target`
* `http.url`

Of course, more than the required attributes can be supplied, but this is recommended only if they cannot be inferred from the sent ones.
For example, `http.server_name` has shown great value in practice, as bogus HTTP Host headers occur often in the wild.

It is strongly recommended to set `http.server_name` to allow associating requests with some logical server entity.

### Example

As an example, if a browser request for `https://example.com:8080/webshop/articles/4?s=1` is invoked from a host with IP 192.0.2.4, we may have the following Span on the client side:

Span name: `/webshop/articles/4` (NOTE: This is subject to change, see [open-telemetry/opentelemetry-specification#270][].)

[open-telemetry/opentelemetry-specification#270]: https://github.com/open-telemetry/opentelemetry-specification/issues/270

|   Attribute name   |                                       Value             |
| :----------------- | :-------------------------------------------------------|
| `component`        | `"http"`                                                |
| `http.method`      | `"GET"`                                                 |
| `http.flavor`      | `"1.1"`                                                 |
| `http.url`         | `"https://example.com:8080/webshop/articles/4?s=1"`     |
| `peer.ip4`         | `"192.0.2.5"`                                           |
| `http.status_code` | `200`                                                   |
| `http.status_text` | `"OK"`                                                  |

The corresponding server Span may look like this:

Span name: `/webshop/articles/:article_id`.

|   Attribute name   |                      Value                      |
| :----------------- | :---------------------------------------------- |
| `component`        | `"http"`                                        |
| `http.method`      | `"GET"`                                         |
| `http.flavor`      | `"1.1"`                                         |
| `http.target`      | `"/webshop/articles/4?s=1"`                     |
| `http.host`        | `"example.com:8080"`                            |
| `http.server_name` | `"example.com"`                                 |
| `host.port`        | `8080`                                          |
| `http.scheme`      | `"https"`                                       |
| `http.route`       | `"/webshop/articles/:article_id"`               |
| `http.status_code` | `200`                                           |
| `http.status_text` | `"OK"`                                          |
| `http.client_ip`   | `"192.0.2.4"`                                   |
| `net.peer.ip`      | `"192.0.2.5"` (the client goes through a proxy) |
| `code.ns`          | `"com.example.webshop.ArticleService"`          |
| `code.func`        | `"showArticleDetails"`                          |

Note that following the recommendations above, `http.url` is not set in the above example.
If set, it would be
`"https://example.com:8080/webshop/articles/4?s=1"`
but due to `http.scheme`, `http.host` and `http.target` being set, it would be redundant.
As explained above, these separate values are preferred but if for some reason the URL is available but the other values are not,
URL can replace `http.scheme`, `http.host` and `http.target`.

## Databases (SQL and NoSQL) client calls

> 🚧 WORK IN PROGRESS

For database client call the `SpanKind` MUST be `Client`.

Span `name` should be set to low cardinality value representing the statement
executed on the database. It may be stored procedure name (without argument), SQL
statement without variable arguments, etc. When it's impossible to get any
meaningful representation of the span `name`, it can be populated using the same
value as `db.instance`.

### Connection-level attributes

These attributes will usually be the same for all operations performed over the same database connection
although some database systems may allow to switch to a different `db.user` within the same connection
and other database systems may not even have the concept of a connection.

| Attribute name | Notes and examples                                           | Required? |
| :------------- | :----------------------------------------------------------- | --------- |
| `component` | Always the string `"db"` | Yes       |
| `db.type` | The lower-case database category, e.g. `"sql"`, `"cassandra"`, `"hbase"`, or `"redis"`. | Yes       |
| `db.tech` | Name of the database technology used, e.g. "SQL Server" or "MySQL". | No     |
| `db.url` | The connection string used to connect to the database | No       |
| `db.user`      | Username for accessing database. E.g., `"readonly_user"` or `"reporting_user"` | No        |
| `db.portname` | A name that identifies the `net.peer.port` used to connect. This property is intended to be used for [MS SQL Server instances][] but can be used for any database system that provides a name to port mapping. | See below        |

[MS SQL Server instances]: https://docs.microsoft.com/en-us/sql/connect/jdbc/building-the-connection-url?view=sql-server-ver15

Additionally at least one of `net.peer.name` or `net.peer.ip` from the [network attributes][] is required and `net.peer.port` is recommended.
If using a non-standard port for the `db.tech`, at least one of `net.peer.port` or `db.portname` is required.

### Call-level attributes

These attributes may be different for each operation performed, even if it uses the same connection
although usually only one `db.name` will be used per connection.

| Attribute name | Notes and examples                                           | Required? |
| :------------- | :----------------------------------------------------------- | --------- |
| `db.name`  | Database name. E.g., In Java, if the jdbc.url=`"jdbc:mysql://db.example.com:3306/customers"`, the name is `"customers"`. Note that this attribute was previously called `db.instance` but has nothing to do with [MS SQL Server instances][]. For commands that switch the database, this should be set to the target database (even if the command fails). | Yes (if applicable) |
| `db.statement` | A database statement for the given database type. Note that the value may be sanitized to exclude sensitive information. E.g., for `db.type="sql"`, `"SELECT * FROM wuser_table"`; for `db.type="redis"`, `"SET mykey 'WuValue'"`. | Yes (if applicable)       |
| `db.op` | The type of operation that is executed, e.g. the [MongoDB command name][] such as `findAndModify`. While it would semantically make sense to set this e.g. to an SQL keyword like `SELECT` or `INSERT`, it is *not* recommended to attempt any client-side parsing of `db.statement` just to get this property (the back end can do that if required). | If `db.statement` is not applicable.       |

[MongoDB command name]: https://docs.mongodb.com/manual/reference/command/#database-operations

### Notes for specific database types

In some **SQL** databases, `db.name` is called "schema name".

In **Cassandra**, `db.name` should be set to the keyspace name.

In **CouchDB**, `db.op` should be set to the HTTP method + the target REST route according to the API reference documentation.
For example, when retrieving a document, `db.op` would be set to (literally, i.e., without replacing the placeholders with concrete values): [`GET /{db}/{docid}`][CouchDB get doc].

[CouchDB get doc]: http://docs.couchdb.org/en/stable/api/document/common.html#get--db-docid

In **HBase**, `db.name` is the [namespace][hbase ns].

[hbase ns]: https://hbase.apache.org/book.html#_namespace

**Redis** does not have a database name.

[rpc]: #remote-procedure-calls

## Remote procedure calls

Also known as Remote Method Invocation (RMI).

Implementations MUST create a span, when the RPC call starts, one on the
client-side and one on the server-side. Outgoing requests should be a span `kind`
of `CLIENT` and incoming requests should be a span `kind` of `SERVER`.

Span `name` MUST be the full RPC method name formatted as:

```
$package.$service/$method
```

(where $service must not contain dots).

Examples of span name: `grpc.test.EchoService/Echo`.

### Attributes

| Attribute name |                          Notes and examples                            | Required? |
| -------------- | ---------------------------------------------------------------------- | --------- |
| `component`    | Denotes the type of the span and needs to be `"rpc"`.                  | Yes       |
| `rpc.service`  | The service name, must be equal to the $service part in the span name. | Yes       |
| `rpc.method`   | The $method part in the span name.                                     | No        |
| `rpc.flavor`   | The name of the used remoting system, e.g. `"grpc"` or `"RMI"`.        | No        |

Additionally, the `net.peer.name` and `net.peer.port` [network attributes][] are required.

Note that `rpc.service` may coincide with `code.ns` and `rpc.method` with `code.func`.
Semantically however there is the fine distinction that the `rpc.*` attributes describe the public,
RPC protocol level name of the service (method),
while the `code.*` attributes describe the names used in the source code that implements it.

### gRPC

gRPC is a special case of [RPC spans][rpc] but has additional conventions described in this section.

`rpc.flavor` must be `"grpc"`.

#### Status

Implementations MUST set status which MUST be the same as the gRPC client/server
status. The mapping between gRPC canonical codes and OpenTelemetry status codes
is 1:1 as OpenTelemetry canonical codes is just a snapshot of grpc codes which
can be found [here](https://github.com/grpc/grpc-go/blob/master/codes/codes.go).

#### Events

In the lifetime of a gRPC stream, an event for each message sent/received on
client and server spans SHOULD be created with the following attributes:

```
-> [time],
    "name" = "message",
    "message.type" = "SENT",
    "message.id" = id
    "message.compressed_size" = <compressed size in bytes>,
    "message.uncompressed_size" = <uncompressed size in bytes>
```

```
-> [time],
    "name" = "message",
    "message.type" = "RECEIVED",
    "message.id" = id
    "message.compressed_size" = <compressed size in bytes>,
    "message.uncompressed_size" = <uncompressed size in bytes>
```

The `message.id` MUST be calculated as two different counters starting from `1`
one for sent messages and one for received message. This way we guarantee that
the values will be consistent between different implementations. In case of
unary calls only one sent and one received message will be recorded for both
client and server spans.

## Messaging systems

> 🚧 WORK IN PROGRESS

### Definitions

Although messaging systems are not as standardized as, e.g., HTTP, it is assumed that the following definitions are applicable to most of them that have similar concepts at all (names borrowed mostly from JMS):

A *message* usually consists of headers (or properties, or meta information) and an optional body. It is sent by a single message *producer* to:

* Physically: some message *broker* (which can be e.g., a single server, or a cluster, or a local process reached via IPC). The broker handles the actual routing, delivery, re-delivery, persistence, etc. In some messaging systems the broker may be identical or co-located with (some) message consumers.
* Logically: some particular message *destination*.

A destination is usually identified by some name unique within the messaging system instance, which might look like an URL or a simple one-word identifier. Two kinds of targets are are distinguished: *topic*s and *queue*s.
A message that is sent (the send-operation is often called "*publish*" in this context) to a *topic* is broadcasted to all *subscribers* of the topic.
A message submitted to a queue is processed by a message *consumer* (usually exactly once although some message systems support a more performant at least once-mode for messages with [idempotent][] processing).

The consumption of a message can happen in multiple steps:
First, the lower-level receiving of a message at a consumer, and then the logical processing of the message.
For receiving, the consumer may have the choice between peeking a destination to get a message back only if one is currently available
(for the purpose of this document, it is not relevant whether the message is removed from the destination or not by the "peek" operation),
or waiting until a message is received.
Often, the waiting for a message is not particularly interesting and hidden away in a framework that only invokes some handler function to process a message once one is received
(in the same way that the listening on a TCP port for an incoming HTTP message is not particularly interesting).
However, in a synchronous conversation, the wait time for a message is important.

In some messaging systems, a message can receive a reply message that answers a particular other message that was sent earlier. All messages that are grouped together by such a reply-relationship are called a *conversation*. The grouping usually happens through some sort of "In-Reply-To:" meta information or an explicit conversation ID. Sometimes a conversation can span multiple message targets (e.g. initiated via a topic, continued on a temporary one-to-one queue).

Some messaging systems support the concept of *temporary destination* (often only temporary queues) that are established just for a particular set of communication partners (often one to one) or conversation. Often such destinations are unnamed or have an auto-generated name.


[idempotent]: https://en.wikipedia.org/wiki/Idempotence


### Conventions

Given these definitions, the remainder of this section describes the semantic conventions that shall be followed for Spans describing interactions with messaging systems.

**Span name:** The span name should usually be set to the message destination name.
The conversation ID should be used instead when it is expected to have lower cardinality.
In particular, the conversation ID must be used if the message destination is unnamed or temporary.

**Span kind:** A producer of a message should set the span kind to `PRODUCER` unless it synchronously waits for a response: then it should use `CLIENT`.
The processor of the message should set the kind to `CONSUMER`, unless it always sends back a reply that is directed to the producer of the message
(as opposed to e.g., a queue on which the producer happens to listen): then it should use `SERVER`.

### Messaging attributes

| Attribute name |                          Notes and examples                            | Required? |
| -------------- | ---------------------------------------------------------------------- | --------- |
| `component`    | Denotes the type of the span and needs to be `"msg"`.                  | Yes       |
| `msg.flavor`   | A string identifying the messaging system kind used. Should be of the form `KIND.TECH`, e.g. `JMS.Web Sphere` or `AMQP.RabbitMQ`. | Yes |
| `msg.dst`      | The message destination name. A deprecated alternative key for this attribute is [`message_bus.destination`][ot-msg]. | Yes       |
| `msg.dst_kind` | The kind of message destination: Either `"queue"` or `"topic"`.        | Yes       |
| `msg.tmp_dst`  | A boolean that is `true` if the message destination is temporary. | If temporary (assumed to be `false` if missing). |
| `msg.id`       | An integer or string used by the messaging system as an identifier for the message. | No |
| `msg.conversation_id` | An integer or string identifying the conversation to which the message belongs. Sometimes called "correlation ID". | No |

It is strongly recommended to also set at least the [network attributes] `net.peer.ip`, `net.peer.name` and `net.peer.port`.

[ot-msg]: https://github.com/opentracing/specification/blob/master/semantic_conventions.md#message-bus

For message consumers, the following additional attributes may be set:

| Attribute name |                          Notes and examples                            | Required? |
| -------------- | ---------------------------------------------------------------------- | --------- |
| `msg.op` | A string identifying which part and kind of message consumption this span describes: `recv`, `peek` or `proc`. (If the operation is `send`, this attribute must not be set: the operation can be inferred from the span kind in that case.) | No |
| `msg.peeked_some` |  A boolean indicating whether a `peek` operation resulted in a received message or not. Must only be added if `msg.op` is `peek`. I missing, assumed to be `true` if `msg.id` is set, otherwise assumed to be `false`. | No |

Note that one or multiple Spans with `msg.op` = `proc` may often be the children of a Span with `msg.op` = `recv` or `peek`.
Even though in that case one might think that the span kind is `INTERNAL`, that kind MUST NOT be used.
Instead span kind should be set to either `CONSUMER` or `SERVER` according to the rules defined above.

## General attributes

The attributes described in this section are not specific to a particular operation but rather generic.
They may be used in any Span they apply to.
Particular operations may refer to or require some of these attributes.

[misc attributes]: #general-miscellaneous-attributes

### General miscellaneous attributes

> 🚧 WORK IN PROGRESS

| Attribute name | Notes and examples                                           |
| :------------- | :----------------------------------------------------------- |
| `tech` | The name of the relevant technology associated with the span, such as a database client tehcnology (e.g., ODBC) or library (e.g., `com.example.sqlite3`) name, an HTTP server framework name (e.g., Java Servlet, JSP, Flask) or similar. This should be the technology used at the side of the span, not at any remote side (e.g. the database client technology if the span represents the client-side of an operation) |

[code attributes]: #general-code-attributes

### General source code attributes

Often a Span is tied closely to a certain unit of code, for example the function that handles an HTTP request. In that case, it might make sense to add the name of the function that is logically responsible for handling the operation that the span describes (usually the method that starts the span) to the span attributes as follows:

| Attribute name | Notes and examples                                           |
| :------------- | :----------------------------------------------------------- |
| `code.func` | The method or function name, or equivalent (usually rightmost part of the code unit's name). |
| `code.ns` | The "namespace" within which `code.func` is defined. Usually the qualified class or module name, such that `code.ns` + some separator + `code.func` form an unique identifier for the code unit. |
| `code.file` | The source code file name where the code unit is defined. |
| `code.basepath` | Optional prefix for `code.file`. If present `code.basepath`/`code.file` must be an absolute path. |
| `code.lineno` | The line number within `code.file` as an integer. |

[network attributes]: #general-network-connection-attributes

### General network connection attributes

These attributes may be used for any network related operation.

|  Attribute name  |                                 Notes and examples                                |
| :--------------- | :-------------------------------------------------------------------------------- |
| `net.transport` | Transport protocol used. See note below.                                           |
| `net.peer.ip`   | Remote address of the peer (dotted decimal for IPv4 or [RFC5952][] for IPv6)       |
| `net.peer.port` | Remote port number as an integer. E.g., `80`.                                      |
| `net.peer.name` | Remote hostname or similar, see note below.                                        |
| `net.host.ip`   | Like `net.peer.ip` but for the host IP. Useful in case of a multi-IP host.         |
| `net.host.port` | Like `net.peer.port` but for the host port.                                        |
| `net.host.name` | Like `net.peer.name` but for the host name. If known, the name that the peer used to connect should be preferred. For IP-based communication, an value otbained via an API like POSIX `gethostname` may be used as fallback. |

[RFC5952]: https://tools.ietf.org/html/rfc5952

**net.transport**: The name of the transport layer protocol (or the relevant protocol below the "application protocol"). Should be one of these strings:

* `IP.TCP`
* `IP.UDP`
* `IP`: Another IP-based protocol.
* `Unix`: Unix Domain socket.
* `pipe`: Named or anonymous pipe.
* `inproc`: Signals that there is only in-process communication not using a "real" network protocol in cases where network attributes would normally be expected. Usually all other network attributes can be left out in that case.
* `other`: Something else (not IP-based).

**net.peer.name**, **net.host.name**:
For IP-based communication, the name should be the host name that was used to look up the IP adress in `peer.ip`
(e.g., `"example.com"` if connecting to an URL `https://example.com/foo`).
If that is not available, reverse-lookup of the IP can be used to obtain it.
If `net.transport` is `"unix"` or `"pipe"`, the absolute path to the file representing it should be used.
If there is no such file (e.g., anonymous pipe),
the name should explicitly be set to the empty string to distinguish it from the case where the name is just unknown or not covered by the instrumentation.
