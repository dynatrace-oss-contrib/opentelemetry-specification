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

## HTTP client

This span type represents an outbound HTTP request.

For a HTTP client span, `SpanKind` MUST be `Client`.

Given an [RFC 3986](https://www.ietf.org/rfc/rfc3986.txt) compliant URI of the form
`scheme:[//authority]path[?query][#fragment]`, the span name of the span SHOULD
be set to to the URI path value.

If a framework can identify a value that represents the identity of the request
and has a lower cardinality than the URI path, this value MUST be used for the span name instead.

| Attribute name | Notes and examples                                           | Required? |
| :------------- | :----------------------------------------------------------- | --------- |
| `component`    | Denotes the type of the span and needs to be `"http"`. | Yes |
| `http.method` | HTTP request method. E.g. `"GET"`. | Yes |
| `http.url` | HTTP URL of this request, represented as `scheme://host:port/path?query#fragment` E.g. `"https://example.com:779/path/12314/?q=ddds#123"`. | Yes |
| `http.status_code` | [HTTP response status code](https://tools.ietf.org/html/rfc7231). E.g. `200` (integer) | No |
| `http.status_text` | [HTTP reason phrase](https://www.ietf.org/rfc/rfc2616.txt). E.g. `"OK"` | No |

It is recommended to also use the general [network attributes][], especially `peer.ip` and `peer.name`.

## HTTP server

This span type represents an inbound HTTP request.

For a HTTP server span, `SpanKind` MUST be `Server`.

Given an inbound request for a route (e.g. `"/users/:userID?"` the `name` attribute of the span SHOULD be set to this route. If the route does not include the application root path, it SHOULD be prepended to the span name.

If the route can not be determined, the `name` attribute MUST be set to the [RFC 3986 URI](https://www.ietf.org/rfc/rfc3986.txt) path value.

If a framework can identify a value that represents the identity of the request
and has a lower cardinality than the URI path or route, this value MUST be used for the span name instead.

| Attribute name | Notes and examples                                           | Required? |
| :------------- | :----------------------------------------------------------- | --------- |
| `component`    | Denotes the type of the span and needs to be `"http"`. | Yes |
| `http.method` | HTTP request method. E.g. `"GET"`. | Yes |
| `http.url` | HTTP URL of this request, represented as `scheme://host:port/path?query#fragment` E.g. `"https://example.com:779/path/12314/?q=ddds#123"`. | Yes |
| `http.route` | The matched route. E.g. `"/users/:userID?"`. | No |
| `http.status_code` | [HTTP response status code](https://tools.ietf.org/html/rfc7231). E.g. `200` (integer) | No |
| `http.status_text` | [HTTP reason phrase](https://www.ietf.org/rfc/rfc2616.txt). E.g. `"OK"` | No |
| `http.app` | An identifier for the whole HTTP application. E.g. Flask app name, `spring.application.name`, etc. | No |
| `http.app_root` |The path prefix of the URL that identifies this `http.app`. If multiple roots exist, the one that was matched in the current URL should be used. | No |
| `http.client_ip` | The IP address of the original client behind all proxies, if known. For syntax, see `peer.ip`. | No |

It is recommended to also use the general [network attributes][], especially `peer.ip` and `host.name` as well as the `code.ns` and `code.func` [code attributes][] to name the logical handler method (`code.ns` + `code.func` will have a lower cardinality than `http.route`).

As an example, if a browser request for `https://example.com/webshop/articles/4` is invoked, we may have:

Span name: `/webshop/articles/:article_id` (`app_root` + `route`).

|   Attribute name   |                      Value                      |
| :----------------- | :---------------------------------------------- |
| `component`        | `"http"`                                        |
| `http.method`      | `"GET"`                                         |
| `http.url`         | `"https://example.com/webshop/articles/4"`      |
| `http.route`       | `/articles/:article_id` (note that the `app_root` part is missing in this case) |
| `http.status_code` | `200`                                           |
| `http.status_text` | `"OK"`                                          |
| `http.app`         | E.g., `"My cool WebShop"` or `"my.webshop`"     |
| `http.app_root`    | `"/webshop"`                                    |
| `http.client_ip`   | `"192.0.2.4"`                                   |
| `peer.ip`          | `"192.0.2.5"` (the client goes through a proxy) |
| `host.name`        | `"example.com"`                                 |
| `code.ns`          | `"com.example.webshop.ArticleService"`          |
| `code.func`        | `"showArticleDetails"`                          |

Note that a naive implementation might set `code.ns` = `com.example.mywebframework.HttpDispatcherServlet` and `code.func` = `service`. If possible, this should be avoided and the logically responsible more specific handler method should be used, even if the span is actually started and ended in the web framework (integration).

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
| `db.type`      | Database type. For any SQL database, `"sql"`. For others, the lower-case database category, e.g. `"cassandra"`, `"hbase"`, or `"redis"`. | Yes       |
| `db.instance`  | Database instance name. E.g., In Java, if the jdbc.url=`"jdbc:mysql://db.example.com:3306/customers"`, the instance name is `"customers"`. | Yes     |
| `db.statement` | A database statement for the given database type. Note, that the value may be sanitized to exclude sensitive information. E.g., for `db.type="sql"`, `"SELECT * FROM wuser_table"`; for `db.type="redis"`, `"SET mykey 'WuValue'"`. | Yes       |
| `db.url` | The connection string used to connect to the database | Yes       |
| `db.clientlib` | Database driver name or database client library name (when known), e.g., `"JDBI"`, `"jdbc"`, `"odbc"`, `"com.example.postresclient"`. | No       |
| `db.tech` | Database technology, e.g. `"PostgreSQL"`, `"SQLite"`, `"SQL Server"` | No       |
| `db.resultcount` | An integer specifying the number of results returned. | No       |
| `db.roundtripcount` | An integer specifying the number of network roundtrips while executing the request. | No       |
| `db.user`      | Username for accessing database. E.g., `"readonly_user"` or `"reporting_user"` | No        |

Additionally the `peer.name` attribute from the [network attributes][] is required and `peer.ip` and `peer.port` are recommended.

## General RPC

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

| Attribute name |                          Notes and examples                           | Required? |
| -------------- | --------------------------------------------------------------------- | --------- |
| `rpc.service`  | The service name, must be equal to the $service part in the span name | Yes       |
| `rpc.method`   | The $method part in the span name.                                    | No        |
| `rpc.flavor`   | The remoting protocol name if it is widely-used, otherwise the library or framework name. E.g. `"grpc"` | Yes       |

Additionally, the `peer.hostname` and `peer.port` [network attributes][] are required.

## gRPC

gRPC is a special case of [RPC spans](#rpc) but has additional conventions described in this section.

### Status

Implementations MUST set status which MUST be the same as the gRPC client/server
status. The mapping between gRPC canonical codes and OpenTelemetry status codes
is 1:1 as OpenTelemetry canonical codes is just a snapshot of grpc codes which
can be found [here](https://github.com/grpc/grpc-go/blob/master/codes/codes.go).

### Events

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

[code attributes]: #general-code-attributes

## General code attributes

Often a Span is tied closely to a certain unit of code, for example the function that handles an HTTP request. In that case, it makes sense to add the name of the function that is logically responsible for handling the operation that traces the span (usually the method that starts the span) to the span attributes as follows:

| Attribute name | Notes and examples                                           |
| :------------- | :----------------------------------------------------------- |
| `code.func` | The method or function name, or equivalent (usually rightmost part of the code unit's name). |
| `code.ns` | The "namespace" within which `code.func` is defined. Usually the qualified class or module name, such that `code.ns` + some separator + `code.func` form an unique identifier for the code unit. |
| `code.file` | The source code file name where the code unit is defined. |
| `code.basepath` | Optional prefix for `code.file`. If present `code.basepath`/`code.file` must be an absolute path. |
| `code.lineno` | The line number within `code.file` as an integer. |

[network attributes]: #general-network-connection-attributes

## General network connection attributes

These attributes may be used for any network related operation.

|  Attribute name  |                                 Notes and examples                                 |
| :--------------- | :--------------------------------------------------------------------------------- |
| `sock.transport` | Transport protocol used. See note below.                                           |
| `peer.ip`        | Remote address of the peer (dotted decimal for IPv4 or RFC5952 for IPv6)           |
| `peer.port`      | Remote port number as an integer. E.g., `80`.                                      |
| `peer.name`      | Remote hostname or similar, see note below.                                        |
| `host.ip`        | Like `peer.ip` but for the host IP. Useful in case of a multi-IP host.             |
| `host.port`      | Like `peer.port` but for the host port.                                            |
| `host.name`      | Like `peer.name` but for the host name. If known, the name that the client used to connect should be preferred.   |

**peer.name**: For IP-based communication, the name should be the host name that was used to look up the IP adress in `peer.ip` (e.g., `"example.com"` if connecting to an URL `https://example.com/foo`). If that is not available, reverse-lookup of the IP can be used to obtain it. If `sock.transport` is `"unix"` or `"pipe"`, the absolute path to the file representing it should be used. If there is no such file (e.g., anonymous pipe), the name should explicitly be set to the empty string to distinguish it from the case where the name is just unknown or not covered by the instrumentation.

**sock.transport**: The name of the transport layer protocol (or the relevant protocol below the "application protocol"). Should be one of these strings:

* `IP.TCP`
* `IP.UDP`
* `IP`: Another IP-based protocol.
* `Unix`: Unix Domain socket.
* `pipe`: Named or anonymous pipe.
* `inproc`: Signals that there is only in-process communication not using a "real" network protocol in cases where network attributes would normally be expected. Usually all other network attributes can be left out in that case.
