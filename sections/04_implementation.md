\newpage

# Creating a Trust Context for the Authentication Mesh {#sec:implementation}

This section gives an overview of the used demo applications, the programming language Rust, and several security topics that are relevant for the implementation of the authentication mesh. Furthermore, the implementation of the shared trust context is described.

## Demo Applications

To demonstrate and test the implementation of the trust context, multiple demo applications are used. All applications are hosted on GitHub in the open source repository https://github.com/WirePact/demo-applications. There exist six different applications that are described below.

The **basic_auth_api** is a simple API application written in Go^[<https://go.dev/>]. It uses HTTP Basic Authentication (RFC7617) to authenticate calls against its endpoints. The API can be configured with three different environment variables (`PORT`, `AUTH_USERNAME`, and `AUTH_PASSWORD`). A HTTP web framework package "[Gin](https://github.com/gin-gonic/gin)" provides the HTTP middleware for Go.

```go
router := gin.Default()
secure := router.Group("/", gin.BasicAuth(gin.Accounts{
	config.Username: config.Password,
}))
secure.GET("swapi/people", getPeopleFromSwapi)
router.OPTIONS("/swapi/people", cors)
```

The static website **basic_auth_app** provides a trivial way of accessing any basic protected API. The site runs within an NGINX and contains minimal code. Since this site is hosted statically and does not call API endpoints through some backend logic, it is not possible to adhere to the `HTTP_PROXY` environment variable to route traffic through a specific proxy.

In contrast to the basic auth app, the **basic_auth_backend_app** is an `ASP.NET` application that also uses the HTTP Basic mechanism to authenticate requests. The application runs in an ASP.NET context. Thus, it is possible to respect the `HTTP_PROXY` variable and route traffic through a specific proxy.

To provide a more complex authentication scheme, the **oidc_api** authenticates requests against its API via OAuth2.0. When the API receives an access token from a client, it uses token introspection (defined by **RFC7662**) to validate the token and authenticate the user [@RFC7662]. The API needs an issuer, a client id, and a client secret to validate the given tokens. The configuration of the C\# application is done as follows:

```csharp
builder.Services
    .AddAuthentication("token")
    .AddOAuth2Introspection("token", o =>
    {
        var section = builder.Configuration.GetSection("Oidc");
        o.Authority = section.GetValue<string>("Issuer");
        o.ClientId = section.GetValue<string>("ClientId");
        o.ClientSecret = section.GetValue<string>("ClientSecret");
        o.DiscoveryPolicy = new()
        {
            RequireHttps = false,
            ValidateEndpoints = false,
            ValidateIssuerName = false,
            RequireKeySet = false,
        };
    });
```

To complement the oidc API, an **oidc_app** provides the means to access an OIDC (OAuth) protected API via an application. This [Next.js](https://nextjs.org/) application authenticates users against the OIDC provider and then renders a simple page. Since this is a hosted application, the `HTTP_PROXY` is respected.

The final demo application is the **oidc_provider**. It is based on a Node.js package that provides oidc server capabilities. This identity provider allows any user with any password and thus is not suitable for production environments. The provider supports OAuth 2.0 Token Exchange (**RFC8693**) to enable the proxy applications to fetch an access token for a specific user [@RFC8693].

## The Rust Programming Language

## Sign and Distribute Contracts between Participants

### Using a Block Chain

#### Introduction

### Using a Master Key

### Distribute Contracts via Git

## Define the Contract
