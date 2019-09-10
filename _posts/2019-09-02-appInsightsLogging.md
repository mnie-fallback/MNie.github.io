---
layout: post
title: AppInsights logging full requests with hidding sensitive data
---

In our application, daily, we are using Application Insights to report all requests to and from us so we could monitor the actual status of a system. Thanks to that we are sure that our clients wouldn't experience unpleasantness because of not working or slowly working application. To look at the body of each request and response, we store them in a separate Log database. As long as the application growth, we have more and more users and regions to handle we thought it would be easier for us to have everything in one place. So we planned to involve everything in ApplicationInsights.

All of this sounds pretty easy, but we encounter a couple of problems.
First of them was that when we listen in middleware for a request/response to/from a server so we could send it body to an Application Insights is not that easy, when we tried to read a data stream we get an exception. This is caused by built-in middleware which already read the body but didn't keep a stream in a state that would be ready for another read ([link to SO](https://stackoverflow.com/questions/42686363/view-post-request-body-in-application-insights)). So as a workaround to this problem, we decided to cache a request/response body in an in-memory cache for a very short period. So we could read it when we want to send it to an Azure. The important thing here is that we cache this data per instance of a service, so every instance has it's own cache for request/response body so the availability time would be very low. Also because of that we used a very short lifetime and bounded cache size, so it wouldn't be growth to the enormous sizes. The caching approach itself we implement like this:

```csharp
public class RewindFilter : ActionFilterAttribute
{
    private readonly IRequestDataAccessor _bodyAccessor;
    private readonly ITelemetryEnricher _telemetryEnricher;
    private readonly ILogger<RewindFilter> _logger;

    public RewindFilter(IRequestDataAccessor bodyAccessor, ILogger<RewindFilter> logger, ITelemetryEnricher telemetryEnricher)
    {
        _bodyAccessor = bodyAccessor;
        _logger = logger;
        _telemetryEnricher = telemetryEnricher;
    }

    public override async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        if (_telemetryEnricher.AttachRequest(context.HttpContext))
            _bodyAccessor.SetBody(context.HttpContext, context.ActionDescriptor, context.ActionArguments)
                .Bind(
                    x => x.ToSuccess(), 
                    err =>
                    {
                        _logger.LogWarning(err.Message, err);
                        return err.Message.ToFailure<Unit>();
                    });
        
        await next();
    }

    public override async Task OnResultExecutionAsync(ResultExecutingContext context, ResultExecutionDelegate next)
    {
        if (_telemetryEnricher.AttachResponse(context.HttpContext))
            _bodyAccessor.SetBody(context.HttpContext, context.Result)
                .Bind(
                    x => x.ToSuccess(), 
                    err =>
                    {
                        _logger.LogWarning(err.Message, err);
                        return err.Message.ToFailure<Unit>();
                    });
        await next();
    }
}
```

We could found here a .net core filter, which is responsible for catching requests/response to/from server and then save them in a cache.

TelemetryEnricher implementation looks like this:

```csharp
public interface ITelemetryEnricher
{
    ITelemetry Enrich(ITelemetry tele);
    bool AttachRequest(HttpContext context);
    bool AttachResponse(HttpContext context);
}

public class DefaultTelemetryEnricher : ITelemetryEnricher
{
    public ITelemetry Enrich(ITelemetry tele) => tele;
    public bool AttachRequest(HttpContext context) => true;
    public bool AttachResponse(HttpContext context) => true;
}
```

Cache implementation looks like this and in fact it encapsulates a IMemoryCache.

```csharp
internal static class KeyGenerator
{
    internal static string Request(string id) => $"Request_{id}";
    internal static string Response(string id) => $"Response_{id}";
}

public class RequestDataAccessor : IRequestDataAccessor
{
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _expirationInMs;

    public RequestDataAccessor(long cacheSize, int scanFrequencyMs, int expirationInMs)
    {
        var cacheOptions = new MemoryCacheOptions
        {
            ExpirationScanFrequency = TimeSpan.FromMilliseconds(scanFrequencyMs),
            SizeLimit = cacheSize
        };
        _cache = new MemoryCache(cacheOptions);
        _expirationInMs = TimeSpan.FromMilliseconds(expirationInMs);
    }

    public Result<string> GetBody(string traceInitializer)
    {
        try
        {
            var success = _cache.TryGetValue(traceInitializer, out var result);
            if (!success)
                return $"record under key {traceInitializer} was not found in a cache".ToFailure<string>();

            switch (result)
            {
                case string s:
                    return s.ToSuccess();
                default:
                    return
                        $"record under key: {traceInitializer} was found but was of a different type: {result.GetType()} than string"
                            .ToFailure<string>();
            }
        }
        catch (Exception e)
        {
            return e.Message.ToFailure<string>();
        }
    }

    public Result<Unit> SetBody(HttpContext context, ActionDescriptor descriptor,
        IDictionary<string, object> args) =>
        ReadRequest(descriptor, args)
            .Bind(body => SetEntryInCache(KeyGenerator.Request(context.TraceIdentifier), body));

    public Result<Unit> SetBody(HttpContext context, IActionResult result) =>
        ReadResponse(result)
            .Bind(body => SetEntryInCache(KeyGenerator.Response(context.TraceIdentifier), body));

    private static Result<string> ReadResponse(IActionResult result)
    {
        try
        {
            switch (result)
            {
                case ObjectResult r when r?.Value != null:
                    return JsonConvert.SerializeObject(
                        r.Value,
                        new JsonSerializerSettings()}
                    ).ToSuccess();
                default:
                    return string.Empty.ToFailure<string>();
            }
        }
        catch (Exception e)
        {
            return e.Message.ToFailure<string>();
        }
    }

    private static Result<string> ReadRequest(ActionDescriptor actionDescriptor,
        IDictionary<string, object> args)
    {
        try
        {
            var methodInfo = ((Microsoft.AspNetCore.Mvc.Controllers.ControllerActionDescriptor)actionDescriptor).MethodInfo;
            var noLogParameters = methodInfo.GetParameters().Where(p => p.GetCustomAttributes(true).Any(t => t.GetType() == typeof(SensitiveAttribute))).Select(p => p.Name);

            var hiddenSensitiveData = args
                .Where(a => !noLogParameters.Contains(a.Key) && a.Value != null)
                .Select(argument => JsonConvert.SerializeObject(argument.Value,
                    new JsonSerializerSettings())
                )
                .ToList();

            return hiddenSensitiveData.Any()
                ? string.Join(Environment.NewLine, hiddenSensitiveData).ToSuccess()
                : "data is empty".ToFailure<string>();
        }
        catch (Exception e)
        {
            return e.Message.ToFailure<string>();
        }
    }

    private Result<Unit> SetEntryInCache(string traceInitializer, string body)
    {
        try
        {
            _cache.Set(traceInitializer, body, new MemoryCacheEntryOptions()
            {
                SlidingExpiration = _expirationInMs,
                Size = 1
            });
            var successSave = _cache.TryGetValue(traceInitializer, out var _);
            return successSave
                ? ResultFactory.CreateSuccess()
                : $"entry under key {traceInitializer} was not saved in a cache".ToFailure<Unit>();
        }
        catch (Exception e)
        {
            return e.Message.ToFailure<Unit>();
        }
    }
}
```

Encapsulation here is not a mistake, because we want to have full control of how an instance of a cache is created in concrete service, so the constructor has a lifetime, size parameter. Also, we could see here how we build keys, this is because we want to distinguish the request from response bodies.

Okay, we listen to requests and responses but how the call to AppInsights looks like?

```csharp
public class Initializer : ITelemetryInitializer
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly IRequestDataAccessor _requestDataAccessor;
    private readonly ITelemetryEnricher _telemetryEnricher;
    public Initializer(IHttpContextAccessor httpContextAccessor, IRequestDataAccessor requestDataAccessor, ITelemetryEnricher telemetryEnricher)
    {
        _httpContextAccessor = httpContextAccessor;
        _requestDataAccessor = requestDataAccessor;
        _telemetryEnricher = telemetryEnricher;
    }

    public void Initialize(ITelemetry telemetry)
    {
        telemetry = _telemetryEnricher.Enrich(telemetry);

        if(_telemetryEnricher.AttachRequest(_httpContextAccessor.HttpContext))
            AddRequestBody(telemetry);
        
        if(_telemetryEnricher.AttachResponse(_httpContextAccessor.HttpContext))
            AddResponseBody(telemetry);
    }

    protected void AddRequestBody(ITelemetry telemetry)
    {
        switch (telemetry)
        {
            case RequestTelemetry requestTelemetry when HasBody() && int.TryParse(requestTelemetry.ResponseCode, out var code):
            {
                if(!requestTelemetry.Properties.ContainsKey("RequestBody") && IsAnError(code))
                    GetBody(KeyGenerator.Request)
                        .Bind(data =>
                        {
                            requestTelemetry.Properties.Add("RequestBody", data);
                            return data.ToSuccess();
                        });
                break;
            }
        }
    }
    
    protected void AddResponseBody(ITelemetry telemetry)
    {
        switch (telemetry)
        {
            case RequestTelemetry requestTelemetry:
            {
                if(!requestTelemetry.Properties.ContainsKey("ResponseBody"))
                    GetBody(KeyGenerator.Response)
                        .Bind(data =>
                        {
                            requestTelemetry.Properties.Add("ResponseBody", data);
                            return data.ToSuccess();
                        });
                break;
            }
        }
    }

    private static bool IsAnError(int code) =>
        (code >= 400
        && code < 600)
        || code == 0;

    private Result<string> GetBody(Func<string, string> decorateIdentifier)
    {
        var context = _httpContextAccessor.HttpContext;
        return _requestDataAccessor.GetBody(decorateIdentifier(context.TraceIdentifier));
    }

    private bool HasBody()
    {
        var verbWhichSupportsBody = (_httpContextAccessor.HttpContext.Request.Method == HttpMethods.Post
                    || _httpContextAccessor.HttpContext.Request.Method == HttpMethods.Put
                    || _httpContextAccessor.HttpContext.Request.Method == HttpMethods.Patch);
        return verbWhichSupportsBody
                && _httpContextAccessor.HttpContext.Request.ContentLength > 0;
    }
}
```

As we could see we have to implement an `ITelemetryInitializer` interface from AppInsights nuget package, which is executed to send data to AppInsights. Here we get data from our cache. We don't delete them since we know the lifetime is set to very short so the delete operation has no sense here.

We also have to implement our `ITelemetryProcessor` to say when we want to send data to Azure:

```csharp
public interface IProcessorApplier
{
    bool Apply(ITelemetry tele);
}

public class DefaultProcessorApplier : IProcessorApplier
{
    public bool Apply(ITelemetry tele) => true;
}

public class TelemetryProcessor : ITelemetryProcessor
{
    private readonly ITelemetryProcessor _next;
    private readonly IProcessorApplier _applier;

    public TelemetryProcessor(ITelemetryProcessor next, IProcessorApplier applier)
    {
        _next = next;
        _applier = applier;
    }

    public void Process(ITelemetry item)
    {
        if (_applier.Apply(item))
            _next.Process(item);
    }
}
```

We run application and we could found records as such on Azure:

![old](https://mnie.github.com/img/02-09-2019AppInsights/old.png)

Everything looks good. But are we sure? As we could look at the data, we don't hide anything so we could send sensitive data to Azure which is not a good idea when we have a RODO/GDPR. Because of that, we introduce an attribute name it `Sensitive`, so we could mark a whole class, field or property we want to hide from logging to ApplicationInsights. We would change it's content to a `PII Data` string, so the similar behavior to hiding sensitive data in Connection Strings by Azure.

Attribute code looks like this:

```csharp
public class SensitiveAttribute : Attribute { }
```

And how it is consumed you could see here:

```csharp
internal class PIIValueProvider : IValueProvider
{
    private readonly object _defaultValue;

    public PIIValueProvider(string defaultValue) => _defaultValue = defaultValue;

    public object GetValue(object target) => _defaultValue;

    public void SetValue(object target, object value)
    {
    }
}

public class NoPIILogContractResolver : DefaultContractResolver
{
    protected override IList<JsonProperty> CreateProperties(Type type, MemberSerialization memberSerialization)
    {
        var properties = new List<JsonProperty>();

        var thereAreSomePropertiesToHide = type.GetCustomAttributes(true).All(t => t.GetType() != typeof(SensitiveAttribute));
        if (thereAreSomePropertiesToHide)
        {
            var props = base.CreateProperties(type, memberSerialization);
            var excludedProperties = type
                .GetProperties()
                .Where(p => p.GetCustomAttributes(true).Any(t => t.GetType() == typeof(SensitiveAttribute)))
                .Select(s => s.Name)
                .ToList();

            var excludedFields = type
                .GetFields()
                .Where(p => p.GetCustomAttributes(true).Any(t => t.GetType() == typeof(SensitiveAttribute)))
                .Select(s => s.Name)
                .ToList();

            var toExclude = excludedProperties.Concat(excludedFields).ToList();
            
            return props
                .Select(property => 
                    toExclude.Contains(property.PropertyName)
                        ? HideValue(property)
                        : property
                )
                .ToList();
        }

        return properties;
    }

    private static JsonProperty HideValue(JsonProperty property)
    {
        property.PropertyType = typeof(string);
        property.ValueProvider = new PIIValueProvider("PII Data");
        return property;
    }
}
```

We get an object we want to log to ApplicationInsights, we check all fields and properties if they should be hidden (based on an attribute) if so we change their value to `PII Data`. The above code works also for nested objects. 

We also need to include NoPIILogContractResolver to `IRequestDataAccessor` where we operate on JsonConvert, we did this like this:

```csharp
JsonConvert.SerializeObject(argument.Value,
                        new JsonSerializerSettings() {ContractResolver = new NoPIILogContractResolver()})
```

We run code once again and we could find records like this in ApplicationInsights:

![result3](https://mnie.github.com/img/02-09-2019AppInsights/new.png)
![result2](https://mnie.github.com/img/02-09-2019AppInsights/new2.png)

To sum up everything. In 3 very easy steps, we were able to add logging to ApplicationInsights which logs almost full request and response without any sensitive data (of course as long as a developer would mark all sensitive parts by attribute). But thanks to the above in a future we could more easily and faster encounter a problem with our application, which would help us save some time for other things. Only 1 thing to consider here is consumption of database under the hood for ApplicationInsights, maybe it would be a good idea to somehow limit the number of data logged per single request/response so some very big bodies wouldn't be logged as a full but only in some small parts.

[Full code here](https://github.com/MNie/AppInsights.Enricher)

Thanks for reading :)
