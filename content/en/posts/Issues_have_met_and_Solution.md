---
title: Azure Kubernetes Service - Issues have met and the solution
date: 2021-05-10T12:00:06+09:00
description: This post just used to note all issues that I have met and my solution
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Nhat Le
authorEmoji: 👽
tags:
- Kubernetes
- cloud
- Azure Kubernetes Service
- Azure
categories:
- Issues and Solution
series:
- Issues and Solution
image: images/post/Issues_have_met_and_Solution/problems-solutions.png
---

{{< alert theme="warning" dir="ltr" >}}
First, thank you for visiting my blog and read my article, I appreciate it. Secondly, this is also a place to keep my idea, my knowledge. At the same time, I also want to share these understandings, in a way it can help you or you can have another perspective on your problem.

Please read and see from many angles that it is suitable for your situation or not.
If you have any suggestion or you see my mistake in any posts, please help me by contacting me via Telegram, Email or LinkedIn. Thank you :heart:
{{< /alert >}}

{{< alert theme="danger" dir="ltr" >}}
This post just used to note all issues that I have met and my solution related to Kubernetes, Azure Kubernetes Service, Helm3, Azure DevOps or Azure. Maybe you can find your solution here. __*This post will be updated by the time*__ :blush:
{{< /alert >}}

## [Mongodb] Restore data in Mongodb:  

{{< img src="/images/post/Issues_have_met_and_Solution/mongodb_restore.png" title="mongodb restore" caption="" alt="mongodb_restore" width="700px" position="center" >}}

Solution:  

* Exec to pod:  kubectl exec -it <POD_NAME> bash  
* Then u are inside the pod and run:  mongo -u <USERNAME> -p <Password>  
* Next, run:  rs.initiate()

---

## [Azure Resource] 1 Insufficient memory, 1 node(s) exceed max volume count.

When deploying resource infrastructure The reason is the limit of the disks can be attached to the node. For example, the size of node_1 is standard_b2s, which can only attach a maximum of 4 disks to the node. So when u deploy a new infra resource to node_1 which has already had 4 disks attach to. Then the error will show 1 node(s) exceeds max volume count

Check the capacity of the disk can attach to the node:

{{< img src="/images/post/Issues_have_met_and_Solution/disk_attach.png" title="disk attach" caption="" alt="disk_attach" width="700px" position="center" >}}

Solution:

* Go Virtual machine scale set on Azure
* **_Resize_** the VMs (The size have more Data disks) **_or increase_** the number of node

---

## [Release] "<release_name>" has no deployed releases

Solution:

* Just need to uninstall that chart with the command "helm uninstall <chart_name>" and release again (Even though when using `helm list`, the chart not show)
* Maybe there is a mismatch between Kubernetes and Helm. when deploying the chart but you immediately interrupted it

---

## [Helm chart] Install another version MongoDB chart

The scenario is you are using version 4 but install version 5 or any version different from the first

Solution

* It can not work as expected. Have to uninstall MongoDB chart, delete PVC. (back up the database before do this). Then create a new PVC and MongoDB with the new version

---

## [Elasticsearch] Unable to start pod after restoring data (Elasticsearch pod not green)

There are more errors show on logs, but the root core is java.lang.OutOfMemoryError: Java heap space

Solution

* Increate JVM heap to 512, or more (as the image below) in the configuration file, if you use helm3, the config in elasticsearch-values.yaml

{{< img src="/images/post/Issues_have_met_and_Solution/java-heap.png" title="Java heap space" caption="" alt="Java heap space" width="700px" position="center" >}}

---

## [Nginx-Ingress] Can not upload file (limit size upload)

Solution

* Add `nginx.ingress.kubernetes.io/proxy-body-size: 6m` to the values.yaml file (this will create an annotation to service's ingress)

for example, add this line to candidate-app-service:  

{{< img src="/images/post/Issues_have_met_and_Solution/increase-body.png" title="increase body size nginx" caption="" alt="increase body size nginx" width="700px" position="center" >}}

---

## [Helm] [Local machine] The system cannot find the file specified stable-index.yaml

Somehow the file `stabel-index.yaml` has been deleted, you can check at path, for example on window:  
C:\Users\xxx\AppData\Local\Temp\helm\repository (the Helm3 path you have install in your machine)

Solution (run on your machine):

* Run: `helm repo add stable https://charts.helm.sh/stable`
* This script add repo "stable" to the path

---

## [Azure DevOps Pipeline] helm task deploy timeout

Somehow there is a problem with Helm3 task deploy option –wait, it not actually waiting to deploy the chart successfully.

**Besides that, If we implement a health Probe for pods, a pod such as `A-client` take time to let the "health check" to green (more than ~10 minutes). this also makes the "helm3 task" wait ~10 minutes, this led to a timeout error. We can increase the timeout in a "task", but seem like it not work as expected**

Solution:

* To workaround, uncheck option –wait on helm deploy task. Then we have to go on Kubernetes dashboard to manual check there is any problem with the pod or use scrip (Kubernetes get pod -w)

NOTE: Will update scenarior and solution later

---

## [Certificate] The certificate is expired (Logs from a service, ex: Identity service)

{{< img src="/images/post/Issues_have_met_and_Solution/accounts-cert-error.png" title="Identity service certificate" caption="" alt="Identity service certificate" width="700px" position="center" >}}

Explanation:  

* The certificate file in Identity pod no longer valid so you need update that file
* The way to put the file to a pod is using "secret" or "configmap". Basically the idea is create a secret file, then "mount" that secret to the path inside the pod. Take a look at this in values.yaml of accounts chart:

```yaml
volumeMounts:
  - name: cert
    mountPath: /tmp/cert
    readOnly: true
volumes:
  - name: cert
    secret:
      secretName: identityserver-certificate
```

Solution:

* Remove the old certificate: `kubectl get secret` and find the secret name "identityserver-certificate" or any your secret name and delete
* `cd` to where you place the certificate on your machine, 
* (**change certificate.pfx to the file name you have**) next run  `kubectl create secret generic identityserver-certificate --from-file=./certificate.pfx`.
* Next, update the password in the Identity service (if your cert have password)

---

## [Certificate] Unable to retrieve document from: xxxx/.well-known/openid configuration

_The cert is still valid but maybe it lacks Chain Cert(CA)._  

{{< img src="/images/post/Issues_have_met_and_Solution/openid_config.png" title="Error from Identity service" caption="" alt="Error from Identity service" width="700px" position="center" >}}

Solution:  

* Check that the certificate is still valid with https://www.sslshopper.com/ssl-checker.html or https://www.ssllabs.com/ssltest/
* The result shows that lacking "Chain Cert" (or rank B). The result get from sslshopper (image below):
  
  {{< img src="/images/post/Issues_have_met_and_Solution/openid_lack_ca.png" title="Check CA domain" caption="" alt="Check CA domain" width="700px" position="center" >}}

* Use https://whatsmychaincert.com/ to generate the Chain Cert, give it our domain and it gives us the Chain Cert.
  Look at the file you has already downloaded (OPEN WITH NOTEPAD), copy that string (only CA string) and add
  to your tls.crt (in k8s secret) (ex: tls.cert in bravotalents-tls-certificate)

{{< alert theme="danger" dir="ltr" >}}
I'm not recommend this solution, you can find your CA with another way.
{{< /alert >}}

---

## [Logging configuration] Cannot scrape logs from some services

* Make sure that all service has already saved logs on the node. 
* You can check by creating a pod, then mount the log's path on the node to that pod.
* Then you can exec to the pod has created, and check is there already have all services logs  
  Example a pod (take a look at `hostPath` and `mountPath`):

  {{< img src="/images/post/Issues_have_met_and_Solution/pod_mount.png" title="pod volume" caption="" alt="pod volume" width="700px" position="center" >}}
* If everything is ok, so the problem maybe comes from the Filebeat service or the format of logs that Filebeat doesn't understand
  * Go to setting of service such as `appsetting.json` (.NET) and remove this line
    `"UseElasticsearchLogFormatter": false` or set to `true`
  * Check that the setting doesn't change the output of logs or turn off the format.

---

## [Application] Service cannot serve traffic

The scenario is when exec to the pod, run command `curl localhost:80` it doesn't response anything. event run command `netstat -apnt` still return the service listen on port 80

NOTE: Will update scenarior and solution later