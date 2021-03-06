---
layout: post
title: Generic HttpClient GET and POST requests
category: coding
description: Find out how to send generic HttpClient GET and POST requests using Newtonsoft.Json
tags: [coding, c#, .net, asp.net]
---

Updated: March 2019:

*I have been doing some reading on the use of HttpClient and it appears to be a common mistake to wrap it in a using block. Although HttpClient implements IDisposable, and many developers would assume it be correct to automatically dispose once the HTTP request has been made (myself included), this does in fact introduce a performance issue if the same HttpClient reference is being called frequently. Please use a singleton pattern for your HttpClient references and only dispose when you're app doesn't require it anymore.*

I've been recently working on a project at work to set up Xamarin build automation for our Android and iOS apps via VSTS. I decided to use ASP.NET Core as the framework
to use and at the time of writing, the .NET client library to interact with VSTS, namely `Microsoft.TeamFoundationServer.Client` does not appear to support .NET Standard 2.0 correctly.
With this hurdle to jump over, I turned to the VSTS REST API and have had success with it so far. Whilst reading the official documentation, MS provided an example
method to get a list of projects for a given MSDN account, and I noticed that I could improve it with generics. You can make use of the methods if your HTTP response returns a JSON string and can be mapped to a POCO you've made.

**GET Request**

For reference, here is the original method:

```csharp
public static async void GetProjects()
{
    try
    {
        var personalaccesstoken = "PAT_FROM_WEBSITE";

        using (HttpClient client = new HttpClient())
        {
            client.DefaultRequestHeaders.Accept.Add(
                new System.Net.Http.Headers.MediaTypeWithQualityHeaderValue("application/json"));

            client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Basic",
                Convert.ToBase64String(
                    System.Text.ASCIIEncoding.ASCII.GetBytes(
                        string.Format("{0}:{1}", "", personalaccesstoken))));

            using (HttpResponseMessage response = await client.GetAsync(
                        "https://{account}.visualstudio.com/DefaultCollection/_apis/projects"))
            {
                response.EnsureSuccessStatusCode();
                string responseBody = await response.Content.ReadAsStringAsync();
                Console.WriteLine(responseBody);
            }
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.ToString());
    }
}
```

And the improved generic version:

```csharp
private async Task<T> GetRequest<T>(string uri)
{
    try
    {
    	using (var client = new HttpClient())
    	{
    	    client.DefaultRequestHeaders.Accept.Add(
    		new MediaTypeWithQualityHeaderValue("application/json"));
    
    	    using (HttpResponseMessage response = await client.GetAsync(uri))
    	    {
    		response.EnsureSuccessStatusCode();
    		string responseBody = await response.Content.ReadAsStringAsync();
    
    		return JsonConvert.DeserializeObject<T>(responseBody);
    	    }
    	}
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.ToString());
    }
}
```

As you can see here, we're making use of the generic `DeserializeObject` method available in the Newtonsoft.Json package. I've removed all the stuff relating to tokens etc. so
this is more of a general purpose method for GET requests via HttpClient.


**POST Request**

For completeness, here is a generic POST request using HttpClient:

```csharp
private async Task<TOut> PostRequest<TIn, TOut>(string uri, TIn content)
{
    try
    {
    	using (var client = new HttpClient())
    	{
            client.DefaultRequestHeaders.Accept.Add(
    		new MediaTypeWithQualityHeaderValue("application/json"));
    
    	    var serialized = new StringContent(JsonConvert.SerializeObject(content), Encoding.UTF8, "application/json");
    
    	    using (HttpResponseMessage response = await client.PostAsync(uri, serialized))
    	    {
    		response.EnsureSuccessStatusCode();
    		string responseBody = await response.Content.ReadAsStringAsync();
    
    		return JsonConvert.DeserializeObject<TOut>(responseBody);
    	    }
    	}
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.ToString());
    }
}
```

This method will accept a generic object and will deserialize the response to a generic type also.
