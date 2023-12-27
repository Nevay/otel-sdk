# OpenTelemetry SDK

Asynchronous OpenTelemetry SDK based on [Revolt].  
Projects using [ReactPHP] libraries can use this SDK together with [`revolt/event-loop-adapter-react`].

While synchronous projects cannot utilize all capabilities of this SDK, they can still profit from concurrent exports
during SDK `::shutdown()` and `::forceFlush()`.

## Installation

```shell
composer require tbachert/otel-sdk
```

### Prerequisites

#### Fiber support

Ensure that your system allows the usage of fibers with [`open-telemetry/context`], refer to the
[official OpenTelemetry documentation](https://opentelemetry.io/docs/instrumentation/php/#ext-ffi) for details.

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

### Initialization from [configuration file](https://opentelemetry.io/docs/specs/otel/configuration/file-configuration/)

```php
$config = yaml_parse_file('kitchen-sink.yaml');
$result = Config::load($config);
```

### Initialization from [environment variables](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/)

See also [PHP SDK configuration](https://opentelemetry.io/docs/instrumentation/php/sdk/#configuration).

```php
$result = Env::load();
```


[Revolt]: https://revolt.run/
[ReactPHP]: https://reactphp.org/
[`revolt/event-loop-adapter-react`]: https://github.com/revoltphp/event-loop-adapter-react
[`open-telemetry/context`]: https://github.com/opentelemetry-php/context
