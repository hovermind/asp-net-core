TOC
* [Custom RequestResponseLoggingMiddleware Using ILogger](https://gist.github.com/elanderson/c50b2107de8ee2ed856353dfed9168a2)
* [Custom Request Response Logging Middleware by datalust (developer of Seq)](https://github.com/datalust/serilog-middleware-example/blob/master/src/Datalust.SerilogMiddlewareExample/Diagnostics/SerilogMiddleware.cs)
* [Routes ASP.NET Core log messages through Serilog](https://github.com/serilog/serilog-aspnetcore)

## Custom RequestResponseLoggingMiddleware Using ILogger
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

