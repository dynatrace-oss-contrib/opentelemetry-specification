# General attributes

The attributes described in this section are not specific to a particular operation but rather generic.
They may be used in any Span they apply to.
Particular operations may refer to or require some of these attributes.

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [General miscellaneous attributes](#general-miscellaneous-attributes)
- [General source code attributes](#general-source-code-attributes)
- [General network connection attributes](#general-network-connection-attributes)

<!-- tocstop -->

[misc attributes]: #general-miscellaneous-attributes

## General miscellaneous attributes

> 🚧 WORK IN PROGRESS

| Attribute name | Notes and examples                                           |
| :------------- | :----------------------------------------------------------- |
| `tech` | The name of the relevant technology associated with the span, such as a database client tehcnology (e.g., ODBC) or library (e.g., `com.example.sqlite3`) name, an HTTP server framework name (e.g., Java Servlet, JSP, Flask) or similar. This should be the technology used at the side of the span, not at any remote side (e.g. the database client technology if the span represents the client-side of an operation) |

[code attributes]: #general-code-attributes

## General source code attributes

Often a Span is tied closely to a certain unit of code, for example the function that handles an HTTP request. In that case, it might make sense to add the name of the function that is logically responsible for handling the operation that the span describes (usually the method that starts the span) to the span attributes as follows:

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