---
title: How to backup database inside kubernetes
date: 2021-05-02T12:00:06+09:00
description: The article to introduce how to use cronjob to backup database inside kubernetes
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Nhat Le
authorEmoji: 👽
tags:
- kubernetes
- backup
- self-idea
- database
- cloud
categories:
- platform
series:
- kubernetes
- Azure cloud
image: images/post/backup_database_inside_kubernetes/bk-database-icon.png
---


This article will show you how to backup database inside a cluster (kubernetes). Here i use Azure kubernetes service (a service of Azure cloud). But you can use this article to back up any kind of Kubernetes approach.

## Scenario
* You are using kubernetes, here is Azure Kubernetes Service (AKS)
* Deploy database, such as mongodb, elasticsearch,...
* Use persistence volume claim (PVC), to mount data from pod to disk place on Azure