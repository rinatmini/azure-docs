---
title: Guide for running C# Azure Functions in an isolated worker process
description: Learn how to use a .NET isolated worker process to run your C# functions in Azure, which supports non-LTS versions of .NET and .NET Framework apps.  
ms.service: azure-functions
ms.topic: conceptual 
ms.date: 01/16/2023
ms.custom: template-concept, devx-track-dotnet
recommendations: false
#Customer intent: As a developer, I need to know how to create functions that run in an isolated worker process so that I can run my function code on current (not LTS) releases of .NET.
---

# Guide for running C# Azure Functions in an isolated worker process

This article is an introduction to working with .NET Functions isolated worker process, which runs your functions in an isolated worker process in Azure. This allows you to run your .NET class library functions on a version of .NET that is different from the version used by the Functions host process. For information about specific .NET versions supported, see [supported version](#supported-versions). 

Use the following links to get started right away building .NET isolated worker process functions.

| Getting started | Concepts| Samples |
|--|--|--| 
| <ul><li>[Using Visual Studio Code](create-first-function-vs-code-csharp.md?tabs=isolated-process)</li><li>[Using command line tools](create-first-function-cli-csharp.md?tabs=isolated-process)</li><li>[Using Visual Studio](functions-create-your-first-function-visual-studio.md?tabs=isolated-process)</li></ul> | <ul><li>[Hosting options](functions-scale.md)</li><li>[Monitoring](functions-monitoring.md)</li> | <ul><li>[Reference samples](https://github.com/Azure/azure-functions-dotnet-worker/tree/main/samples)</li></ul> |

If you still need to run your functions in the same process as the host, see [In-process C# class library functions](functions-dotnet-class-library.md). 

For a comprehensive comparison between isolated worker process and in-process .NET Functions, see [Differences between in-process and isolate worker process .NET Azure Functions](dotnet-isolated-in-process-differences.md).

## Why .NET Functions isolated worker process?

When it was introduced, Azure Functions only supported a tightly integrated mode for .NET functions. In this _in-process_ mode, your [.NET class library functions](functions-dotnet-class-library.md) run in the same process as the host. This mode provides deep integration between the host process and the functions. For example, when running in the same process .NET class library functions can share binding APIs and types. However, this integration also requires a tight coupling between the host process and the .NET function. For example, .NET functions running in-process are required to run on the same version of .NET as the Functions runtime. This means that your in-process functions can only run on version of .NET with Long Term Support (LTS). To enable you to run on non-LTS version of .NET, you can instead choose to run in an isolated worker process. This process isolation lets you develop functions that use current .NET releases not natively supported by the Functions runtime, including .NET Framework. Both isolated worker process and in-process C# class library functions run on LTS versions. To learn more, see [Supported versions][supported-versions]. 

Because these functions run in a separate process, there are some [feature and functionality differences](./dotnet-isolated-in-process-differences.md) between .NET isolated function apps and .NET class library function apps.

### Benefits of isolated worker process

When your .NET functions run in an isolated worker process, you can take advantage of the following benefits: 

+ Fewer conflicts: because the functions run in a separate process, assemblies used in your app won't conflict with different version of the same assemblies used by the host process.  
+ Full control of the process: you control the start-up of the app and can control the configurations used and the middleware started.
+ Dependency injection: because you have full control of the process, you can use current .NET behaviors for dependency injection and incorporating middleware into your function app. 

[!INCLUDE [functions-dotnet-supported-versions](../../includes/functions-dotnet-supported-versions.md)]

## .NET isolated worker process project

A .NET isolated function project is basically a .NET console app project that targets a supported .NET runtime. The following are the basic files required in any .NET isolated project:

+ [host.json](functions-host-json.md) file.
+ [local.settings.json](functions-develop-local.md#local-settings-file) file.
+ C# project file (.csproj) that defines the project and dependencies.
+ Program.cs file that's the entry point for the app.
+ Any code files [defining your functions](#bindings).
 
For complete examples, see the [.NET 6 isolated sample project](https://github.com/Azure/azure-functions-dotnet-worker/tree/main/samples/FunctionApp) and the [.NET Framework 4.8 isolated sample project](https://github.com/Azure/azure-functions-dotnet-worker/tree/main/samples/NetFxWorker).

> [!NOTE]
> To be able to publish your isolated function project to either a Windows or a Linux function app in Azure, you must set a value of `dotnet-isolated` in the remote [FUNCTIONS_WORKER_RUNTIME](functions-app-settings.md#functions_worker_runtime) application setting. To support [zip deployment](deployment-zip-push.md) and [running from the deployment package](run-functions-from-deployment-package.md) on Linux, you also need to update the `linuxFxVersion` site config setting to `DOTNET-ISOLATED|7.0`. To learn more, see [Manual version updates on Linux](set-runtime-version.md#manual-version-updates-on-linux). 

## Package references

A .NET Functions isolated worker process project uses a unique set of packages, for both core functionality and binding extensions. 

### Core packages 

The following packages are required to run your .NET functions in an isolated worker process:

+ [Microsoft.Azure.Functions.Worker](https://www.nuget.org/packages/Microsoft.Azure.Functions.Worker/)
+ [Microsoft.Azure.Functions.Worker.Sdk](https://www.nuget.org/packages/Microsoft.Azure.Functions.Worker.Sdk/)

### Extension packages

Because .NET isolated worker process functions use different binding types, they require a unique set of binding extension packages. 

You'll find these extension packages under [Microsoft.Azure.Functions.Worker.Extensions](https://www.nuget.org/packages?q=Microsoft.Azure.Functions.Worker.Extensions).

## Start-up and configuration 

When using .NET isolated functions, you have access to the start-up of your function app, which is usually in Program.cs. You're responsible for creating and starting your own host instance. As such, you also have direct access to the configuration pipeline for your app. With .NET Functions isolated worker process, you can much more easily add configurations, inject dependencies, and run your own middleware. 

The following code shows an example of a [HostBuilder] pipeline:

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/FunctionApp/Program.cs" id="docsnippet_startup":::

This code requires `using Microsoft.Extensions.DependencyInjection;`. 

A [HostBuilder] is used to build and return a fully initialized [IHost] instance, which you run asynchronously to start your function app. 

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/FunctionApp/Program.cs" id="docsnippet_host_run":::

> [!IMPORTANT]
> If your project targets .NET Framework 4.8, you also need to add `FunctionsDebugger.Enable();` before creating the HostBuilder. It should be the first line of your `Main()` method. For more information, see [Debugging when targeting .NET Framework](#debugging-when-targeting-net-framework).

### Configuration

The [ConfigureFunctionsWorkerDefaults] method is used to add the settings required for the function app to run in an isolated worker process, which includes the following functionality:

+ Default set of converters.
+ Set the default [JsonSerializerOptions] to ignore casing on property names.
+ Integrate with Azure Functions logging.
+ Output binding middleware and features.
+ Function execution middleware.
+ Default gRPC support. 

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/FunctionApp/Program.cs" id="docsnippet_configure_defaults" :::   

Having access to the host builder pipeline means that you can also set any app-specific configurations during initialization. You can call the [ConfigureAppConfiguration] method on [HostBuilder] one or more times to add the configurations required by your function app. To learn more about app configuration, see [Configuration in ASP.NET Core](/aspnet/core/fundamentals/configuration/?view=aspnetcore-5.0&preserve-view=true). 

These configurations apply to your function app running in a separate process. To make changes to the functions host or trigger and binding configuration, you'll still need to use the [host.json file](functions-host-json.md).   

### Dependency injection

Dependency injection is simplified, compared to .NET class libraries. Rather than having to create a startup class to register services, you just have to call [ConfigureServices] on the host builder and use the extension methods on [IServiceCollection] to inject specific services. 

The following example injects a singleton service dependency:  
 
:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/FunctionApp/Program.cs" id="docsnippet_dependency_injection" :::

This code requires `using Microsoft.Extensions.DependencyInjection;`. To learn more, see [Dependency injection in ASP.NET Core](/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-5.0&preserve-view=true).

### Middleware

.NET isolated also supports middleware registration, again by using a model similar to what exists in ASP.NET. This model gives you the ability to inject logic into the invocation pipeline, and before and after functions execute.

The [ConfigureFunctionsWorkerDefaults] extension method has an overload that lets you register your own middleware, as you can see in the following example.  

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/CustomMiddleware/Program.cs" id="docsnippet_middleware_register" :::

 The `UseWhen` extension method can be used to register a middleware that gets executed conditionally. You must pass to this method a predicate that returns a boolean value, and the middleware participates in the invocation processing pipeline when the return value of the predicate is `true`.

The following extension methods on [FunctionContext] make it easier to work with middleware in the isolated model.

| Method | Description |
| ---- | ---- |
| **`GetHttpRequestDataAsync`** | Gets the `HttpRequestData` instance when called by an HTTP trigger. This method returns an instance of `ValueTask<HttpRequestData?>`, which is useful when you want to read message data, such as request headers and cookies. |
| **`GetHttpResponseData`** | Gets the `HttpResponseData` instance when called by an HTTP trigger. |
|  **`GetInvocationResult`** | Gets an instance of `InvocationResult`, which represents the result of the current function execution. Use the `Value` property to get or set the value as needed. |
|  **` GetOutputBindings`** | Gets the output binding entries for the current function execution. Each entry in the result of this method is of type `OutputBindingData`. You can use the `Value` property to get or set the value as needed. | 
|  **` BindInputAsync`** | Binds an input binding item for the requested `BindingMetadata` instance. For example, you can use this method when you have a function with a `BlobInput` input binding that needs to be accessed or updated by your middleware. |

The following is an example of a middleware implementation that reads the `HttpRequestData` instance and updates the `HttpResponseData` instance during function execution. This middleware checks for the presence of a specific request header(x-correlationId), and when present uses the header value to stamp a response header. Otherwise, it generates a new GUID value and uses that for stamping the response header.
 
:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/CustomMiddleware/StampHttpHeaderMiddleware.cs" id="docsnippet_middleware_example_stampheader" :::
 
For a more complete example of using custom middleware in your function app, see the [custom middleware reference sample](https://github.com/Azure/azure-functions-dotnet-worker/blob/main/samples/CustomMiddleware).

## Cancellation tokens

A function can accept a [CancellationToken](/dotnet/api/system.threading.cancellationtoken) parameter, which enables the operating system to notify your code when the function is about to be terminated. You can use this notification to make sure the function doesn't terminate unexpectedly in a way that leaves data in an inconsistent state.

Cancellation tokens are supported in .NET functions when running in an isolated worker process. The following example raises an exception when a cancellation request has been received:

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/Net7Worker/EventHubCancellationToken.cs" id="docsnippet_cancellation_token_throw":::
 
The following example performs clean-up actions if a cancellation request has been received:

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/Net7Worker/EventHubCancellationToken.cs" id="docsnippet_cancellation_token_cleanup":::

## ReadyToRun

You can compile your function app as [ReadyToRun binaries](/dotnet/core/deploying/ready-to-run). ReadyToRun is a form of ahead-of-time compilation that can improve startup performance to help reduce the effect of [cold-start](event-driven-scaling.md#cold-start) when running in a [Consumption plan](consumption-plan.md).

ReadyToRun is available in .NET 6 and later versions and requires [version 4.0 or later](functions-versions.md) of the Azure Functions runtime.

To compile your project as ReadyToRun, update your project file by adding the `<PublishReadyToRun>` and `<RuntimeIdentifier>` elements. The following is the configuration for publishing to a Windows 32-bit function app.

```xml
<PropertyGroup>
  <TargetFramework>net6.0</TargetFramework>
  <AzureFunctionsVersion>v4</AzureFunctionsVersion>
  <RuntimeIdentifier>win-x86</RuntimeIdentifier>
  <PublishReadyToRun>true</PublishReadyToRun>
</PropertyGroup>
```

## Execution context

.NET isolated passes a [FunctionContext] object to your function methods. This object lets you get an [ILogger] instance to write to the logs by calling the [GetLogger] method and supplying a `categoryName` string. To learn more, see [Logging](#logging). 

## Bindings 

Bindings are defined by using attributes on methods, parameters, and return types. A function method is a method with a `Function` attribute and a trigger attribute applied to an input parameter, as shown in the following example:

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/Extensions/Queue/QueueFunction.cs" id="docsnippet_queue_trigger" :::

The trigger attribute specifies the trigger type and binds input data to a method parameter. The previous example function is triggered by a queue message, and the queue message is passed to the method in the `myQueueItem` parameter.

The `Function` attribute marks the method as a function entry point. The name must be unique within a project, start with a letter and only contain letters, numbers, `_`, and `-`, up to 127 characters in length. Project templates often create a method named `Run`, but the method name can be any valid C# method name.

Bindings can provide data as strings, arrays, and serializable types, such as plain old class objects (POCOs). You can also bind to [types from some service SDKs](#sdk-types-preview).

For HTTP triggers, you must use [HttpRequestData] and [HttpResponseData] to access the request and response data. This is because you don't have access to the original HTTP request and response objects when using .NET Functions isolated worker process.

For a complete set of reference samples for using triggers and bindings with isolated worker process functions, see the [binding extensions reference sample](https://github.com/Azure/azure-functions-dotnet-worker/blob/main/samples/Extensions). 

### Input bindings

A function can have zero or more input bindings that can pass data to a function. Like triggers, input bindings are defined by applying a binding attribute to an input parameter. When the function executes, the runtime tries to get data specified in the binding. The data being requested is often dependent on information provided by the trigger using binding parameters.  

### Output bindings

To write to an output binding, you must apply an output binding attribute to the function method, which defined how to write to the bound service. The value returned by the method is written to the output binding. For example, the following example writes a string value to a message queue named `output-queue` by using an output binding:

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/Extensions/Queue/QueueFunction.cs" id="docsnippet_queue_output_binding" :::

### Multiple output bindings

The data written to an output binding is always the return value of the function. If you need to write to more than one output binding, you must create a custom return type. This return type must have the output binding attribute applied to one or more properties of the class. The following example from an HTTP trigger writes to both the HTTP response and a queue output binding:

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/Extensions/MultiOutput/MultiOutput.cs" id="docsnippet_multiple_outputs":::

The response from an HTTP trigger is always considered an output, so a return value attribute isn't required.

### SDK types (preview)

For some service-specific binding types, binding data can be provided using types from service SDKs and frameworks. These provide additional capability beyond what a serialized string or plain-old CLR object (POCO) may offer. Support for SDK types is currently in preview with limited scenario coverage.

To use SDK type bindings, your project must reference [Microsoft.Azure.Functions.Worker 1.12.1-preview1 or later][sdk-types-worker-version] and [Microsoft.Azure.Functions.Worker.Sdk 1.9.0-preview1 or later][sdk-types-worker-sdk-version]. Specific package versions will be needed for each of the service extensions as well. When testing SDK types locally on your machine, you will also need to use [Azure Functions Core Tools version 4.0.5000 or later](./functions-run-local.md). You can check your current version using the command `func version`.

The following service-specific bindings are currently included in the preview:

| Service | Trigger | Input binding | Output binding |
|-|-|-|-|
| [Azure Blobs][blob-sdk-types] | Preview support | Preview support  |  Not yet supported<sup>1</sup> | 
| [Azure Cosmos DB][cosmos-sdk-types] | SDK types not used<sup>2</sup> | Preview support  |  Not yet supported<sup>1</sup> | 

[blob-sdk-types]: ./functions-bindings-storage-blob.md?tabs=isolated-process%2Cextensionv5&pivots=programming-language-csharp#binding-types
[cosmos-sdk-types]: ./functions-bindings-cosmosdb-v2.md?tabs=isolated-process%2Cextensionv4&pivots=programming-language-csharp#binding-types

<sup>1</sup> Support for SDK type bindings does not presently extend to output bindings.

<sup>2</sup> The Cosmos DB trigger uses the [Azure Cosmos DB change feed](../cosmos-db/change-feed.md) and exposes change feed items as JSON-serializable types. The absence of SDK types is by-design for this scenario.

The [SDK type binding samples](https://github.com/Azure/azure-functions-dotnet-worker/tree/main/samples/WorkerBindingSamples) show examples of working with the various supported types.

> [!NOTE]
> When using [binding expressions](./functions-bindings-expressions-patterns.md) that rely on trigger data, SDK types for the trigger itself are not supported.

[sdk-types-worker-version]: https://www.nuget.org/packages/Microsoft.Azure.Functions.Worker/1.12.1-preview1
[sdk-types-worker-sdk-version]: https://www.nuget.org/packages/Microsoft.Azure.Functions.Worker.Sdk/1.9.0-preview1

### HTTP trigger

[HTTP triggers](./functions-bindings-http-webhook-trigger.md) allow a function to be invoked by an HTTP request. There are two different approaches that can be used:

- A [built-in model](#built-in-http-model) which does not require additional dependencies and uses custom types for HTTP requests and responses
- An [ASP.NET Core integration model (Preview)](#aspnet-core-integration-preview) that uses concepts familiar to ASP.NET Core developers

#### Built-in HTTP model

In the built-in model, the system translates the incoming HTTP request message into an [HttpRequestData] object that is passed to the function. This object provides data from the request, including `Headers`, `Cookies`, `Identities`, `URL`, and optionally a message `Body`. This object is a representation of the HTTP request but is not directly connected to the underlying HTTP listener or the received message. 

Likewise, the function returns an [HttpResponseData] object, which provides data used to create the HTTP response, including message `StatusCode`, `Headers`, and optionally a message `Body`.  

The following example demonstrates the use of `HttpRequestData` and `HttpResponseData`:

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/Extensions/Http/HttpFunction.cs" id="docsnippet_http_trigger" :::

#### ASP.NET Core integration (preview)

This section shows how to work with the underlying HTTP request and response objects using types from ASP.NET Core including [HttpRequest], [HttpResponse], and [IActionResult]. Use of this feature for local testing requires [Core Tools version 4.0.5198 or later](./functions-run-local.md). This model is not available to [apps targeting .NET Framework][supported-versions], which should instead leverage the [built-in model](#built-in-http-model).

> [!NOTE]
> Not all features of ASP.NET Core are exposed by this model. Specifically, the ASP.NET Core middleware pipeline and routing capabilities are not available.

1. Add a reference to the [Microsoft.Azure.Functions.Worker.Extensions.Http.AspNetCore NuGet package, version 1.0.0-preview2 or later](https://www.nuget.org/packages/Microsoft.Azure.Functions.Worker.Extensions.Http.AspNetCore/1.0.0-preview2) to your project.

  You must also update your project to use [version 1.11.0 or later of Microsoft.Azure.Functions.Worker.Sdk](https://www.nuget.org/packages/Microsoft.Azure.Functions.Worker.Sdk/1.11.0) and [version 1.16.0 or later of Microsoft.Azure.Functions.Worker](https://www.nuget.org/packages/Microsoft.Azure.Functions.Worker/1.16.0).

2. In your `Program.cs` file, update the host builder configuration to use `ConfigureFunctionsWebApplication()` instead of `ConfigureFunctionsWorkerDefaults()`. The following example shows a minimal setup without other customizations:

    ```csharp
    using Microsoft.Extensions.Hosting;
    using Microsoft.Azure.Functions.Worker;
    
    var host = new HostBuilder()
        .ConfigureFunctionsWebApplication()
        .Build();
    
    host.Run();
    ```

3. You can then update your HTTP-triggered functions to use the ASP.NET Core types. The following example shows `HttpRequest` and an `IActionResult` used for a simple "hello, world" function:

    ```csharp
    [Function("HttpFunction")]
    public IActionResult Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get")] HttpRequest req)
    {
        return new OkObjectResult($"Welcome to Azure Functions, {req.Query["name"]}!");
    }
    ```

4. Enable the feature by setting `AzureWebJobsFeatureFlags` to include "EnableHttpProxying". When hosted in a function app, configure this as an application setting. When running locally, set this value in `local.settings.json`.

## Logging

In .NET isolated, you can write to logs by using an [ILogger] instance obtained from a [FunctionContext] object passed to your function. Call the [GetLogger] method, passing a string value that is the name for the category in which the logs are written. The category is usually the name of the specific function from which the logs are written. To learn more about categories, see the [monitoring article](functions-monitoring.md#log-levels-and-categories). 

The following example shows how to get an [ILogger] and write logs inside a function:

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/Extensions/Http/HttpFunction.cs" id="docsnippet_logging" ::: 

Use various methods of [ILogger] to write various log levels, such as `LogWarning` or `LogError`. To learn more about log levels, see the [monitoring article](functions-monitoring.md#log-levels-and-categories).

An [ILogger] is also provided when using [dependency injection](#dependency-injection).

## Debugging when targeting .NET Framework

If your isolated project targets .NET Framework 4.8, the current preview scope requires manual steps to enable debugging. These steps aren't required if using another target framework.

Your app should start with a call to `FunctionsDebugger.Enable();` as its first operation. This occurs in the `Main()` method before initializing a HostBuilder. Your `Program.cs` file should look similar to the following:

```csharp
using System;
using System.Diagnostics;
using Microsoft.Extensions.Hosting;
using Microsoft.Azure.Functions.Worker;
using NetFxWorker;

namespace MyDotnetFrameworkProject
{
    internal class Program
    {
        static void Main(string[] args)
        {
            FunctionsDebugger.Enable();

            var host = new HostBuilder()
                .ConfigureFunctionsWorkerDefaults()
                .Build();

            host.Run();
        }
    }
}
```

Next, you need to manually attach to the process using a .NET Framework debugger. Visual Studio doesn't do this automatically for isolated worker process .NET Framework apps yet, and the "Start Debugging" operation should be avoided.

In your project directory (or its build output directory), run:

```azurecli
func host start --dotnet-isolated-debug
```

This will start your worker, and the process will stop with the following message:

```azurecli
Azure Functions .NET Worker (PID: <process id>) initialized in debug mode. Waiting for debugger to attach...
```

Where `<process id>` is the ID for your worker process. You can now use Visual Studio to manually attach to the process. For instructions on this operation, see [How to attach to a running process](/visualstudio/debugger/attach-to-running-processes-with-the-visual-studio-debugger#BKMK_Attach_to_a_running_process).

After the debugger is attached, the process execution resumes, and you'll be able to debug.

## Remote Debugging using Visual Studio

Because your isolated worker process app runs outside the Functions runtime, you need to attach the remote debugger to a separate process. To learn more about debugging using Visual Studio, see [Remote Debugging](functions-develop-vs.md?tabs=isolated-process#remote-debugging).

## Next steps

+ [Learn more about triggers and bindings](functions-triggers-bindings.md)
+ [Learn more about best practices for Azure Functions](functions-best-practices.md)


[supported-versions]: #supported-versions
[HostBuilder]: /dotnet/api/microsoft.extensions.hosting.hostbuilder
[IHost]: /dotnet/api/microsoft.extensions.hosting.ihost
[ConfigureFunctionsWorkerDefaults]: /dotnet/api/microsoft.extensions.hosting.workerhostbuilderextensions.configurefunctionsworkerdefaults?view=azure-dotnet&preserve-view=true#Microsoft_Extensions_Hosting_WorkerHostBuilderExtensions_ConfigureFunctionsWorkerDefaults_Microsoft_Extensions_Hosting_IHostBuilder_
[ConfigureAppConfiguration]: /dotnet/api/microsoft.extensions.hosting.hostbuilder.configureappconfiguration
[IServiceCollection]: /dotnet/api/microsoft.extensions.dependencyinjection.iservicecollection
[ConfigureServices]: /dotnet/api/microsoft.extensions.hosting.hostbuilder.configureservices
[FunctionContext]: /dotnet/api/microsoft.azure.functions.worker.functioncontext?view=azure-dotnet&preserve-view=true
[ILogger]: /dotnet/api/microsoft.extensions.logging.ilogger
[ILogger&lt;T&gt;]: /dotnet/api/microsoft.extensions.logging.ilogger-1
[GetLogger]: /dotnet/api/microsoft.azure.functions.worker.functioncontextloggerextensions.getlogger?view=azure-dotnet&preserve-view=true
[BlobClient]: /dotnet/api/azure.storage.blobs.blobclient?view=azure-dotnet&preserve-view=true
[DocumentClient]: /dotnet/api/microsoft.azure.documents.client.documentclient
[BrokeredMessage]: /dotnet/api/microsoft.servicebus.messaging.brokeredmessage
[HttpRequestData]: /dotnet/api/microsoft.azure.functions.worker.http.httprequestdata?view=azure-dotnet&preserve-view=true
[HttpResponseData]: /dotnet/api/microsoft.azure.functions.worker.http.httpresponsedata?view=azure-dotnet&preserve-view=true
[HttpRequest]: /dotnet/api/microsoft.aspnetcore.http.httprequest
[HttpResponse]: /dotnet/api/microsoft.aspnetcore.http.httpresponse
[IActionResult]: /dotnet/api/microsoft.aspnetcore.mvc.iactionresult
[JsonSerializerOptions]: /dotnet/api/system.text.json.jsonserializeroptions
