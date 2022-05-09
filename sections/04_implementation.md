\newpage

# Creating a Trust Context for the Authentication Mesh {#sec:implementation}

This section gives an overview of the used demo applications, the programming language Rust, and several security topics that are relevant for the implementation of the authentication mesh. Furthermore, the implementation of the shared trust context is described.

## Demo Applications

To demonstrate and test the implementation of the trust context, multiple demo applications are used. All applications are hosted on GitHub in the open source repository <https://github.com/WirePact/demo-applications>. There exist six different applications that are described below.

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

### Using a Blockchain

One possibility to create and share such contracts is the usage of Blockchain. Blockchain and smart contracts allow participants to validate the transaction history of the chain and therefore give a possibility to create trust between the parties.

#### Introduction into Blockchain

![Basic Principle of a Blockchain](images/04_blockchain_overview.png){#fig:04_blockchain_overview}

The basic principle, stated in {@fig:04_blockchain_overview}, shows how new blocks in the chain come to existence. The first block is called the "genesis block" and has no information about any previous blocks. All blocks down the chain contain information about the previous block. Along with the previous hash, each block contains a hashed history of all transactions [@nofer:blockchain].

The transaction history is encoded in a Merkle tree, a data structure where all leaf nodes are values of one-way functions. Merkle trees are often found in cryptography. However, the Merkle tree has a particular downside: traversing the tree requires a large amount of computation [@jakobsson:MerkleTree].

A blockchain allows transactions without the need for a third party authority. This enables smart contracts, a technology that executes certain contract clauses when specified conditions are met. The contracts and their specifics are published on a blockchain and can be verified by other participants [@zheng:SmartContracts].

#### Using Blockchain to Create a Contract

One possible way to create trust between the arbitrary PKIs in the authentication mesh is the use of a smart contract. The PKIs of the authentication mesh would be connected to a blockchain that spans over all participants in the mesh.

![Blockchain Smart Contract between PKIs](diagrams/04_blockchain_contract.puml){#fig:04_blockchain_contract}

{@fig:04_blockchain_contract} shows the necessary steps to form trust between two PKIs in the authentication mesh. Since all operations are performed on a blockchain, the contract and the steps to form it are verified by other participants as well.

With the smart contract, both parties can exchange their public key material and form a trust anchor between them without the need of a third party authority. As soon as the contract is voided by any of the parties, the trust anchor is revoked.

#### Using a Blockchain PKI to Create Certificates

Another possibility to create trust between the distributed participants of the authentication mesh is the usage of a distributed PKI (dPKI). The distributed PKI would act as a mediator between the different PKI that exist in each trust zone.

![Using a Decentralized Public Key Infrastructure (dPKI) as root PKI to ensure that all participants are able to create trust between them.](images/04_blockchain_dPKI.png){#fig:04_blockchain_dPKI short-caption="Decentralized Public Key Infrastructure on Blockchain"}

With a dPKI deployed on a blockchain, as shown in {@fig:04_blockchain_dPKI}, each specialized PKI in a trust zone could request a certificate that acts as the root for the trust zone of that PKI. The PKI fulfills its role as key material provider for the specific zone and has knowledge about the other PKIs in the mesh through the blockchain. If two zones are to trust each other, a configuration on the blockchain defines that two parties must create trust. Since the specific PKIs already have the information about the other certificates, they can validate the public key material of services in other zones.

An example of such a distributed PKI for blockchain is "ETHERST". However, deploying the PKI on the blockchain has the disadvantage of raising prices for the PKI. The participants need to pay the Ethereum gas prices to request and sign a certificate in ETHERST [@koa:ETHERST].

#### Security Issues with Blockchain

When considering the CIA triad in {@sec:definitions}, only _integrity_ and _availability_ can be provided. No information that is published to the blockchain is confidential and can be read by all participants in the chain.

While the blockchain approach seems elegant, it also bears some security issues. A blockchain can be attacked by a "majority attack" where an attacker holds more than 51% of the computing power in the blockchain. If this happens, the next calculation for the Proof of Work algorithm can be found faster than the rest of the network is able to validate the calculation. Therefore, an attacker can decide which blocks are valid and which are not [@lin:BlockchainSecurityIssues]. There exist other issues and attack vectors, but the majority attack would be the most threatening one for the distributed authentication mesh.

### Using a Master Node

A more centralized approach to form trust between participants is the usage of a master node.

![Centralized Trust Manager for Participants](images/04_central_master.png){#fig:04_central_master}

{@fig:04_central_master} shows the basic concept. While the trust zones remain decentralized, the master node must be central to manage the trust between the PKIs. The master node creates contracts between the PKIs of the participants. This could happen via API calls or via configuration stored in a store. However, this creates a single point of failure since the master node must also validate the trust. Trust revocation is done via the master node as well. If the master node is the target of an attack, the whole trust in the mesh is threatened. The master node is the single point of failure for inter-zonal communication.

### Distribute Contracts via Git

A third option to establish contracts between PKIs in the authentication mesh is the usage of a git repository. Git is a distributed version control system. It consists of a central repository server and a set of clients that clone the repository locally [@spinellis:Git].

![Use Git Repository for Trust Management](images/04_git_repo.png){#fig:04_git_repo}

The basic principle is depicted in {@fig:04_git_repo}. A central git repository acts as distribution node for contracts between the parties and therefore between the trust zones. The contract is either created via some application or via manual creation by an administrator. The contract is then pushed into the central repository. All participants can periodically check for new or revoked contracts in the repository. A contract is only valid as long as the file is physically present in the repository. To revoke a contract, the file is deleted from the repository.

With a central repository, other security concerns arise. The repository is not crucial for the communication between participants, but it is relevant for the management of the contracts. While a denial of service attack may not impact the communication itself, it can disable the possibility to check for revoked contracts. Furthermore, the history of a git repository is not secure since the clients can hold a local clone.

## Define the Contract
