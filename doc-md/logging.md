TOC
* [Request-Response Logging Middleware Using Serilog](#SerilogMiddleware)
* [Request Logging Middleware](#)
* [Response Logging Middleware](#)
* Request-Response Logging Middleware Using ILogger
    * [Sample 1](#Custom-RequestResponseLoggingMiddleware-Using-ILogger)
    * [Sample 2](https://stackoverflow.com/a/51305912/4802664)
* [Request-Response Logging Middleware Using ILoggerService](#ApiLoggingMiddleware)
* [Routes ASP.NET Core log messages through Serilog](https://github.com/serilog/serilog-aspnetcore)

## SerilogMiddleware
Courtesy: [this](https://github.com/datalust/serilog-middleware-example/blob/master/src/Datalust.SerilogMiddlewareExample/Diagnostics/SerilogMiddleware.cs)   
`SerilogMiddleware.cs`
```cs
using Microsoft.AspNetCore.Http;
using Serilog;
using Serilog.Events;
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http.Features;

namespace Foo
{
    class SerilogMiddleware
    {
        const string MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";

        static readonly ILogger Log = Serilog.Log.ForContext<SerilogMiddleware>();

        static readonly HashSet<string> HeaderWhitelist = new HashSet<string> {"Content-Type", "Content-Length", "User-Agent"};

        readonly RequestDelegate _next;

        public SerilogMiddleware(RequestDelegate next)
        {
            _next = next ?? throw new ArgumentNullException(nameof(next));
        }

        // ReSharper disable once UnusedMember.Global
        public async Task Invoke(HttpContext httpContext)
        {
            if (httpContext == null) throw new ArgumentNullException(nameof(httpContext));

            var start = Stopwatch.GetTimestamp();
            try
            {
                await _next(httpContext);
                var elapsedMs = GetElapsedMilliseconds(start, Stopwatch.GetTimestamp());

                var statusCode = httpContext.Response?.StatusCode;
                var level = statusCode > 499 ? LogEventLevel.Error : LogEventLevel.Information;

                var log = level == LogEventLevel.Error ? LogForErrorContext(httpContext) : Log;
                log.Write(level, MessageTemplate, httpContext.Request.Method, GetPath(httpContext), statusCode, elapsedMs);
            }
            // Never caught, because `LogException()` returns false.
            catch (Exception ex) when (LogException(httpContext, GetElapsedMilliseconds(start, Stopwatch.GetTimestamp()), ex)) { }
        }

        static bool LogException(HttpContext httpContext, double elapsedMs, Exception ex)
        {
            LogForErrorContext(httpContext)
                .Error(ex, MessageTemplate, httpContext.Request.Method, GetPath(httpContext), 500, elapsedMs);

            return false;
        }

        static ILogger LogForErrorContext(HttpContext httpContext)
        {
            var request = httpContext.Request;

            var loggedHeaders = request.Headers
                .Where(h => HeaderWhitelist.Contains(h.Key))
                .ToDictionary(h => h.Key, h => h.Value.ToString());

            var result = Log
                .ForContext("RequestHeaders", loggedHeaders, destructureObjects: true)
                .ForContext("RequestHost", request.Host)
                .ForContext("RequestProtocol", request.Protocol);

            return result;
        }

        static double GetElapsedMilliseconds(long start, long stop)
        {
            return (stop - start) * 1000 / (double)Stopwatch.Frequency;
        }
        
        static string GetPath(HttpContext httpContext)
        {
            return httpContext.Features.Get<IHttpRequestFeature>()?.RawTarget ?? httpContext.Request.Path.ToString();
        }
    }
}
```

## Custom RequestResponseLoggingMiddleware Using ILogger
Courtesy: [this](https://gist.github.com/elanderson/c50b2107de8ee2ed856353dfed9168a2)   
`RequestResponseLoggingMiddleware.cs`
```cs
public class RequestResponseLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger _logger;

    public RequestResponseLoggingMiddleware(RequestDelegate next,
                                            ILoggerFactory loggerFactory)
    {
        _next = next;
        _logger = loggerFactory
                  .CreateLogger<RequestResponseLoggingMiddleware>();
    }
    
    public async Task Invoke(HttpContext context)
    {  
        _logger.LogInformation(await FormatRequest(context.Request));

        var originalBodyStream = context.Response.Body;

        using (var responseBody = new MemoryStream())
        {
            context.Response.Body = responseBody;

            await _next(context);

            _logger.LogInformation(await FormatResponse(context.Response));
            await responseBody.CopyToAsync(originalBodyStream);
        }
    }
    
    private async Task<string> FormatRequest(HttpRequest request)
    {
        var body = request.Body;
        request.EnableRewind();

        var buffer = new byte[Convert.ToInt32(request.ContentLength)];
        await request.Body.ReadAsync(buffer, 0, buffer.Length);
        var bodyAsText = Encoding.UTF8.GetString(buffer);
        request.Body = body;

       return $"{request.Scheme} {request.Host}{request.Path} {request.QueryString} {bodyAsText}";
    }

    private async Task<string> FormatResponse(HttpResponse response)
    {
        response.Body.Seek(0, SeekOrigin.Begin);
        var text = await new StreamReader(response.Body).ReadToEndAsync(); 
        response.Body.Seek(0, SeekOrigin.Begin);

        return $"Response {text}";
    }
}
```

`RequestResponseLoggingMiddlewareExtensions.cs`
```cs
public static class RequestResponseLoggingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestResponseLogging(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<RequestResponseLoggingMiddleware>();
    }
}
```

`Startup.cs`
```cs
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
  // Middleware order matters => add before other Middleware
  app.Use Middleware<ApiLoggingMiddleware>();
  
  // ... ... ...
}
```

## ApiLoggingMiddleware
* Courtesy: [this](https://salslab.com/a/safely-logging-api-requests-and-responses-in-asp-net-core)
* Dependency
   * `ApiLogService.cs`
   * `ApiLogItem.cs`
   
`ApiLogService.cs`
```cs
public class ApiLogService
{
   // ...

  public async Task Log(ApiLogItem apiLogItem)
  {
      // ...
  }
}
```

`ApiLogItem.cs`
```cs
public class ApiLogItem
{
  public long Id { get; set; }

  [Required]
  public DateTime RequestTime { get; set; }

  [Required]
  public long ResponseMillis { get; set; }

  [Required]
  public int StatusCode { get; set; }

  [Required]
  public string Method { get; set; }

  [Required]
  public string Path { get; set; }

  public string QueryString { get; set; }

  public string RequestBody { get; set; }

  public string ResponseBody { get; set; }

}
```


`ApiLoggingMiddleware.cs`
```cs
using AspNetCoreApiLoggingSample.Data;
using AspNetCoreApiLoggingSample.Services;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Http.Internal;
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace AspNetCoreApiLoggingSample.Middleware
{
    public class ApiLoggingMiddleware
    {
        private readonly RequestDelegate _next;
        private ApiLogService _apiLogService;

        public ApiLoggingMiddleware(RequestDelegate next)
        {
            _next = next;
        }

        public async Task Invoke(HttpContext httpContext, ApiLogService apiLogService)
        {
            try
            {
                _apiLogService = apiLogService;

                var request = httpContext.Request;
                if (request.Path.StartsWithSegments(new PathString("/api")))
                {
                    var stopWatch = Stopwatch.StartNew();
                    var requestTime = DateTime.UtcNow;
                    var requestBodyContent = await ReadRequestBody(request);
                    var originalBodyStream = httpContext.Response.Body;
                    using (var responseBody = new MemoryStream())
                    {
                        var response = httpContext.Response;
                        response.Body = responseBody;
                        await _next(httpContext);
                        stopWatch.Stop();

                        string responseBodyContent = null;
                        responseBodyContent = await ReadResponseBody(response);
                        await responseBody.CopyToAsync(originalBodyStream);

                        await SafeLog(requestTime,
                            stopWatch.ElapsedMilliseconds,
                            response.StatusCode,
                            request.Method,
                            request.Path,
                            request.QueryString.ToString(),
                            requestBodyContent,
                            responseBodyContent);
                    }
                }
                else
                {
                    await _next(httpContext);
                }
            }
            catch (Exception ex)
            {
                await _next(httpContext);
            }
        }

        private async Task<string> ReadRequestBody(HttpRequest request)
        {
            request.EnableRewind();

            var buffer = new byte[Convert.ToInt32(request.ContentLength)];
            await request.Body.ReadAsync(buffer, 0, buffer.Length);
            var bodyAsText = Encoding.UTF8.GetString(buffer);
            request.Body.Seek(0, SeekOrigin.Begin);

            return bodyAsText;
        }

        private async Task<string> ReadResponseBody(HttpResponse response)
        {
            response.Body.Seek(0, SeekOrigin.Begin);
            var bodyAsText = await new StreamReader(response.Body).ReadToEndAsync();
            response.Body.Seek(0, SeekOrigin.Begin);

            return bodyAsText;
        }

        private async Task SafeLog(DateTime requestTime,
                            long responseMillis,
                            int statusCode,
                            string method,
                            string path,
                            string queryString,
                            string requestBody,
                            string responseBody)
        {
            if (path.ToLower().StartsWith("/api/login"))
            {
                requestBody = "(Request logging disabled for /api/login)";
                responseBody = "(Response logging disabled for /api/login)";
            }

            if (requestBody.Length > 100)
            {
                requestBody = $"(Truncated to 100 chars) {requestBody.Substring(0, 100)}";
            }

            if (responseBody.Length > 100)
            {
                responseBody = $"(Truncated to 100 chars) {responseBody.Substring(0, 100)}";
            }

            if (queryString.Length > 100)
            {
                queryString = $"(Truncated to 100 chars) {queryString.Substring(0, 100)}";
            }

            await _apiLogService.Log(new ApiLogItem
            {
                RequestTime = requestTime,
                ResponseMillis = responseMillis,
                StatusCode = statusCode,
                Method = method,
                Path = path,
                QueryString = queryString,
                RequestBody = requestBody,
                ResponseBody = responseBody
            });
        }
    }
}
```

`Startup.cs`
```cs
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
  // Middleware order matters => add before other Middleware
  app.UseMiddleware<ApiLoggingMiddleware>();
  
  // ... ... ...
}
```
