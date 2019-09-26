# Semantic Conventions

This document defines reserved attributes that can be used to add operation and
protocol specific information.

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
  * [HTTP client](#http-client)
  * [HTTP server](#http-server)
- [Databases client calls](#databases-client-calls)
- [Remote procedure calls](#remote-procedure-calls)
  * [Attributes](#attributes)
  * [gRPC](#grpc)
    + [Status](#status)
    + [Events](#events)
- [Messaging systems](#messaging-systems)
  * [Definitions](#definitions)
  * [Conventions](#conventions)
  * [Messaging attributes](#messaging-attributes)
- [General attributes](#general-attributes)
  * [General miscellaneous attributes](#general-miscellaneous-attributes)
  * [General source code attributes](#general-source-code-attributes)
  * [General network connection attributes](#general-network-connection-attributes)

## HTTP

This span types represents HTTP requests. They can be used for http and https
schemes and various HTTP versions like 1.1, 2 and SPDY.

Given an [RFC 3986](https://tools.ietf.org/html/rfc3986) compliant URI of the form
`scheme:[//authority]path[?query][#fragment]`, the span name of the span SHOULD
be set to to the URI path value.

If a value that represents the identity of the request
and has a lower cardinality than the URI path can be identified, this value MUST
be used for the span name instead.

| Attribute name | Notes and examples                                           | Required? |
| :------------- | :----------------------------------------------------------- | --------- |
| `component`    | Denotes the type of the span and needs to be `"http"`. | Yes |
| `http.method` | HTTP request method. E.g. `"GET"`. | Yes |
| `http.url` | Full HTTP request URL in the form `scheme://host:port/path?query#fragment`. Usually the fragment is not transmitted over HTTP, but if it is known, it should be included nevertheless. | Defined later. |
| `http.status_code` | [HTTP response status code][]. E.g. `200` (integer) | No |
| `http.status_text` | [HTTP reason phrase][]. E.g. `"OK"` | No |
| `http.flavor` | Kind of HTTP protocol used: `"1.0"`, `"1.1"`, `"2"`, `"SPDY"` or `"QUIC"`. |  If not TCP-based (`QUIC`). |

[HTTP response status code]: https://tools.ietf.org/html/rfc7231#section-6
[HTTP reason phrase]: https://tools.ietf.org/html/rfc7230#section-3.1.2

It is recommended to also use the general [network attributes][], especially `peer.ip`. If `sock.transport` is not specified, it can be assumed to be `IP.TCP` except if `http.flavor` is `QUIC`, in which case `IP.UDP` is assumed.

### HTTP client

This span type represents an outbound HTTP request.

For a HTTP client span, `SpanKind` MUST be `Client`.

`http.url` is required and represents the HTTP URL used to (initially) make this request.

### HTTP server

This span type represents an inbound HTTP request.

For a HTTP server span, `SpanKind` MUST be `Server`.

Given an inbound request for a route (e.g. `"/users/:userID?"` the `name` attribute of the span SHOULD be set to this route. If the route does not include the application root path, it SHOULD be prepended to the span name.

If the route cannot be determined, the `name` attribute MUST be set as defined in the general semantic conventions for HTTP.

| Attribute name | Notes and examples                                           | Required? |
| :------------- | :----------------------------------------------------------- | --------- |
| `http.target` | The full request target as passed in a [HTTP request line][] or equivalent, e.g. `/path/12314/?q=ddds#123"`. | [1] |
| `http.host` | The value of the [HTTP host header][]. Note that this might be empty or not present. | [1] |
| `http.scheme` | The URI scheme identifying the used protocol: `"http"` or `"https"` | [1] |
| `http.server_name` | The server name (not including port). This should be obtained via configuration. If no such configuration can be obtained, this attribute MUST NOT be set (`host.name` from the [network attributes][] should be used instead). | [1] |
| `http.route` | The matched route (path template). E.g. `"/users/:userID?"`. | No |
| `http.app` | An identifier for the whole HTTP application. E.g. Flask app name, `spring.application.name`, etc. | No |
| `http.app_root` |The path prefix of the URL that identifies this `http.app`. Also known as "context root". If multiple roots exist, the one that was matched for this request should be used. | No |
| `http.client_ip` | The IP address of the original client behind all proxies, if known (e.g. from [X-Forwarded-For][]). For syntax, see `peer.ip`. | No |

[HTTP request line]: https://tools.ietf.org/html/rfc7230#section-3.1.1
[HTTP host header]: https://tools.ietf.org/html/rfc7230#section-5.4
[X-Forwarded-For]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For

**[1]**: `http.url` is usually not readily available on the server side but would have to be assembled in a cumbersome and sometimes lossy process from other information (see e.g. <https://github.com/open-telemetry/opentelemetry-python/pull/148>).
It is thus preferred to supply the raw data that *is* available.
Namely, one of the following sets is required (in order of preference, all strings must be non-empty):

* `http.scheme`, `http.host`, `http.target`
* `http.scheme`, `http.server_name`, `host.port`, `http.target`
* `http.scheme`, `host.name`, `host.port`, `http.target`
* `http.url`

Of course, more than the required attributes can be supplied, but this is recommended only if they cannot be inferred from the sent ones. For example, `http.server_name` might be valuable in a scenario where multiple "virtual" hosts are served by a server under the same IP.

It is recommended to also use the `code.ns` and `code.func` [code attributes][] to name the logical handler method (`code.ns` + `code.func` will have a lower cardinality than `http.route`).

As an example, if a browser request for `https://example.com:8080/webshop/articles/4?s=1` is invoked, we may have:

Span name: `/webshop/articles/:article_id` (`app_root` + `route`).

|   Attribute name   |                                       Value                                       |
| :----------------- | :-------------------------------------------------------------------------------- |
| `component`        | `"http"`                                                                          |
| `http.method`      | `"GET"`                                                                           |
| `http.url`         | `"https://example.com:8080/webshop/articles/4?s=1"` (or not set)                  |
| `http.target`      | `"/webshop/articles/4?s=1"`                                                       |
| `http.host`        | `"example.com:8080"`                                                              |
| `http.server_name` | `"example.org"` (in that case, the canonical server name does not match the host) |
| `host.port`        | `8080`                                                                            |
| `http.scheme`      | `"https"`                                                                         |
| `http.route`       | `"/articles/:article_id"` (note that the `app_root` part is missing in this case) |
| `http.status_code` | `200`                                                                             |
| `http.status_text` | `"OK"`                                                                            |
| `http.app`         | E.g., `"My cool WebShop"` or `"com.example.webshop"`                              |
| `http.app_root`    | `"/webshop"`                                                                      |
| `http.client_ip`   | `"192.0.2.4"`                                                                     |
| `peer.ip`          | `"192.0.2.5"` (the client goes through a proxy)                                   |
| `code.ns`          | `"com.example.webshop.ArticleService"`                                            |
| `code.func`        | `"showArticleDetails"`                                                            |

Note that a naive implementation might set `code.ns` = `com.example.mywebframework.HttpDispatcherServlet` and `code.func` = `service`.
If possible, this should be avoided and the logically responsible more specific handler method should be used, even if the span is actually started and ended in the web framework (integration).

## Databases client calls

For database client call the `SpanKind` MUST be `Client`.

Span `name` should be set to low cardinality value representing the statement
executed on the database. It may be stored procedure name (without argument), sql
statement without variable arguments, etc. When it's impossible to get any
meaningful representation of the span `name`, it can be populated using the same
value as `db.instance`.

Note, Redis, Cassandra, HBase and other storage systems may reuse the same
attribute names.

| Attribute name | Notes and examples                                           | Required? |
| :------------- | :----------------------------------------------------------- | --------- |
| `component` | Always the string `"db"` | Yes       |
| `db.tech`      | The lower-case database category, e.g. `"sql"`, `"cassandra"`, `"hbase"`, or `"redis"`, optionally followed by a dot `"."` and the more specific technology name (e.g. `"sql.MySQL"`). | Yes       |
| `db.instance`  | Database instance name. E.g., In Java, if the jdbc.url=`"jdbc:mysql://db.example.com:3306/customers"`, the instance name is `"customers"`. | Yes     |
| `db.statement` | A database statement for the given database type. Note, that the value may be sanitized to exclude sensitive information. E.g., for `db.type="sql"`, `"SELECT * FROM wuser_table"`; for `db.type="redis"`, `"SET mykey 'WuValue'"`. | Yes       |
| `db.url` | The connection string used to connect to the database | Yes       |
| `db.resultcount` | An integer specifying the number of results returned. | No       |
| `db.roundtripcount` | An integer specifying the number of network roundtrips while executing the request. | No       |
| `db.user`      | Username for accessing database. E.g., `"readonly_user"` or `"reporting_user"` | No        |

Additionally the `peer.name` attribute from the [network attributes][] is required and `peer.ip` and `peer.port` are recommended. Also, `tech` ([misc attributes][]) is recommended.

[rpc]: #remote-procedure-calls

## Remote procedure calls

Also known as Remote Method Invocation (RMI).

Implementations MUST create a span, when the RPC call starts, one for
client-side and one for server-side. Outgoing requests should be a span `kind`
of `CLIENT` and incoming requests should be a span `kind` of `SERVER`.

Span `name` MUST be full RPC method name formatted as:

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
| `rpc.flavor`   | The remoting protocol name, if it is widely-used. Otherwise the RPC library name must be specified in `tech`. | Yes |

Additionally, the `peer.hostname` and `peer.port` [network attributes][] are required.

Note that `rpc.service` may coincide with `code.ns` and `rpc.method` with `code.func`.
Semantically however there is the fine distinction that, the `rpc.*` attributes describe the public, RPC protocol level name of the service (method),
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
| `msg.tmp_dst`  | A boolean that is `true` if the message destination is temporary. | If temporary (assumed to be `false` if missing). |
| `msg.id`       | An integer or string used by the messaging system as an identifier for the message. | No |
| `msg.conversation_id` | An integer or string identifying the conversation to which the message belongs. Sometimes called "correlation ID". | No |

It is strongly recommended to also set at least the [network attributes] `peer.ip`, `peer.name` and `peer.port`.

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

| Attribute name | Notes and examples                                           |
| :------------- | :----------------------------------------------------------- |
| `tech` | The name of the relevant technology associated with the span, such as a database client tehcnology (e.g., ODBC) or library (e.g., `com.example.sqlite3`) name, a HTTP server framework name (e.g., Java Servlet, JSP, Flask) or similar. This should be the technology used at the side of the span, not at any remote side (e.g. the database client technology if the span represents the client-side of an operation) |

[code attributes]: #general-code-attributes

### General source code attributes

Often a Span is tied closely to a certain unit of code, for example the function that handles an HTTP request. In that case, it makes sense to add the name of the function that is logically responsible for handling the operation that traces the span (usually the method that starts the span) to the span attributes as follows:

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

|  Attribute name  |                                 Notes and examples                                 |
| :--------------- | :--------------------------------------------------------------------------------- |
| `sock.transport` | Transport protocol used. See note below.                                           |
| `peer.ip`        | Remote address of the peer (dotted decimal for IPv4 or [RFC5952][] for IPv6)       |
| `peer.port`      | Remote port number as an integer. E.g., `80`.                                      |
| `peer.name`      | Remote hostname or similar, see note below.                                        |
| `host.ip`        | Like `peer.ip` but for the host IP. Useful in case of a multi-IP host.             |
| `host.port`      | Like `peer.port` but for the host port.                                            |
| `host.name`      | Like `peer.name` but for the host name. If known, the name that the client used to connect should be preferred. For IP-based communication, an value otbained via an API like POSIX `gethostname` may be used as fallback.  |

[RFC5952]: https://tools.ietf.org/html/rfc5952

**sock.transport**: The name of the transport layer protocol (or the relevant protocol below the "application protocol"). Should be one of these strings:

* `IP.TCP`
* `IP.UDP`
* `IP`: Another IP-based protocol.
* `Unix`: Unix Domain socket.
* `pipe`: Named or anonymous pipe.
* `inproc`: Signals that there is only in-process communication not using a "real" network protocol in cases where network attributes would normally be expected. Usually all other network attributes can be left out in that case.
* `other`: Something else (not IP-based).

**peer.name**, **host.name**: For IP-based communication, the name should be the host name that was used to look up the IP adress in `peer.ip` (e.g., `"example.com"` if connecting to an URL `https://example.com/foo`). If that is not available, reverse-lookup of the IP can be used to obtain it. If `sock.transport` is `"unix"` or `"pipe"`, the absolute path to the file representing it should be used. If there is no such file (e.g., anonymous pipe), the name should explicitly be set to the empty string to distinguish it from the case where the name is just unknown or not covered by the instrumentation.