---
layout: post
title: VSTS Rest API - Send user defined variables when queuing build
category: coding
tags: [coding, c#, .net, asp.net]
---

Whilst developing an ASP.NET Core app for building and releasing Xamarin projects via the VSTS REST API, I came across a problem when trying to pass variables to the build 
server. For reference, I was trying to call the v4.1 API, and specifically the Queue build API which can be seen [here](https://docs.microsoft.com/en-us/rest/api/vsts/build/builds/queue?view=vsts-rest-4.1).
I needed to pass some user-defined variables to the build definition, and looking at the API docs, I could see an option to pass `parameters` in the Request Body which is defined as "The parameters for the build" and accepts a string; sounds fairly straightforward
at first glimpse, but no matter what data assigned to the `parameters` argument I received a Bad Request (400) from the server. After hours of looking online, I came across this StackOverflow [post](https://stackoverflow.com/questions/34343084/start-a-build-and-passing-variables-through-vsts-rest-api)
which mentions passing a JSON formatted string - this was slightly annoying as I'd tried previously to send a JSON object defined as a POCO but had no luck.

Bingo!

This was exactly the issue I was having, and below you can see the solution I ended up with:

The simplified controller endpoint:

```
[HttpPost]
public async Task<IActionResult> BuildApp(string appRequestId, [FromBody] MobileAppRequestViewModel vm)
{	
	var parameters = new Dictionary<string, string>
	{
		{ "VARIABLE_ONE", "DATA" },
		{ "VARIABLE_TWO", "DATA" }
	};

	var json = JsonConvert.SerializeObject(parameters);

	var appRequest = new BuildMobileAppRequest
	{
		// Replace '1' with your definition id.
		Definition = new Definitition(1),
		Parameters = json,
		Project = new AppProject("YOUR PROJECT ID"),
		// Replace '1' with your queue id.
		Queue = new AppQueue(1)
	};

	// Build your app using the VSTS API
	
	return Ok();
}
```

The POCOs to serialize:

```
public class BuildMobileAppRequest
{
	public Definitition Definition { get; set; }		
	public string Parameters { get; set; }
}

public class Definitition
{
	public int Id { get; set; }

	public Definitition(int id)
	{
		this.Id = id;
	}
}
```

It would be really helpful for the Microsoft guys to add a note against their docs stating what format they require a string to be sent in as I couldn't find any official documentation
anywhere stating a JSON formatted string was required for this argument.

Hope this saves someone some time!