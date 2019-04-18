TOC
* [API -> Custom Exception Handling Middleware](#custom-middleware-for-api)
* [API -> UseExceptionHandler](#useexceptionhandler-for-api)
* [WebApp -> UseExceptionHandler](#useExceptionHandler-for-webApp)
* [WebApp -> UseStatusCodePagesWithReExecute](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/error-handling?view=aspnetcore-2.2#usestatuscodepageswithreexecute)

## UseExceptionHandler for API
`Startup..cs`
```cs
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment()) {
    
        app.UseDeveloperExceptionPage();
        
    } else {
    
        app.UseExceptionHandler(a => a.Run(async context =>
        {
            var feature = context.Features.Get<IExceptionHandlerPathFeature>();
            var exception = feature.Error;

            var result = JsonConvert.SerializeObject(new { error = exception.Message });
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsync(result);
        }));
    }
    
    // More config...
}
```
**Important:**  `UseExceptionHandler()` must be before `UseMvc()` 

## Custom Middleware for API
1. Error Details Model
`ErrorDetails.cs`
```cs
public class ErrorDetails
{
    public int StatusCode { get; set; }
    public string Message { get; set; }

    public override string ToString()
    {
        return JsonConvert.SerializeObject(this);
    }
}
```

2. Custom Exception Class
`HttpStatusCodeException.cs`
```cs
public class HttpStatusCodeException : Exception
{
    public HttpStatusCode StatusCode { get; set; }
    public string ContentType { get; set; } = @"text/plain";

    public HttpStatusCodeException(HttpStatusCode statusCode)
    {
        this.StatusCode = statusCode;
    }

    public HttpStatusCodeException(HttpStatusCode statusCode, string message) : base(message)
    {
        this.StatusCode = statusCode;
    }

    public HttpStatusCodeException(HttpStatusCode statusCode, Exception inner) : this(statusCode, inner.ToString()) { }

    public HttpStatusCodeException(HttpStatusCode statusCode, JObject errorObject) : this(statusCode, errorObject.ToString())
    {
        this.ContentType = @"application/json";
    }
}
```

3. Custom Exception Middleware
`CustomExceptionMiddleware.cs`
```cs
public class CustomExceptionMiddleware
    {
        private readonly RequestDelegate next;

    public CustomExceptionMiddleware(RequestDelegate next)
    {
        this.next = next;
    }

    public async Task Invoke(HttpContext context /* other dependencies */)
    {
        try
        {
            await next(context);
        }
        catch (HttpStatusCodeException ex)
        {
            await HandleExceptionAsync(context, ex);
        }
        catch (Exception exceptionObj)
        {
            await HandleExceptionAsync(context, exceptionObj);
        }
    }

    private Task HandleExceptionAsync(HttpContext context, HttpStatusCodeException exception)
    {
        string result = null;
        context.Response.ContentType = "application/json";
        if (exception is HttpStatusCodeException)
        {
            result = new ErrorDetails() { Message = exception.Message, StatusCode = (int)exception.StatusCode }.ToString();
            context.Response.StatusCode = (int)exception.StatusCode;
        }
        else
        {
            result = new ErrorDetails() { Message = "Runtime Error", StatusCode = (int)HttpStatusCode.BadRequest }.ToString();
            context.Response.StatusCode = (int)HttpStatusCode.BadRequest;
        }
        return context.Response.WriteAsync(result);
    }

    private Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        string result = new ErrorDetails() { Message = exception.Message, StatusCode = (int)HttpStatusCode.InternalServerError }.ToString();
        context.Response.StatusCode = (int)HttpStatusCode.BadRequest;
        return context.Response.WriteAsync(result);
    }
}
```

4. Middleware Extensions
`HttpStatusCodeExceptionMiddlewareExtensions.cs`
```cs
public static class HttpStatusCodeExceptionMiddlewareExtensions
{
  public static void ConfigureCustomExceptionMiddleware(this IApplicationBuilder app)
  {
    app.UseMiddleware<CustomExceptionMiddleware>();
  }
}
```

5. Configure in `Startup.cs`
```cs
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment()) {
    
        app.UseDeveloperExceptionPage();
        app.ConfigureCustomExceptionMiddleware();
        
    } else {
    
        app.ConfigureCustomExceptionMiddleware();
        app.UseExceptionHandler();
    }
    
    // More config...
}
```
**Courtesy:** https://blogs.msdn.microsoft.com/brandonh/2017/07/31/using-middleware-to-trap-exceptions-in-asp-net-core/

## UseExceptionHandler for WebApp
`Startup..cs`
```cs
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment()) {
    
         app.UseDeveloperExceptionPage();
         
    } else {
    
        app.UseStatusCodePagesWithReExecute("/Error");
        app.UseExceptionHandler("/Error");
        //app.UseHsts();
    }
    
    // More config...
}
```
