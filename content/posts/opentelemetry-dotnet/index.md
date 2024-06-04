+++
author = "Shawn Vause"
title = "OpenTelemetry for .NET Solutions with Grafana Tooling"
date = "2024-02-24"
summary = """
An overview on setting up OpenTelemetry and OpenTelemetry Collector for .NET Core solutions, along with the necessary configuration to setup a cohesive observability solution utilizing \
OSS offerings Grafana, Grafana Tempo, Grafana Loki and Prometheus."""
tags = [
    "dotnet",
    "opentelemetry",
    "observability"
]
+++

Observability of solutions has become a critical component of modern distributed applications that deserves more forethought and consideration than often afforded to it. Typically, implementing visibility to application functionality is relegated to a last minute implementation detail for the sole purpose of checking a box to indicate the system has been delivered with core expectations. While I can understand developers are strapped with a never-ending backlog of features to deliver (often features with perceived higher value to stakeholders) there is frankly no feature of higher value for development teams and those supporting their applications than observability. When you are awakened in the middle of the night from your slumber to solve a system problem, you want to see where the issue is quickly and with the "right" level of detail to indicate what the solution is likely to be. Not only is this beneficial to restoring functionality for your customers, but let's be honest, we need to get our asses back to bed quickly to handle the upcoming day. Enter <a href="https://opentelemetry.io" title="OpenTelemetry">OpenTelemetry</a>, a tool to make our jobs easier and get the visibility we need quickly, effectively and with a relatively easy implementation process.
{{<preview_ph>}}

OpenTelemetry (OTEL) is rapidly becoming the defacto solution for collecting observability data from our applications in the form of traces, metrics and logs (primer available <a href="https://opentelemetry.io/docs/concepts/observability-primer/" title="Observability Primer">here</a>). A key benefit of using OTEL, is that it is vendor agnostic and thus enables us to self-host our observability stack or choose from multiple vendors that support OTEL telemetry. By standardizing how we instrument our applications we can move faster and get intelligent insights that correlate events in complex distributed systems built with a variety of different technologies and programming languages. Luckily, OTEL's rise in popularity has lead to most modern languages providing an SDK to instrument across heterogenous distributed solutions. This post focuses on .NET based solutions, but be aware of the robust offerings available <a href="https://opentelemetry.io/docs/languages" title="Language APIs & SDKs">here</a>. OTEL helps with the collection process of telemetry and getting it out of our way quickly and easily, so we can focus on instrumenting our apps and providing value for ourselves and support teams from the start of a development project.

### Setup

For this post, we are going to instrument a simple Web API solution using ASP.NET Core. Create a new ASP.NET core Web API project utilizing .NET 8 and install the following nuget packages (at time of writing this post):

<table>
    <tr>
        <th>Package Name</th>
        <th>Version</th>
        <th>Description</th>
    </tr>
    <tr>
        <td>OpenTelemetry</td>
        <td>1.7.0</td>
        <td>Core OTEL Functionality.</td>
    </tr>
        <td>OpenTelemetry.Exporter.Console</td>
        <td>1.7.0</td>
        <td>Basic data exporter displaying telemetry in stdout.</td>
    </tr>
        <td>OpenTelemetry.Exporter.OpenTelemetryProtocol</td>
        <td>1.7.0</td>
        <td>Data exporter using OTEL Protocol to transmit observability data (to OTEL Collector).</td>
    </tr>
        <td>OpenTelemetry.Extensions.Hosting</td>
        <td>1.7.0</td>
        <td>Extensions to utilize OTEL in the Microsoft Hosting environment.</td>
    </tr>
        <td>OpenTelemetry.Instrumentation.AspNetCore</td>
        <td>1.7.1</td>
        <td>Enables collection of ASP.NET Core observability data already built into it's libraries.</td>
    </tr>
        <td>OpenTelemetry.Instrumentation.Http</td>
        <td>1.7.1</td>
        <td>Enables collection of HTTPClient observability data (again already built into the libraries).</td>
    </tr>
        <td>OpenTelemetry.Instrumentation.Runtime</td>
        <td>1.7.0</td>
        <td>This is a cool one, get all the interesting observability data (allocations, garbage collections, etc.) from the dotnet runtime itself!</td>
    </tr>
</table>

### Implementation

I am not sure if this is considered official best practice, but anytime I add a lot of "noise" to the dependency injection system, I always create a class(es) of extension methods to encapsulate core feature sets or group similar "services". This helps keep the registration pipeline clean and readable. Create the following class as follows and we will step through it:

```csharp
using OpenTelemetry.Instrumentation.AspNetCore;
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using System.Diagnostics.Metrics;

namespace WebApplication1;

public static class OpenTelemetryExtensions
{
	private static readonly Action<ResourceBuilder> _ConfigureResource = 
		r => r.AddService(
			serviceName: "web-app-1",
			serviceVersion: typeof(Program).Assembly.GetName()
					.Version?.ToString()
						?? "unknown",
			serviceInstanceId: Environment.MachineName);

	public static IServiceCollection AddOpenTelemetryObservability(
		this IServiceCollection services,
		IConfiguration configuration)
	{
		services.AddOpenTelemetry()
			.ConfigureResource(_ConfigureResource)
			.WithTracing(builder =>
			{
				builder.AddSource()
					.SetSampler(new AlwaysOnSampler())
					.AddHttpClientInstrumentation()
					.AddAspNetCoreInstrumentation();

				services.Configure<AspNetCoreTraceInstrumentationOptions>(
					configuration.GetSection("AspNetCoreInstrumentation"));

				builder.SetupTracingExporter(configuration);
			})
			.WithMetrics(builder =>
			{
				builder
					// Used to record/export specific types 
					// of metrics (counters, gagues, etc.)
					//.AddMeter()
					.AddRuntimeInstrumentation()
					.AddHttpClientInstrumentation()
					.AddAspNetCoreInstrumentation();

				builder.SetupMetricsView(configuration);
				builder.SetupMetricsExporter(configuration);
			});

		return services;
	}

	public static void AddOpenTelemetryLogging(this ILoggingBuilder builder,
		IConfiguration configuration)
	{
		var logExporter = configuration.GetValue(
			"UseLogExporter", defaultValue: "console")!.ToLowerInvariant();

		builder.ClearProviders();

		builder.AddOpenTelemetry(options =>
		{
			var resourceBuilder = ResourceBuilder.CreateDefault();
			_ConfigureResource(resourceBuilder);
			options.SetResourceBuilder(resourceBuilder);

			options.SetupLogsExporter(logExporter, configuration);
		});
	}

	private static void SetupTracingExporter(
		this TracerProviderBuilder builder, 
		IConfiguration configuration)
	{
		var tracingExporter = configuration.GetValue(
			"UseTracingExporter", defaultValue: "console")!.ToLowerInvariant();

		switch (tracingExporter)
		{
			case "otlp":
				builder.AddOtlpExporter(otlpOptions =>
				{
					otlpOptions.Endpoint = new Uri(
						configuration.GetValue(
						"Otlp:Endpoint",
						defaultValue: " http://localhost:4317")!);
				});
				break;

			default:
				builder.AddConsoleExporter();
				break;
		}
	}

	private static void SetupMetricsExporter(
		this MeterProviderBuilder builder,
		IConfiguration configuration)
	{
		var metricsExporter = 
			configuration.GetValue("UseMetricsExporter", defaultValue: "console")!
				.ToLowerInvariant();

		switch (metricsExporter)
		{
			case "otlp":
				builder.AddOtlpExporter(otlpOptions =>
				{
					otlpOptions.Endpoint = new Uri(
						configuration.GetValue(
						"Otlp:Endpoint",
						defaultValue: " http://localhost:4317")!);
				});
				break;

			default:
				builder.AddConsoleExporter();
				break;
		}
	}

	private static void SetupMetricsView(
		this MeterProviderBuilder builder,
		IConfiguration configuration)
	{
		var histogramAggregation = 
			configuration.GetValue(
				"HistogramAggregation", defaultValue: "explicit")!
					.ToLowerInvariant();

		switch (histogramAggregation)
		{
			case "exponential":
				builder.AddView(instrument =>
				{
					return instrument.GetType()
						.GetGenericTypeDefinition() == 
						typeof(Histogram<>)
					? new Base2ExponentialBucketHistogramConfiguration()
					: null;
				});
				break;

			default:
				// Explicit bounds histogram is default, nothing to do.
				break;
		}
	}

	private static void SetupLogsExporter(
		this OpenTelemetryLoggerOptions options, 
		string logExporter,
		IConfiguration configuration)
	{
		switch (logExporter)
		{
			case "otlp":
				options.AddOtlpExporter(otlpOptions =>
				{
					otlpOptions.Endpoint = new Uri(
						configuration.GetValue(
						"Otlp:Endpoint",
						defaultValue: " http://localhost:4317")!);
				});
				break;

			default:
				options.AddConsoleExporter();
				break;
		}
	}
}
```

#### Configuration

The code is pretty readable in my opinion, but essentially we are using a configuration file with the following structure to drive the appropriate OTEL behaviors:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "UseTracingExporter": "otlp",
  "UseMetricsExporter": "otlp",
  "UseLogExporter": "otlp",
  "HistogramAggregation": "explicit",
  "AspNetCoreInstrumentation": {
    "RecordException": "true"
  },
  "Otlp": {
    "Endpoint": "http://localhost:4317"
  },
  "AllowedHosts": "*"
}
```

For each "*Use\<Data Type\>Exporter*" property we take two string values currently "console" or "otlp". This configures the system to use either a console exporter (great for testing) or the OpenTelemetry Protocol which is a vendor/tool agnostic protocol for transmitting traces, metrics and logs telemetry to another service. We will use something called the OTEL Collector as this service, but be aware that logging tools like DataDog and others support OTLP protocol natively in some cases. In addition, we provide an OTLP endpoint for sending data to the OTEL collector from our API. The last parameter controls how the histogram buckets metric data (explicit boundaries vs exponential scales). 

#### Code Explanation

There are two public extension methods in the class defined above. One controls setting up the traces and metrics collection process for OTEL and the other (while similar) sets up the application logging provider. Using the settings from the `appsettings.json` file we add appropriate exporters for getting observability data points out of our API. You will also notice that we add instrumentation for the various .NET core functionality that is made available to us (HTTPClient, Dotnet Runtime, AspNetCore) for trace and metrics. This is incredibly powerful by itself as a ton of telemetry for our solution is already built-in just waiting for us to export it somewhere we can make use of it!

Another important call-out, is the use of the `ConfigureResource` action method. OTEL will use the value configured as a service name tag value. This is really useful for grouping telemetry in an environment where maybe there are many distributed services working independently/interactively. It also allows us to tag our telemetry with a software version (think about handling blue/green deployments and rolling updates where multiple version of a service are live at the same time) and a machine name where the data originated from (load balanced environments).

#### Hook Up

To utilize the extension methods above it is easy to wire them into the HTTP builder pipeline for your web application. In `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add this line below the above builder assignment to add OpenTelemetry for tracing and metrics.
builder.Services.AddOpenTelemetryObservability(builder.Configuration);

// Add this line to add OpenTelemetry for logging.
builder.Logging.AddOpenTelemetryLogging(builder.Configuration);
```

I would recommend starting with the *console* exporter and verifying that you start to see OTEL yaml for logs and metrics/trace data being written out to the console window.

In my next post, I will discuss taking this data from the console and pushing it to the OpenTelemetry Collector for display in Grafana!

Part 2 can be found [here](/posts/opentelemetry-dotnet-2)!
