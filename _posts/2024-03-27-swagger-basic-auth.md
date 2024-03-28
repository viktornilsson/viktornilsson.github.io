---
layout: post
title: "Secure your Swagger docs with Basic Auth in .NET"
categories: dotnet
---

## Introduction
In this tutorial, we will guide you through securing your Swagger documentation using basic authentication in .NET. 
By implementing a custom middleware, you can restrict access to your Swagger docs based on a username and password.

## Prerequisites
- Basic knowledge of C# and .NET development.
- A .NET API Project with Swagger.

## Steps

### 1. Create a custom middleware

Create a class with the following code. Double check that you use te same path for swagger.
Add your own username and password. You could also use environment variables for this, instead of static ones.

```cs
using Microsoft.AspNetCore.Http;
using System.Net;
using System.Text;
using System.Threading.Tasks;
using System;

namespace DotNetSwaggerBasicAuth
{
    public class SwaggerBasicAuthMiddleware(RequestDelegate next)
    {
        private readonly RequestDelegate next = next;

        public async Task InvokeAsync(HttpContext context)
        {
            if (context.Request.Path.StartsWithSegments("/swagger"))
            {
                string authHeader = context.Request.Headers.Authorization;

                if (authHeader != null && authHeader.StartsWith("Basic "))
                {
                    var encodedUsernamePassword = authHeader.Split(' ', 2, StringSplitOptions.RemoveEmptyEntries)[1]?.Trim();

                    var decodedUsernamePassword = Encoding.UTF8.GetString(Convert.FromBase64String(encodedUsernamePassword));

                    var username = decodedUsernamePassword.Split(':', 2)[0];
                    var password = decodedUsernamePassword.Split(':', 2)[1];

                    if (IsAuthorized(username, password))
                    {
                        await next.Invoke(context);
                        return;
                    }
                }

                context.Response.Headers.WWWAuthenticate = "Basic";
                context.Response.StatusCode = (int)HttpStatusCode.Unauthorized;
            }
            else
            {
                await next.Invoke(context);
            }
        }

        public bool IsAuthorized(string username, string password)
        {
            return username.Equals("[[username]]", StringComparison.InvariantCultureIgnoreCase) && password.Equals("[[password]]");
        }
    }
}
```


### 2. Register your custom middleware

A bit depending on how you have configured your `Program.cs` file. 
But it should be registered like this, before `app.UseSwagger();`.

```cs
app.UseMiddleware<SwaggerBasicAuthMiddleware>();

app.UseSwagger();
app.UseSwaggerUI(c =>
{
    c.SwaggerEndpoint("/swagger/v1/swagger.json", "Api");
});
```

### 3. Conclusion
By following this tutorial, you have successfully secured your Swagger documentation behind basic authentication in C#. This extra layer of security helps protect your API documentation from unauthorized access. Feel free to customize the authentication logic to suit your specific requirements.
