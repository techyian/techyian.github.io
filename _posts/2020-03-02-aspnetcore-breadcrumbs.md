---
layout: post
title: Super simple breadcrumbs in ASP.NET Core
category: coding
description: Set up simple breadcrumbs in your ASP.NET Core web application with Bootstrap 4.
tags: [coding, c#, .net, aspnetcore]
---

### Intro

When designing a new web application, breadcrumbs regrettably tend to be one of the last things I think about. Depending on where your interests lie, you can find yourself concentrating more on the back-end of your application rather than the usability but it's important to remember that usability is key to its uptake and continued use.

I was recently asked to implement breadcrumbs in a new ASP.NET Core web application and looked for a quick solution on StackOverflow. A NuGet package called SmartBreadcrumbs was mentioned quite a bit but I was hoping for a more simple solution. Unfortunately, I didn't find anything concrete on the topic so began implementing my own super simple breadcrumb system using an `ActionFilterAttribute` and the ASP.NET `ViewBag`.

### The code

The logic behind this system was to make use of the `OnActionExecuted` override and use the `HttpContext.Request.Path` value to configure the breadcrumb entries. I began by creating a new Action Filter called `BreadcrumbActionFilter` which extends the `ActionFilterAttribute` abstract class and overrides the `OnActionExecuted` method.

```csharp
public class BreadcrumbActionFilter : ActionFilterAttribute
{
    public override void OnActionExecuted(ActionExecutedContext context)
    {
        if (context.HttpContext.Request.Path.HasValue && context.HttpContext.Request.Path.Value.Contains("Api"))
        {
            // This is an API request.  
            base.OnActionExecuted(context);
            return;
        }

        var breadcrumbs = this.ConfigureBreadcrumb(context);

        var controller = context.Controller as Controller;
        controller.ViewBag.Breadcrumbs = breadcrumbs;

        base.OnActionExecuted(context);
    }
}
```

My web application features a mix of traditional MVC controller and 'WebAPI' endpoints suffixed by "ApiController". I only wanted the MVC controller endpoints to feature in my breadcrumb system so it was important to strip these out.

A typical website will feature a "Home Page" which will be used as a starting page to navigate to all other areas from. When configuring your breadcrumbs, this will be the first entry in your trail.

```csharp
private List<Breadcrumb> ConfigureBreadcrumb(ActionExecutedContext context)
{
    var breadcrumbList = new List<Breadcrumb>();
    var homeControllerName = "Home";

    breadcrumbList.Add(new Breadcrumb
    {
        Text = "Home",
        Action = "Index",
        Controller = homeControllerName, // Change this controller name to match your Home Controller.
        Active = true
    });

    if (context.HttpContext.Request.Path.HasValue)
    {
        var pathSplit = context.HttpContext.Request.Path.Value.Split("/");

        for (var i = 0; i < pathSplit.Length; i++)
        {
            // Check if first element is equal to our Index (home) page.
            if (string.IsNullOrEmpty(pathSplit[i]) || string.Compare(pathSplit[i], homeControllerName, true) == 0)
            {
                continue;
            }

            // First check if path is a Controller class.
            var controller = this.GetControllerType(pathSplit[i] + "Controller");

            // If this is a controller, does it have a default Index method? If so, that needs adding as a breadcrumb. Is the next path element called Index?
            if (controller != null)
            {
                var indexMethod = controller.GetMethod("Index");

                if (indexMethod != null)
                {
                    breadcrumbList.Add(new Breadcrumb
                    {
                        Text = this.CamelCaseSpacing(pathSplit[i]),
                        Action = "Index",
                        Controller = pathSplit[i],
                        Active = true
                    });

                    if (i + 1 < pathSplit.Length && string.Compare(pathSplit[i + 1], "Index", true) == 0)
                    {
                        // This is the last element in the breadcrumb list. This should be disabled.
                        breadcrumbList.LastOrDefault().Active = false;

                        // Next path item is the Index method. We can escape from this method now.
                        return breadcrumbList;
                    }
                }
            }

            // If not a Controller, check if this is a method on the previous path element.
            if (i - 1 > 0)
            {
                var controllerName = pathSplit[i - 1] + "Controller";
                var prevController = this.GetControllerType(controllerName);

                if (prevController != null)
                {
                    var method = prevController.GetMethod(pathSplit[i]);

                    if (method != null)
                    {
                        // We've found an endpoint on the previous controller.
                        breadcrumbList.Add(new Breadcrumb
                        {
                            Text = this.CamelCaseSpacing(pathSplit[i]),
                            Action = pathSplit[i],
                            Controller = pathSplit[i - 1]
                        });
                    }
                }
            }
        }
    }

    // There will always be at least 1 entry in the breadcrumb list. The last element should always be disabled.
    breadcrumbList.LastOrDefault().Active = false;

    return breadcrumbList;
}
```

Above is the main method which will be used to configure your breadcrumb trail. You can see here that we're first adding an initial entry to our list of breadcrumbs which is the "Home Page". 

The next stage of the process is to determine if the request's path has a value, and if so, we want to extract each part of that path so we can build up our trail. On each iteration of the path members, we want to determine if that member is the name of a MVC Controller class and it has a default Index endpoint (a controller with an Index endpoint can be accessed by using the controller's name in a URL without explicitly having to reference the Index endpoint).

If the path member we're processing isn't the name of a controller class, we want to determine if it is the name of an endpoint on the previous path member - if it is, it gets added to the breadcrumb trail.

There are a few helper methods which are used during this process which assist in checking for a type via reflection and also adding spacing in camel cased strings.

```csharp
private Type GetControllerType(string name)
{
    Type controller = null;

    try
    {
        controller = Assembly.GetCallingAssembly().GetType("WebApp.Web.Controllers." + name);
    }
    catch
    { }

    return controller;
}

private string CamelCaseSpacing(string s)
{
    // Sourced from https://stackoverflow.com/questions/4488969/split-a-string-by-capital-letters.
    var r = new Regex(@"
        (?<=[A-Z])(?=[A-Z][a-z]) |
         (?<=[^A-Z])(?=[A-Z]) |
         (?<=[A-Za-z])(?=[^A-Za-z])", RegexOptions.IgnorePatternWhitespace);

    return r.Replace(s, " ");
}
```

You will also want to create the `Breadcrumb` class:

```
public class Breadcrumb
{
    public string Text { get; set; }
    public string Action { get; set; }
    public string Controller { get; set; }
    public bool Active { get; set; }

    public Breadcrumb() { }

    public Breadcrumb(string text, string action, string controller, bool active)
    {
        this.Text = text;
        this.Action = action;
        this.Controller = controller;
        this.Active = active;
    }
}
```

This solution is based around using an `ActionFilterAttribute` and we need to instruct ASP.NET Core to now use our filter. In the web application I was working on, all controllers extended a common `BaseController` class so it was relatively trivial to add the attribute to the base class:

```csharp
[BreadcrumbActionFilter]
public class BaseController : Controller
```

If you don't have a common 'BaseController' class, you will need to add the attribute to each controller you want to participate in showing the breadcrumb trail.

The last stage of the process is to implement the client side changes necessary to actually see your breadcrumbs in action. The web application I was working on used Bootstrap 4 but you should easily be able to convert this into whatever framework you're using. The following code should ideally be placed in your `_Layout.cshtml` view if you want breadcrumbs to show on every page.

```
@if (ViewBag.Breadcrumbs != null)
{
    <nav aria-label="breadcrumb">
        <ol class="breadcrumb bg-white justify-content-center justify-content-md-start justify-content-lg-start justify-content-xl-start">
            @foreach (var breadcrumb in ViewBag.Breadcrumbs as List<Breadcrumb>)
            {
                if (breadcrumb.Active)
                {
                    <li class="breadcrumb-item active"><a asp-action="@breadcrumb.Action" asp-controller="@breadcrumb.Controller">@breadcrumb.Text</a></li>
                }
                else
                {
                    <li class="breadcrumb-item" aria-current="page">@breadcrumb.Text</li>
                }
            }
        </ol>
    </nav>
}
```

### GitHub Repository

I've uploaded an example ASP.NET Core web application to [GitHub](https://github.com/techyian/SuperSimpleBreadcrumbs) which shows how the breadcrumbs function in practice; there is also an example of how you can override the breadcrumbs in a Razor view.

### Conclusion

In this post I have shown how to configure a basic breadcrumb system using ASP.NET Core. The system heavily relies on reflection to determine whether members of a request's path are names of controller classes or endpoints within a class. It is important to remember that this system will only produce a maximum breadcrumb trail of 3 entries so it won't be suitable for complex web applications which require a larger trail (solutions such as SmartBreadcrumbs may be able to help you there or perhaps expanding on the solution here) - this was however perfectly adequate for the web app I was working on.

I hope this post helps you with implementing breadcrumbs in your web app!