# CloudEvents

This document defines how to apply semantic conventions when instrumenting code that uses [CloudEvents](https://cloudevents.io/). 
These semantic conventions are general and not tied to any protocol used to send/receive the events.

**Status**: [Experimental](../../document-status.md)

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [Span name](#span-name)
- [Span kind](#span-kind)
- [Attributes](#attributes)

<!-- tocstop -->

## Span name

The span name SHOULD be set to the `cloudevents.` prefix, followed by the event type in the following format:

```
cloudevents.<event type name>
```

The event type name SHOULD only be used for the span name if it is known to be of low cardinality (cf. [general span name guidelines](../api.md#span)).

Examples:

* `cloudevents.com.github.pull_request.opened`

## Span kind

A producer of an event should set the span kind to `PRODUCER` unless it synchronously waits for a response: then it should use `CLIENT`.
The processor of the message should set the kind to `CONSUMER`, unless it always sends back a reply that is directed to the producer of the event
(as opposed to e.g., a queue on which the producer happens to listen): then it should use `SERVER`.

## Attributes

<!-- semconv cloudevents -->
| Attribute  | Type | Description  | Examples  | Required |
|---|---|---|---|---|
| `cloudevents.event_id` | string | The [event_id](https://github.com/cloudevents/spec/blob/master/spec.md#id) identifies the event within the scope of a producer. | `123e4567-e89b-12d3-a456-426614174000`; `producer-1` | No |
| `cloudevents.source` | string | The [source](https://github.com/cloudevents/spec/blob/master/spec.md#source-1) identifies the context in which an event happened. | `https://github.com/cloudevents`; `/cloudevents/spec/pull/123`; `my-service` | No |
| `cloudevents.spec_version` | string | The [version of the CloudEvents specification](https://github.com/cloudevents/spec/blob/master/spec.md#specversion) which the event uses. | `1.0` | No |
| `cloudevents.event_type` | string | The [event_type](https://github.com/cloudevents/spec/blob/master/spec.md#type) contains a value describing the type of event related to the originating occurrence. | `com.github.pull_request.opened`; `com.example.object.deleted.v2` | No |
| `cloudevents.data_content_type` | string | The [content type](https://github.com/cloudevents/spec/blob/master/spec.md#datacontenttype) of the event `data` value. [1] | `application/json` | No |
| `cloudevents.subject` | string | The [subject](https://github.com/cloudevents/spec/blob/master/spec.md#subject) of the event in the context of the event producer (identified by source). | `mynewfile.jpg` | No |

**[1]:** If present, MUST adhere to the format specified in [RFC 2046](https://datatracker.ietf.org/doc/html/rfc2046)
<!-- endsemconv -->
