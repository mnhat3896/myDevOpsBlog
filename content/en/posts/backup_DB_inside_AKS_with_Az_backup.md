---
title: Backup database run inside Azure Kubernetes Service with Azure Backup vault
date: 2021-05-08T12:00:06+09:00
description: The article to introduce about Azure backup vault to backup database
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
image: images/post/backup_DB_inside_AKS_with_Az_backup/Azure-Backup.png
---

{{< alert theme="warning" dir="ltr" >}}
First, thank you for visiting my blog and read my article, I appreciate it. Secondly, this is also a place to keep my idea, my knowledge. At the same time, I also want to share these understandings, in a way it can help you or you can have another perspective on your problem.

Please read and see from many angles that it is suitable for your situation or not.
If you have any suggestion or you see my mistake in any posts, please help me by contacting me via Telegram, Email or LinkedIn. Thank you :heart:
{{< /alert >}}

*In the previous section I used kubernetes cronjob to do back up by schedule. It has certain pros and cons. You can read here [Backup database run inside Azure Kubernetes Service with Kubernetes cronjob](https://dev2ops.net/en/posts/backup_db_inside_aks_with_cronjob/)*

{{< alert theme="danger" dir="ltr" >}}
In this article, I just give you a short description, the benefit and the different name of service when you use Azure Disk Backup. I will not show step on how to configuration here because it has been done in Microsoft document (have images and step by step for easy to read and to implement, so I will not rewrite it here) . I've already linked the reference.
{{< /alert >}}

## Scenario

* You are using kubernetes, here is Azure Kubernetes Service (AKS)
* Deploy database, such as mongodb, elasticsearch,...
* __Containerized applications running on Azure Kubernetes Service (AKS cluster) are using managed disks as persistent storage__
* You need to back up data but you want an easy solution, centralized management place,not using scripting.

## Azure Disk Backup

* Azure Disk Backup offers a solution that provides snapshot lifecycle management for managed disks by automating periodic creation of snapshots and retaining it for configured duration using backup policy. 
* You can manage the disk snapshots without the need for custom scripting or any management overhead. 
* This backup solution that takes point-in-time backup of a managed disk using [incremental snapshots](https://docs.microsoft.com/en-us/azure/virtual-machines/disks-incremental-snapshots?tabs=azure-powershell) with support for multiple backups per day. 
* It supports backup and restore of both OS and data disks (including shared disks), whether or not they're currently attached to a running Azure virtual machine.

{{< alert theme="info" dir="ltr" >}}
So as you can see, the Azure Disk Backup will give you a easy solution to backup. A gathering place for backup management, easy control and delete the old snapshot, without using scripting. 
But the tradeoff is you sticky or depend on a cloud provider, on another cloud provider you have to do research more, find a service as same as or near same with Azure Disk Backup. __*But this tradeoff is not a big deal if your purpose is using Azure for a lifetime of application*__
[Reference Azure Disk Backup here](https://docs.microsoft.com/en-us/azure/backup/disk-backup-overview)
{{< /alert >}}

So we know the benefit of Azure Disk Backup, but how to we implement it? 
* That is Backup vault. The vault gives you a consolidated view of the backups configured across different workloads.

## Backup Vault

> A Backup vault is a storage entity in Azure that holds backup data for various newer workloads that Azure Backup supports, such as Azure Database for PostgreSQL servers and Azure Disks. Backup vaults make it easy to organize your backup data, while minimizing management overhead. Backup vaults are based on the Azure Resource Manager model of Azure, which provides enhanced capabilities to help secure backup data.

To configure backup, 
1. Go to the Backup vault
2. Assign a backup policy
3. Select the managed disk that needs to be backed up and provide a resource group where the snapshots are to be stored and managed. 
4. Azure Backup automatically triggers scheduled backup jobs that create an incremental snapshot of the disk according to the backup frequency.
5. Older snapshots are deleted according to the retention duration specified by the backup policy.

Once you configure the backup of a managed disk, a backup instance will be created within the backup vault. Using the backup instance, you can find health of backup operations, trigger on-demand backups, and perform restore operations. You can also view health of backups across multiple vaults and backup instances using Backup Center, which provides a single pane of glass view.

May be you can heard or meet the name Backup Center, what it is?
* Basically, it just a single place for unified management. __*To govern, monitor, operate, and analyze backups at scale.*__. Which mean you can create Backup vault in Backup Center (as the image below).
{{< img src="/images/post/backup_DB_inside_AKS_with_Az_backup/backup-center.png" title="backup center" caption="create vault" alt="image alt" width="700px" position="center" >}}

You can create a vault in Backup Vault or Backup Center. Check both reference here
[Backup Vault](https://docs.microsoft.com/en-us/azure/backup/backup-vault-overview#create-a-backup-vault)
[Backup Center](https://docs.microsoft.com/en-us/azure/backup/backup-managed-disks)


## Conclusion

Backup is a high requirement that needs to implement carefully. Depend on your scenario, your requirement that the strategy is different. 
For me, I would recommend using the existing backup service on the cloud for quick, easy implementation, have centralized management. I think we can do some research, read some practical then we can apply it, somehow it can help you recognize the difference between that provider's service