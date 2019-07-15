---
layout: post
title: Using the VSTS REST API with ASP.NET Core
category: coding
description: Learn how to use the VSTS REST API with ASP.NET Core
tags: [coding, c#, .net, asp.net]
---

Over the last 12 months I've been maintaining an in-house Xamarin app builder wizard that I wrote in ASP.NET Core which helps streamline the building and deploying of Android and iOS applications. I've written a few blog posts in the past about this project, but never explained how the VSTS authentication is carried out - I intend on doing so in this post.

Authentication with VSTS involves linking a web project with VSTS and subsequently requesting a Bearer token which will be used when communicating with the REST API.

### The code and UI

To begin, we will add a new controller to your web application called `OAuthController`:

```
public class OAuthController : Controller
{
    private readonly IStorageConfig _storage;
    private readonly IHttpContextAccessor _context;
    
    public OAuthController(IStorageConfig storage, IHttpContextAccessor context)
    {
        _storage = storage;
        _context = context;
    }
    
    public IActionResult LinkVstsPage()
    {
        var token = _context.HttpContext.Session.Get<TokenModel>(Constants.TokenSessionKey);
        ViewBag.Token = token;
        
        return View("YourView", token != null);
    }
    
    public IActionResult RequestToken()
    {
        return new RedirectResult(this.GenerateAuthorizeUrl());
    }
    
    public IActionResult RefreshToken()
    {
        var token = new TokenModel();
        string error = null;

        var sessionToken = _context.HttpContext.Session.Get<TokenModel>(Constants.TokenSessionKey);

        if (sessionToken != null)
        {
            error = PerformTokenRequest(this.GenerateRefreshPostData(sessionToken.RefreshToken), out sessionToken);
            if (string.IsNullOrEmpty(error))
            {
                token.Expiration = DateTime.Now.AddSeconds(int.Parse(sessionToken.ExpiresIn));
                _context.HttpContext.Session.Set(Constants.TokenSessionKey, sessionToken);
            }
        }
        
        TempData["OAuthError"] = error;
        return View("YourView", sessionToken != null);
    }
    
    public IActionResult Callback(string code, string state)
    {
        var token = new TokenModel();
        string error = null;

        if (!string.IsNullOrEmpty(code))
        {
            error = PerformTokenRequest(this.GenerateRequestPostData(code), out token);
            if (string.IsNullOrEmpty(error))
            {
                token.Expiration = DateTime.Now.AddSeconds(int.Parse(token.ExpiresIn));
                _context.HttpContext.Session.Set(Constants.TokenSessionKey, token);
            }
        }

        TempData["OAuthError"] = error;
        return View("YourView", true);
    }
    
    public string GenerateAuthorizeUrl()
    {
        var uriBuilder = new UriBuilder(_storage.OAuthUrl);
        var queryDictionary = QueryHelpers.ParseQuery(uriBuilder.Query);
        
        var items = queryDictionary.SelectMany(x => x.Value, (col, value) => new KeyValuePair<string, string>(col.Key, value)).ToList();
        var builder = new QueryBuilder(items)
        {
            { "client_id", _storage.ClientAppId },
            { "response_type", "Assertion" },
            { "state", "state" },
            { "scope", _storage.OAuthScope },
            { "redirect_uri", _storage.OAuthCallbackUrl }
        };

        return _storage.OAuthUrl + builder.ToQueryString().ToString().Replace("%2B", "%20");
    }
    
    private string GenerateRequestPostData(string code)
    {
        return string.Format("client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer&client_assertion={0}&grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion={1}&redirect_uri={2}",
            WebUtility.UrlEncode(_storage.ClientAppSecret),
            WebUtility.UrlEncode(code),
            _storage.OAuthCallbackUrl
        );
    }

    private string GenerateRefreshPostData(string refreshToken)
    {
        return string.Format("client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer&client_assertion={0}&grant_type=refresh_token&assertion={1}&redirect_uri={2}",
            WebUtility.UrlEncode(_storage.ClientAppSecret),
            WebUtility.UrlEncode(refreshToken),
            _storage.OAuthCallbackUrl
        );
    }
    
    private string PerformTokenRequest(string postData, out TokenModel token)
    {
        var error = string.Empty;
        var strResponseData = string.Empty;

        var webRequest = (HttpWebRequest)WebRequest.Create(
            _storage.OAuthTokenUrl
        );

        webRequest.Method = "POST";
        webRequest.ContentLength = postData.Length;
        webRequest.ContentType = "application/x-www-form-urlencoded";

        using (var swRequestWriter = new StreamWriter(webRequest.GetRequestStream()))
        {
            swRequestWriter.Write(postData);
        }

        try
        {
            var hwrWebResponse = (HttpWebResponse)webRequest.GetResponse();

            if (hwrWebResponse.StatusCode == HttpStatusCode.OK)
            {
                using (var srResponseReader = new StreamReader(hwrWebResponse.GetResponseStream()))
                {
                    strResponseData = srResponseReader.ReadToEnd();
                }

                token = JsonConvert.DeserializeObject<TokenModel>(strResponseData);
                return null;
            }
        }
        catch (WebException wex)
        {
            error = "Request Issue: " + wex.Message;
        }
        catch (Exception ex)
        {
            error = "Issue: " + ex.Message;
        }

        token = new TokenModel();
        return error;
    }

}

```

**Note:** In this controller for the purposes of simplicity, we are storing the token data to the session state - in a real web application, the tokens should be stored in the database.

The supporting classes of this controller are documented below:

```
public class TokenModel
{
    [JsonProperty(PropertyName = "access_token")]
    public string AccessToken { get; set; }

    [JsonProperty(PropertyName = "token_type")]
    public string TokenType { get; set; }

    [JsonProperty(PropertyName = "expires_in")]
    public string ExpiresIn { get; set; }

    [JsonProperty(PropertyName = "refresh_token")]
    public string RefreshToken { get; set; }

    public DateTime Expiration { get; set; }
}
```

```
public static class Constants
{
    public const string TokenSessionKey = "_VstsToken";
    
    // Local
    public const string Local_AuthUrl = "Configuration:AuthUrl";
    public const string Local_CallbackUrl = "Configuration:CallbackUrl";
    public const string Local_ClientAppId = "Configuration:ClientAppId";
    public const string Local_ClientAppSecret = "Configuration:ClientAppSecret";
    public const string Local_TokenUrl = "Configuration:TokenUrl";
    public const string Local_Scope = "Configuration:Scope";
    
    // Azure - add the following APPSETTINGS properties (delete if unnecessary for your workflow)
    public const string Azure_AuthUrl = "APPSETTING_AuthUrl";
    public const string Azure_CallbackUrl = "APPSETTING_CallbackUrl";
    public const string Azure_ClientAppId = "APPSETTING_ClientAppId";
    public const string Azure_ClientAppSecret = "APPSETTING_ClientAppSecret";
    public const string Azure_TokenUrl = "APPSETTING_TokenUrl";
    public const string Azure_Scope = "APPSETTING_Scope";
}
```

```
public interface IStorageConfig
{
    string OAuthUrl { get; }
    string OAuthCallbackUrl { get; }
    string ClientAppId { get; }
    string ClientAppSecret { get; }
    string OAuthTokenUrl { get; }
    string OAuthScope { get; }
}
```

```
public class StorageConfig : IStorageConfig
{
    private readonly IConfiguration _config;

    public StorageConfig(IConfiguration config)
    {
        _config = config;
    }
    
    public string OAuthUrl
    {
        get
        {
            if (!string.IsNullOrEmpty(_config[Constants.Local_AuthUrl]))
            {
                return _config[Constants.Local_AuthUrl];
            }

            return Environment.GetEnvironmentVariable(Constants.Azure_AuthUrl);
        }
    }

    public string OAuthCallbackUrl
    {
        get
        {
            if (!string.IsNullOrEmpty(_config[Constants.Local_CallbackUrl]))
            {
                return _config[Constants.Local_CallbackUrl];
            }

            return Environment.GetEnvironmentVariable(Constants.Azure_CallbackUrl);
        }
    }

    public string ClientAppId
    {
        get
        {
            if (!string.IsNullOrEmpty(_config[Constants.Local_ClientAppId]))
            {
                return _config[Constants.Local_ClientAppId];
            }

            return Environment.GetEnvironmentVariable(Constants.Azure_ClientAppId);
        }
    }

    public string ClientAppSecret
    {
        get
        {
            if (!string.IsNullOrEmpty(_config[Constants.Local_ClientAppSecret]))
            {
                return _config[Constants.Local_ClientAppSecret];
            }

            return Environment.GetEnvironmentVariable(Constants.Azure_ClientAppSecret);
        }
    }
    
    public string OAuthTokenUrl
    {
        get
        {
            if (!string.IsNullOrEmpty(_config[Constants.Local_TokenUrl]))
            {
                return _config[Constants.Local_TokenUrl];
            }

            return Environment.GetEnvironmentVariable(Constants.Azure_TokenUrl);
        }
    }

    public string OAuthScope
    {
        get
        {
            if (!string.IsNullOrEmpty(_config[Constants.Local_Scope]))
            {
                return _config[Constants.Local_Scope];
            }

            return Environment.GetEnvironmentVariable(Constants.Azure_Scope);
        }
    }
}
```

Your `appsettings.json` file will need to include the relevant configuration properties and we will fill these in shortly:

```
{
    "Configuration": {
        "ClientAppId": "",
        "ClientAppSecret": "",
        "Scope": "",
        "CallbackUrl": "",
        "AuthUrl": "",
        "TokenUrl": ""
    }
}
```

Finally, you will need a View to support your controller and a very basic example can be found below (add a reference to the `TokenModel` object in your view):

```
@model bool
<div class="row">
    @if (ViewBag.Token != null)
    {
        <div class="col-12 mt-3">
            @if (DateTime.Now > ((TokenModel)ViewBag.Token).Expiration)
            {
                <div class="alert alert-danger">
                    <p>Your access token has expired. Please refresh it now.</p>
                </div>
            }
            else
            {
                var expires = ((TokenModel)ViewBag.Token).Expiration;
                var diffInSeconds = (int)(expires - DateTime.Now).TotalSeconds;

                <div class="alert alert-success">
                    <h5>You have a valid access token.</h5>

                    <div class="form-group row">
                        <div class="col-2">
                            <p>Expiration (seconds):</p>
                        </div>
                        <div class="col-6">
                            <input type="text" class="form-control" id="expiration" value="@diffInSeconds" readonly />
                        </div>
                    </div>
                </div>
            }
        </div>
    }

    <div class="col-12 mt-3">
        <div class="btn-group">
            <a class="btn btn-default" role="button" asp-controller="OAuth" asp-action="RequestToken">Link</a>

            @if (Model)
            {
                <a class="btn btn-default" role="button" asp-controller="OAuth" asp-action="RefreshToken">Refresh Token</a>
            }
        </div>
    </div>
</div>
```

### Link your application with VSTS

Now that we have all the relevant code in place, we now need to associate your web application with VSTS. To do this, head over to VSTS and log in. From there, access your profile and find the **Applications and services** header which underneath will have a link to **"Create new application"**. At this stage you should have been redirected to another page which features a form to fill out - below are a few pointers to what you should enter:

1) **Application website** - The base url to your hosted web application
1) **Authorization callback URL** - The base url followed by `/OAuth/Callback` (if you called your controller or endpoint something different, replace as appropriate).
1) **Authorized scopes** - Select the following: Build (read and execute), Release (read, write and execute). You are free to add additional scopes if your application requires them, these are just the ones I needed.

Once complete, you should be presented with a summary of the information that you need to add into your `appconfiguration.json` file. The authorized scopes should be `vso.build_execute vso.release_execute`.

### Send requests to VSTS

At this stage, your web application is ready to start sending requests to the VSTS REST API. We now need to add the code required to send these requests. Add the following which will act as a service to communicate with VSTS:

```
public interface ISourceCodeService
{
    Task<ProjectResponse> GetProjects();
    Task<BuildDefinitionList> GetBuildDefinitions();
    Task<AppBuildResponse> PostBuildRequest(BuildMobileAppRequest request);
    Task<ArtifactResponse> GetBuildArtifacts(string buildDef);
}
```

```
public class SourceCodeService : ISourceCodeService
{
    private const string VstsUri = "https://yourvsts.visualstudio.com";
    private const string CollectionUri = "https://yourvsts.visualstudio.com/DefaultCollection";
    private const string ProjectName = "YOUR PROJECT NAME";
    private const string RepoName = "YOUR REPO NAME";
    private const string BranchName = "YOUR BRANCH NAME";
    private readonly IHttpContextAccessor _context;
    private HttpClient _client;

    public SourceCodeService(IHttpContextAccessor context)
    {
        _context = context;
        
        // Singleton, make sure DI is setup correctly.
        _client = new HttpClient();
    }

    public async Task<ProjectResponse> GetProjects()
    {
        return await GetRequest<ProjectResponse>($"{CollectionUri}/_apis/projects");
    }

    public async Task<BuildDefinitionList> GetBuildDefinitions()
    {
        return await GetRequest<BuildDefinitionList>($"{VstsUri}/{ProjectName}/_apis/build/definitions?api-version=4.1");
    }

    public async Task<ArtifactResponse> GetBuildArtifacts(string buildDef)
    {
        return await GetRequest<ArtifactResponse>($"{VstsUri}/{ProjectName}/_apis/build/builds/{buildDef}/artifacts?api-version=4.1");
    }

    private async Task<T> GetRequest<T>(string uri)
    {
        try
        {
            var token = _context.HttpContext.Session.Get<TokenModel>(Constants.TokenSessionKey);
            var accessToken = token.AccessToken;
                            
            _client.DefaultRequestHeaders.Accept.Add(
                new MediaTypeWithQualityHeaderValue("application/json"));

            _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);

            using (HttpResponseMessage response = await _client.GetAsync(
                $"{uri}"))
            {
                response.EnsureSuccessStatusCode();
                string responseBody = await response.Content.ReadAsStringAsync();

                return JsonConvert.DeserializeObject<T>(responseBody);
            }
            
        }
        catch (Exception ex)
        {
            // Log your exception accordingly.
        }
    }

    private async Task<TOut> PostRequest<TIn, TOut>(string uri, TIn content)
    {
        try
        {
            var token = _context.HttpContext.Session.Get<TokenModel>(Constants.TokenSessionKey);
            var accessToken = token.AccessToken;
                            
            _client.DefaultRequestHeaders.Accept.Add(
                new MediaTypeWithQualityHeaderValue("application/json"));

            _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);

            var serialized = new StringContent(JsonConvert.SerializeObject(content), Encoding.UTF8, "application/json");

            using (HttpResponseMessage response = await _client.PostAsync(
                $"{uri}", serialized))
            {
                response.EnsureSuccessStatusCode();
                string responseBody = await response.Content.ReadAsStringAsync();

                return JsonConvert.DeserializeObject<TOut>(responseBody);
            }
            
        }
        catch (Exception ex)
        {
            // Log your exception accordingly.
        }
    }
}
```

The following classes will support your service:

```
public class ProjectResponse
{
    public List<VstsProject> Value { get; set; }
    public int Count { get; set; }
}
```

```

public class VstsProject
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string Url { get; set; }
    public string Description { get; set; }
    public List<VstsCollection> Collection { get; set; }
}
```

```
public class VstsCollection
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string Url { get; set; }
    public string CollectionUrl { get; set; }
}
```

```
public class ArtifactResponse
{
    public int Count { get; set; }
    public List<Artifact> Value { get; set; }
}
```

```
public class Artifact
{
    public int Id { get; set; }
    public string Name { get; set; }
    public ArtifactResource Resource { get; set; }
}
```

```
public class ArtifactResource
{
    public string Type { get; set; }
    public string Data { get; set; }
    public string Url { get; set; }
    public string DownloadUrl { get; set; }
}
```

```
public class AppBuildResponse
{
    public int Id { get; set; }
    public string BuildNumber { get; set; }
    public string Status { get; set; }
    public DateTime QueueTime { get; set; }
    public string Url { get; set; }
    public BuildDefinition Definition { get; set; }
    public int BuildNumberRevision { get; set; }
}
```

```
public class BuildDefinitionList
{
    public int Count { get; set; }
    public List<BuildDefinition> Value { get; set; }
}
```

```
public class BuildDefinition
{
    public string Name { get; set; }
    public string Url { get; set; }
    public string Uri { get; set; }
    public string Path { get; set; }
    public DateTime CreatedDate { get; set; }
    public string BuildNumber { get; set; }
    public int BuildNumberRevision { get; set; }
    public bool Deleted { get; set; }
    public string DeletedDate { get; set; }
    public string DeletedReason { get; set; }
    public string FinishTime { get; set; }
    public int Id { get; set; }
    public bool KeepForever { get; set; }
    public string Parameters { get; set; }
    public Project Project { get; set; }
    public BuildQueue Queue { get; set; }
}
```

```
public class BuildQueue
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Url { get; set; }
}
```

```
public class Project
{
    public string Abbreviation { get; set; }
    public string Description { get; set; }
    public string Id { get; set; }
    public string Name { get; set; }
    public string Revision { get; set; }
    public string State { get; set; }
    public string Url { get; set; }
    public string Visibility { get; set; }
}
```

At this stage you're ready to carry out requests against VSTS. In the `SourceCodeService` above, I've added a few methods to return Projects, Build Definitions and Build Artifacts - there are many more endpoints available which you can find in the v4.1 [API](https://docs.microsoft.com/en-us/rest/api/azure/devops/?view=azure-devops-rest-4.1).

If you've read my previous article on [Sending user defined variables when queuing builds](https://techyian.github.io/2018-07-02-vsts-build-variables/), you will see that the format in which to send the variables isn't straightforward at first glimpse. If you have trouble sending user defined variables when queuing builds, I recommend reading that first.

I hope this has been helpful. If you need to reach out to me, you can contact me on Twitter [@techyian](https://twitter.com/techyian).