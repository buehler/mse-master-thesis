\newpage

# Definitions and Clarification of the Scope {#sec:definitions}

This section provides the scope, context and prerequisite knowledge for this project. It also gives an overview of the used technologies as well as an introduction into the security topic of the project. Note that a deeper introduction into other security related technologies is given in the implementation section.

## Scope of this Project

This project builds upon the two former projects "Distributed Authentication Mesh" [@buehler:DistAuthMesh] and "Common Identities in a Distributed Authentication Mesh" [@buehler:CommonIdentity]. The past work did define a general concept for a distributed authentication [@buehler:DistAuthMesh] and the definition and implementation of a common identity that is shared between the applications in the mesh [@buehler:CommonIdentity].

The goal of this project is to achieve a distributed mesh. To reach a distributed state in the mesh and to be able to trust other trust zones, a contract between each zone must exist. This project defines and implements the contract and provides the tools that are necessary to run such a mesh in Kubernetes. In this project, we analyze different options to actually form a contract between distant parties and define the specific properties of the contract. After the analyzation and definition, an open-source implementation shall show the feasibility and the usability of the distributed authentication mesh.

Service mesh functionality, such as service discovery even for distant services, is not part of the authentication mesh nor of this project. While the authentication mesh is able to run alongside with a service mesh, it must not interfere with the resolution of the communication. The applications that are part of the mesh must be able to respect the `HTTP_PROXY` and `HTTPS_PROXY` variables, since the Kubernetes Operator will inject those variables into the application. This technique allows the mesh to configure a local sidecar as the proxy for the application.

## Kubernetes

### Basic Terminology

### What is an Operator

### What is a Sidecar

## Trust Zones and Secure Communication

### Trust is Important

### Zones and Zero Trust

### Securing Communication between Parties
