---
title: Some common step to double-check service still working inside Kubernetes
date: 2021-06-27T12:00:06+09:00
description: Checking services run inside Kubernetes
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Nhat Le
authorEmoji: 👽
tags:
- Kubernetes
categories:
- platform
series:
- Issues and Solution
image: images/post/doublecheck_service_inside_kubernetes/kubernetes-pod.png
---

{{< alert theme="warning" dir="ltr" >}}
First, thank you for visiting my blog and read my article, I appreciate it. Secondly, this is also a place to keep my idea, my knowledge. At the same time, I also want to share these understandings, in a way it can help you or you can have another perspective on your problem.

Please read and see from many angles that it is suitable for your situation or not.
If you have any suggestion or you see my mistake in any posts, please help me by contacting me via Telegram, Email or LinkedIn. Thank you :heart:
{{< /alert >}}

## Scenario

In Kubernetes, problems can occur in a different way, There are a few places to consider (from lower to upper):

- The application (by code)
- The pod (by configmap, secret)
- The service (mapping between pod and service unstable, service cannot see a pod,...)
- The ingress (mapping between service and ingress does not define, not config TLS certificate)
- The certificate (is expired)
- The nodes (not enough CPU, RAM to serve pods. Normally, the pod is evicted. Sometimes the node will be updating to maintenance by Microsoft)
- The cluster (the VM scale set is turned off)

---

### To know the pod is still working

Get logs from the pod by: `kubectl logs <pod_name>` (add -n <NAMESPACE> if the pod in different namespace)

- If the error from the application, get help from the developer
- Some error like `connection timeout, can't connect to Database,..etc`. 
  1. Try to connect to Database by `kubectl port-forward svc/<service_pod> port:port -n infrastructure`. Then use a tool such as Microsoft SQL     Server Management for SQL, robo3T for MongoDB,...etc. to connect to Database. To see the DB still work. For easier we can try `kubectl get pod/<mongodb> -n infrastructure` or `kubectl logs pod/<mongodb> -n infrastructure`
  2. If destination pod still working , next try to `ping` from a pod (any pod which has a ping command) to destination pod.
     - get IP of destination pod:
       - `kubectl get pod -n infrastructure -o wide`
           {{< img src="/images/post/doublecheck_service_inside_kubernetes/ip_pod.png" title="get pod's ip" caption="" alt="get pod's ip" width="700px" position="center" >}}
       - Exec to a service pod, any pod : `kubectl exec -it <pod_name> /bin/sh`
       - Then, run: `ping 10.244.0.xxx`, If the pod responds. Which mean the pod still reach to destination pod.
     - get IP of destination service (**_this way also test mapping between pod and service of the destination (because service will reoute traffic to pod)_**) **
       - `kubectl get svc -n infrastructure`, Then exec to a pod as above.
         Then try to curl, for example: `kubectl curl elasticsearch-master.infrastructure:9200` (remember to change port for each service)
       - **_NOTE_**: It depend on DB you using, for example you can use `curl` with elasticsearch but cannot for postgresql, mongodb,...
       - Create your own image contain `netcat, nslookup, dig, ping,...etc.` `**`

`**` You can build your own image which one contain any tool you want to troubleshoot. Then you can create a pod base on that image, after testing we can delete that pod, not effect anything

If the connection between pods is still Ok with manual check, So maybe the connection string in service pod is not correct.

---

### Check service still mapping pod

- `kubectl get svc`
- Inside a pod: `curl -I http://<service_name>` (the service name you get which above command). If you get `HTTP/1.1 200 OK`, the service still mapping with the pod.
- Or which `nc -vz <servicename>.<namespace> <port>` (namespace is optional). Look like this

    ``` command
    / # nc -v -z infra-postgresql
        infra-postgresql (10.108.164.176:0) open
    ```

    Reference netcat: <https://docs.rackspace.com/support/how-to/testing-network-services-with-netcat/>

- If notthing work try to delete the pod, **_let Kubernetes recreate that pod_**. Then try again.

---

### Check ingress still mapping service

- `kubectl get ingress` to see the ingress of service exists
- `kubectl get ingress/<ingress_name> -o yaml` to check that ingress is mapped with a service
- `curl -I https://<ingress_name>`, and check the status. If the status is any number except 404, which mean still map with service but the problem can be from the application
- Another reason is your certificate. If it expired, the ingress will no longer work as expected. You can check the certificate with some online tool <https://www.sslshopper.com/ssl-checker.html>, <https://www.ssllabs.com/ssltest/>. Or get logs from "service" as the example below

  {{< img src="/images/post/doublecheck_service_inside_kubernetes/accounts-cert-error.png" title="cert expired" caption="" alt="cert expired" width="700px" position="center" >}}

---

### Use Dashboard to have an overview

- az account set --subscription <subscription_id>
- az aks get-credentials --resource-group <resource_group_name> --name <subscription_name>
- az aks browse --resource-group <resource_group_name> --name <aks_name>
  
Another tool are octant (<https://octant.dev/>) , lens (<https://k8slens.dev/>)

---

### Error 502 from server

- Make sure the VM scale set (AKS) still run. In your side that can be GKE or EKS,...
- Go to Instances of VM scale set click refresh button **a few times** to sure that these nodes still Running, not Updating

  {{< img src="/images/post/doublecheck_service_inside_kubernetes/refresh_instance.png" title="refresh status of node" caption="" alt="refresh status of node" width="700px" position="center" >}}

- You also can run `kubectl get node` or `kubectl top node` to see if the node still running