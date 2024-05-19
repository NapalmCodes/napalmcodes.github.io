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

The OpenTelemetry Collector is a service used to glue disparate observability solutions together. Think of it as the hub in a wheel and spoke architecture. Our web application will use the [OpenTelemetry Protocol (OTLP)](https://opentelemetry.io/docs/specs/otel/protocol/) supported by the OTEL collector to send all observability data (logs, traces, metrics) in a consistent way. This means our application only needs to take on the exporter dependency mentioned in part one for OTLP. All the data is thus sent to the collector "speaking the same language" vs having different protocols and specifications that would need maintained and supported. An added benefit of this approach is that if each application "speaks OTLP" we can spin up an auto scaling/load balanced environment that all our web applications and services (potentially written with a microservices architectural style) send observability data to, allowing for reuse of code, configuration and potentially infrastructure across our environment.

Configuring the OTEL collector allows us to use solutions potentially outside the OpenTelemetry ecosystem. For instance, metrics can be stored in a variety of data stores such as time series databases like Prometheus. Rather than have our application talk to the database directly, this intermediary gives us the opportunity to transform the data potentially amongst other tasks. Typically this is setup using some YAMl based configuration that OTEl collector understands and utilizes to forward observability data to the down stream solution (such as Prometheus).
