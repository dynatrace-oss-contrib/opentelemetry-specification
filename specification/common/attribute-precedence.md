# Attribute Precedence on transformation to non-OTLP formats

This document defines the attribute precedence that exporters should follow when 
translating from the hierarchical OTLP format to non-hierarchical formats.

This mapping is required when flattening out attributes from the structured OTLP
format that allows adding attributes at different levels (e.g., Resource 
attributes, InstrumentationLibraryScope attributes, Attributes on 
Spans/Metrics/Logs) to a non-hierarchical representation (e.g., OpenMetrics 
labels).
In the case of OpenMetrics, the set of labels is completely flat and must have 
unique labels only (https://github.com/OpenObservability/OpenMetrics/blob/main/specification/OpenMetrics.md#labelset).
Since OpenTelemetry allows for different levels of attributes, it is feasible
that the same attribute appears multiple times on different levels.

This document aims to provide guidance around consistently transforming 
OpenTelemetry attributes to flat sets.

## Attribute hierarchy in the OTLP message

Since the OTLP format is a hierarchical format, there is an inherent order in 
the attributes.
This document assumes that Resource attributes are at the top of the hierarchy, 
since they are the most general attributes. 
Attributes on individual Spans/Metric data points/Logs are at the bottom of the
hierarchy, as they are most specialized and only apply to a subset of all data.

**A more specialized attribute that shares an attribute key with more general 
attribute will take precedence.** 

Therefore, more specialized attributes will overwrite more general ones.
In some cases it might be desirable to overwrite an attribute like this.
<!-- TODO example -->

When de-normalizing the structured OTLP message to a flat set of key-value pairs,
attributes that are present at Resource and InstrumentationLibraryScope level
will be duplicated for each Span/Metric data point/Log.

## Precedence per Signal

Below, the precedence for each of the signals is spelled out explicitly.
Some attributes (e.g., Span Link attributes) should only be used when flattening
out attributes to transform the respective concept (Span Link attributes should
not be added when transforming attributes to export spans, for example).
`A > B` means that `A` will overwrite `B` if the keys clash.

### Traces

```
Span.attributes > ScopeSpans.scope.attributes > ResourceSpans.resource.attributes
```

#### Span Events

```
Span.events.attributes > Span.attributes > ScopeSpans.scope.attributes > ResourceSpans.resource.attributes
```

#### Span links

```
Span.links.attributes > Span.attributes > ScopeSpans.scope.attributes > ResourceSpans.resource.attributes
```

### Metrics

Metrics are different from Spans and LogRecords, as each Metric has a data field
which can contain one or more data points.
Each data point has a set of attributes, which need to be considered 
independently.

```
Metric.data.data_points.attributes > ScopeMetrics.scope.attributes > ResourceMetrics.resource.attributes
```

#### Metric exemplars

```
Metric.data.data_points.exemplars.filtered_attributes > Metric.data.data_points.attributes > ScopeMetrics.scope.attributes > ResourceMetrics.resource.attributes
```

### Logs

```
LogRecord.log_records.attributes > ScopeLogs.scope.attributes > ResourceLogs.resource.attributes
```

## Considerations

Note that this precedence is a strong suggestion, not a requirement.
Exporters should follow this mode of flattening, but might diverge if they have
a reason to do so. 
Furthermore, exporters can apply clash prevention, e.g., by prefixing all 
Resource attributes with `resource.`.
Note that even then, a Span attribute can overwrite the resource attribute 
`attribute_name`, if it is called `resource.attribute_name`.
Therefore, extra care needs to be taken when prefixing attributes.

## Example

De-duplication can be thought of as a map with unique keys to which the 
attributes are added, from most general to most specialized.
First, the resource attributes are added, then the InstrumentationLibraryScope 
attributes, which overwrite the resource attributes if they share a key.
Then the attributes on the Span/Metric data point/LogRecord are added, which
again overwrite keys that are already present.
The final set of key-value pairs are all the pairs in the map.

This YAML-like representation of a theoretical OTLP message has attributes
with attribute names that clash on multiple levels.

```yaml
ResourceMetrics:
    resource:
        attributes:
            # key-value pairs (attributes) on the resource
            attribute1: resource-attribute-1
            attribute2: resource-attribute-2
            attribute3: resource-attribute-3
            service.name: my-service

    scope_metrics:
        scope:
            attributes:
                attribute1: scope-attribute-1
                attribute2: scope-attribute-2
        
        metrics:
            # there can be multiple data entries here.
            data/0:
                data_points:
                    # each data can have multiple data points:
                    data_point/0:
                        attributes: 
                            attribute1: data-point-0-attribute-1

                    data_point/1:
                        attributes: 
                            attribute1: data-point-1-attribute-1
```

The structure above contains two data points, thus there will be two data points
in the output.
Their attributes will be:

```yaml
# data point 0
service.name: my-service
attribute1: data-point-0-attribute-1     # only this attribute is different
attribute2: scope-attribute-2
attribute3: resource-attribute-3

# data point 1
service.name: my-service
attribute1: data-point-1-attribute-1     # only this attribute is different
attribute2: scope-attribute-2
attribute3: resource-attribute-3
```


<!-- TODO Links -->

