# CloudEvents

**Status**: [Experimental](../../document-status.md)

This specification defines semantic conventions for [CloudEvents](https://cloudevents.io/). It covers creation, processing and context propagation between producer and consumer. It does not cover transport aspects of publishing and receiving cloud events.

## Overview

To be individually traceable, every CloudEvent has to have individual trace context, which is populated on [Distributed Context Extension](https://github.com/cloudevents/spec/blob/v1.0.1/extensions/distributed-tracing.md).

Trace context is not intended to be modified once populated - it flows through any intermediaries from producer to consumer.

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [Attributes](#attributes)

<!-- tocstop -->

## Spans

### Creation

Instrumentation SHOULD create a new span and populate 'Distributed Context Extension' on the event. It applies when:

- CloudEvent is created by instrumented library
- CloudEvent created outside of instrumented library, but passed without 'Distributed Context Extension'

In case CouldEvent is passed to instrumentation with a valid distributed context, instrumentation MUST NOT create a span and MUST NOT modify distributes context extension.

**Span name:** `CloudEvent Create <event_type>`
**Span kind:** PRODUCER

### Processing

To trace CloudEvent(s) processing, instrumentation SHOULD create a new span.
If single event is processed, instrumentation SHOULD use remote context from 'distributed context extension' as a parent. Instrumentation MAY use links in case of single message processing.
If multiple events are processed together, instrumentation MUST set links with all distributed tracing contexts being processed on a span.

**Span name:** `CloudEvent Process <event_type>`
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
