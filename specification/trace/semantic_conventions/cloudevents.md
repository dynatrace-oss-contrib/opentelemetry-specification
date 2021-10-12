# CloudEvents

**Status**: [Experimental](../../document-status.md)

This specification defines semantic conventions for [CloudEvents](https://cloudevents.io/). It covers creation, processing and context propagation between producer and consumer. It does not cover transport aspects of publishing and receiving cloud events.

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [Attributes](#attributes)

<!-- tocstop -->

## Spans

### Creation

When CloudEvent is created and does not have [Distributed Context Extension](https://github.com/cloudevents/spec/blob/v1.0.1/extensions/distributed-tracing.md) populated, instrumentation SHOULD create a new PRODUCER span.

**Note:** to be individually traceable, every CloudEvent has to have individual trace context (which is achieved with span creation).

If PRODUCER span for CloudEvent was created, instrumentation MUST populate [Distributed Context Extension](https://github.com/cloudevents/spec/blob/v1.0.1/extensions/distributed-tracing.md) using [W3C Trace Context](https://w3c.github.io/trace-context/) propagator.

**Span name:** `create <event_type>`

### Processing

To trace CloudEvent(s) processing, instrumentation SHOULD create a new CONSUMER span.
If single event is processed, instrumentation SHOULD use remote context from distributed context extension as a parent. Instrumentation MAY use links in case of single message processing.
If multiple events are processed together, instrumentation MUST set links with all distributed tracing contexts being processed on CONSUMER span.

**Span name:** `process <event_type>`

## Attributes

Attributes are applicable to creation and processing spans.

<!-- semconv cloudevents -->
| Attribute  | Type | Description  | Examples  | Required |
|---|---|---|---|---|
| `cloudevents.event_id` | string | The [event_id](https://github.com/cloudevents/spec/blob/master/spec.md#id) uniquely identifies the event. [1] | `123e4567-e89b-12d3-a456-426614174000`; `0001` | No |
| `cloudevents.event_source` | string | The [source](https://github.com/cloudevents/spec/blob/master/spec.md#source-1) identifies the context in which an event happened. | `https://github.com/cloudevents`; `/cloudevents/spec/pull/123`; `my-service` | No |
| `cloudevents.event_spec_version` | string | The [version of the CloudEvents specification](https://github.com/cloudevents/spec/blob/master/spec.md#specversion) which the event uses. | `1.0` | No |
| `cloudevents.event_type` | string | The [event_type](https://github.com/cloudevents/spec/blob/master/spec.md#type) contains a value describing the type of event related to the originating occurrence. | `com.github.pull_request.opened`; `com.example.object.deleted.v2` | No |
| `cloudevents.event_subject` | string | The [subject](https://github.com/cloudevents/spec/blob/master/spec.md#subject) of the event in the context of the event producer (identified by source). | `mynewfile.jpg` | No |

**[1]:** Producers MUST ensure that event_source + event_id is unique for each distinct event. If a duplicate event is re-sent (e.g. due to a network error) it MAY have the same id. Consumers MAY assume that Events with identical event_source and event_id are duplicates.
<!-- endsemconv -->
