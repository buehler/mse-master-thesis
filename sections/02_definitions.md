\newpage

# Definitions and Clarification of the Scope {#sec:definitions}

This section provides the scope, context and prerequisite knowledge for this thesis. It also gives an overview of the used technologies as well as an introduction into the security topic of the project. Note that a deeper introduction into other security related topics is given in the implementation section.

## Scope of this Project

This project builds upon the prior projects "Distributed Authentication Mesh" [@buehler:DistAuthMesh] and "Common Identities in a Distributed Authentication Mesh" [@buehler:CommonIdentity]. The past work defined a general concept for distributed authentication [@buehler:DistAuthMesh] and the definition and implementation of a common identity that is shared between the applications in the mesh [@buehler:CommonIdentity].

The goal of this project is to achieve a truly distributed mesh. To reach a distributed state in the mesh and to be able to trust other trust zones, a contract between each zone must exist. This project defines and implements the contract and provides the tools necessary to run such a mesh in a Proof of Concept. In this project, we analyze different options to form a contract between distant parties and define the specific properties of the contract. After the analysis and definition, an open-source implementation shall show the feasibility and the usability of the Distributed Authentication Mesh.

Service mesh functionality, such as service discovery, is not part of the authentication mesh nor of this project. While the authentication mesh is able to run alongside with a service mesh, it must not interfere with the resolution of the communication. The applications that are part of the mesh must be able to respect the `HTTP_PROXY` and `HTTPS_PROXY` variables. Past work has introduced a Kubernetes Operator, that will inject those variables into the application. This technique allows the mesh to configure a local sidecar as the proxy for the application. However, the concept of the mesh or this thesis does not rely on Kubernetes.

## Introduction into Kubernetes

Since the provided implementation of the Distributed Authentication Mesh is able to run on Kubernetes, this section gives a brief overview of Kubernetes and the used patterns. The PoC of this thesis runs purely in Docker, but past work created a Kubernetes Operator that allows running the mesh in a Kubernetes Cluster. Kubernetes is an orchestration system that can distribute tasks on several nodes (servers). The explained patterns allow developers to extend the basic Kubernetes functionality.

### Basic Terminology

To understand further concepts and Kubernetes in general, some basic terminology and concepts around Kubernetes must be understood.

![Basic Buildingblocks in Kubernetes](images/02_kubernetes_parts.png){#fig:02_kubernetes_parts width="50%"}

A **Pod** is the smallest possible deployment unit and contains a collection of application containers and volumes [@burns:KubernetesBook, ch. 5]. {@fig:02_kubernetes_parts} shows a Pod that contains two containers. Containers are definitions for workloads that must be run. To enable Kubernetes to run such a container, a containerized application and a container image must be present. Such an image-format is "Docker"^[<https://www.docker.com/>], a container runtime for various platforms.

**Deployments** manage multiple Pods. A Deployment object manages new releases and represent a deployed application. They enable developers to move up to new versions of an application [@burns:KubernetesBook, ch. 10]. In {@fig:02_kubernetes_parts}, a Deployment contains the Pod which in turn holds containers. There exist multiple deployment specifications, such as `Deployment` and `Stateful Set` which have their own use-cases depending on the specification.

A **Service** makes ports in Pods accessible to the Kubernetes world. They provide service discovery via Kubernetes internal DNS services [@burns:KubernetesBook, ch. 7]. The service in {@fig:02_kubernetes_parts} enables access to one of the containers in the Pod. A service load balances access if multiple containers match the service description.

**Ingress** objects define external access to objects within Kubernetes. Kubernetes uses "Ingress Controllers" that configure the access to services and/or containers [@burns:KubernetesBook, ch. 8]. As an example, "NGINX"^[<https://www.nginx.com/>] is an ingress controller that is popular. When an Ingress is configured to allow access to the service in {@fig:02_kubernetes_parts}, NGINX is configured that the respective virtual host forwards communication to the given service (reverse-proxying).

### What is an Operator

Site Reliability Engineering (SRE) is a specific software engineering technique to automate complex software. A team of experts uses certain practices and principles to run scalable and highly available applications [@beyer:SRE]. The "Operator pattern" provides a way to automate complex applications in Kubernetes. An Operator embodies the knowledge of SRE teams in software to automate certain tasks [@dobies:Operators].

An Operator makes use of "Custom Resource Definitions" (CRD) in Kubernetes. These definitions extend the Kubernetes API with custom objects that can be manipulated by a user of the Kubernetes instance [@burns:KubernetesBook, ch. 16]. The Operator "watches" for events regarding objects in Kubernetes. The events can contain the creation, modification, and deletion of such a watched resource. As an example, the "Postgres"^[<https://www.postgresql.org/>] database operator reacts to the `Postgres` custom entity. When such an entity is created within Kubernetes, the Operator starts and configures the Postgres database system.

![Interaction of the Distributed Authentication Mesh Operator in Kubernetes](diagrams/02_operator_in_mesh.puml){#fig:02_operator_in_mesh}

In the Distributed Authentication Mesh, an Operator is used to automatically include a deployment into the mesh and configure the corresponding services accordingly. The original application is enhanced with several sidecar containers. As {@fig:02_operator_in_mesh} shows, the Operator injects the credential translator and the Envoy^[<https://www.envoyproxy.io/>] proxy into the application (Deployment) and modifies the ports of the service to target the Envoy proxy [@buehler:DistAuthMesh].

### What is a Sidecar

A Sidecar is an extension to an existing Pod. Some controller (for example an Operator) can inject a Sidecar into a Pod or the Sidecar gets configured in the Deployment in the first place. [@burns:DesignPatterns]

![An Example of a Sidecar](images/02_sidecar_example.png){#fig:02_sidecar_example width=80%}

{@fig:02_sidecar_example} shows an example of a Sidecar. An application runs a Pod and writes log messages to `/var/logs/app.log` in the shared file system. A specialized "Log Collector" Sidecar can be injected into the Pod and read those log messages. Then the Sidecar forwards the parsed logs to some logging software like Graylog^[<https://www.graylog.org/>].

Sidecars can fulfil multiple use-cases. A service mesh may use Sidecars to provide proxies for service discovery. Logging operators may inject Sidecars into applications to grab and parse logs from applications. Sidecars are a symbiotic extension to an application [@burns:KubernetesBook, ch. 5].

## Introduction into Security, Trust Zones, and Secure Communication

The Distributed Authentication Mesh is a security application. Therefore, security is an important topic in this work. This section gives an overview of the relevant topics to understand further security related concepts. More in-depth knowledge is provided in {@sec:implementation}.

### The CIA Triad

The three pillars of information security: **Confidentiality**, **Integrity**, and **Availability**. These three elements form the foundation of security in information systems. The CIA triad is, despite the fact that it was first mentioned around the year 1980, still relevant for security practitioners and in general security management [@samonas:CIA].

Confidentiality addresses the topic of gaining access where one is not allowed to. If someone is able to read certain information without being authorized to do so, the confidentiality is breached. An example could be that some attacker is able to forge login credentials and thus has access to files they should not be able to see.

Integrity covers proving that some information was not modified. When an attacker is able to modify information in a system, even when the attacker is not able to read the information, the integrity of the information is compromised. For example, with a man in the middle (MITM) attack, the integrity of the communication is corrupted and the attacker may forge or change information that the users are sending/receiving [@mallik:MITM].

Availability handles the possibility to get the information from the particular system. If an attacker can prevent an authorized user to gain access to their information, the availability is impaired. This could happen, if an attacker uses a DDoS (distributed denial of service) attack to prevent access to a resource.

### Trust Zones and Zero Trust

Trust zones are the areas where applications "can trust each other". When an application verifies the presented credentials of a user and allows a request, it may access other resources (such as APIs) on the users' behalf. When the concept of trust zones is applied, other APIs may trust the original requester that the identity has authenticated itself. Typically, this is used in microservice architectures where only one point of access (the gateway into the zone) is exposed to the outside world. The APIs behind the application can then share the trust that the gateway created.

In contrast to trust zones, **"Zero Trust"** is a security model that focuses on protecting (sensitive) data [@iftekhar:ProtectDataWithZeroTrust]. Zero trust assumes that every call could be intercepted by an attacker. Thus, for the concept of zero trust, it is irrelevant if the application resides in an enterprise network or if it is publicly accessible. As a consequence of zero trust, user credentials must be presented and validated for each access to a resource [@rose:zero-trust].

### Securing Communication between Parties

The key focus of the Distributed Authentication Mesh is the possibility to provide a secured identity over a service landscape that has heterogeneous authentication schemes [@buehler:DistAuthMesh]. Thus, securing communication between participants is of most utter importance. A wide range of security mechanisms and authentication schemes exist. To demonstrate the Distributed Authentication Mesh and the contracts between the trust zones, the following schemes/techniques are used.

#### HTTP Basic Authentication

The "Basic" authentication scheme is defined in **RFC7617**. `Basic` is a trivial authentication scheme which provides an extremely low security when used without HTTPS. Even with HTTPS, Basic Authentication does not provide solid security for applications. It does not use any real form of encryption, nor can any party validate the source of the data. To transmit basic credentials, the username and the password are combined with a colon (`:`) and then encoded with Base64. The encoded result is transmitted via the HTTP header `Authorization` and the prefix `Basic` [@RFC7617].

#### OpenID Connect

OpenID Connect (OIDC) is not defined in an RFC. The specification is provided by the OpenID Foundation (OIDF). OIDC extends OAuth, which is defined by **RFC6749**. The OAuth framework only defines the authorization part and how access is granted to data and applications. OAuth does not define how the credentials are transmitted [@RFC6749].

![OIDC code authorization flow [@spec:OIDC]. Only contains the credential flow, without the explicit OAuth part. OAuth handles the authorization whereas OIDC handles the authentication.](diagrams/02_oidc_code_flow.puml){#fig:02_oidc_code_flow short-caption="OpenID Connect (OIDC) Authorization Code Flow"}

{@fig:02_oidc_code_flow} shows an example where a user wants to access a protected application. The user is forwarded to an external login page (Identity Provider) and enters his credentials. When they are correct, the user gets redirected to the web application with an authorization code. The code is used to fetch an access and ID token for the user. These tokens identify, authenticate and authorize the user. The application is now able to provide the access token to the API. The API itself is able to verify the presented token to validate and authorize the user.

#### Mutual Transport Layer Security (mTLS)

An mTLS connection is essentially a TLS connection, like in HTTPS requests, but both parties present an X509 certificate. The connection is only allowed to open if both parties present a valid and trusted certificate. Thus, it enables both parties to verify their corresponding partner and prevents man in the middle attacks [@siriwardena:mTLS].

![The mTLS Handshake for Client and Server](diagrams/02_tls_handshake.puml){#fig:02_tls_handshake width="40%"}

To establish an mTLS connection, the TLS handshake defined in **RFC5246** is used. {@fig:02_tls_handshake} shows such a handshake. The client sends its `Client Hello` and is then greeted by the server with a `Server Hello` response. The server response also includes the servers certificate to authenticate itself. The server requests a certificate from the client in the same response. The client can then verify the certificate of the server and returns its own certificate with other TLS related messages. When both parties verify the identity of the other party, the handshake is completed and the connection is established [@RFC5246]. The biggest advantage of mTLS is that both parties can verify the identity of the other party. Thus, it is not possible to impersonate a client or a server.

## Introduction into the Distributed Authentication Mesh

The concept of the "Distributed Authentication Mesh", as described in [@buehler:DistAuthMesh], is a practical attempt to shared authentication information in a distributed environment. With modern cloud environments, like Kubernetes or Docker, some problems like discovery of services and data transfer are solved in general [@burns:KubernetesBook, ch. 7]. Modern cloud-native applications (CNA) still have to deal with authentication. Legacy software, however, often only supports older authentication schemes. But with the current state in digitalization, they tend to be moved into the cloud as well. This leads to the issue of mismatching authentication, as the old authentication schemes are not compatible with the new cloud-native applications.

### Accessing Legacy Software with Cloud-Native Applications

The Distributed Authentication Mesh addresses the conversion of user credentials from one authentication scheme to another. When multiple services or applications with diverging authentication schemes are required to communicate, the credentials (such as access tokens or user/password combinations) need to be translated.

![The Problem with Diverging Authentication Mechanisms](images/02_is_dist_auth_mesh.png){#fig:02_is_dist_auth_mesh width=85%}

{@fig:02_is_dist_auth_mesh} shows an example: a service with the capability of handling OIDC access tokens wants to communicate with a legacy service that is only able to handle HTTP Basic Authentication. Either software is required to receive a change in code to be able to communicate with the other one.

To translate this into a real world example: a legacy customer relationship management (CRM) system, that has its own web GUI as well as an API is deployed on a Kubernetes cluster. The API is able to handle HTTP Basic Authentication but no modern schemes. The company in question creates a modern web application that supports OIDC. The company already has an Identity and Access Management (IAM) system deployed and uses the IAM for other applications as well. The modern web application communicates with a modern API that understands OIDC but then must fetch some data about a customer from the legacy CRM system. The legacy CRM system is not able to handle OIDC and thus the modern API must translate the OIDC access token into an HTTP Basic Authentication header.

For various reasons like budget, time or technical risks and skill availability, legacy applications are not always refactored before they are deployed into the cloud. Following the assumption about the reasons, the code change will most likely be introduced into the modern application, because it is presumably better maintainable and deployable than the legacy software. The modern software now receives changes that are not part of its core functionality and may introduce new bugs and security vulnerabilities. In small applications that consist of one or two services, implementing this conversion may be a feasible option. However, in large scale applications with several services, this is error-prone and time-consuming work. In practice, the stated scenario was encountered at various points in time. Legacy services may not be the primary use case. Another example is the usage of third-party applications without any access to the source code.

### The Contrast to Security Assertion Markup Language

"Security Assertion Markup Language" (SAML) is a Federated Identity Management (FIdM) standard. In modern applications, SAML, OAuth, and OIDC are the most popular FIdM standards. SAML is an XML framework for transmitting user data, such as authentication, entitlement, and other attributes, between services and organizations [@naik:SAMLandFIdM].

In contrast to SAML, the Distributed Authentication Mesh does not require any changes to the software. SAML does not cover the case when the authentication mechanisms do not match. In order for SAML to work, all participating applications and services need to be able to understand SAML. The same goal could be achieved, if all services would understand OIDC. The concept of the authentication mesh is built around already deployed software. This allows the translation of the authentication method without any changes to the applications.

### The Concept of Distributed Authentication

The architecture of the Distributed Authentication Mesh is not bound to a specific platform or a specific implementation. It is not bound to cloud environments as well. The concept is generalized and can be implemented in all digital environments [@buehler:DistAuthMesh].

![Abstract Architecture of the Distributed Authentication Mesh](diagrams/02_dist_auth_solution_architecture.puml){#fig:02_dist_auth_solution_architecture width=85%}

{@fig:02_dist_auth_solution_architecture} shows the abstract solution architecture of the original authentication mesh in [@buehler:DistAuthMesh]. There are several objects that support the general solution like a public key infrastructure and a config and secret storage. An optional operator can automate managing the additional software parts of the mesh.

The main part of the concept evolves around the proxies that are attached to deployed applications. An application service is composed out of three parts. The service (application), a translator that manages the transformation between the common identity and the specific authentication information, and a proxy that is responsible for the communication.

The communication between the instances is handled by the proxies. The proxies communicate with the translators to transform the credentials and modify HTTP headers in requests. However, the mesh must not interfere with the data communication. Handling errors on the data plane is not part of the mesh and must be done by the implementation of the proxy.

![Outbound Networking Sequence](diagrams/02_dist_auth_outbound_networking.puml){#fig:02_dist_auth_outbound_networking width=60%}

In {@fig:02_dist_auth_outbound_networking} the outgoing communication flow is depicted. When a source wants to communicate with another service, the communication is routed through the proxy and the proxy forwards the HTTP headers to the translator. The translator in turn transforms the credentials and returns a modification instruction to the proxy. The proxy then attaches the modified HTTP headers to the request and forwards it to the destination. The result of the call is then returned to the source [@buehler:DistAuthMesh].
