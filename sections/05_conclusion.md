\newpage

# Conclusions and Outlook

In this thesis, we presented a solution to truly distribute the authentication mesh over distant clusters and trust zones. The Distributed Authentication Mesh in conjunction with the common identity was not able to run safely across trust zones [@buehler:DistAuthMesh; @buehler:CommonIdentity]. Inside the same trust zone, the mesh did provide its functionality, but as soon as the communication spans multiple clusters with their own gateways, the trust between the participants of the mesh could not be guaranteed. There was no mechanism in place to verify the sender of a common identity. The goals of this work were to analyze the situation of distributed trust systems and provide a solution for the Distributed Authentication Mesh.

{@sec:introduction} introduces the reader into the topic and gives references and introductions to past work. The section also briefly describes the general problem with the Distributed Authentication Mesh.

{@sec:definitions} defines the scope of this thesis and presents prerequisite knowledge to the reader. The section gives an introduction into Kubernetes, the container orchestration software, since it is used to show the use-case of the Distributed Authentication Mesh. Furthermore, several security related topics are described to allow the reader to understand the general problems at hand. {@sec:definitions} also explains two basic authentication and authorization mechanisms: "HTTP Basic" and "OpenID Connect".

{@sec:state_of_the_art} then describes the current state of the Distributed Authentication Mesh and its shortcomings. It shows how the mesh uses the common identity from [@buehler:CommonIdentity] to encode authorization information into a JSON Web Token (JWT). The section then shows a general hypothesis on how the communication between distant trust zones can be achieved.

{@sec:implementation}

This thesis analyzed the issue and gives a detailed solution for the problem. The solution is based on the idea of a trust contract between the public key infrastructures (PKIs) of the mesh. Each PKI creates its own root certificate authority (CA) which is used by the services inside the same trust zone to communicate with each other. To allow
