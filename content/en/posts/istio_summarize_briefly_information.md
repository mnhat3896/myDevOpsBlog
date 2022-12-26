---
title: "[Istio] Summarize information briefly for re-reading"
date: 2022-12-25T11:13:41+07:00
description: Some knowledge/notes when using Istio
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Nhat Le
authorEmoji: 👽
tags:
- Kubernetes
- Service Mesh
categories:
- service mesh
series:
- Issues and Solution
image: images/post/istio_issues_and_solution/Istio.png
---

> :warning: **Note**: This page is for simple summary concepts, and quick note information about Istio for those who already know about Istio then we can easy to recall without re-reading the whole docs, if you need to read more completely, please visit the official [docs](https://istio.io/latest/docs/)

### What is service mesh?

from the [docs](https://istio.io/latest/about/service-mesh/)

> A service mesh is a dedicated infrastructure layer that you can add to your applications. It allows you to transparently add capabilities like observability, traffic management, and security, without adding them to your own code.

### What is Istio?
> Istio is an open source service mesh that layers transparently onto existing distributed applications (*Roughly, it will run a sidecar proxy along with the pod's app container*). Istio’s powerful features provide a uniform and more efficient way to secure, connect, and monitor services

### How it Works
`Istio has two components: the data plane and the control plane.` [Architecture](https://istio.io/latest/docs/ops/deployment/architecture/)
- `The data plane` is the communication between services
- `The control plane` takes your desired configuration, and its view of the services, and dynamically programs the proxy servers, updating them as the rules or the environment changes

{{< img src="/images/post/istio_issues_and_solution/service-mesh-how-it-works.png" title="after install istio" caption="" alt="istio flow" width="700px" position="center" >}}


### Components
[docs](https://istio.io/latest/docs/ops/deployment/architecture/#components)
#### I.Envoy
>Istio uses an extended version of the Envoy proxy. `Envoy proxies are the only Istio components that interact with data plane traffic`.

Envoy proxies are deployed as sidecars to services, logically augmenting the services with Envoy’s many built-in features, for example:
* Dynamic service discovery
* Load balancing
* TLS termination
* HTTP/2 and gRPC proxies
* Circuit breakers
* Health checks
* Staged rollouts with %-based traffic split
* Fault injection
* Rich metrics

#### II.Istiod

> Istiod provides service discovery, configuration and certificate management.

Istiod converts high level routing rules that control traffic behavior into Envoy-specific configurations, and propagates them to the sidecars at runtime. Pilot abstracts platform-specific service discovery mechanisms and synthesizes them into a standard format that any sidecar conforming with the Envoy API can consume.

### Concepts

#### I.Traffic Management
[docs](https://istio.io/latest/docs/concepts/traffic-management/)

In order to direct traffic within your mesh, Istio needs to know where all your endpoints are, and which services they belong to. To populate its own service registry, Istio connects to a service discovery system. For example, if you’ve installed Istio on a Kubernetes cluster, then Istio automatically detects the services and endpoints in that cluster.

resources need to note:
- Virtual services
- Destination rules
- Gateways
- Service entries
- Sidecars

1. Virtual services
> Virtual services, along with destination rules, are the key building blocks of Istio’s traffic routing functionality. A virtual service lets you configure how requests are routed to a service within an Istio service mesh

- Virtual services (The hosts field): [ref](https://istio.io/latest/docs/concepts/traffic-management/#the-hosts-field) 
    * Virtual service hosts don't actually have to be part of the Istio service registry, they are simply virtual destinations. This lets you model traffic for virtual hosts that don't have routable entries inside the mesh.
        ```
        hosts:
        - reviews
        ```
    * The virtual service hostname can be an IP address, a DNS name, or, depending on the platform, a short name (such as a Kubernetes service short name) 
2. Destination rule: [ref](https://istio.io/latest/docs/concepts/traffic-management/#destination-rules)
    * The route section's destination field specifies the actual destination for traffic that matches this condition. Unlike the virtual service's host(s), `the destination's host must be a real destination that exists in Istio's service registry` or Envoy won't know where to send traffic to it. `This can be a mesh service with proxies or a non-mesh service added using a service entry`.
    * When this rule is evaluated, Istio adds a domain suffix based on the namespace of the virtual service that contains the routing rule to get the fully qualified name for the host
    ```
    route:
    - destination:
        host: reviews
    Using short names only works if the destination hosts and the virtual service are actually in the same Kubernetes namespace.
    ```
    You can think of virtual services as how you route your traffic to a given destination, and then you use destination rules to configure what happens to traffic for that destination. `Destination rules are applied after virtual service routing rules are evaluated`, so they apply to the traffic’s “real” destination.

    You use destination rules to specify named service subsets, such as grouping all a given service’s instances by version. You can then use these service subsets in the routing rules of virtual services to control the traffic to different instances of your services.

    ``` You use destination rules to configure what happens to traffic for that destination ```
    Destination rules also let you customize Envoy’s traffic policies when calling the entire destination service or a particular service subset, such as your preferred load balancing model, `TLS security mode`, or circuit breaker settings
    
    Load balancing options:
    - Random: Requests are forwarded at random to instances in the pool.
    - Weighted: Requests are forwarded to instances in the pool according to a specific percentage.
    - Least requests: Requests are forwarded to instances with the least number of requests.

3. Gateway [ref](https://istio.io/latest/docs/concepts/traffic-management/#gateways):
    
    `Gateway configurations are applied to standalone Envoy proxies that are running at the edge of the mesh`, rather than sidecar Envoy proxies running alongside your service workloads.

    Gateways are primarily used to manage ingress traffic, but you can also configure egress gateways. An egress gateway lets you configure a dedicated exit node for the traffic leaving the mesh

4. Service entries [ref](https://istio.io/latest/docs/concepts/traffic-management/#service-entries):

    `You use a service entry to add an entry to the service registry that Istio maintains internally`. After you add the service entry, the Envoy proxies can send traffic to the service as if it was a service in your mesh. Configuring service entries allows you to manage traffic for services running outside of the mesh, including the following tasks:
    * Redirect and forward traffic for external destinations, such as APIs consumed from the web, or traffic to services in legacy infrastructure.
    * Define retry, timeout, and fault injection policies for external destinations.
    * Run a mesh service in a Virtual Machine (VM) by adding VMs to your mesh.

5. Sidecars [ref](https://istio.io/latest/docs/concepts/traffic-management/#sidecars):

    `By default, Istio configures every Envoy proxy to accept traffic on all the ports of its associated workload, and to reach every workload in the mesh when forwarding traffic`. You can use a sidecar configuration to do the following:

    * Fine-tune the set of ports and protocols that an Envoy proxy accepts.
    * Limit the set of services that the Envoy proxy can reach.
    ---
    You might want to limit sidecar reachability like this in larger applications, `where having every proxy configured to reach every other service in the mesh can potentially affect mesh performance due to high memory usage.`

    You can specify that you want a sidecar configuration to apply to all workloads in a particular namespace, or choose specific workloads using a workloadSelector. For example, the following sidecar configuration configures all services in the bookinfo namespace to only reach services running in the same namespace and the Istio control plane (needed by Istio’s egress and telemetry features):
    ```
    apiVersion: networking.istio.io/v1alpha3
    kind: Sidecar
    metadata:
    name: default
    namespace: bookinfo
    spec:
    egress:
    - hosts:
        - "./*"
        - "istio-system/*"
    ```
#### II.Security
[docs](https://istio.io/latest/docs/concepts/security/)

The Istio security features provide strong identity, powerful policy, transparent TLS encryption, and authentication, authorization and audit (AAA) tools to protect your services and data. The goals of Istio security are:

- Security by default: no changes needed to application code and infrastructure
- Defense in depth: integrate with existing security systems to provide multiple layers of defense
- Zero-trust network: build security solutions on distrusted networks

The control plane handles configuration from the API server and configures the [PEPs](https://www.oreilly.com/library/view/network-access-control/9780470398678/9780470398678_policy_enforcement_point.html) in the data plane. The PEPs are implemented using Envoy. The following diagram shows the architecture.

{{< img src="/images/post/istio_summarize_briefly_information/security_architecture.png" title="Security Architecture" caption="" alt="Security Architecture" width="700px" position="center" >}}

##### 1. Istio identity
Identity is a fundamental concept of any security infrastructure. At the beginning of a workload-to-workload communication, the two parties must exchange credentials with their identity information for mutual authentication purposes

On the client side, the server’s identity is checked against the [secure naming](https://istio.io/latest/docs/concepts/security/#secure-naming) information to see if it is an authorized runner of the workload. On the server side, the server can determine what information the client can access based on the [authorization policies](https://istio.io/latest/docs/concepts/security/#authorization-policies

The Istio identity model uses the first-class `service identity` to determine the identity of a request’s origin
The following list shows examples of service identities that you can use on different platforms:
- <mark>Kubernetes: Kubernetes service account</mark>
- GCE: GCP service account
- On-premises (non-Kubernetes): user account, custom service account, service name, Istio service account, or GCP service account. The custom service account refers to the existing service account just like the identities that the customer’s Identity Directory manages.)


##### 2. Identity and certificate management

Istio agents, running alongside each Envoy proxy, work together with istiod to automate key and certificate rotation at scale

{{< img src="/images/post/istio_summarize_briefly_information/identity_provisioning_workflow.png" title="Security Architecture" caption="" alt="Security Architecture" width="700px" position="center" >}}

Istio provisions keys and certificates through the following flow:
  1. istiod offers a gRPC service to take certificate signing requests (CSRs).
  2. When started, the Istio agent(container init) creates the private key and CSR, and then sends the CSR with its credentials to istiod for signing.
  3. The CA in istiod validates the credentials carried in the CSR. Upon successful validation, it signs the CSR to generate the certificate.
  4. When a workload is started, Envoy requests the certificate and key from the Istio agent in the same container via the Envoy secret discovery service (SDS) API.
  5. The Istio agent sends the certificates received from istiod and the private key to Envoy via the Envoy SDS API.
  6. Istio agent monitors the expiration of the workload certificate. The above process repeats periodically for certificate and key rotation.

##### 3. Authentication

- Security in Istio involves multiple components:
  - A Certificate Authority (CA) for key and certificate management
  - The configuration API server distributes to the proxies:
    - authentication policies
    - authorization policies
    - secure naming information
  - Sidecar and perimeter proxies work as Policy Enforcement Points [PEPs](https://www.jerichosystems.com/technology/glossaryterms/policy_enforcement_point.html) to secure communication between clients and servers.
  - A set of Envoy proxy extensions to manage telemetry and auditing

- Istio provides two types of authentication:
  - [Peer authentication](https://istio.io/latest/docs/concepts/security/#permissive-mode): used for service-to-service authentication to verify the client making the connection.
    - Istio mutual TLS has a permissive mode, which allows a service to accept both plaintext traffic and mutual TLS traffic at the same time. This feature greatly improves the mutual TLS onboarding experience(migrate that server to Istio with mutual TLS enabled.).
    - If authentication policies disable mutual TLS mode, Istio continues to use plain text between PEPs.
    - [Peer policy](https://istio.io/latest/docs/concepts/security/#peer-authentication)

  - Request authentication: Used for end-user authentication to verify the credential attached to the request

- When a workload sends a request to another workload using mutual TLS authentication, the request is handled as follows `(Mutual TLS authentication)`:
  1. Istio re-routes the outbound traffic from a client to the client’s local sidecar Envoy.
  2. The client side Envoy starts a mutual TLS handshake with the server side Envoy. During the handshake, the client side Envoy also does a secure naming check to verify that the service account presented in the server certificate is authorized to run the target service.
  3. The client side Envoy and the server side Envoy establish a mutual TLS connection, and Istio forwards the traffic from the client side Envoy to the server side Envoy.
  4. The server side Envoy authorizes the request. If authorized, it forwards the traffic to the backend service through local TCP connections.


##### 3.1 Authentication policies
Authentication policies apply to requests that a service receives. `To specify client-side authentication` rules in mutual TLS, you need to specify the TLSSettings in the `DestinationRule`. [TLS settings reference docs](https://istio.io/latest/docs/reference/config/networking/destination-rule/#ClientTLSSettings)

- Policy storage
  <mark>Istio stores mesh-scope policies in the root namespace. These policies have an empty selector apply to all workloads in the mesh</mark>. Policies that have a namespace scope are stored in the corresponding namespace. They only apply to workloads within their namespace. If you configure a selector field, the authentication policy only applies to workloads matching the conditions you configured.
- Selector field
  Peer and request authentication policies use selector fields to specify the label of the workloads to which the policy applies. The following example shows the selector field of a policy that applies to workloads with the app:product-page label:
  ```yaml
  selector:
    matchLabels:
      app: product-page
  ```
  `If you don’t provide a value for the selector field, Istio matches the policy to all workloads in the **storage scope** of the policy.` Thus, the selector fields help you specify the scope of the policies:
  - Mesh-wide policy: A policy specified for the root namespace without or with an empty selector field.
  - Namespace-wide policy: A policy specified for a non-root namespace without or with an empty selector field.
  - Workload-specific policy: a policy defined in the regular namespace, with non-empty selector field.

  For the policy order apply, check the [docs](https://istio.io/latest/docs/concepts/security/#selector-field)