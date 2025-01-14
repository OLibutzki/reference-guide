# Metrics

Interesting metrics in a message centric system come in several forms and flavors, like count, capacity and latency for example. Axon Framework allows you to retrieve such measurements through the use of the `axon-metrics` or `axon-micrometer` module. With these modules you can register a number of `MessageMonitor` implementations to your messaging components, like the [`CommandBus`](axon-framework-commands/command-dispatchers.md#the-command-bus), [`EventBus`](events/event-bus-and-event-store.md#event-bus), [`QueryBus`](queries/query-dispatchers.md#the-query-bus-and-query-gateway) and [`EventProcessors`](events/event-processors/README.md).

`axon-metrics` module uses [Dropwizard Metrics](https://metrics.dropwizard.io/) for registering the measurements correctly. That means that `MessageMonitors` are registered against the Dropwizard `MetricRegistry`.

`axon-micrometer` module uses [Micrometer](https://micrometer.io/) which is a dimensional-first metrics collection facade whose aim is to allow you to time, count, and gauge your code with a vendor neutral API. That means that `MessageMonitors` are registered against the Micrometer `MeterRegistry`.‌

The following monitor implementations are currently provided:

1. `CapacityMonitor` - Measures message capacity
   by keeping track of the total time spent on message handling compared to total time it is active.
   This returns a number between 0 and n number of threads. 
   Thus, if there are 4 threads working, the maximum capacity is 4 if every thread is active 100% of the time.
2. `EventProcessorLatencyMonitor` - Measures the difference in message timestamps between the last ingested and the last processed event message.
3. `MessageCountingMonitor` - Counts the number of ingested, successful, failed, ignored and processed messages.
4. `MessageTimerMonitor` - Keeps a timer for all successful, failed and ignored messages, as well as an overall timer for all three combined.
5. `PayloadTypeMessageMonitorWrapper` - A special `MessageMonitor` implementation which allows setting a monitor per message type instead of per message publishing/handling component.

You are free to configure any combination of `MessageMonitors` through constructors on your messaging components, simply by using the Configuration API. The `GlobalMetricRegistry` contained in the `axon-metrics` and `axon-micrometer` modules provides a set of sensible defaults per type of messaging component. The following example shows you how to configure default metrics for your message handling components:‌

## Dropwizard

{% tabs %}
{% tab title="Axon Configuration API" %}
```java
public class MetricsConfiguration {

    public Configurer buildConfigurer() {
        return DefaultConfigurer.defaultConfiguration();
    }

    // The MetricRegistry is a class from the Dropwizard Metrics framework
    public void configureDefaultMetrics(Configurer configurer, MetricRegistry metricRegistry) {
        GlobalMetricRegistry globalMetricRegistry = new GlobalMetricRegistry(metricRegistry);
        // We register the default monitors to our messaging components by doing the following
        globalMetricRegistry.registerWithConfigurer(configurer);
    }
}
```
{% endtab %}

{% tab title="Spring Boot AutoConfiguration" %}
```text
# The default value is `true`. Thus you will have Metrics configured if `axon-metrics` and `io.dropwizard.metrics` are on your classpath.
axon.metrics.auto-configuration.enabled=true
```
{% endtab %}
{% endtabs %}

## Micrometer

{% tabs %}
{% tab title="Axon Configuration API - Without Tags" %}
```java
public class MetricsConfiguration {

    public Configurer buildConfigurer() {
        return DefaultConfigurer.defaultConfiguration();
    }

    // The MeterRegistry is a class from the Micrometer library
    public void configureDefaultMetrics(Configurer configurer, MeterRegistry meterRegistry) {
        GlobalMetricRegistry globalMetricRegistry = new GlobalMetricRegistry(meterRegistry);
        globalMetricRegistry.registerWithConfigurer(configurer);
    }
}
```
{% endtab %}

{% tab title="Axon Configuration API - With Tags" %}
```java
public class MetricsConfiguration {

    public Configurer buildConfigurer() {
        return DefaultConfigurer.defaultConfiguration();
    }

    // The MeterRegistry is a class from the Micrometer library
    public void configureDefaultMetrics(Configurer configurer, MeterRegistry meterRegistry) {
        GlobalMetricRegistry globalMetricRegistry = new GlobalMetricRegistry(meterRegistry);
        globalMetricRegistry.registerWithConfigurerWithDefaultTags(configurer);
    }
}
```
{% endtab %}

{% tab title="Spring Boot AutoConfiguration - Without Tags" %}
```text
# The default value is `true`. 
# Thus you will have Metrics configured if `axon-micrometer` and 
#  appropriate metric implementation (for example: `micrometer-registry-prometheus`) are on your classpath.
axon.metrics.auto-configuration.enabled=true

# Spring Boot metrics enabled
management.endpoint.metrics.enabled=true

# Spring Boot (Prometheus) endpoint (`/actuator/prometheus`) enabled and exposed
management.metrics.export.prometheus.enabled=true
management.endpoint.prometheus.enabled=true
```
{% endtab %}
{% tab title="Spring Boot AutoConfiguration - With Tags" %}
```text
# The default value is `true`. 
# Thus you will have Metrics configured if `axon-micrometer` and 
#  appropriate metric implementation (for example: `micrometer-registry-prometheus`) are on your classpath.
axon.metrics.auto-configuration.enabled=true

# The default value is `false`. 
# By enabling this property you will have message (event, command, query)
#   payload type set as a micrometer tag/dimension by default. 
# Additionally, the processor name will be a tag/dimension instead of it being part of the metric name.
axon.metrics.micrometer.dimensional=true

# Spring Boot metrics enabled
management.endpoint.metrics.enabled=true

# Spring Boot (Prometheus) endpoint (`/actuator/prometheus`) enabled and exposed
management.metrics.export.prometheus.enabled=true
management.endpoint.prometheus.enabled=true
```
{% endtab %}
{% endtabs %}

The scenario might occur that more fine-grained control over which `MessageMonitor` instance are defined is necessary.
The following snippet provides as sample if you want to have more specific metrics on any of the message handling components:

```java
// Java (Spring Boot Configuration) - Micrometer example
@Configuration
public class MetricsConfig {

    @Bean
    public ConfigurerModule metricConfigurer(MeterRegistry meterRegistry) {
        return configurer -> {
            instrumentEventStore(meterRegistry, configurer);
            instrumentEventProcessors(meterRegistry, configurer);
            instrumentCommandBus(meterRegistry, configurer);
            instrumentQueryBus(meterRegistry, configurer);
        };
    }

    private void instrumentEventStore(MeterRegistry meterRegistry, Configurer configurer) {
        MessageMonitorFactory messageMonitorFactory = (configuration, componentType, componentName) -> {
            MessageCountingMonitor messageCounter = MessageCountingMonitor.buildMonitor(
                    componentName, meterRegistry,
                    message -> Tags.of(TagsUtil.PAYLOAD_TYPE_TAG, message.getPayloadType().getSimpleName())
                                   .and(message.getMetaData().entrySet().stream()
                                               .map(s -> Tag.of(s.getKey(), s.getValue().toString()))
                                               .collect(Collectors.toList()))
            );
            // Naming the Timer monitor/meter with the name of the component (eventStore)
            // Registering the Timer with custom tags: payloadType.
            MessageTimerMonitor messageTimer = MessageTimerMonitor.buildMonitor(
                    componentName, meterRegistry,
                    message -> Tags.of(TagsUtil.PAYLOAD_TYPE_TAG, message.getPayloadType().getSimpleName())
            );
            return new MultiMessageMonitor<>(messageCounter, messageTimer);
        };
        configurer.configureMessageMonitor(EventStore.class, messageMonitorFactory);
    }

    private void instrumentEventProcessors(MeterRegistry meterRegistry, Configurer configurer) {
        MessageMonitorFactory messageMonitorFactory = (configuration, componentType, componentName) -> {

            // Naming the Counter monitor/meter with the fixed name `eventProcessor`.
            // Registering the Counter with custom tags: payloadType and processorName.
            MessageCountingMonitor messageCounter = MessageCountingMonitor.buildMonitor(
                    "eventProcessor", meterRegistry,
                    message -> Tags.of(
                            TagsUtil.PAYLOAD_TYPE_TAG, message.getPayloadType().getSimpleName(),
                            TagsUtil.PROCESSOR_NAME_TAG, componentName
                    )
            );
            // Naming the Timer monitor/meter with the fixed name `eventProcessor`.
            // Registering the Timer with custom tags: payloadType and processorName.
            MessageTimerMonitor messageTimer = MessageTimerMonitor.buildMonitor(
                    "eventProcessor", meterRegistry,
                    message -> Tags.of(
                            TagsUtil.PAYLOAD_TYPE_TAG, message.getPayloadType().getSimpleName(),
                            TagsUtil.PROCESSOR_NAME_TAG, componentName
                    )
            );
            // Naming the Capacity/Gauge monitor/meter with the fixed name `eventProcessor`.
            // Registering the Capacity/Gauge with custom tags: payloadType and processorName.
            CapacityMonitor capacityMonitor1Minute = CapacityMonitor.buildMonitor(
                    "eventProcessor", meterRegistry,
                    message -> Tags.of(
                            TagsUtil.PAYLOAD_TYPE_TAG, message.getPayloadType().getSimpleName(),
                            TagsUtil.PROCESSOR_NAME_TAG, componentName
                    )
            );

            return new MultiMessageMonitor<>(messageCounter, messageTimer, capacityMonitor1Minute);
        };
        configurer.configureMessageMonitor(TrackingEventProcessor.class, messageMonitorFactory);
    }

    private void instrumentCommandBus(MeterRegistry meterRegistry, Configurer configurer) {
        MessageMonitorFactory messageMonitorFactory = (configuration, componentType, componentName) -> {
            MessageCountingMonitor messageCounter = MessageCountingMonitor.buildMonitor(
                    componentName, meterRegistry,
                    message -> Tags.of(
                            TagsUtil.PAYLOAD_TYPE_TAG, message.getPayloadType().getSimpleName(),
                            "messageId", message.getIdentifier()
                    )
            );
            MessageTimerMonitor messageTimer = MessageTimerMonitor.buildMonitor(
                    componentName, meterRegistry,
                    message -> Tags.of(TagsUtil.PAYLOAD_TYPE_TAG, message.getPayloadType().getSimpleName())
            );

            CapacityMonitor capacityMonitor1Minute = CapacityMonitor.buildMonitor(
                    componentName, meterRegistry,
                    message -> Tags.of(TagsUtil.PAYLOAD_TYPE_TAG, message.getPayloadType().getSimpleName())
            );

            return new MultiMessageMonitor<>(messageCounter, messageTimer, capacityMonitor1Minute);
        };
        configurer.configureMessageMonitor(CommandBus.class, messageMonitorFactory);
    }

    private void instrumentQueryBus(MeterRegistry meterRegistry, Configurer configurer) {
        MessageMonitorFactory messageMonitorFactory = (configuration, componentType, componentName) -> {
            MessageCountingMonitor messageCounter = MessageCountingMonitor.buildMonitor(
                    componentName, meterRegistry,
                    message -> Tags.of(
                            TagsUtil.PAYLOAD_TYPE_TAG, message.getPayloadType().getSimpleName(),
                            "messageId", message.getIdentifier()
                    )
            );
            MessageTimerMonitor messageTimer = MessageTimerMonitor.buildMonitor(
                    componentName, meterRegistry,
                    message -> Tags.of(TagsUtil.PAYLOAD_TYPE_TAG, message.getPayloadType().getSimpleName())
            );
            CapacityMonitor capacityMonitor1Minute = CapacityMonitor.buildMonitor(
                    componentName, meterRegistry,
                    message -> Tags.of(TagsUtil.PAYLOAD_TYPE_TAG, message.getPayloadType().getSimpleName())
            );

            return new MultiMessageMonitor<>(messageCounter, messageTimer, capacityMonitor1Minute);
        };
        configurer.configureMessageMonitor(QueryBus.class, messageMonitorFactory);
    }
}
```
