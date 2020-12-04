---
title: "Using Dependency Injection in Asp.Net Core Filters & Attributes"
date: 2020-12-04T01:41:00-08:00
# publish-date: 2020-04-11T22:30:00-07:00
draft: false
categories: ["development"]
tags: ["dotnet", "asp.net"]
---

Sometimes when working with Asp.Net MVC or Web Apis, you'll want to add a Filter Attribute to a class or an endpoint. A common use case would be adding a custom Authorization filter. Usually these filters would be defined as:

```cs
public class CustomAuthorizeAttribute: AuthorizeAttribute, IAuthorizationFilter
{
    public void OnAuthorization(AuthorizationFilterContext filterContext)
    {
        // Some authorization code.
    }
}
```

and then used like this:

```cs
[HttpGet]
[CustomAuthorize]
public async IActionResult GetItem(string id)
{
    // Some implementation.
}
```

This allows us to move the authorization logic to a shared piece of code, to cut down on reuse and keep our controller endpoints clean. The big downside to this approach is that we can't use Dependency Injection. If we add a Constructor to the Attribute, we'll have to pass it a value when we define the controller. 

Let's say we wanted to pass in an ILogger so we could capture telemetry in our Authorization code. In our Controller we could inject it:

```cs
public class SuperAwesomeController : ControllerBase
{

    private readonly ILogger logger;

    public SuperAwesomeController(ILogger logger)
    {
        this.logger = logger;
    }

}
```

But in our Attribute this won't work because we would need to pass in the logger when we use the attribute as a compile-time constant, which it can never be.  So now we need to come up with a solution that allows us to inject services from the DI container. Asp.Net Core actually gives us two ways to do this, each with their own advantages and disadvantages: the `TypeFilter` and the `ServiceFilter`.

## TypeFilters

Type filters are a convenient way to create an attribute that instantiates the attribute per request and inject services from the container. The way we do this is by making two classes, one attribute that implements `TypeFilterAttribute` and one that is defined as the `ImplementationType' in the Type Filter.

```cs
public class CustomAuthorizeAttribute : TypeFilterAttribute
{
    public CustomAuthorizeAttribute()
        : base(typeof(CustomAuthorizeFilter))
    {
        
    }
}

public class CustomAuthorizeFilter : IAuthorizationFilter
{
    private readonly ILogger logger;

    public CustomAuthorizeFilter(ILogger logger)
    {
        this.logger = logger;
    }

    public void OnAuthorization(AuthorizationFilterContext filterContext)
    {
        // Some authorization code.
    }
}
```

Now when we add the `[CustomAuthorize]` attribute to our controller, we're actually adding a TypeFilter that will instantiate our `CustomAuthorizeFilter` and inject the `ILogger`. One of the reasons this is so powerful is that we can extend the capabilities here a little bit. Say we wanted to pass a role into the Authorization Filter to specify more granular access to the endpoint. With a traditional attribute we would implement it as:

```cs
public class CustomAuthorizeAttribute: AuthorizeAttribute, IAuthorizationFilter
{
    private readonly string role;

    public CustomAuthorizeAttribute(string role)
    {
        this.role = role;
    }
    ...
}
```

With the power of the `TypeFilter` we can still do that, just with a slight modification:

```cs
public class CustomAuthorizeAttribute : TypeFilterAttribute
{
    public CustomAuthorizeAttribute(string role)
        : base(typeof(CustomAuthorizeFilter))
    {
        Arguments = new object[] { role }
    }
}

public class CustomAuthorizeFilter : IAuthorizationFilter
{
    private readonly string role;
    private readonly ILogger logger;

    public CustomAuthorizeFilter(string role, ILogger logger)
    {
        this.role = role;
        this.logger = logger;
    }

    ...
}
```

### TypeFilter Benefits and Downsides.

The `TypeFilter` allows us to define our Filter in a way that gives us Dependency Injection and still allows us to pass in arguments like a normal Attribute would. The main downside here is that it has to instantiate an instance per request to our controller, adding some overhead to every request. If you don't need per-instance properties like this, it might be more beneficial to use a `ServiceFilter` instead.

## ServiceFilters

Similar to a `TypeFilter`, a `ServiceFilter` allows us to add Dependency Injection to our Filters, but it goes one step further and actually pulls the Filter from our DI container, instead of making a new instance per request. This allows us to register our filter in the DI container, giving us some more control over the lifetime of it. An example implementation might look like this:

```cs
public class CustomAuthorizeAttribute : ServiceFilterAttribute
{
    public CustomAuthorizeAttribute()
        : base(typeof(CustomAuthorizeFilter))
    {
        
    }
}

public class CustomAuthorizeFilter : IAuthorizationFilter
{
    private readonly ILogger logger;

    public CustomAuthorizeFilter(ILogger logger)
    {
        this.logger = logger;
    }

    public void OnAuthorization(AuthorizationFilterContext filterContext)
    {
        // Some authorization code.
    }
}
```

Notice how the only difference is the base type of `CustomAuthorizeFilter`. If we only did this though, we would get an exception as `CustomAuthorizeFilter` has not yet been registered in the DI container. This can easily be done by adding it to the `IServiceCollection` in `Startup`:

```cs
services.AddSingleton<CustomAuthorizeFilter>();
```

This would register it as a Singleton, meaning only one instance will ever be created. You can register it with whatever scope fits your needs.


### ServiceFilter Benefits and Downsides.

The `ServiceFilter` gives us the added advantage of using a Filter registered in the DI container, saving us the overhead of creating an instance per-request. The main downside is losing the ability to pass in arguments from the Attribute like `TypeFilter` gives us. 

## Conclusion

Both `TypeFilter` and `ServiceFilter` give us powerful tools for creating custom Attributes and Filters, along with the flexibility to choose the option that best suits our needs. Hopefully you found this post helpful and if you see any errors or issues, please reach out to me on [twitter](https://twitter.com/iamdavidfrancis). 

## References

* https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.typefilterattribute?view=aspnetcore-5.0
* https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.servicefilterattribute?view=aspnetcore-5.0
