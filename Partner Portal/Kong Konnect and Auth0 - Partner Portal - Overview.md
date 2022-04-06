# Kong Konnect

Implement your OIDC based authetication processes to protect your APIs with Kong and Auth0


## Overview
API gateway authentication is an important way to control the data that is allowed to be transmitted using your APIs. Basically, it checks that a particular consumer has permission to access the API, using a predefined set of credentials.

Kong Gateway has a library of plugins that provide simple ways to implement the best known and most widely used methods of API gateway authentication. Here are some of the commonly used ones:

Basic Authentication
Key Authentication
OAuth 2.0 Authentication
LDAP Authentication Advanced
OpenID Connect

Please, refer to the following link to read more about [API Gateway authentication](https://konghq.com/learning-center/api-gateway/api-gateway-authentication)

### OpenID Connect
Of all authentication mechanisms listed before, OpenID Connect (OIDC) is the preferred one for advanced requirements. In fact, when applying OIDC to secure the APIs, we're delegating the Authentication process to an external entity, called Identity Provider.

OIDC is a standard built on top of OAuth and JWT (JSON Web Token). Please, refer to the following link to learn more about [OAuth/OIDC](https://auth0.com/docs/authenticate/protocols/openid-connect-protocol) and its Flows:
Authorization Code
Implicit
Resource Owner Password
Client Credentials

Auth0 is one of the main IdPs in the marketplace today. Kong, on the other hand, is the world’s most popular API gateway, built for hybrid and multi-cloud and optimized for microservices and distributed architectures. The Auth0 and Kong integration provides a powerful and flexible solution for secure API Management infrastructures.

### Kong Konnect and Auth0 Reference Architecture
The following diagram shows a Reference Architecture including Kong Konnect and Auth0
![Reference Architecture](images/Reference_Architecture.png)

