# OpenTelemetry SDK

Asynchronous OpenTelemetry SDK for PHP.

Built on [Revolt], it can be used with any project that uses the Revolt event loop, including [AMPHP] projects and
[ReactPHP] projects when using [`revolt/event-loop-adapter-react`]. All asynchronous features of the SDK are
available in these environments. Synchronous applications can also use this SDK without changes, but features that
require an asynchronous runtime, such as periodic exports and the Prometheus exporter, are unavailable; exports will
instead be triggered during shutdown.

## Requirements

* PHP >= 8.2 (64-bit)
* For file-based configuration: [`symfony/yaml`] or `ext-yaml`
* [`ext-protobuf`] is recommended for the OTLP exporters (significantly better performance)

## Installation

```shell
composer require tbachert/otel-sdk
```

## Usage

Refer to the [official OpenTelemetry documentation](https://opentelemetry.io/docs/instrumentation/php/) for
general usage of the OpenTelemetry API.

### Initialization from a configuration file

Set the [`OTEL_CONFIG_FILE`] environment variable to a [file-based configuration][opentelemetry-configuration] to
initialize the SDK on startup. The `Globals` instances are registered, and all providers shut down automatically when
the process exits. SDK self-diagnostic messages (e.g. export failures) are written to `error_log`.

A good starting point is the official [`otel-getting-started.yaml`]:

```yaml
file_format: "1.2"

resource:
  attributes_list: ${OTEL_RESOURCE_ATTRIBUTES}
  detection/development:
    detectors:
      - service:
      - host:
      - process:
      - container:

propagator:
  composite:
    - tracecontext:
    - baggage:

tracer_provider:
  sampler:
    parent_based:
      root:
        always_on:
  processors:
    - batch:
        exporter:
          otlp_http:
            endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:-http://localhost:4318}/v1/traces

meter_provider:
  readers:
    - periodic:
        exporter:
          otlp_http:
            endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:-http://localhost:4318}/v1/metrics

logger_provider:
  processors:
    - batch:
        exporter:
          otlp_http:
            endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:-http://localhost:4318}/v1/logs
```

All configuration options are defined by the [OpenTelemetry configuration specification][opentelemetry-configuration].
Configuration files support environment variable substitution with the `${ENV_VAR}` and `${ENV_VAR:-default}` syntax.
Except for substitution, environment variables (including `OTEL_*`) are ignored in configuration files.

Alternatively, the file can be loaded explicitly:

```php
$config = Config::loadFile(__DIR__ . '/otel-sdk-config.yaml');
```

### Initialization from environment variables

Set [`OTEL_PHP_AUTOLOAD_ENABLED`] to `true` to initialize the SDK on startup from the standard [environment
variables][env-variables]. The behavior is the same as for configuration files. If both are set, `OTEL_CONFIG_FILE`
takes precedence.

Alternatively, the components can be created explicitly:

```php
$config = Config::loadFromEnv();
```

### SDK-specific configuration

#### Configuring the automatic shutdown timeout

The automatic shutdown timeout can be configured with the `distribution.tbachert/otel-sdk.shutdown_timeout` property or
the `OTEL_PHP_SHUTDOWN_TIMEOUT` environment variable (in milliseconds). If no limit is configured, a failed export during
shutdown keeps the process alive for the duration of the exporter's full retry backoff sequence.

```yaml
distribution:
  tbachert/otel-sdk:
    shutdown_timeout: 10000
```

#### Reloading the configuration in long-running processes

The `distribution.tbachert/otel-sdk.watcher/development` plugin watches loaded configuration files for changes and
reloads them automatically.

```yaml
distribution:
  tbachert/otel-sdk:
    watcher/development:
      inotify:
```

### Manual SDK initialization

```php
$resource = Resource::create(['foo' => 'bar']);

$tracerProvider = (new TracerProviderBuilder())
    ->setResource($resource)
    ->addSpanProcessor(new BatchSpanProcessor(new OtlpStreamSpanExporter(getStdout())))
    ->build();
$meterProvider = (new MeterProviderBuilder())
    ->setResource($resource)
    ->addMetricReader(new PeriodicExportingMetricReader(new OtlpStreamMetricExporter(getStdout())))
    ->build();
$loggerProvider = (new LoggerProviderBuilder())
    ->setResource($resource)
    ->addLogRecordProcessor(new BatchLogRecordProcessor(new OtlpStreamLogRecordExporter(getStdout())))
    ->build();
```

```php
$cancellation = new TimeoutCancellation(10);
await([
    async($tracerProvider->shutdown(...), $cancellation),
    async($meterProvider->shutdown(...), $cancellation),
    async($loggerProvider->shutdown(...), $cancellation),
]);
```

## Specification compliance

This SDK implements all stable [environment variables][env-variables] from the specification, and file-based
configuration following the official [opentelemetry-configuration] data model (version 1.2).

Compliance is verified by [`tbachert/otel-integration-tests`], an SDK-agnostic test suite. It interacts with the SDK
only through the documented configuration interface (environment variables or configuration file) and verifies the
exported telemetry over the OTLP wire protocol. All tests pass against this SDK.

## Stability

This package is pre-1.0. The zero-code configuration mechanism (environment variables and configuration files,
following the OpenTelemetry specification) is considered stable. Other APIs may change without notice.


[Revolt]: https://revolt.run/
[AMPHP]: https://amphp.org/
[ReactPHP]: https://reactphp.org/
[`revolt/event-loop-adapter-react`]: https://github.com/revoltphp/event-loop-adapter-react
[`ext-protobuf`]: https://opentelemetry.io/docs/instrumentation/php/#ext-protobuf
[`symfony/yaml`]: https://packagist.org/packages/symfony/yaml
[env-variables]: https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/
[`OTEL_CONFIG_FILE`]: https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/#declarative-configuration
[`OTEL_PHP_AUTOLOAD_ENABLED`]: https://opentelemetry.io/docs/languages/php/sdk/#configuration
[`otel-getting-started.yaml`]: https://github.com/open-telemetry/opentelemetry-configuration/blob/main/examples/otel-getting-started.yaml
[opentelemetry-configuration]: https://github.com/open-telemetry/opentelemetry-configuration
[`tbachert/otel-integration-tests`]: https://github.com/Nevay/otel-sdk-integration-tests
