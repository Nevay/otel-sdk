# OpenTelemetry SDK

Asynchronous OpenTelemetry SDK based on [Revolt].  

This metapackage contains the basic components that are required for creating traces/metrics/logs through the
official [OpenTelemetry API](https://packagist.org/packages/open-telemetry/api) and sending them to an [`OTLP/HTTP`]
compatible collector[^1]. Additional components (e.g. resource detectors, non-OTLP exporters) can be installed
as separate packages.

Projects using [ReactPHP] libraries can use this SDK together with [`revolt/event-loop-adapter-react`].

[^1]: It is highly recommended to install the [`ext-protobuf`] extension if using one of the `OTLP` exporters due to
its significantly better performance.

## Installation

```shell
composer require tbachert/otel-sdk
```

## Usage

Refer to the [official OpenTelemetry documentation](https://opentelemetry.io/docs/instrumentation/php/) for general usage of the OpenTelemetry API.

### Manual SDK initialization

```php
$resource = Resource::detect()
    ->merge(Resource::create(['foo' => 'bar']));

$tracerProvider = (new TracerProviderBuilder())
    ->addResource($resource)
    ->addSpanProcessor(new BatchSpanProcessor(new OtlpStreamSpanExporter(getStdout())))
    ->build($logger);
$meterProvider = (new MeterProviderBuilder())
    ->addResource($resource)
    ->addMetricReader(new PeriodicExportingMetricReader(new OtlpStreamMetricExporter(getStdout())))
    ->build($logger);
$loggerProvider = (new LoggerProviderBuilder())
    ->addResource($resource)
    ->addLogRecordProcessor(new BatchLogRecordProcessor(new OtlpStreamLogRecordExporter(getStdout())))
    ->build($logger);
```

```php
awaitAll([
    async($tracerProvider->shutdown(...)),
    async($meterProvider->shutdown(...)),
    async($loggerProvider->shutdown(...)),
]);
```

### Initialization from [configuration file](https://opentelemetry.io/docs/specs/otel/configuration/data-model/#file-based-configuration-model)

```php
$result = Config::loadFile(__DIR__ . '/sdk-config.yaml');
```

###### Automatic initialization from configuration file

The [`OTEL_EXPERIMENTAL_CONFIG_FILE`](https://opentelemetry.io/docs/languages/php/sdk/#configuration) environment
variable can be set to initialize the global instrumentation instances on startup.

### Initialization from [environment variables](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/)

```php
$result = Env::load();
```

###### Automatic initialization from environment variables

The [`OTEL_PHP_AUTOLOAD_ENABLED`](https://opentelemetry.io/docs/languages/php/sdk/#configuration) environment variable
can be set to `true` to initialize the global instrumentation instances on startup.


[Revolt]: https://revolt.run/
[ReactPHP]: https://reactphp.org/
[`revolt/event-loop-adapter-react`]: https://github.com/revoltphp/event-loop-adapter-react
[`OTLP/HTTP`]: https://opentelemetry.io/docs/specs/otlp/#otlphttp
[`ext-protobuf`]: https://opentelemetry.io/docs/instrumentation/php/#ext-protobuf
[`open-telemetry/context`]: https://github.com/opentelemetry-php/context
