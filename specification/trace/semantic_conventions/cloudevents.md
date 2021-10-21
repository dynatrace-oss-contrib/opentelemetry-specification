# CloudEvents

**Status**: [Experimental](../../document-status.md)

This specification defines semantic conventions for [CloudEvents](https://cloudevents.io/). It covers creation, processing and context propagation between producer and consumer. It does not cover transport aspects of publishing and receiving cloud events.

## Overview

To be individually traceable, each CloudEvent has to have its own trace context. The trace context is populated in the event using the [Distributed Tracing Extension](https://github.com/cloudevents/spec/blob/v1.0.1/extensions/distributed-tracing.md).

Once the trace context is set on the event, it MUST not be modified.

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [Attributes](#attributes)

<!-- tocstop -->

## Spans

### Creation

Instrumentation SHOULD create a new span and populate the [Distributed Tracing Extension](https://github.com/cloudevents/spec/blob/v1.0.1/extensions/distributed-tracing.md) on the event. This applies when:

- CloudEvent is created by instrumented library. It may be impossible or impractical to create span in CloudEvent constructor (or factory) - instrumentation MAY create span later when passing event to transport layer.
- CloudEvent is created outside of instrumented library, but passed without 'Distributed Tracing Extension'

In case CloudEvent is passed to instrumented library with the Distributed Tracing Extension populated, instrumentation MUST NOT create a span and MUST NOT modify Distributed Tracing Extension on the event.

**Span name:** `CloudEvents Create <event_type>`
**Span kind:** PRODUCER

### Processing

To trace the processing of one or more CloudEvents, instrumentation SHOULD create a new span.
If a single event is processed, instrumentation SHOULD use the remote trace context from Distributed Tracing Extension as a parent or MAY instead add it as a link on the processing span.
If multiple events are processed together, for each event being processed, if the event has Distributed Tracing Extension populated, instrumentation MUST add a link to the trace context from the extension on the processing span.

**Span name:** `CloudEvents Process <event_type>`
**Span kind:** CONSUMER

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