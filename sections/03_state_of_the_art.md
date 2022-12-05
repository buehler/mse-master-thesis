\newpage

# The State of Distributed Authentication {#sec:state_of_the_art}

This section briefly explains the concept of the Distributed Authentication Mesh. Further, it shows the current state of the art of the mesh, and describes the deficiencies that this project solves.

## The Distributed Authentication Mesh in a Single Trust Zone

The concept "Distributed Authentication Mesh", as described in [@buehler:DistAuthMesh], allows applications to communicate with each other, even if they do not share the same authentication schemes.

![Two applications can communicate with an API, despite the fact, that the API only supports HTTP Basic authentication. The possibility to access an API with diverging authentication schemes is the basic principle of the Distributed Authentication Mesh [@buehler:DistAuthMesh].](images/03_single_context.png){#fig:03_single_context short-caption="Distributed Authentication Mesh in Single Trust Zone" width="80%"}

{@fig:03_single_context} shows the concept of the "Distributed Authentication Mesh". Both applications can communicate with the API, but they do not necessarily share the same authentication and authorization mechanisms. The mesh provides the means to translate authentication information into a common identity and transmit it to the receiving application. There, the common identity is translated back into the required authentication credentials (an HTTP Basic authorization header for example) [@buehler:DistAuthMesh].

![Network Architecture in the Distributed Authentication Mesh](diagrams/03_mesh_network_architecture.puml){#fig:03_mesh_network_architecture}

The "Distributed Authentication Mesh" builds upon the idea, that a proxy acts as mediator between source and destination. This proxy then uses an external service, the "Translator". The translator receives incoming and outgoing calls and has the ability to modify the requests. However, it must not interfere with the data plane. It shall only modify HTTP headers and allow or reject a connection. The translator is able to convert the provided authentication information (if any) into a common format.

The common identity is defined as a simple user ID. The ID is encapsulated into a JSON Web Token (JWT) and then signed by the client certificate that the translator receives from the PKI (public key infrastructure). The JWT is then sent to the destination application where the JWT is parsed and validated. The ID is extracted from the JWT and the information can be translated into the corresponding authentication credentials (for example, username/password combination for HTTP Basic) [@buehler:CommonIdentity].

## Multiple Trust Zones and Distribution

In its current state, the Distributed Authentication Mesh is able to run inside the same trust zone with a shared common identity [@buehler:DistAuthMesh; @buehler:CommonIdentity]. The mesh handles the conversion of authentication information (such as an access token or a login/password combination) by transforming it into a shared format. A sender encodes the user ID in a JWT and signs it with its own private key. The receiver can then verify that the information is not modified and that the sender is part of the authentication mesh.

However, the connection between the participants is prone to attacks in multiple ways. The concept only works, if all applications of the mesh are within the same trust zone (for example in the same Kubernetes cluster behind the same API gateway). If part of the application runs on a different cluster, the same trust cannot be applied. An attacker may get their own key material from a mesh PKI and can pose as a valid participant of the mesh. Therefore, the confidentiality and integrity are violated. Further, the receiving end of the communication has no possibility to verify the sender of the message for certain.

![Distributed Authentication Mesh with Multiple Trust Zones](images/03_multi_context.png){#fig:03_multi_context width="80%"}

The situation in {@fig:03_multi_context} shows the basic problem of the "Distributed Authentication Mesh". It is not truly distributed over multiple clusters and trust zones. It can only be used within a single trust zone, as {@fig:03_single_context} showed. The communication between the application and the API could be intercepted by an attacker. An attacker could fetch its own key material from either PKI and then pose as a valid member of the mesh since the common identity only stores the user ID in the JWT [@buehler:CommonIdentity].

## Contracts for Distribution

To achieve true distribution in the authentication mesh, the mesh needs a possibility to form trust between different trust zones. Various trust zones must establish contracts between them that function as a trust anchor. Trusting another "zone" shall result in an exchange of the public keys of their respective PKIs. With that contract, the mesh can allow its participants to use mutual TLS (mTLS) instead of normal HTTP connections. When mTLS is in place, sender and receiver of the communication can verify that they communicate with the correct entity and thus can verify if a trust anchor between the two exists.

![Creating Trust with a Contract](images/03_trust_contract.png){#fig:03_trust_contract width="80%"}

Regarding {@fig:03_trust_contract}, a contract between the two trust zones creates the trust anchor between the zones. This trust further allows the mTLS connection between the applications to be established. If the connection can be created (i.e. it is not rejected by either side) the participants trust each other and are who they pretend to be.
