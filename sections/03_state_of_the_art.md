\newpage

# The State of Distributed Authentication {#sec:state_of_the_art}

This section shows the current state of the art of the Distributed Authentication Mesh. Further, it describes the deficiencies that this project solves.

## Multiple Trust Zones and Distribution

In its current state, the Distributed Authentication Mesh is able to run inside the same trust zone with a shared common identity [@buehler:DistAuthMesh; @buehler:CommonIdentity]. The mesh handles the conversion of authentication information (such as an access token or a login/password combination) by transforming it into a shared format. A sender encodes the user ID in a JSON Web Token (JWT) and signs it with its own private key. The receiver can then verify that the information is not modified and that the sender is part of the authentication mesh.

However, the connection between the participants is prone to attacks in multiple ways. The concept only works, if all applications of the mesh are within the same trust zone (for example in the same Kubernetes cluster behind the same API gateway). If part of the application runs on a different cluster, the same trust cannot be applied. An attacker may get their own key material from a mesh PKI (public key infrastructure) and can pose as a valid participant of the mesh. Therefore, the confidentiality and integrity are violated. Further, the receiving end of the communication has no possibility to verify the sender of the message for certain.

## Contracts for Distribution

To achieve true distribution in the authentication mesh, the mesh needs a possibility to form trust between different trust zones. Various trust zones must establish contracts between them that function as a trust anchor. Trusting another "zone" shall result in an exchange of the public keys of their respective PKIs. With that contract, the mesh can allow its participants to use mutual TLS (mTLS) instead of normal HTTP connections. When mTLS is in place, sender and receiver of the communication can verify they "speak" with the correct entity and thus can verify if a trust anchor between the two exists.

![Creating Trust with a Contract](images/03_trust_contract.png){#fig:03_trust_contract}

Regarding {@fig:03_trust_contract}, a contract between the two trust zones creates the trust anchor between the zones. This trust further allows an mTLS connection to be established. If the connection can be created (i.e. it is not rejected by either side) the participants trust each other and are who they pretend to be.
