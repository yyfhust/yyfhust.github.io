---
layout: post
title: How to sign GCS URL in cloud-hosted instances (cloud run, cloud functions, compute engine etc..)
tags: GCP 
---

## How to sign GCS URL in Cloud-hosted instances (cloud run, cloud functions, compute engine etc..)

### 1. Share files via Google Cloud Storage (GCS)
Google Cloud Storage (GCS) is a cloud-hosted and fully-managed online file storage web service.  
Simply speaking, you can assume GCS as an online giant "disk", where you can save any type of files in folders
(so-called bucket in GCP's terminology). You can retrieve/download/upload/delete etc... the files or share the files with 
other people/applications via some RESTful URL. You do not need to be worried about sclability and creating replica across different regions
to reduce latency , as it is all handled by Google. 

One use case of GCS is to share resources such as images videos etc to users via web applications. The backend forwards the GCS URL 
to the frontend, then the browser "clicks" the url to, for example, render images.
The question is: how to share the URL to the end user in a safe manner ? 
If the web application is an internal tool, which is only used by few people internally, 
we can use "Authenticated URL", so that "Only users granted permission can access the object with this link". 
<p align="center">
![1](/resources/images/post3/1.png)
</p>
It is workable as we can manage IAM internally and only grant access to employees who could have access.

However, if the application is towards the whole world or many external customers, it will be painful to manage access. 
- You cannot expect every user to have a Google account to access the resources;
- It is not possible to control access of thousands and millions users, within which someone could even be anonymous. 

It is easy to come up with the solution: using pubic URL. 
Indeed we can publicize the gcs files / buckets by granting "allUsers" principals access to the gcs, so that anyone on the internet 
can access the URL. In the following example, I granted "allUsers" access on the object-level as I turn on the fine-grained
access control mode. You could also grant "allUsers" access on the bucket level if uniform access control mode is chosen.
<p align="center">
![2](/resources/images/post3/2.png)
</p>
Then public url is auto-generated!
<p align="center">
![3](/resources/images/post3/3.png)
</p>

It is as easy as pie to come up with and implement such solution, 
while it is also natural to have security concern. 
Any security engineer will cry if they know that you publicized URLs to anyone permanently without encryption LoL. 


### 2. Signed URL
Signed URL is the approach to the aforementioned scenario. 
A signed URL is a URL that provides limited permission and time to make a request. 
Signed URLs contain authentication information in their query string, 
allowing users without credentials to perform specific actions on a resource [1](https://cloud.google.com/storage/docs/access-control/signed-urls). 

### 3. Implementation 


