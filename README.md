# Master Thesis MSE

Spring and Autumn Semester 2022\
Eastern Switzerland University of Applied Science (OST)

## Abstract

The "Distributed Authentication Mesh" is a concept to authenticate and authorize an identity over multiple services that do not share an authentication scheme. The mesh uses a common identity to encode the authorization information into a JSON Web Token (JWT) that is signed by a certificate of the system. The JWT is then used to authenticate the user at the participating services. However, the current concept and implementation of the mesh does not allow the trusted, secure communication between distant trust zones.

This thesis analyzes the current state of the mesh and provides a solution to spread the "Distributed Authentication Mesh" over multiple trust zones and environments. The project analyzes several possibilities to form a trust contract between trust zones of the mesh. After the analysis, a contract is designed and implemented. The contract is then used to distribute the mesh over multiple trust zones and allow secure communication between the zones. The thesis also provides a working demo setup of the mesh that can be used to validate the concept. The conclusions of the thesis provide a detailed summary of the project and possible extensions to the mesh in follow-up work.

## Thanks

I would like to express my appreciation to [Mirko Stocker](https://github.com/misto) for guiding and reviewing this work. Furthermore, special thanks to [Florian Forster](https://github.com/fforootd), who provided the initial inspiration and technical expertise of the topic.

## Full Report

To view the full project report please visit:
[Trust in a Distributed Authentication Mesh](https://buehler.github.io/mse-master-thesis/report.pdf)
