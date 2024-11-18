# JsonStreamer

JSON Lines streaming serializer/deserializer on .NET and ASP.NET Core.

![JsonStreamer](Images/JsonStreamer.100.png)

# Status

[![Project Status: WIP – Initial development is in progress, but there has not yet been a stable, usable release suitable for the public.](https://www.repostatus.org/badges/latest/wip.svg)](https://www.repostatus.org/#wip)

## ASP.NET Core packages

|Target serializer|Pakcage|
|:----|:----|
|Newtonsoft.Json|[![NuGet JsonStreamer.NewtonsoftJson](https://img.shields.io/nuget/v/JsonStreamer.NewtonsoftJson.svg?style=flat)](https://www.nuget.org/packages/JsonStreamer.NewtonsoftJson)|
|System.Text.Json|[![NuGet JsonStreamer.NewtonsoftJson](https://img.shields.io/nuget/v/JsonStreamer.SystemTextJson.svg?style=flat)](https://www.nuget.org/packages/JsonStreamer.SystemTextJson)|

## Client package

|Target serializer|Pakcage|
|:----|:----|
|Newtonsoft.Json|[![NuGet JsonStreamer.NewtonsoftJson.Client](https://img.shields.io/nuget/v/JsonStreamer.NewtonsoftJson.Client.svg?style=flat)](https://www.nuget.org/packages/JsonStreamer.NewtonsoftJson.Client)|

----

## What is this?

Has anyone else noticed that ASP.NET Core can send streaming data using asynchronous iterators `IAsyncEnumerable<T>` ?
It is code like this:

```csharp
[HttpGet]
public IAsyncEnumerable<WeatherForecast> Get() =>
    AsyncEnumerable.Range(1, 1000000).
    Select(index => new WeatherForecast
    {
        Date = DateTime.Now.AddDays(index),
        TemperatureC = Random.Shared.Next(-20, 55),
        Summary = Summaries[Random.Shared.Next(Summaries.Length)]
    });
```

In fact, this works as expected on the server side of ASP.NET Core and does not consume memory for buffering.
Were you aware that this code returns a JSON array?

```json
[
    {
        "date": "2023-09-20T20:23:49.5736146+09:00",
        "temperatureC": 14,
        "temperatureF": 57,
        "summary": "Mild"
    },
    {
        "date": "2023-09-21T20:23:49.5768618+09:00",
        "temperatureC": 34,
        "temperatureF": 93,
        "summary": "Freezing"
    },
    {
        "date": "2023-09-22T20:23:49.5768924+09:00",
        "temperatureC": 1,
        "temperatureF": 33,
        "summary": "Mild"
    },

    // (continues a lot of JObject)
]
```

What about on the browser side receiving this?

In .NET, JavaScript and other platforms, there are few libraries that can streaming deserialize huge JSON arrays.
Streaming sending is easy, but streaming receiving is very difficult.

On the other hand, there is a variant of the JSON format designed for this purpose.
These are [JSON Lines (NDJSON) format](https://jsonlines.org/).

It is a variant, but very simple and easy to understand:

```json
{"date":"2023-09-20T20:23:49.5736146+09:00","temperatureC":14,"temperatureF":57,"summary":"Mild"}
{"date":"2023-09-21T20:23:49.5768618+09:00","temperatureC":34,"temperatureF":93,"summary":"Freezing"}
{"date":"2023-09-22T20:23:49.5768924+09:00","temperatureC":1,"temperatureF":33,"summary":"Mild"}

// (continues a lot of JObject)
```

That is instead of JSON array, JObjects are sent only separated by LF (Newline delimited).
This means that deserializer implementation is easier.

For example, in the case of JavaScript, the package [can-ndjson-stream](https://github.com/canjs/can-ndjson-stream) exists and can be used immediately.

JsonStreamer overrides the serializer so that when returning asynchronous iterators, they are automatically sent in JSON Lines.
(Other types use the default serializer)
And now supports on client-side .NET package!

----

## Target .NET platforms (ASP.NET Core)

|Serializer|ASP.NET Core versions|
|:----|:----|
|Newtonsoft.Json|ASP.NET Core 9 to 3|
|System.Text.Json|ASP.NET Core 9 to 5|

* We tested on only ASP.NET Core 9, 8 and 6.

## Target .NET platforms (Client)

|Serializer|.NET versions|
|:----|:----|
|Newtonsoft.Json|.NET 9 to 5, .NET Core 3.1, 2.2, .NET Standard 2.1, 2.0, .NET Framework 4.8.1 to 4.6.1|

* Newtonsoft.Json version is 13.0.1 or higher.

----

## How to use (ASP.NET Core)

It is very easy to use, just install this package and set it up in the builder as follows.

|Serializer|Application|
|:----|:----|
|Newtonsoft.Json|Install package [JsonStreamer.NewtonsoftJson](https://www.nuget.org/packages/JsonStreamer.NewtonsoftJson) and call `AddNewtonsoftJsonStreamer()` instead of `AddNewtonsoftJson()`.|
|System.Text.Json|Install package [JsonStreamer.SystemTextJson](https://www.nuget.org/packages/JsonStreamer.SystemTextJson) and call `AddJsonStreamer()`.|

For builder configuration example:

```csharp
public static void Main(string[] args)
{
    // ...

    // Add services to the container.
    builder.Services.AddControllers().
        AddNewtonsoftJsonStreamer();    // Enable JsonStreamer.

    // ...

    var app = builder.Build();

    app.MapControllers();
    app.Run();
}
```

And you have to use with asynchronous iterator `IAsyncEnumerable<T>` on the webapi result type:

```csharp
// Use IAsyncEnumerable<T> type on entry point result.
[HttpGet]
public IAsyncEnumerable<WeatherForecast> Get()
{
    // (Asynchronous iteration implementation)
}
```

The sample fragment use with `HttpGet`, but this serializer can use any other http methods.

Complete ASP.NET Core example projects are located in the [samples directory](samples/).

----

## How to use (Client)

It is very easy to use, just install this package.

|Serializer|Application|
|:----|:----|
|Newtonsoft.Json|Install package [JsonStreamer.NewtonsoftJson.Client](https://www.nuget.org/packages/JsonStreamer.NewtonsoftJson.Client).|
|System.Text.Json|TODO:|

### For using with HttpClient

Available varies of many overloads.

```csharp
using System.Net.Http;
using JsonStreamer;

var httpClient = new HttpClient();

// ...

// HTTP GET
await foreach (var order in
    await httpClient.GetStreamingAsync<Order>(
        "https://example.com/api/orders"))
{
    // Already deserialized the "Order" object.
}

// HTTP POST
await foreach (var order in
    await httpClient.PostStreamingAsync<Order>(
        "https://example.com/api/orders",
        new StringContent("ORDER DATA")))
{
    // Already deserialized the "Order" object.
}
```

```csharp
var httpClient = new HttpClient();

// ...

// Get from the response interface
await using var response = await httpClient.GetAsync(...);

await foreach (var order in
    await response.Content.ReadStreamingAsync<Order>())
{
    // Already deserialized the "Order" object.
}
```

### For using with Stream

You can use JsonStreamer outsite HTTP access.

```csharp
using System.IO;
using JsonStreamer;

using var fs = new FileStream(
    "huge_data.jsonl",
    FileMode.Open, FileAccess.Read, FileShare.Read,
    65536, true);

// ...

await foreach (var order in
    await fs.ReadStreamingAsync<Order>())
{
    // Already deserialized the "Order" object.
}
```

----

## Tips and Notes

### Newtonsoft.Json version is NOT buffering

ASP.NET Core official package [Microsoft.AspNetCore.Mvc.NewtonsoftJson](https://www.nuget.org/packages/Microsoft.AspNetCore.Mvc.NewtonsoftJson/) has a problem for serializing `IAsyncEnumerable<T>` as a JSON array actually buffers the data entirely.

* See also: [ASP.NET Core `AsyncEnumerableReader.ReadInternal<T>()`](https://github.com/dotnet/aspnetcore/blob/e6c7c01bce4fce79bf5bc84098ea8d347ef358cc/src/Mvc/Mvc.Core/src/Infrastructure/AsyncEnumerableReader.cs#L79)

However, the JsonStreamer implementation avoids this limitation because the operation of serializing JObject elements is performed in each iterative asynchronous iteration.

### JavaScript deserializer

We introduced [can-ndjson-stream](https://github.com/canjs/can-ndjson-stream) for deserialization in JavaScript,
which can be used to configure the following asynchronous iterator (in TypeScript):

```typescript
import ndjsonStream from 'can-ndjson-stream';

async function* streamingFetch<T>(url: string) {
    const response = await fetch(url);
    const reader =
        ndjsonStream(response.body).
        getReader();
    while (true) {
        const result = await reader.read();
        if (result.done) {
            break;
        }
        yield result.value as T;
    }
}
```

The `streamingFetch()` function can be used to drive an asynchronous iterator as follows:

```typescript
interface WeatherForecast {
    date: string;
    temperatureC: number;
    temperatureF: number;
    summary: string;
}

async function fetchWeatherForeastItems() {
    const url = "http://localhost:4242/weatherforecast";

    for await (const item of streamingFetch<WeatherForecast>(url)) {

        // ...
    }
}
```


----

## TODO

* Enabling attribute-based control.


----

## License

Apache-v2

## History

* 1.1.0:
  * Supported .NET 9.0.
  * Supported ASP.NET Core 9.0.
* 1.0.0:
  * Supported ASP.NET Core 8.0 and 3.0 (netcoreapp3.0).
  * Downgraded ASP.NET Core Newtonsoft.Json package requirements.
* 0.4.0:
  * (Breaking change) Renamed package name `JsonStreamer.*` instead of `AspNetCore.JsonStreamer.*`.
  * Added JsonStreamer client package.
* 0.3.0:
  * Supported System.Text.Json based serializing.
  * Improved cancellation.
* 0.2.0:
  * Refactored.
* 0.1.0:
  * Initial release for Newtonsoft.Json serializer.
