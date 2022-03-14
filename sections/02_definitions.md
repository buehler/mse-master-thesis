\newpage

# Definitions and Clarification of the Scope {#sec:definitions}

This section provides the scope, context and prerequisite knowledge for this project. It also gives an overview of the used technologies as well as an introduction into the security topic of the project. Note that a deeper introduction into other security related technologies is given in the implementation section.

## Scope of this Project

This project builds upon two former projects "Distributed Authentication Mesh" [@buehler:DistAuthMesh] and "Common Identities in a Distributed Authentication Mesh" [@buehler:CommonIdentity]. The past work did define a general concept for a distributed authentication [@buehler:DistAuthMesh] and the definition and implementation of a common identity that is shared between the applications in the mesh [@buehler:CommonIdentity].

The goal of this project is to achieve a truly distributed mesh. To reach a distributed state in the mesh and to be able to trust other trust zones, a contract between each zone must exist. This project defines and implements the contract and provides the tools that are necessary to run such a mesh in Kubernetes. In this project, we analyze different options to form a contract between distant parties and define the specific properties of the contract. After the analyzation and definition, an open-source implementation shall show the feasibility and the usability of the distributed authentication mesh.

Service mesh functionality, such as service discovery even for distant services, is not part of the authentication mesh nor of this project. While the authentication mesh is able to run alongside with a service mesh, it must not interfere with the resolution of the communication. The applications that are part of the mesh must be able to respect the `HTTP_PROXY` and `HTTPS_PROXY` variables, since the Kubernetes Operator will inject those variables into the application. This technique allows the mesh to configure a local sidecar as the proxy for the application.

## Introduction into Kubernetes

Since the provided implementation of the distributed authentication mesh runs on Kubernetes, this section gives a brief overview of Kubernetes and the used patterns. Kubernetes is a workload manager that can load balance tasks on several nodes (servers). The explained patterns allow developers to extend the basic Kubernetes functionality.

### Basic Terminology

To understand further concepts and Kubernetes in general, some basic terminology and concepts around Kubernetes must be understood.

![Basic Buildingblocks in Kubernetes](images/02_kubernetes_parts.png){#fig:02_kubernetes_parts}

A **Pod** is the smallest possible deployment unit and contains a collection of application containers and volumes [@burns:KubernetesBook, ch. 5]. {@fig:02_kubernetes_parts} shows a Pod that contains two containers. Containers are definitions for workloads that must be run. To enable Kubernetes to run such a container, a containerized application and a container image must be present. Such an image-format is "Docker"^[<https://www.docker.com/>], a container runtime for various platforms.

**Deployments** manage multiple Pods. A Deployment object manages new releases and represent a deployed application. They enable developers to move up to new versions of an application [@burns:KubernetesBook, ch. 10]. In {@fig:02_kubernetes_parts}, a Deployment holds the Pod which in turn holds the containers.

A **Service** makes ports in Pods accessible to the Kubernetes world. They provide service discovery via Kubernetes internal DNS services [@burns:KubernetesBook, ch. 7]. The service in {@fig:02_kubernetes_parts} enables access to one of the containers in the Pod. A service load balances access if multiple containers match the service description.

**Ingress** objects define external access to objects within Kubernetes. Kubernetes uses "Ingress Controllers" that configure the access to services and/or containers [@burns:KubernetesBook, ch. 8]. As an example, "NGINX"^[<https://www.nginx.com/>] is an ingress controller that is often used. When an Ingress is configured to allow access to the service in {@fig:02_kubernetes_parts}, NGINX is configured that the respective virtual host forwards communication to the given service (reverse-proxying).

### What is an Operator

The "Operator pattern" provides a way to automate complex applications in Kubernetes. An Operator can be compared to a Site Reliability Engineer (SRE) because the Operator manages and automates complex applications with expert knowledge [@dobies:Operators].

An Operator makes use of "Custom Resource Definitions" (CRD) in Kubernetes. These definitions extend the Kubernetes API with custom objects that can be manipulated by a user of the Kubernetes instance. The Operator "watches" for events regarding objects in Kubernetes. The events can contain the creation, modification, and deletion of such a watched resource.

In the distributed authentication mesh, an Operator is used to automatically attach a deployment to the mesh and configure the corresponding services accordingly. The Operator modifies the ports of the service to target an Envoy^[<https://www.envoyproxy.io/>] proxy and injects the credential translator into the application (Deployment) in question [@buehler:DistAuthMesh].

A partial list of operators available to use is viewable on <https://operatorhub.io>.

### What is a Sidecar

A Sidecar is an extension to an existing Pod. Some controller (for example an Operator) can inject a Sidecar into a Pod or the Sidecar gets configured in the Deployment in the first place. [@burns:DesignPatterns]

## Security, Trust Zones, and Secure Communication

### The CIA Triad

### Trust is Important

### Zones and Zero Trust

### Securing Communication between Parties
