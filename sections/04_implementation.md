\newpage

# Creating a Trust Context for the Authentication Mesh {#sec:implementation}

This section gives an overview of the created demo applications, the programming language Rust, and security topics that are relevant for the implementation of the authentication mesh. Furthermore, the section describes the implementation of the trust contract and the relation to the authentication mesh.

## Demo Applications

To demonstrate and test the implementation of the trust context and the mesh, multiple demo applications are used. All applications are hosted on GitHub in the open-source repository <https://github.com/WirePact/demo-applications>. There exist six different applications that are described below.

The **basic_auth_api** is a simple API application written in Go^[<https://go.dev/>]. It uses HTTP Basic Authentication (RFC7617) to authenticate calls against its endpoints. The API can be configured with three different environment variables (`PORT`, `AUTH_USERNAME`, and `AUTH_PASSWORD`). An HTTP web framework package "[Gin](https://github.com/gin-gonic/gin)" provides the HTTP middleware for Go.

```go
router := gin.Default()
secure := router.Group("/", gin.BasicAuth(gin.Accounts{
	config.Username: config.Password,
}))
secure.GET("swapi/people", getPeopleFromSwapi)
router.OPTIONS("/swapi/people", cors)
```

The code above shows the implementation of the HTTP Basic Authentication in the Go application. The `gin.BasicAuth` function is used to create a middleware that is applied to the `secure` group. The middleware checks the HTTP request for the `Authorization` header and validates the credentials against the given accounts. The named map `gin.Accounts` is a map that contains username / password combinations. The `getPeopleFromSwapi` function is called if the authentication was successful.

The static website **basic_auth_app** provides a trivial way of accessing any basic protected API. The site runs within an NGINX and contains minimal code. Since this site is hosted statically and does not call API endpoints through some backend logic, it is not possible to adhere to the `HTTP(S)_PROXY` environment variable to route traffic through a specific proxy.

In contrast to the basic auth app, the **basic_auth_backend_app** is an `ASP.NET` application that also uses the HTTP Basic mechanism to authenticate requests. However, the application runs in an ASP.NET context. Thus, it is possible to respect the `HTTP_PROXY` and `HTTPS_PROXY` variable and route traffic through a specific proxy. The application shows a trivial GUI in which the user can specify an API endpoint and a username / password combination.

To provide a more complex authentication scheme, the **oidc_api** authenticates requests against its API via `OAuth2.0`. When the API receives an access token from a client, it uses token introspection (defined by **RFC7662**) to validate the token and authenticate the user [@RFC7662]. The API needs an issuer, a client ID, and a client secret to validate the given tokens.

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

The code above shows the configuration of the C\# API application. It enables the API to verify an incoming access token by using the introspection endpoint of the OIDC provider. The introspection endpoint is defined in **RFC7662** [@RFC7662].

To complement the OIDC API, an **oidc_app** provides the means to access an OIDC (OAuth2.0) protected API via an application. This [Next.js](https://nextjs.org/) application authenticates users against the OIDC provider and then renders a simple page. Since this is a hosted application, the `HTTP(S)_PROXY` variable is respected. The app calls the API and attaches the access token in the `HTTP Authorization` header. The API validates the token and returns the requested data or denies the request.

The final demo application is the **oidc_provider**. It is based on a Node.js package that provides OIDC server capabilities. This identity provider allows any user with any password and thus is not suitable for production environments. The provider supports OAuth 2.0 Token Exchange (**RFC8693**) to enable the proxy applications to fetch an access token for a specific user [@RFC8693].

## The Rust Programming Language

To achieve the goals of this work, the programming language "Rust" provides a solid base to implement the contract repository and other system relevant parts. Rust itself is a multi-paradigm language that supports object-oriented features as well as functional components. Rust further allows low-level memory management without the need for garbage collection. Despite the absence of garbage collection, Rust guarantees memory safety. To achieve it, Rust uses a special type checking mechanism that allows the compiler to calculate the lifetime of references and the ownership of the data [@Klabnik:Rust].

Since the compiler of Rust ensures that data can only be modified once and that code has no side effects, the language enables developers to create reliable and secure software. The strict compiler and the vast speed of the compiled results were the primary reasons for choosing Rust as the programming language for this work. The Rust language has comparable performance to C and C++ and is therefore suitable for fast reacting systems like the authentication mesh [@ivanov:IsRustFast].

With the calculation of ownership and the transfer of ownership, Rust ensures that data can only ever be manipulated by one instance (its owner). No object can be modified without specifically taking ownership. Even though Rust allows an `unsafe` keyword, the code that it contains must be safe and is checked like normal Rust code. This was proven by Ralf Jung et al. by giving a formal safety proof for the language (and the `unsafe` parts in its standard library) [@jung:RustBelt].

To demonstrate the advantages of Rust and its compiler, consider the following code examples taken from the article "Safe Systems Programming in Rust" [@jung:Rust]:

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

During this project, all existing elements of the Distributed Authentication Mesh were rewritten to the Rust programming language. Since the communication between the moving parts of the system use gRPC to communicate, the framework or language behind the system does not really matter.

## Sign and Distribute Contracts between Participants

This section shows how a contract between two parts of the authentication mesh can be created and distributed. To enable the authentication mesh to be truly distributed, the PKI of each trust zone must have a contract to create trust between them. Since each PKI creates its own root certificate, other PKIs must be able to verify and trust the root CA of other PKIs.

### Using a Blockchain

One possibility to create and share such contracts is the usage of Blockchain. Blockchain and smart contracts allow participants to validate the transaction history of the chain and therefore give a possibility to create trust between the parties.

#### Introduction into Blockchain

![Basic Principle of a Blockchain](images/04_blockchain_overview.png){#fig:04_blockchain_overview width="80%"}

The basic principle, stated in {@fig:04_blockchain_overview}, shows how new blocks in the chain come to existence. The first block is called the "genesis block" and has no information about any previous blocks. All blocks down the chain contain information about the previous block. Along with the previous hash, each block contains a hashed history of all transactions [@nofer:blockchain].

The transaction history is typically encoded in a Merkle tree, a data structure where all leaf nodes are values of one-way functions. Merkle trees are often found in cryptography. However, the Merkle tree has a particular downside: traversing the tree requires a large amount of computation [@jakobsson:MerkleTree].

A blockchain allows transactions without the need for a third party authority. The chain itself achieves a consensus if a new block is valid or not. This enables smart contracts, a technology that executes certain contract clauses when specified conditions are met. The contracts and their specifics are published on a blockchain and can be verified by other participants [@zheng:SmartContracts].

#### Using Blockchain to Create a Contract

One possible way to create trust between the arbitrary PKIs in the authentication mesh is the use of a smart contract. The PKIs of the authentication mesh would be connected to a blockchain that spans over all participants in the mesh.

![Blockchain Smart Contract between PKIs](diagrams/04_blockchain_contract.puml){#fig:04_blockchain_contract width="60%"}

{@fig:04_blockchain_contract} shows the necessary steps to form trust between two PKIs in the authentication mesh. Since all operations are performed on a blockchain, the contract and the steps to create it are verified by other participants as well.

With the smart contract, both parties can exchange their public key material and generate a trust anchor between them without the need of a third party authority. As soon as the contract is voided by any of the parties, the trust anchor is revoked.

#### Using a Blockchain PKI to Create Certificates

Another possibility to create trust between the distributed participants of the authentication mesh is the usage of a distributed PKI (dPKI). The distributed PKI would act as a mediator between the different PKI that exist in each trust zone.

![Using a Decentralized Public Key Infrastructure (dPKI) as root PKI to ensure that all participants are able to create trust between them.](images/04_blockchain_dPKI.png){#fig:04_blockchain_dPKI short-caption="Decentralized Public Key Infrastructure on Blockchain" width="70%"}

With a dPKI deployed on a blockchain, as shown in {@fig:04_blockchain_dPKI}, each specialized PKI in a trust zone could request a certificate that acts as the root for the trust zone of that PKI. The PKI fulfills its role as key material provider for the specific zone and has knowledge about the other PKIs in the mesh through the blockchain. If two zones are to trust each other, a configuration on the blockchain defines that two parties must create trust. Since the specific PKIs already have the information about the other certificates, they can validate the public key material of services in other zones.

An example of such a distributed PKI for blockchain is "ETHERST". However, deploying the PKI on the blockchain has the disadvantage of raising prices for the PKI. The participants need to pay the Ethereum gas fees to request and sign a certificate in ETHERST [@koa:ETHERST].

#### Security Issues with Blockchain

When considering the CIA triad in {@sec:definitions}, only _integrity_ and _availability_ can be provided. No information that is published to the blockchain is confidential and can be read by all participants in the chain.

While the blockchain approach seems elegant, it also bears some security issues. A blockchain can be attacked by a "majority attack" where an attacker holds more than 51% of the computing power in the blockchain. If this happens, the next calculation for the Proof of Work algorithm can be found faster than the rest of the network is able to validate the calculation. Therefore, an attacker can decide which blocks are valid and which are not [@lin:BlockchainSecurityIssues]. There exist other issues and attack vectors, but the majority attack would be the most threatening one for the Distributed Authentication Mesh.

### Using a Master Node

A more centralized approach to form trust between participants is the usage of a master node.

![Centralized Trust Manager for Participants](images/04_central_master.png){#fig:04_central_master width="60%"}

{@fig:04_central_master} shows the basic concept. While the trust zones remain decentralized, the master node must be central to manage the trust between the PKIs. The master node creates contracts between the PKIs of the participants. This could happen via API calls or via configuration in a secure storage location. However, this creates a single point of failure since the master node must also validate the trust. Trust revocation is done via the master node as well. If the master node is the target of an attack, the whole trust in the mesh is threatened. The master node is the single point of failure for inter-zonal communication.

### Distribute Contracts via Git

A third option to establish contracts between PKIs in the authentication mesh is the usage of a git repository. Git is a distributed version control system. It consists of a central repository server and a set of clients that clone the repository locally [@spinellis:Git].

![Use Git Repository for Trust Management](images/04_git_repo.png){#fig:04_git_repo width="60%"}

The basic principle is depicted in {@fig:04_git_repo}. A central git repository acts as distribution node for contracts between the parties and therefore between the trust zones. The contract is either created via some application or via manual creation by an administrator. The contract is then pushed into the central repository. All participants can periodically check for new or revoked contracts in the repository. A contract is only valid as long as the file is physically present in the repository. To revoke a contract, the file is deleted from the repository.

With a central repository, other security concerns arise. The repository is not crucial for the communication between participants, but it is relevant for the management of the contracts. While a denial of service attack may not impact the communication itself, it can disable the possibility to check for revoked contracts. Furthermore, the history of a git repository is not secure since the clients can hold a local clone.

## Define the Contract

When considering all options in the previous section, using a combination between fetching contracts and having a master access point is a solid compromise. It does not require payment of blockchain gas fees nor the setup of a private blockchain. Furthermore, it does provide the possibility to create and revoke contracts while not being the single point of failure if the server does not respond for a certain time period. However, the central repository is not secure against denial of service attacks. Such attacks can disable the possibility to check for contract updates.

The most basic information that is required in the trust contract is the public certificate of the PKIs. The public certificate is the root certificate of the specific trust zone. When both parties have the public key of the other party, they are able to verify certificates of the other PKI and therefore are enabled to create mTLS (mutual TLS) connections. The usage of mTLS in the authentication mesh does ensure that only trusted connections are allowed and all other attempts to connect to a service are rejected. This further enables the authentication mesh to guarantee that only trusted participants can send the custom HTTP header that authenticates the user.

![Trust Contract between PKIs](diagrams/04_define_contract.puml){#fig:04_define_contract width="60%"}

The contract between two parties is simple. As {@fig:04_define_contract} shows, the only parts required to form a contract is the public key of the respective partners. With the public key, either PKI can verify the other PKIs certificates and thus allow an mTLS connection. The contract can be extended in future work to enable other use-cases like rule based access control or other security features.

To enable serialization and to create a data scheme for the contracts, Protobuf^[<https://developers.google.com/protocol-buffers>] is used. Protobuf is a serialization format that defines the messages and calls in a `proto` file. The format is used by gRPC^[Google Remote Procedure Call, <https://grpc.io/>], a well-known RPC framework in microservice architecture. The `proto` files can be used to create client implementations and server stubs for programming languages.

```protobuf
message Participant {
    string name = 1;
    string public_key = 2;
    string hash = 3;
}

message Contract {
    repeated Participant participants = 1;
}
```

The `proto` definition above shows the structure of a contract. In principle, a contract is just a list of participants that trust each other. A participant may be involved in multiple contracts. All contracts that include the own participant, are fetched and installed into the local trust store. As soon as this is done, the Envoy proxy of the authentication mesh is able to connect to distant services with an mTLS connection.

## Implementing the Contract Repository

The (open-source) implementation of the contract repository resides in the GitHub repository <https://github.com/WirePact/k8s-contract-repository>. The contract repository consists of two parts: "API" and "GUI". The separation of these parts is done to enable the usage of the API without the user interface. The contract provider only needs access to the API while an administrator could use the gRPC API or the graphical interface to manage the contracts.

### Provide a High-Performance API for Contracts

The API is a gRPC based application that provides the means to fetch, create, and revoke contracts. The GUI is a web application that allows direct access to that API via web browser.

In contrast to a git based approach that is described in the previous sections, the local or Kubernetes storage provides a deterministic approach to store the contracts. Further, it improves the testability of the overall system. Using a git repository to store the contracts would not improve the security nor the distribution of the system. However, the basic concept of a git repository is used to distribute the contracts. The opposing part - the contract provider - fetches the contracts from the repository in a regular interval. The repository is not the single point of failure, but could be targeted with a denial of service attack.

The contracts do not contain any sensitive information. Therefore, the API does not need to encrypt them in any way. The contracts can be stored in two possible ways: "Local" and "Kubernetes". While the local storage repository just uses the local file system to store the serialized `proto` files, the "Kubernetes" storage adapter uses Kubernetes Secrets to store the contracts.

![Use-cases for the Contract Repository](diagrams/04_repo_usecases.puml){#fig:04_repo_usecases}

The use-cases shown in {@fig:04_repo_usecases} show the basic functionality of the contract repository. Admins use the GUI or the gRPC API to create, fetch, and revoke contracts in the system. Providers then use the gRPC API to fetch a list of all involved public certificates. This allows the contract provider to create a certificate chain that contains all involved parties and therefore allows mTLS connections to corresponding services.

![Provider fetching relevant contracts from the repository](diagrams/04_repo_get_certs.puml){#fig:04_repo_get_certs}

The application sequence in {@fig:04_repo_get_certs} depicts the process when a provider fetches the relevant list of contracts for itself. The provider calls the repository with its own public certificate (which it fetches from its own PKI). The repository then returns a list of all contracts that the provider is part of.

### Administrate Contracts via Graphical Web Interface

The GUI application is based on the "Lit"^[<https://lit.dev/>] framework, which uses native web components to create applications instead of an engine like "React" and "Angular". Web components are a mix between different technologies to create reusable custom HTML elements. They consist of three main technologies ("Custom HTML Elements", "Shadow DOM", and "HTML Templates") to create reusable elements with encapsulated functionality [@mdn:WebComponents].

```typescript
import { html, css, LitElement } from 'lit';
import { customElement, property } from 'lit/decorators.js';

@customElement('demo-element')
export class DemoElement extends LitElement {
  static styles = css`
    p {
      color: pink;
    }
  `;

  @property()
  name = 'World';

  render() {
    return html`<p>Hello ${this.name}!</p>`;
  }
}
```

The code above creates a custom "demo-element" that just prints "Hello World!" in pink. Note that the CSS style is not interfering with any other styles. The CSS block is encapsulated in this particular component only. To use the component above, one needs to include the "demo-element" in their HTML code.

```html
<div>
  <demo-element></demo-element>
</div>
```

The HTML above will render the demo element component inside the `<div>` and print "Hello World!" in pink. If multiple of these components are rendered, each has its own root DOM such that there is no interference between them.

The GUI application of the contract repository will allow administrators to create and delete contracts in the repository. The GUI directly interacts with the repository via gRPC-web calls. In contrast to gRPC, gRPC-web is a protocol that allows the usage of gRPC in web applications. It allows HTTP/1.1 and HTTP/2 calls and requires the API to understand gRPC-web or any form of translation layer between the two protocols.

## Implementing a Contract Provider

The contract provider is an application that fetches the contracts from the repository in a defined interval. The implementation can be found on the GitHub repository <https://github.com/WirePact/k8s-contract-provider>.

![Activity of the provider during each interval](diagrams/04_provider_interval.puml){#fig:04_provider_interval width="30%"}

During each interval, the provider executes the steps in {@fig:04_provider_interval}:

1. Connect to its own PKI.
2. Connect to the contract repository.
3. Check if the public key of the PKI is stored, if not, download and store it.
4. Check if a client certificate and key are stored, if not, create a key and fetch a certificate from the PKI.
5. Fetch all public certificates that the "own PKI" is involved it and store the certificates.

The following code blocks describe the actions that the provider takes to achieve the steps above.

```rust
debug!("Check PKI public certificate.");
if !storage.has_ca().await {
    info!("Fetching PKI public certificate.");
    let response = pki.get_ca(Request::new(())).await?.into_inner();
    storage.store_ca(&response.certificate).await?;
}
```

The first step after connecting to the PKI and the contract repository is to check if the configured storage location contains the public certificate of the "own" PKI. If not, the provider fetches the public certificate from the PKI and stores it in the storage adapter.

```rust
debug!("Check private certificate.");
if !storage.has_certificate().await {
    info!("Sign private certificate.");
    let (key, csr) = create_csr(&config.common_name)?;
    let response = pki
        .sign_csr(Request::new(grpc::pki::SignCsrRequest {
            csr: csr.to_pem()?,
        }))
        .await?
        .into_inner();
    storage
        .store_certificate(
            &response.certificate,
            &key.private_key_to_pem_pkcs8()?,
        )
        .await?;
}
```

Next, the provider validates if a client certificate and key are present in the storage adapter. This client certificate is required to enable the Envoy proxy to present it for the mTLS connection to the distant serivce. If no certificate and/or key is found, the provider creates a new key and a certificate signing request (CSR) and sends it to the PKI. The PKI then signs the CSR and returns the signed certificate. The provider now stores the certificate and the key in the storage adapter.

```rust
debug!("Fetch certificate chain.");
let (ca, ca_hash) = storage.get_ca().await?;
let response = repo
    .get_certificates(Request::new(
        grpc::contracts::GetCertificatesRequest {
            participant_identifier: Some(
                ParticipantIdentifier::Hash(ca_hash)
            ),
        }
    ))
    .await?
    .into_inner();
let mut certificates = response.certificates;
certificates.push(ca);
storage.store_chain(&certificates).await?;
info!("Stored {} certificates in chain.", certificates.len());
```

The last step is to fetch all certificates that are involved in the contracts that the provider is part of. The provider loads the public certificate of the "own" PKI and uses the hash of the certificate to fetch all participants that share a contract with its own PKI. The provider then attaches its own PKI root CA into the chain (since the API only returns "other" certificates) and stores the chain in the storage adapter.

Like other applications in this project and the Distributed Authentication Mesh, the provider is able to store the certificates in a local or Kubernetes storage adapter. The main goal of the provider is to fetch all public keys of participating PKIs to enable mutual TLS (mTLS) connections between participants.

Since there are multiple possible ways to inject additional trusted root certificates (all participant PKIs), the provider does only store the certificate in the defined storage adapter. In Kubernetes and its ingress controllers, the TLS context must be configured to use the certificate, the key, and the trusted root certificates. The NGINX ingress controller must know where the client certificate resides to connect to an internal service.

## Create Secure Communication between Services

With the Distributed Authentication Mesh and the additional extensions of this project, we are now able to create fully trusted communication between distant services. Even if the applications are not running in the same trust context. The Distributed Authentication Mesh provides the means to create a signed identity that can be used to authenticate a user [@buehler:DistAuthMesh]. The common identity allows participating systems to restore required authorization information for the targeted service [@buehler:CommonIdentity].

The contract repository and provider now allow the PKIs to form a trust contract with each other. This in turn allows services to establish mTLS connections with each other. When participants of the mesh communicate with other services in distant trust contexts, mTLS ensures that only allowed connections can be created. This mitigates the risk of external services forging an identity and connect to internal services. The secured connection proofs that the PKIs are trusted and therefore no further encryption for the common identity is required. The mTLS connection cannot be successfully created if the service (respectively its PKI) is not involved in a contract with the destination.

![The Contract Repository and the Trust Zones](diagrams/04_trusted_comm_contracts.puml){#fig:04_trusted_comm_contracts width="80%"}

{@fig:04_trusted_comm_contracts} shows how the parts interact with the contract repository. There are two different trust zones, each of which contains its own "main" PKI. The PKI generate a CA certificate root and create client certificates for the services within the same trust zone. An admin can create a trust contract between the two trust zones and stores the contract in the repository. Contract providers (for each service) can then fetch the contracts and provide a client certificate and a certificate chain to validate incoming client certificates.

## A Trusted Distributed Authentication Mesh

One issue with the Distributed Authentication Mesh is that the identity of a user is sent to a specific target service. The destination then translates this identity into valid authentication credentials [@buehler:DistAuthMesh]. This target service has no means to verify that the sender is actually part of the mesh itself [@buehler:CommonIdentity]. Inside the same trust zone, the service can trust the sender if it is not publicly exposed. But, the use-case of the mesh includes communication between different trust zones. Therefore, the service must be able to verify that the sender is part of the mesh. With the mentioned contracts and the contract repository, it is possible for all participants to fetch a list of contracts. The contracts include the public certificates of all participating PKIs. Thus, it is possible for an application to call an API in a distant trust zone and verify that the sender is part of the mesh.

To show and verify the statement, a demo application setup in Docker is provided in the GitHub repository "<https://github.com/WirePact/docker-demo>". This demo proofs that it is possible to create a connection between two applications via mTLS connection.

The Docker demo consists of various containers that are required for the mesh. To verify the setup and the system itself, this section provides a step by step analysis of the demo and the functionality of the mesh in conjunction with the contract repository.

![Trust Zone Alice](diagrams/04_proof_pki_alice.puml){#fig:04_proof_pki_alice}

{@fig:04_proof_pki_alice} shows the setup for the first trust zone, "Trust Zone Alice". It consists of a PKI, a contract provider, the application, an application proxy and the translator for WirePact. The PKI creates its own root certificate authority (CA) and creates a client certificate for the contract provider and the translator. The translator is responsible for the extraction and translation of the WirePact common identity [@buehler:CommonIdentity]. The proxy manages all incoming and outgoing communication of the application itself. To enable general access to the application, a public gateway allows incoming communication and passes it to the application proxy.

![Trust Zone Bob](diagrams/04_proof_pki_bob.puml){#fig:04_proof_pki_bob}

The second trust zone, depicted in {@fig:04_proof_pki_bob}, is similar. It contains the same elements except for a public gateway since the demo system resides in Docker. A real world example would include another gateway that limits the access to other containers in the system.

![Communication between Trust Zones](diagrams/04_proof_communication.puml){#fig:04_proof_communication width="80%"}

Without a contract, communication as shown in {@fig:04_proof_communication} is not possible. The HTTPS / mTLS connection between the two proxies cannot be established since they have totally different root CAs. To enable communication between the parties, both proxies must now all public certificates of the involved parties to allow verification of the certificates. When the contract is created, the public certificates of both PKIs are inserted and then stored in the contract repository. Both contract providers will fetch the contract and deliver the full certificate chain to their respective proxies. The proxies can now verify the certificates and establish a connection.

![mTLS Connection between Proxies](images/04_proof_tls_handshake.png){#fig:04_proof_tls_handshake}

To proof that the connection is secured via mTLS, the network traffic of the demo Docker setup was recorded^[With "termshark", a terminal only alternative to Wireshark (<https://github.com/gcla/termshark>)]. {@fig:04_proof_tls_handshake} shows the TLS handshake between the two proxies. All other communication is HTTP, while the communication between the proxies is HTTPS. We can see that the server does present its own certificate accompanied by the certificate request for the client. The client in turn does present its own certificate and then, the connection is established.
