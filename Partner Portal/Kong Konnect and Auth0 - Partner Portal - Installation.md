## OpenID Connect
For Enterprise Class Applications, OpenID Connect (OIDC) is the preferred mechanisms for advanced requirements. In fact, when applying OIDC to secure the APIs, we're delegating the Authentication process to an external entity, called Identity Provider.

OIDC is a standard built on top of OAuth and JWT (JSON Web Token). Please, refer to the following link to learn more about [OAuth/OIDC](https://auth0.com/docs/authenticate/protocols/openid-connect-protocol) and its Flows:
* Authorization Code
* Implicit
* Resource Owner Password
* Client Credentials

### Kong Konnect and Auth0 Reference Architecture
The following diagram shows a Reference Architecture including Kong Konnect and Auth0
![Reference Architecture](images/Reference_Architecture.png)


## Prerequisites

1. An Auth0 account and tenant. [Sign up for free here](https://auth0.com/signup).
2. Subscribe to [Kong Konnect Plus](https://konnect.konghq.com/register?_ga=2.127542733.1909926982.1645531351-1264875266.1638464050) to get a 30-day trial with all Kong Gateway Enterprise capabilities enabled including IdP integrations with OpenID Connect

For the Client Credentials Flow, the Kong OpenID Connect plugin forwards the credentials passed by the client to the Auth0’s token endpoint. The Client Credentials grant is visualized below:

![Client Credentials](images/ClientCredentials.png)

1. Consumer sends a request with a credential to Kong Gateway.
2. Kong Gateway forwards the credential to Auth0 which returns tokens to the Gateway.
3. Kong Gateway routes the requests with Access Token injected.



## The Data Plane
The Kong+Auth0 Environment comprehends a Kong Data Plane, responsible for handling the requests coming from the Consumer and protect the Upstream Services and Application sitting behind it. The Data Plane can be deployment in any available platform including Linux-based Operation Systems, Container, Kubernetes, etc. For the purpose of this guide we assume the Kong Data Plane is running in a Kubernetes cluster.


### Kong Certificates and Key
Login to [Kong Konnect](https://konnect.konghq.com) and go to "Runtimes" -> "Configure Runtime" -> "Kubernetes".

Click on "Generate Certificate" and copy them.
* Save the "Cluster Certificate" as tls.crt
* Save the "Certificate Key" as tls.key
* Save the "Root CA Certificate" as ca.crt


### Injecting Key and DCs
Create a namespace and secrets for the Digital Certificates and Key
<pre>
kubectl create namespace kong-dp

kubectl create secret tls kong-cluster-cert --cert=./tls.crt --key=./tls.key -n kong-dp

kubectl create secret generic kong-cluster-ca --from-file=ca.crt=./ca.crt -n kong-dp
</pre>

### Deploying the Data Plane
Use the following Helm Chart command to deploy the Data Plane. Replace the Control Plane and Telemetry endpoints with the ones provided by Konnect Control Plane.

<pre>
helm install kong-dp kong/kong -n kong-dp \
--set ingressController.enabled=false \
--set image.repository=kong/kong-gateway \
--set image.tag=2.8.0.0-alpine \
--set admin.enabled=false \
--set env.role=data_plane \
--set env.database=off \
--set env.anonymous_reports=off \
--set env.vitals_ttl_days=732 \
--set env.cluster_mtls=pki \
--set env.cluster_control_plane=<b>your-control-plane-endpoint</b>:443 \
--set env.cluster_server_name=<b>your-control-plane-endpoint</b> \
--set env.cluster_telemetry_endpoint=<b>your-control-plane-telemetry-endpoint</b>:443 \
--set env.cluster_telemetry_server_name=<b>your-control-plane-telemetry-endpoint</b> \
--set env.cluster_ca_cert=/etc/secrets/kong-cluster-ca/ca.crt \
--set env.cluster_cert=/etc/secrets/kong-cluster-cert/tls.crt \
--set env.cluster_cert_key=/etc/secrets/kong-cluster-cert/tls.key \
--set env.lua_ssl_trusted_certificate=/etc/secrets/kong-cluster-cert/tls.crt \
--set proxy.enabled=true \
--set proxy.type=LoadBalancer \
--set enterprise.enabled=true \
--set enterprise.portal.enabled=false \
--set enterprise.rbac.enabled=false \
--set enterprise.smtp.enabled=false \
--set manager.enabled=false \
--set portal.enabled=false \
--set portalapi.enabled=false \
--set env.status_listen=0.0.0.0:8100 \
--set secretVolumes[0]=kong-cluster-cert \
--set secretVolumes[1]=kong-cluster-ca \
--set env.log_level=debug
</pre>



### Kong Gateway Service and Route Creation and Policies
### Create the Service Package
* Login to Konnect and click on <b>Add New Service</b>

* Create a new <b>httpbinservice</b> Service with <b>v1</b> version

* Click on <b>Create</b>



### Create the Service's Implementation
* Click on version <b>v1</b>

* Click on <b>New Implementation</b>

* Type <b>http://httpbin.org</b> for URL and click on <b>Next</b>

* Type <b>httpbinroute</b> for <b>Name</b>. Click on <b>+ Add Path</b> and type <b>/httpbin</b>. Click on <b>Create</b>.

### Accessing the Kong Data Plane Cluster
<pre>
$ kubectl get service --all-namespaces
NAMESPACE     NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)                      AGE
default       kubernetes           ClusterIP      10.100.0.1       <none>                                                                   443/TCP                      63m
kong-dp       kong-dp-kong-proxy   LoadBalancer   10.100.201.216   a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com   80:32015/TCP,443:32038/TCP   108s
kube-system   kube-dns             ClusterIP      10.100.0.10      <none>                                                                   53/UDP,53/TCP                63m
</pre>

Use the External IP Address provided by your Kubernetes cluster.



### Consume the Kong Route
Open a local terminal and send a request to the Data Plane using the AWS ELB already provisioned:

<pre>
$ kubectl get service kong-dp-kong-proxy -n kong-dp -o json | jq -r .status.loadBalancer.ingress[].hostname
a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com

$ http a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com/httpbin/get
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 550
Content-Type: application/json
Date: Tue, 15 Feb 2022 18:49:03 GMT
Server: gunicorn/19.9.0
Via: kong/2.7.1.1-enterprise-edition
X-Kong-Proxy-Latency: 13
X-Kong-Upstream-Latency: 165

{
    "args": {},
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Host": "httpbin.org",
        "User-Agent": "HTTPie/3.0.2",
        "X-Amzn-Trace-Id": "Root=1-620bf59f-725aac5a4390f9cc708a05e7",
        "X-Forwarded-Host": "a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com",
        "X-Forwarded-Path": "/httpbin/get",
        "X-Forwarded-Prefix": "/httpbin"
    },
    "origin": "192.168.37.246, 35.180.228.21",
    "url": "http://a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com/get"
}
</pre>



## Kong Konnect Rate Limiting Policy

The API has been exposed and can be consumed as many times as we want. Now it is time to define policies to control the exposure. The first policy we are going to create is Rate Limiting.

### Enable the Rate Limiting plugin to the Route
Generally speaking, a plugin can be enabled to specific objects like Services, Routes and Consumers or globally. Check the [documentation](https://docs.konghq.com/konnect/manage-plugins/) for more information.

* Click on the Route <b>httpbinroute</b>

* Click on <b>Add a Plugin</b>

* Scroll down to the <b>Traffic Control</b> section:

* Click on <b>Rate Limiting</b>:

For the <b>Config.Minute</b> parameter type <b>5</b> and for <b>config.policy</b> choose <b>local</b>. Click on <b>Create</b>.

As you can see we can define Rate Limiting policies by seconds, minutes, hours, days, months or years. An external Redis infrastructure can be used to share the Rate Limiting counters across multiple Data Plane instances. Please check the [Rate Limiting plugin](https://docs.konghq.com/hub/kong-inc/rate-limiting/) page for more information.


### Consume the Kong Route again
Send a new request to the Data Plane. As you can see new Headers have been added to the response showing the Rate Limiting policy status:

<pre>
$ http a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com/httpbin/get
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 550
Content-Type: application/json
Date: Tue, 15 Feb 2022 18:52:07 GMT
<b>RateLimit-Limit: 5
RateLimit-Remaining: 4
RateLimit-Reset: 53</b>
Server: gunicorn/19.9.0
Via: kong/2.7.1.1-enterprise-edition
X-Kong-Proxy-Latency: 12
X-Kong-Upstream-Latency: 165
<b>X-RateLimit-Limit-Minute: 5
X-RateLimit-Remaining-Minute: 4</b>

{
    "args": {},
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Host": "httpbin.org",
        "User-Agent": "HTTPie/3.0.2",
        "X-Amzn-Trace-Id": "Root=1-620bf657-5f8c8aba6a94c8b81018a394",
        "X-Forwarded-Host": "a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com",
        "X-Forwarded-Path": "/httpbin/get",
        "X-Forwarded-Prefix": "/httpbin"
    },
    "origin": "192.168.37.246, 35.180.228.21",
    "url": "http://a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com/get"
}
</pre>

Since we have defined a 5-request a minute policy, you should receive a specific 429 error code when trying to send more requests.

<pre>
$ http a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com/httpbin/get
<b>HTTP/1.1 429 Too Many Requests</b>
Connection: keep-alive
Content-Length: 41
Content-Type: application/json; charset=utf-8
Date: Tue, 15 Feb 2022 18:52:23 GMT
RateLimit-Limit: 5
RateLimit-Remaining: 0
RateLimit-Reset: 37
Retry-After: 37
Server: kong/2.7.1.1-enterprise-edition
X-Kong-Response-Latency: 1
X-RateLimit-Limit-Minute: 5
X-RateLimit-Remaining-Minute: 0

{
    "message": "API rate limit exceeded"
}
</pre>


## Kong Konnect Proxy Caching Policy
The second policy we are going to create is the Proxy Caching to get better performance and reduce latency times.

### Enable the Proxy Caching plugin to the Route
For the same Route we're going to enable the new policy based on the Proxy Caching plugin.

* Click on the Route <b>httpbinroute</b> again:

* Click on <b>Add Plugin</b>. Scroll down to the <b>Traffic Control</b> section again and choose the <b>Proxy Caching</b> plugin:

Similarly to what we did before, we going to set the Proxy Caching plugin with its specific parameters. For example, for <b>Config.CacheTtll</b> parameter type <b>30</b> and for <b>Config.Strategy</b> choose <b>memory</b>.

Check the [Proxy Caching Plugin](https://docs.konghq.com/hub/kong-inc/proxy-cache/) page for more information about it.

### Consume the Kong Route again
Send a new request to the Data Plane. Besides the Rate Limiting Headers, our response shows the Proxy Caching Status. The Status says the Gateway didn't have any data available to satisfy the request, so it had to go to the Upstream.

<pre>
$ http a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com/httpbin/get
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 550
Content-Type: application/json
Date: Tue, 15 Feb 2022 18:57:06 GMT
RateLimit-Limit: 5
RateLimit-Remaining: 4
RateLimit-Reset: 55
Server: gunicorn/19.9.0
Via: kong/2.7.1.1-enterprise-edition
X-Cache-Key: 299e9cfd9462b37fc817946be53b791e
<b>X-Cache-Status: Miss</b>
X-Kong-Proxy-Latency: 21
X-Kong-Upstream-Latency: 165
X-RateLimit-Limit-Minute: 5
X-RateLimit-Remaining-Minute: 4

{
    "args": {},
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Host": "httpbin.org",
        "User-Agent": "HTTPie/3.0.2",
        "X-Amzn-Trace-Id": "Root=1-620bf782-67cfa29c58087fb21a4b4b07",
        "X-Forwarded-Host": "a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com",
        "X-Forwarded-Path": "/httpbin/get",
        "X-Forwarded-Prefix": "/httpbin"
    },
    "origin": "192.168.37.246, 35.180.228.21",
    "url": "http://a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com/get"
}
</pre>

For the second request, the Status says we got a <b>Hit</b> meaning that the Gateway had everything it needed to respond to the consumer. Moreover, the latency times are considerably reduced.

<pre>
$ http a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com/httpbin/get
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Age: 3
Connection: keep-alive
Content-Length: 550
Content-Type: application/json
Date: Tue, 15 Feb 2022 18:57:06 GMT
RateLimit-Limit: 5
RateLimit-Remaining: 3
RateLimit-Reset: 55
Server: gunicorn/19.9.0
Via: kong/2.7.1.1-enterprise-edition
X-Cache-Key: 299e9cfd9462b37fc817946be53b791e
<b>X-Cache-Status: Hit
X-Kong-Proxy-Latency: 0
X-Kong-Upstream-Latency: 0</b>
X-RateLimit-Limit-Minute: 5
X-RateLimit-Remaining-Minute: 3

{
    "args": {},
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Host": "httpbin.org",
        "User-Agent": "HTTPie/3.0.2",
        "X-Amzn-Trace-Id": "Root=1-620bf782-67cfa29c58087fb21a4b4b07",
        "X-Forwarded-Host": "a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com",
        "X-Forwarded-Path": "/httpbin/get",
        "X-Forwarded-Prefix": "/httpbin"
    },
    "origin": "192.168.37.246, 35.180.228.21",
    "url": "http://a3337b1604fb9419190b54634515fcb3-183430464.eu-west-3.elb.amazonaws.com/get"
}
</pre>

If we wait for 30 seconds, since it's been our TTL for the Caching, the Gateway will purge all data from the Cache and report a <b>Miss</b> again.


## Kong + Auth0 - OIDC Client Credentials Grant
The <b>Client Credentials</b> Flow is defined by OAuth to address scenarios like Application Authentication, Machine-to-Machine, Microservices Accounts, etc. Please refer to this link to learn more about [Client Credentials](https://auth0.com/docs/get-started/authentication-and-authorization-flow/client-credentials-flow).

### Auth0 API Creation
Let's create our first Auth0 API:

* Login to Auth0 console using your endpoint like this: http://auth0.com<p>

* Click on <b>Applications</b> -> <b>APIs</b> -> <b>+ Create API</b>. Create a new API named <b>KongClientCredentials</b> with </b>https://clientcredentials.com</b> as its Identifier.<p>

* Click on <b>Create</b>

* Click on the <b>Machine to Machine Applications</b> tab. As you can see Auth0 automatically created a Test Application for us. The Application is also already authorized to use the API.

* Click on the <b>KongClientCredentials (Test Application)</b> Application link. Click on the "eye" button to see the <b>client secret</b>. Save <b>Domain</b>,  <b>Client ID</b> and <b>Client secret</b>. They'll be used to configure the Kong OIDC plugin and consume the API. In our case:<p>
<b>Domain: dev-xxxxxx.us.auth0.com</b><p>
<b>Client ID: r7lgArWHmm3E12elLTRDPVnBHAExruEg</b><p>
<b>Client Secret: 6F3nCynbE6Hgs9cUSWfMYvE2_urz_xPqOsAI0y6K3sob-ZT1AC_DGdn4j2pno7G_</b><p>





### Enable the OIDC plugin to the Route
Let's apply the OIDC plugin to the Route with the specific Auth0 settings:<p>
* <b>Issuer Endpoint</b><p>
* <b>Scope</b><p>

Check the [OIDC Plugin](https://docs.konghq.com/hub/kong-inc/openid-connect/) page for more information about the plugin.<p>

Go back to Kong Konnect Control Plane and click on <b>httpbinroute</b> again. Click on <b>Add Plugin</b>. Choose <b>OpenID Connect</b> plugin

Fill in the form with the following configuration:<p>
* <b>Config.Issuer</b>: use the Auth0 Domain we saved previously <b>https://dev-xxxxxx.us.auth0.com/.well-known/openid-configuration</b><p>
* <b>Config.Scopes</b>: delete the openid scope set by default<p>
* <b>Config.Token Post Args Names</b>: it should be set with <b>audience</b><p>
* <b>Config.Token Post Args Values</b>: it should be set with the API Identifier we used to create the Auth0 API: </b>https://clientcredentials.com</b><p>

Click on <b>Create</b>.

Consume the Route with Client Id and Client Secret. We should be able to consume the Kong Route with our Client Id and Client Secret:




<pre>
$ http a131cf848dbd44e45af06511412201fe-1032110377.eu-west-3.elb.amazonaws.com/httpbin/get -a r7lgArWHmm3E12elLTRDPVnBHAExruEg:6F3nCynbE6Hgs9cUSWfMYvE2_urz_xPqOsAI0y6K3sob-ZT1AC_DGdn4j2pno7G_
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 1316
Content-Type: application/json
Date: Wed, 16 Feb 2022 11:15:10 GMT
RateLimit-Limit: 5
RateLimit-Remaining: 4
RateLimit-Reset: 50
Server: gunicorn/19.9.0
Set-Cookie: session=Zhu08guEpxppKI_E8vTRXg|1645013709|rIDhFNGnbGlOsNM0sIyeqMkBg3tN-Kt6TVu-Ej7KGrT4PC2d8iBNeONvv6xutAjWGXHJHCHM_ShQfMCpb7xMWtDXGTB4aZ3gGMtRZLR2rm3UaSTJRuoiwOMi__fGGWyImAdj7vAcnOowAsAfvRmoTba1Tn4ub0spnvDJaHpEgyOwkqL7xTTwJDxJDPZaSiDjDMlgUz9bJv5Etky-O60SOr-Fxm7I4U9jxVl0mnuN1Q2JUyKhwoyD6GEXtydMXcQ2C39X2k67Lg2OnEEjnop1lMAX-rJ4SFElzYsPWwDGayFdPtZoqPJFN4OTAkcrwvJp1vJ38WwudJhbKBk6tSuE6FTrzEPwqbB0CtgPNWwUl1VP-lIjQ8hXJvkVAGzj-ugDUAL1c_dboqNjT666kXcbtX9ABWPlCLfddcZl4MU09V3mSBJLPoZD5Y02aUOVfbKdn4Vm66XMZ0MZZpLkpcPDz3FLYnKuAvHa34J9UdHZOsz611WeEhFXS4zLwUB959P9O5q5rGWQNfgj1hhA-yTkEJxczYyFfyvqOG8xAJ7XEeqVPgq_JfMvX86TtNY4IsMvtr8yBt5tfjYqUKLokUcZdwKrCjYyOq3ru_gfu0-7EjnCDcUOw_sLx1a3lRhxVVASVX1P6mwCBoCQVqR0_MS83_gE9AQrCFvwEziZqFUpjdVqIDDjRzGy_2yqANWlVZk_QDXSJCOZbeEoscNGmNKMWlVgLiN2rQK64rkHYduwyBjpa3QxfKCJ4HYyRWgcwIK_-VnPy8-v3olBeSe9lSI7zUVZUEZLtEtml2F7-AKFg_xdHcrRjGQIaISzvcidohzwK585eD6kQQw7__x2ZKlRz-6EZDSHiD5naOjOQoiliQYN8sND7iz8rTVWWCx4-IS-tDcuU-rNnhI-7Gk_65_2kGOgDFKG5yMVLzp1AOSP2z4A_lnvkn2Ts9jqokPdfRJcKduX4A5_lJPqJQ2HdYh8eqP6zRZimSVhzxpd2b4aA4gSc_GHayxThzv74S0hljrWfqKsUGIY5q9L56o1xXnqwWN_yR6KlJwg3ryrkTJZf9cezfF8N7HP2N4KZS13aOiqpQxjIY54-VRVC7qOl16O607nAfKVuBUREx_ywbOptJw|Gb9mscNQ0YGSPrBej8OBJHn44IU; Path=/; SameSite=Lax; HttpOnly
Via: kong/2.7.1.1-enterprise-edition
X-Cache-Key: 299e9cfd9462b37fc817946be53b791e
X-Cache-Status: Miss
X-Kong-Proxy-Latency: 1074
X-Kong-Upstream-Latency: 169
X-RateLimit-Limit-Minute: 5
X-RateLimit-Remaining-Minute: 4

{
    "args": {},
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        <b>"Authorization": "Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InlpdFBpTml2LXRQWkxGdjV2YlBNWCJ9.eyJpc3MiOiJodHRwczovL2Rldi0zeWd6a2J6Ny51cy5hdXRoMC5jb20vIiwic3ViIjoicjdsZ0FyV0htbTNFMTJlbExUUkRQVm5CSEFFeHJ1RWdAY2xpZW50cyIsImF1ZCI6Imh0dHBzOi8vY2xpZW50Y3JlZGVudGlhbHMuY29tIiwiaWF0IjoxNjQ1MDEwMTEwLCJleHAiOjE2NDUwOTY1MTAsImF6cCI6InI3bGdBcldIbW0zRTEyZWxMVFJEUFZuQkhBRXhydUVnIiwiZ3R5IjoiY2xpZW50LWNyZWRlbnRpYWxzIn0.XbbNUX36sG-pg7UGoDk5bfeexU4a1CuGCESAsDhIur5gmmMKPNq5YKFZNyLHQqTKEFwlBwyoofyTE5yBY1ES8M_skMAfG0w--AkJ_zE3hChOi-wOC7SKMmc70-JzK0wvz_QcyuKnPT2NUHMNUNC7hwJ0Vd4nOsnd7ZtUldBdhYOZkqdy4t8k4CK6IiOUSAewg0TkRIRMDg6hFcwhWflUxfNZigO2vP80c1G1KAhgaaHPmJcXxLcnE57aRPdc-3w5R6bXt0yiOJu3roBr3AYkksuruqGdRMDqCeOKbZNNMTRKhXfPNmYPNNOvxIu1Faw2ETcH0PXdSNbQrtryNN2tCQ"<b>,
        "Host": "httpbin.org",
        "User-Agent": "HTTPie/3.0.2",
        "X-Amzn-Trace-Id": "Root=1-620cdcbe-3e2ecc8c33302c050a43de4b",
        "X-Forwarded-Host": "a131cf848dbd44e45af06511412201fe-1032110377.eu-west-3.elb.amazonaws.com",
        "X-Forwarded-Path": "/httpbin/get",
        "X-Forwarded-Prefix": "/httpbin"
    },
    "origin": "192.168.37.246, 35.180.228.21",
    "url": "http://a131cf848dbd44e45af06511412201fe-1032110377.eu-west-3.elb.amazonaws.com/get"
}
</pre>

The <b>-a</b> option is a shortcut for <b>--auth</b>

If you try to consumer the Route with invalid ClientId/ClientSecret pair you get a 401 error code:

<pre>
$ http a131cf848dbd44e45af06511412201fe-1032110377.eu-west-3.elb.amazonaws.com/httpbin/get -a r7lgArWHmm3E12elLTRDPVnBHAExruEg:6F3nCynbE6Hgs9cUSWfMYvE2_
HTTP/1.1 401 Unauthorized
Connection: keep-alive
Content-Length: 26
Content-Type: application/json; charset=utf-8
Date: Wed, 16 Feb 2022 11:17:10 GMT
Server: kong/2.7.1.1-enterprise-edition
WWW-Authenticate: Bearer realm="dev-xxxxxx.us.auth0.com"
X-Kong-Response-Latency: 642

{
    "message": "Unauthorized"
}
</pre>



To configure the integration with ((TODO: Your service name)), follow the steps below.

((TODO: Add steps as an ordered list for configuring your service))

## Set up Auth0

((TODO: Add steps as an ordered list for configuring Auth0 for your service))

## Results

((TODO: Explain what the customer should expect when the integration is configured and deployed))

## Troubleshooting

((TODO: Common issues or links to troubleshooting resources))
