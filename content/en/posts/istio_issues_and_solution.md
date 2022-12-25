---
title: "[Istio] Issues and solution/workaround"
date: 2022-12-19T20:13:41+07:00
description: Some issues have met and solution for those things
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

{{< alert theme="warning" dir="ltr" >}}
First, thank you for visiting my blog and read my article, I appreciate it. Secondly, this is also a place to keep my idea, my knowledge. At the same time, I also want to share these understandings, in a way it can help you or you can have another perspective on your problem.

Please read and see from many angles that it is suitable for your situation or not.
If you have any suggestion or you see my mistake in any posts, please help me by contacting me via Telegram, Email or LinkedIn. Thank you :heart:
{{< /alert >}}

### A brief of information about Istio
[Link]({{< ref "istio_summarize_briefly_information.md" >}})

### Issues and solution
#### I.Configuration

1. The config seem correct but the container(istio ingress/egress pod) cannot start the `port`
Check the Protocol http, https or tcp in `virtual service app` and in `istio (ops-tools)` define same protocol or not
Example:

Virtual service app
```
tcp: <-- HERE
- route:
- destination:
    host: example-service
    port:
        number: 2525
```
istio (ops-tools)
```
- name: istio-example-ingressgateway
  selector:
    service.istio.io/canonical-name: istio-example-ingressgateway
  ports:
    - protocol: TCP <-- HERE
      number: 25
      name: example
      hosts:
        - '"*"'
```

2. Istiod show some logs related to `BlackHoleCluster`, `duplicate listener`
since we're using `exportTo`, this can be reduce error but should be check again carefully if you meet this logs
```
[2022-03-30T04:51:00.701Z] "- - -" 0 UH - - "-" 0 0 7 - "-" "-" "-" "-" "-" BlackHoleCluster - 100.125.6.29:443 172.16.0.71:57754 - -
```
By default without exportTo, the egress rule will be applied for all namespaces (*mean you only define URL in an egress rule and other egress rules in another namespace can also use it*). Afterward you apply exportTo to all egress rules, then you have to explicit define the URL that you want to reach, here is the missing egress to *.googleapis.com after use exportTo
```
[2022-11-10T10:45:45.958Z] "- - -" 0 UH - - "-" 0 0 0 - "-" "-" "-" "-" "-" BlackHoleCluster - 199.36.153.9:443 100.70.45.178:48682 sqladmin.googleapis.com -
```

Or duplicate port 
```
warning	envoy config	gRPC config for type.googleapis.com/envoy.config.listener.v3.Listener rejected: Error adding/updating listener(s) 0.0.0.0_8081: duplicate listener 0.0.0.0_8081 found
```

3. Istio Ingress gateway show logs related to `503 NC cluster_not_found`
```
[2022-11-13T13:19:49.192Z] "GET / HTTP/1.1" 503 NC cluster_not_found - "-" 0 0 0 - "172.17.0.1" "Mozilla/5.0 (X11; Linux x86_64; rv:106.0) Gecko/20100101 Firefox/106.0" "d19be249-35c8-9dd4-95a4-c0dd5eb30edf" "rollouts-demo-vsvc1.local:32406" "-" - - 172.17.0.7:8080 172.17.0.1:42675 - primary
```
- Check the port mapping is correct to Service from Virtual Service
```
  http:
  - name: primary
    route:
    - destination:
        host: rollouts-demo-stable
        port:
          number: 80 # Have to be same as Service's port
      weight: 100
```

#### II. Ingress gateway
1. `Error OpenSSL SSL_connect: Connection reset by peer`
When you do a curl to and ingress endpoint if you see the error like this 
```
curl: (35) OpenSSL SSL_connect: Connection reset by peer in connection to app.service.example.com:443
```
So far we met this because the certificate for ingress gateway is not correct, so check in the secret which one the ingress gateway object is using correct or not. 
example the correct one: secret ingress-gw-tls
tls.crt
```
-----BEGIN CERTIFICATE-----
MIIC0DC
098koA==
-----END CERTIFICATE-----
```
tls.key
```
-----BEGIN PRIVATE KEY-----
MIIEvQ..........
HZAFWC..........
-----END PRIVATE KEY-----
```


#### II. Egress gateway
Be awared of changing/adding new listener port. When picking/choosing a new service port, It must be an unique port, otherwise the outging traffic won't work
```yaml
service_ports = [
...
{
"name"        = "https-ext",
"protocol"    = "TCP",
"port"        = 8080,
"target_port" = "8444"
}
...
]

deployment_ports = [
...
{
container_port = 8444, # for HTTPS traffic outgoing
protocol       = "TCP"
}
...
]

```

somehow the deployment_port couldn't listen even though we did run terraform apply. we have to restart the deployment manually. Ensure the newly listener is in LISTEN status by exec into egress-gateway pod
```shell
istio-proxy@istio-egressgateway-intportal-f65b8585c-lmtlq:/$ netstat -lptn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:15000         0.0.0.0:*               LISTEN      16/envoy            
tcp        0      0 0.0.0.0:8443            0.0.0.0:*               LISTEN      16/envoy            
tcp        0      0 0.0.0.0:8443            0.0.0.0:*               LISTEN      16/envoy            
tcp        0      0 0.0.0.0:8444            0.0.0.0:*               LISTEN      16/envoy            
tcp        0      0 0.0.0.0:8444            0.0.0.0:*               LISTEN      16/envoy            
tcp        0      0 127.0.0.1:15004         0.0.0.0:*               LISTEN      1/pilot-agent   

```


---

## Commands useful to troubleshoot issue
Some common commands are used for troubleshoot
###### Get an overview of your mesh: 
The proxy-status command allows you to get an overview of your mesh. If you suspect one of your sidecars isn’t receiving configuration or is out of sync then proxy-status will tell you this
cmd: `istioctl proxy-status`
Example:
```
NAME                                                                                    CDS        LDS        EDS        RDS          ISTIOD                                VERSION
example-app-frontend-api-798dfd579f-gpvwz.obs01-community-uat                SYNCED     SYNCED     SYNCED     SYNCED       istiod-asm-1106-2-ffb486458-8qmkl     1.10.6-asm.2
example-app-frontend-api-sidekiq-cff9887db-p9qnf.obs01-community-uat         SYNCED     SYNCED     SYNCED     SYNCED       istiod-asm-1106-2-ffb486458-8qmkl     1.10.6-asm.2
example-app-mgmt-api-6c67c84767-9mhbp.obs01-community-uat                    SYNCED     SYNCED     SYNCED     SYNCED       istiod-asm-1106-2-ffb486458-9tgl9     1.10.6-asm.2
example-app-mgmt-proxy-7bb685d7c4-mlbqn.obs01-community-uat                  SYNCED     SYNCED     SYNCED     SYNCED       istiod-asm-1106-2-ffb486458-8qmkl     1.10.6-asm.2
example-app-mgmt-worker-6465cb474-2hvjf.obs01-community-uat                  SYNCED     SYNCED     SYNCED     SYNCED       istiod-asm-1106-2-ffb486458-8qmkl     1.10.6-asm.2
example-app-mgmt-worker-6465cb474-zgh5b.obs01-community-uat                  SYNCED     SYNCED     SYNCED     SYNCED       istiod-asm-1106-2-ffb486458-8qmkl     1.10.6-asm.2
example-app-processing-intake-worker-859dd8d57cn2xml.obs01-community-uat     SYNCED     SYNCED     SYNCED     SYNCED       istiod-asm-1106-2-ffb486458-9tgl9     1.10.6-asm.2
```

###### Retrieve diffs between Envoy and Istiod
The command can also be used to retrieve a diff between the configuration Envoy has loaded and the configuration Istiod would send
cmd: `istioctl proxy-status <the_proxy_name>`
Example: 
`istioctl proxy-status example-app-frontend-api-798dfd579f-gpvwz.obs01-community-uat`
```
cloud@workstation-cce:~/SAAS_WORKSPACE/istio$ ./istioctl proxy-status example-app-frontend-api-798dfd579f-gpvwz.obs01-community-uat
Clusters Match
Listeners Match
Routes Match (RDS last loaded at Tue, 23 Aug 2022 07:49:34 UTC)
```

###### Deep dive into Envoy configuration
This can then be used to pinpoint any issues you are unable to detect by just looking through your Istio configuration and custom resources. To get a basic summary of clusters, listeners or routes for a given pod use the command as follows (changing `clusters` for `listeners` or `routes` when required):
cmd: `istioctl proxy-config cluster -n istio-system <pod_istio_gateway>`
Example:
`istioctl proxy-config cluster -n istio-system istio-egressgateway-7548f5f7c-5c42h`
```
cloud@workstation-cce:~/SAAS_WORKSPACE/istio$ istioctl proxy-config cluster -n istio-system istio-egressgateway-7548f5f7c-5c42h
SERVICE FQDN                                                                              PORT      SUBSET     DIRECTION     TYPE           DESTINATION RULE
4543447-sb1.suitetalk.api.netsuite.com                                                    443       -          outbound      STRICT_DNS     
BlackHoleCluster                                                                          -         -          -             STATIC         
accounting.uat.unifiedpost.fr                                                             443       -          outbound      STRICT_DNS     
accounts.google.com                                                                       443       -          outbound      STRICT_DNS     
agent                                                                                     -         -          -             STATIC         
api-eu.pusher.com                                                                         443       -          outbound      STRICT_DNS 
```
Example for query listener on a pod
`istioctl proxy-config listeners istio-egressgateway-7548f5f7c-5c42h`
```
cloud@workstation-cce:~/SAAS_WORKSPACE/istio$ ./istioctl proxy-config listeners istio-egressgateway-7548f5f7c-5c42h -n istio-system
ADDRESS PORT  MATCH                                                                                     DESTINATION
0.0.0.0 8443  SNI: www1.gsis.gr                                                                         Cluster: outbound|443||www1.gsis.gr
0.0.0.0 8443  SNI: www.test.peppolsmp.sg                                                                Cluster: outbound|443||www.test.peppolsmp.sg
0.0.0.0 8443  SNI: www.registeruz.sk                                                                    Cluster: outbound|443||www.registeruz.sk
0.0.0.0 8443  SNI: www.rabobank.nl                                                                      Cluster: outbound|443||www.rabobank.nl
0.0.0.0 8443  SNI: www.googletagmanager.com                                                             Cluster: outbound|443||www.googletagmanager.com
0.0.0.0 8443  SNI: www.googleapis.com                                                                   Cluster: outbound|443||www.googleapis.com
```

###### Verifying connectivity to Istiod
Verifying connectivity to Istiod is a useful troubleshooting step. Every proxy container in the service mesh should be able to communicate with Istiod. This can be accomplished in a few simple steps
suppose you are in some app pods and run this
`curl -sS <istio_service_name>.istio-system:15014/version`
Example: `curl -sS istiod-asm-1106-2.istio-system:15014/version`

