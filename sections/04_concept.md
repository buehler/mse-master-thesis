\newpage

# Creating a Trust Context for the Authentication Mesh {#sec:concept}

This section gives a description of the required concept to create trust between distant parties in the mesh. It further briefly describes considered technologies and their limitations.

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
