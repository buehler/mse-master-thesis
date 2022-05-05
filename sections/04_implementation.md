\newpage

# Creating a Trust Context for the Authentication Mesh {#sec:implementation}

This section gives an overview of the used demo applications, the programming language Rust, and several security topics that are relevant for the implementation of the authentication mesh. Furthermore, the implementation of the shared trust context is described.

## Demo Applications

To demonstrate and test the implementation of the trust context, multiple demo applications are used. All applications are hosted on GitHub in the open source repository https://github.com/WirePact/demo-applications. There exist six different applications that are described below.

The **basic_auth_api** is a simple API application written in Go^[<https://go.dev/>]. It uses HTTP Basic Authentication (RFC7617) to authenticate calls against its endpoints. The API can be configured with three different environment variables (`PORT`, `AUTH_USERNAME`, and `AUTH_PASSWORD`). An HTTP web framework package "[Gin](https://github.com/gin-gonic/gin)" provides the HTTP middleware for Go.

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

To provide a more complex authentication scheme, the **oidc_api** authenticates requests against its API via OAuth2.0. When the API receives an access token from a client, it uses token introspection (defined by **RFC7662**) to validate the token and authenticate the user [@RFC7662]. The API needs an issuer, a client ID, and a client secret to validate the given tokens. The configuration of the C\# application is done as follows:

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

To complement the OIDC API, an **oidc_app** provides the means to access an OIDC (OAuth) protected API via an application. This [Next.js](https://nextjs.org/) application authenticates users against the OIDC provider and then renders a simple page. Since this is a hosted application, the `HTTP_PROXY` is respected.

The final demo application is the **oidc_provider**. It is based on a Node.js package that provides OIDC server capabilities. This identity provider allows any user with any password and thus is not suitable for production environments. The provider supports OAuth 2.0 Token Exchange (**RFC8693**) to enable the proxy applications to fetch an access token for a specific user [@RFC8693].

## The Rust Programming Language

To achieve the goals of this work, the programming language "Rust" provides the necessary features to implement the authentication mesh. Rust itself is a multi-paradigm language that supports object-oriented features as well as functional components. Rust allows low-level memory management without the need for garbage collection and with guaranteed memory safety. To achieve this, Rust uses a special type checking mechanism that allows the compiler to calculate the lifetime of references and the ownership of the data [@Klabnik:Rust].

With the calculation of ownership and the transfer of ownership, Rust ensures that data can only ever be manipulated by one instance (its owner). No object can be modified without specifically taking ownership. Even though Rust allows an `unsafe` keyword, the code that it contains must be safe and is checked like normal Rust code. This was proven by Ralf Jung et al. by giving a formal safety proof for the language [@jung:RustBelt].

To demonstrate the advantages of Rust, consider the following code examples taken from the article "Safe Systems Programming in Rust" [@jung:Rust]:

```c++
std::vector<int> vec {10, 11};
// Create a pointer into the vector.
int *vectorPointer = &vec[1];
v.push_back(12);

// Bug ("use-after-free")
std::cout << *vectorPointer;
```

The C++ code above creates a vector of integers with two initial elements. Next, a pointer to the second element in the growable array is created. When the new content (`12`) is added to the vector, the backing memory buffer may be reallocated to allow the new object to be stored. The pointer now still points to the old memory address and therefore is a "dangling pointer" [@jung:Rust].

```rust
let mut vec = vec![10, 11];
let vector_pointer = &mut vec[1];
vec.push(12);

// This creates a compile error, since the vector is moved.
println!("{}", *vector_pointer);
```

The Rust compiler does check usage of data and references statically and therefore does not allow the use of a dangling pointer. The compiler will give the following error message for the code above: "cannot borrow vec as mutable more than once at a time." [@jung:Rust].

The safety of the Rust programming language and the C++-like performance are the primary reasons for the choice of the language.

## Sign and Distribute Contracts between Participants

This section shows how a contract between two parts of the authentication mesh can be created and distributed. To enable the authentication mesh to be truly distributed, the PKI of the separated parts must have a contract to create trust between the parties. Since the PKI creates its own root certificate, the other PKIs must be able to verify and trust the root CA of other PKIs.

### Using a Block Chain

One possibility to create and share such contracts is the usage of Block Chain.

#### Introduction

### Using a Master Key

### Distribute Contracts via Git

## Define the Contract
