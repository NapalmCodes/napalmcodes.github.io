+++
author = "Shawn Vause"
title = "OpenTelemetry for .NET Solutions with Grafana Tooling - Services Setup"
date = "2024-04-01"
summary = """
An overview on setting up OpenTelemetry and OpenTelemetry Collector for .NET Core solutions, along with the necessary configuration to setup a cohesive observability solution utilizing \
OSS offerings Grafana, Grafana Tempo, Grafana Loki and Prometheus."""
tags = [
    "dotnet",
    "opentelemetry",
    "observability"
]
+++

Welcome to part two of the series I started [here](/posts/opentelemetry-dotnet)! We will pick up with going over the necessary services to support the OpenTelemetry data being pumped in from our sample web application. We will utilize Docker compose as it provides a simple and easy way to orchestrate the necessary containers to send our observability data to. We will utilize [OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector) as a central hub for receiving the observablity data from our application. From there, we can distribute data across a variety of services including [Grafana Loki](https://grafana.com/oss/loki/) for logs, [Grafana Tempo](https://grafana.com/oss/tempo/) for traces and the [Prometheus](https://prometheus.io/) time series database for metrics. Finally, viewing the observability data in meaningful ways will require use of the popular [Grafana](https://grafana.com/oss/grafana/) visualization platform which provides us with browsing tools for the aforementioned data types as well as graphical tooling to create graphs and dashboards from. Ultimately, the tools at our disposal enable us to build the elusive "single pane of glass" view for our application and how it is performing. Let's jump in!
{{<preview_ph>}}

### OpenTelemetry Collector

The OpenTelemetry Collector is a service used to glue disparate observability solutions together. Think of it as the "hub" in a wheel and spoke architecture. Our web application will use the [OpenTelemetry Protocol (OTLP)](https://opentelemetry.io/docs/specs/otel/protocol/) supported by the OTEL collector to send all observability data (logs, traces, metrics) in a consistent way. This means our application only needs to take on the exporter dependency mentioned in part one for OTLP. All the data is thus sent to the collector "speaking the same language" vs having different protocols and specifications that would need maintained and supported. An added benefit of this approach is that if each application "speaks OTLP" we can spin up an auto scaling/load balanced environment that all our web applications and services (potentially written with a microservices architectural style) send observability data to, allowing for reuse of code, configuration and potentially infrastructure across our environment.

Configuring the OTEL collector allows us to use solutions potentially outside the OpenTelemetry ecosystem. For instance, metrics can be stored in a variety of data stores such as time series databases like Prometheus, Traces can be stored in Grafana Tempo and logs in Grafana Loki. Rather than have our application directly talk to these tools, the OTEL collector intermediary gives us the opportunity to transform the data or perform some other pre-load activities. Typically this is setup using some YAML based configuration that OTEl collector understands and utilizes to forward observability data to the down stream solution (such as Prometheus).

#### OpenTelemetry Collector Configuration

I recommend you start by creating a folder in your solution for these configuration files called `observability` or something similar. Inside that folder create a file called `otel-config.yaml` Let's walk though the core sections of the yaml file typically present. For a more comprehensive look examine the official documents found [here](https://opentelemetry.io/docs/collector/configuration/https://opentelemetry.io/docs/collector/configuration/). The YAML below is broken up into four key sections: receivers, processors, exporters and services.

Receivers are observability data providers. Our applications wire up to the OTLP protocol endpoint(s) documented below and sent all their OTEL data for handling.

Processors are an optional but recommended configuration element for OTEL collector. These items give us a change to modify or transform observability data before they are exported/delivered to downstream tools and databases. The *batch* processor noted in the configuration sample is responsible for compressing and grouping observability data. This makes more efficent use of the networking connections in our solution architecture.

Exporters simply send data to the desired destination(s) after processing has been performed. Note we are setting the default logging level for all exporters to "Debug" in this section. This ensures you will get some data in this demo, but it may not be the desired level in production or when you are paying for log volume in a cloud solution for example.

Services are essentially an index of the receivers, processors and exporters configured within the other sections. Ultimately, the configuration here determines what services are enabled in the OTEL collector. The sub-section `pipelines` allows us to wire up configuration for logs, metrics and trace data in different ways. You will notice the configuration utilizes OTLP for all the observability data points as the receiver. Traces are also exported to an OTLP exporter. The batch processors also ensure efficient use of network resources for metrics and logs.

Let's now walk through each tool we will utilize for each type of observability data.

```yaml
receivers:
  otlp:
    protocols:
     http:
       endpoint: 0.0.0.0:4318
     grpc:
       endpoint: 0.0.0.0:4317

processors:
  batch:

exporters:
  logging:
    loglevel: debug

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlp]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: []
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: []
```

### Grafana Loki

TODO