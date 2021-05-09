---
title: Backup database run inside Azure Kubernetes Service with Kubernetes cronjob
date: 2021-04-15T12:00:06+09:00
description: The article to introduce how to use cronjob to back up database (statefulset) inside Azure Kubernetes Service (Kubernetes)
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Nhat Le
authorEmoji: 👽
tags:
- Kubernetes
- backup
- database
- cloud
- Azure Kubernetes Service
- Azure
categories:
- platform
series:
- Azure Kubernetes Service
- Kubernetes
- Azure cloud
image: images/post/backup_DB_inside_AKS_with_cronjob/bk-database-icon.png
---

{{< alert theme="warning" dir="ltr" >}}
First, thank you for visiting my blog and read my article, I appreciate it. Secondly, this is also a place to keep my idea, my knowledge. At the same time, I also want to share these understandings, in a way it can help you or you can have another perspective on your problem.

Please read and see from many angles that it is suitable for your situation or not.
If you have any suggestion or you see my mistake in any posts, please help me by contacting me via Telegram, Email or LinkedIn. Thank you :heart:
{{< /alert >}}

*This article will show you how to back up a database run inside a cluster (Kubernetes) without a dependency on the service of cloud. If you want to have a centralized place to view all your snapshot, maintain without scripting, you can check this post [Backup database run inside Azure Kubernetes Service with Azure Backup vault](https://dev2ops.net/en/posts/backup_db_inside_aks_with_az_backup/)*

Here i use Azure kubernetes service (a service of Azure cloud). But you can use this article to back up any kind of Kubernetes approach. Not depend on service of Azure, so you can use this article on Amazon Elastic Kubernetes Service or Google Kubernetes Engine and so on. I haven't tested on another cloud yet, but i think we can. You can try and tell me if it works :smile:

So why i have to use cronjob? For creating periodic and recurring tasks.

> A CronJob creates Jobs on a repeating schedule.
>
>One CronJob object is like one line of a crontab (cron table) file. It runs a job periodically on a given schedule, written in Cron format.

Reference: <https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/>

### NOTE

I will use acronym.  

* SC is Storage Class
* PVC is Persistent Volume Claim
* SA is Azure Storage Account
* ACR is Azure Container Registry

## Scenario

* You are using kubernetes, here is Azure Kubernetes Service (AKS)
* Deploy database, such as mongodb, elasticsearch,...
* __Containerized applications running on Azure Kubernetes Service (AKS cluster) are using managed disks as persistent storage__
* You need to back up data but you don't want to depend on service back up of cloud

## Idea to back up

### For [Mongodb]

  1. Create a cronjob run every midnight.
  2. This cronjob will run a script to back up. (you will see these script in file yaml below)
  3. Then mount these data back up to Azure file share (for easy download or mount back to pod for restore)

### For [Elasticsearch]

  1. Custom Dockerfile to register repo for elasticsearch
  2. Build the image and push to ACR
  3. Create SC -> Create PVC
  4. Create a secret have ACR authentication
  5. Create a cronjob to run every midnight
  6. Cronjob run a script to back up data to a path inside cronjob, this path will mount to Azure file share

## [Mongodb]

* With mongodb you need to create SC, then create PVC base on SC.  
  *So, for what?*.  
  When you start a cronjob, you create a path inside it, this path used for save data back up. this path also mount to Azure file share. To do this, you create a custom SC base on your Azure Storage Account name, then create PVC.

  Create a file `sc.yaml` and run command: `kubectl apply -f sc.yaml`

  ```yaml
    kind: StorageClass
    apiVersion: storage.k8s.io/v1
    metadata:
      name: azurefile-backup-xxxx # NOTE: CHANGE THIS VALUE AS YOUR NEED
    provisioner: kubernetes.io/azure-file
    mountOptions:
      - dir_mode=0777
      - file_mode=0777
      - uid=1001
      - gid=1001
    reclaimPolicy: Delete
    parameters:
      skuName: Standard_LRS
      storageAccount: # NOTE: CHANGE THIS VALUE AS YOUR NEED (Your storage account name on azure)
  ```

* Create PVC base on Storage Account (SC)

  Create a file `pvc.yaml` and run command: `kubectl apply -f pvc.yaml`  
  
  Let see the yaml file. Do you see `namespace: infrastructure`. One is you run your cronjob in the same namespace your database run, second, you don't want running in the same namespace, then you have to define namespace after your service url. `ex: mongodb.infrastructure` (see option `--host=mongodb` in the `cronjob.yaml` below to understand)

  ```yaml
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: pvc-azure-fileshare-mongodb # NOTE: CHANGE THIS VALUE AS YOUR NEED
    namespace: infrastructure # NOTE: CHANGE THIS VALUE AS YOUR NEED
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 30Gi # NOTE: CHANGE THIS VALUE AS YOUR NEED (how many you need)
    storageClassName: azurefile-backup-xxxx # NOTE: CHANGE THIS VALUE AS YOUR NEED (this is your SC you have created above)
  ```

* Create cronjob

  Command: `kubectl apply -f cronjob.yaml`

  Take a look at `args:` you need to change your database name, username, password in your scenario. You also see that the PVC name i have create above.

  ```yaml
  apiVersion: batch/v1beta1
  kind: CronJob
  metadata:
    name: mongodb-cronjob
  spec:
    schedule: "0 23 * * 0-6"
    successfulJobsHistoryLimit: 1
    failedJobsHistoryLimit: 1
    # "concurrencyPolicy" should be set to "Forbid"
    # otherwise "successfulJobsHistoryLimit" and "failedJobsHistoryLimit" are meaningless
    # use "concurrencyPolicy: Replace" if you don't need to retain job history
    concurrencyPolicy: Forbid
    jobTemplate:
      spec:
        # terminate a Job
        # takes precedence over its .spec.backoffLimit
        # a Job that is retrying one or more failed Pods will not deploy additional Pods \
        # once it reaches the time limit specified by activeDeadlineSeconds, even if the backoffLimit is not yet reached.
        # activeDeadlineSeconds: 600
        backoffLimit: 2
        template:
          spec:
            # using "restartPolicy: Never" ensures failed pods are around to check logs, etc..
            # job controller will start new pods to replace failed ones
            restartPolicy: Never
            containers:
              - name: mongodb
                image: mongo:4.2
                imagePullPolicy: IfNotPresent
                volumeMounts:
                  - mountPath: "/var/backup"
                    name: mount-to-azure-file
                args:
                    - /bin/sh
                    - -c
                    - cd /var/backup; date +"%d-%m-%y" | xargs -I% mkdir %; folderbyday=$(date +"%d-%m-%y");
                      mongodump --host=mongodb --port=27017 --username= --password="" --out="/var/backup/$folderbyday"  --authenticationDatabase=admin -d <DBNAME> &&
                      mongodump --host=mongodb --port=27017 --username= --password="" --out="/var/backup/$folderbyday"  --authenticationDatabase=admin -d <DBNAME>
                      echo "======done dump======"
            volumes:
              - name: mount-to-azure-file
                persistentVolumeClaim:
                  claimName: pvc-azure-fileshare-mongodb

  ```

* Get cronjob you have deploy: `kubectl get cronjob -n <namespace>` (you can skip -n if you don't deploy on any namespace)

Example:
| NAME     | SCHEDULE  | SUSPEND | ACTIVE    | LAST SCHEDULE | AGE|
| ---------- | --------- | ----------------- | ---------- | ---------- | ---------- |
| elasticsearch-cronjob  |  0 23 * * 0-6  | False | 0     | 15h | 67d |
| mongodb-cronjob  |  0 23 * * 0-6  | False | 0     | 15h | 25d|

* For testing, you can also trigger cronjob manually: `kubectl create job --from=cronjob/<cronjob_name> <job_name> -n <namespace>` (you can skip -n if you don't deploy on any namespace)

* Cronjob will create a job, this job will run a pod to excute your script. You now can get pod and watch logs of that pod. Pod name is something like this: `mongodb-cronjob-1619478000-8j82h`

---

## [Elasticsearch]

* Build Dockerfile

  Command: `docker build -t elasticsearch-oss:v1 -f Dockerfile --build-arg elastic_version=6.7.8 .`

  ```Dockerfile
  ARG elastic_version
  FROM docker.elastic.co/elasticsearch/elasticsearch-oss:${elastic_version}

  # RUN bin/elasticsearch-plugin install --batch repository-azure

  USER root
  RUN mkdir -p /usr/share/elasticsearch/snapshot
  RUN chown elasticsearch:elasticsearch /usr/share/elasticsearch/snapshot
  RUN chmod 777 /usr/share/elasticsearch/snapshot

  ENTRYPOINT ["docker-entrypoint.sh"]
  ```

  docker-entrypoint

  ```docker-entrypoint.sh
    #!/bin/sh
    #register repo for elasticsearch on local
    curl -XPUT http://localhost:9200/_snapshot/snapshot --header "Content-Type: application/json" --data '{"type": "fs", "settings": { "location": "/usr/share/elasticsearch/snapshot", "chunk_size": "32MB", "compress": true } }'

  ```

* If you have already created SC above, then you don't need to create again. Because this approach still uses the same Storage Account but the different folder in Azure file share

* Create PVC base on Storage Account (sc) to store data to azure file share

  Command: `kubectl apply -f pvc.yaml`

  ``` yaml
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: pvc-azure-fileshare-elasticsearch
      namespace: infrastructure
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 20Gi
      storageClassName: # azurefile-backup-xxxx # NOTE: CHANGE THIS VALUE
  ```

* Create a secret to let AKS have permission to pull image from your ACR

  You need to change these arguments to your ACR authentication or your docker hub account:

  * --docker-server
  * --docker-username
  * --docker-password
  * --docker-email

  ```bash
    kubectl create secret docker-registry regcred --docker-server= --docker-username= --docker-password= --docker-email=
  ```

* After you build your own image, you can push to the azure container registry to store your image or docker hub. Then create a `elastic-values.yaml` as below

  Command: `kubectl apply -f elastic-values.yaml`

  Take a look at
  * path.repo ( set repo path which one you have mkdir in Dockerfile )
  * imagePullSecrets ( if your image is public. then you can skip this property )
  * extraVolumeMounts ( also mounth the repo path to azure file share )
  * extraVolumes

  ```yaml
  imageTag: "6.8.7-v2"
  image: "<your_ACR>/<image>"
  # ================
  replicas: 3
  minimumMasterNodes: 2

  # Shrink default JVM heap.
  esJavaOpts: "-Xmx512m -Xms512m"

  esConfig:
    elasticsearch.yml: |
      http.cors.enabled: true
      http.cors.allow-origin: /https?:\/\/localhost(:[0-9]+)?/
      http.cors.allow-headers: X-Requested-With,X-Auth-Token,Content-Type,Content-Length,Authorization
      http.cors.allow-credentials: true
      path.repo: ['/usr/share/elasticsearch/snapshot']

  # Allocate smaller chunks of memory per pod.
  resources:
    requests:
      cpu: 100m
      memory: 100Mi
    limits:
      cpu: 1000m
      memory: 1Gi

  volumeClaimTemplate:
    accessModes: [ "ReadWriteOnce" ]
    resources:
      requests:
        storage: 100Gi
    storageClassName: "managed-premium"

  #clusterHealthCheckParams: "wait_for_status=yellow&timeout=1s"
  esMajorVersion: 6
  imagePullSecrets:
    - name: regcred

  extraVolumeMounts:
    - name: mount-to-azure-file
      mountPath: "/usr/share/elasticsearch/snapshot"
      readOnly: false

  extraVolumes:
    - name: mount-to-azure-file
      persistentVolumeClaim:
        claimName: pvc-azure-fileshare-elasticsearch
  ```

* Create cronjob

  Because the script still run inside the cronjob, __*BUT WILL BE EXECUTE*__ inside elasticsearch (not the cronjob). So you don't need to mount the path of cronjob to azure file. This have been done in `elastic-values.yaml`

  ```yaml 
  apiVersion: batch/v1beta1
  kind: CronJob
  metadata:
    name: elasticsearch-cronjob
    namespace: infrastructure
  spec:
    schedule: "0 23 * * 0-6"
    successfulJobsHistoryLimit: 1
    failedJobsHistoryLimit: 1
    # "concurrencyPolicy" should be set to "Forbid"
    # otherwise "successfulJobsHistoryLimit" and "failedJobsHistoryLimit" are meaningless
    # use "concurrencyPolicy: Replace" if you don't need to retain job history
    concurrencyPolicy: Forbid
    jobTemplate:
      spec:
        # terminate a Job
        # takes precedence over its .spec.backoffLimit
        # a Job that is retrying one or more failed Pods will not deploy additional Pods \
        # once it reaches the time limit specified by activeDeadlineSeconds, even if the backoffLimit is not yet reached.
        # activeDeadlineSeconds: 600
        backoffLimit: 2
        template:
          spec:
            # using "restartPolicy: Never" ensures failed pods are around to check logs, etc..
            # job controller will start new pods to replace failed ones
            restartPolicy: Never
            containers:
            - name: elasticsearch-dump
              image: curlimages/curl:7.71.1
              # Dont need to mount here, have to mount in elasticsearch pod
              # volumeMounts:
              #   - mountPath: "/usr/share/elasticsearch/snapshot"
              #     name: mount-to-azure-file
              args:
                - /bin/sh
                - -c
                - date +"%d-%m-%y" | xargs -I% curl -XPUT "elasticsearch-master:9200/_snapshot/snapshot/snapshot-%?wait_for_completion=true"
            # volumes:
            #   - name: mount-to-azure-file
            #     persistentVolumeClaim:
            #       claimName: pvc-azure-fileshare-elasticsearch
  ```

## Conclusion

* Pros: With this approach, you are not depend on service back up of cloud, because it use the native resource run inside AKS.
* Cons: The tradeoff is difficulty managing the snapshot, the back up data. For example: if you want to delete the old back up data.

With MongoDB, I have created a folder to let back up by day so we can easily delete the folder by day, but with Elasticsearch it is a different story. 

For Elasticsearch, have to use port-forward then use a tool to support to get list, edit, delete such as ElasticSearch Head (extension chrome)

So each solution will have pros and cons, depend on your scenario to apply the solution. 

If you want to easier, use the resource that exists on Azure, let take a look on my post "How to backup database run inside Azure Kubernetes Service with Azure Backup"

__*Thank you for reading my article. If you have any questions or suggestions, please Email or chat with me via Telegram or LinkedIn. (You can find my information at the footer or at writers information)*__
