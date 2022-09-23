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

### 3. Google Sample Python Implementations and Their drawbacks
GCP official documentation provided two sample implementations. 


[sample program 1](https://cloud.google.com/storage/docs/access-control/signing-urls-with-helpers#storage-signed-url-object-python):
```python
from google.cloud import storage
import datetime
from google.cloud import storage
def generate_download_signed_url_v4(bucket_name, blob_name):
    """Generates a v4 signed URL for downloading a blob.
    Note that this method requires a service account key file. You can not use
    this if you are using Application Default Credentials from Google Compute
    Engine or from the Google Cloud SDK.
    """
    # bucket_name = 'your-bucket-name'
    # blob_name = 'your-object-name'
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(blob_name)
    url = blob.generate_signed_url(
        version="v4",
        # This URL is valid for 15 minutes
        expiration=datetime.timedelta(minutes=15),
        # Allow GET requests using this URL.
        method="GET",
    )
    print("Generated GET signed URL:")
    print(url)
    print("You can use this URL with any user agent, for example:")
    print(f"curl '{url}'")
    return url
```

Tested the code in Jupyter Lab and it did create the signed URL.(note: do not forget to run `os.environ["GOOGLE_APPLICATION_CREDENTIALS"] = "key.json"`. key.json is generated from 
a service account, which has sufficient permission for generating signed URL.) 
<p align="center">
![4](/resources/images/post3/4.png)
</p>

However, it failed when I run the same snippet of code in cloud run / function ...
Let's take cloud functions as an example. 

Create a cloud function instance which runs the same code as above. 
- Specify a service account with at least the permission `serviceAccountTokenCreator` and `storage.objectCreator`
<p align="center">
![5](/resources/images/post3/5.png)
</p>

- Allow all traffic in ingress settings (make it easier for testing)

After all set, I triggered the cloud functions. Cloud function raised the following exceptions
<p align="center">
![7](/resources/images/post3/7.png)
</p>

The root cause of the above exception is explained in the comment of the python code :  `Note that this method requires a service account key file. You can not use this if you are using Application Default Credentials from Google Compute
Engine or from the Google Cloud SDK.`
We have to explicitly provide the private key to make it work, even though we specify the runtime service account. To put it in another work, we need to "upload"
the private key of the service account to the cloud function instance , and run `os.environ["GOOGLE_APPLICATION_CREDENTIALS"] = "key.json"`



Google provides a 
[sample program 2](https://cloud.google.com/storage/docs/access-control/signing-urls-manually), which works in cloud run. 
```python
import binascii
import collections
import datetime
import hashlib
import sys
# pip install google-auth
from google.oauth2 import service_account
# pip install six
import six
from six.moves.urllib.parse import quote
def generate_signed_url(service_account_file, bucket_name, object_name,
                        subresource=None, expiration=604800, http_method='GET',
                        query_parameters=None, headers=None):

    if expiration > 604800:
        print('Expiration Time can\'t be longer than 604800 seconds (7 days).')
        sys.exit(1)

    escaped_object_name = quote(six.ensure_binary(object_name), safe=b'/~')
    canonical_uri = '/{}'.format(escaped_object_name)

    datetime_now = datetime.datetime.now(tz=datetime.timezone.utc)
    request_timestamp = datetime_now.strftime('%Y%m%dT%H%M%SZ')
    datestamp = datetime_now.strftime('%Y%m%d')

    google_credentials = service_account.Credentials.from_service_account_file(
        service_account_file)
    client_email = google_credentials.service_account_email
    credential_scope = '{}/auto/storage/goog4_request'.format(datestamp)
    credential = '{}/{}'.format(client_email, credential_scope)

    if headers is None:
        headers = dict()
    host = '{}.storage.googleapis.com'.format(bucket_name)
    headers['host'] = host

    canonical_headers = ''
    ordered_headers = collections.OrderedDict(sorted(headers.items()))
    for k, v in ordered_headers.items():
        lower_k = str(k).lower()
        strip_v = str(v).lower()
        canonical_headers += '{}:{}\n'.format(lower_k, strip_v)

    signed_headers = ''
    for k, _ in ordered_headers.items():
        lower_k = str(k).lower()
        signed_headers += '{};'.format(lower_k)
    signed_headers = signed_headers[:-1]  # remove trailing ';'

    if query_parameters is None:
        query_parameters = dict()
    query_parameters['X-Goog-Algorithm'] = 'GOOG4-RSA-SHA256'
    query_parameters['X-Goog-Credential'] = credential
    query_parameters['X-Goog-Date'] = request_timestamp
    query_parameters['X-Goog-Expires'] = expiration
    query_parameters['X-Goog-SignedHeaders'] = signed_headers
    if subresource:
        query_parameters[subresource] = ''

    canonical_query_string = ''
    ordered_query_parameters = collections.OrderedDict(
        sorted(query_parameters.items()))
    for k, v in ordered_query_parameters.items():
        encoded_k = quote(str(k), safe='')
        encoded_v = quote(str(v), safe='')
        canonical_query_string += '{}={}&'.format(encoded_k, encoded_v)
    canonical_query_string = canonical_query_string[:-1]  # remove trailing '&'

    canonical_request = '\n'.join([http_method,
                                   canonical_uri,
                                   canonical_query_string,
                                   canonical_headers,
                                   signed_headers,
                                   'UNSIGNED-PAYLOAD'])

    canonical_request_hash = hashlib.sha256(
        canonical_request.encode()).hexdigest()

    string_to_sign = '\n'.join(['GOOG4-RSA-SHA256',
                                request_timestamp,
                                credential_scope,
                                canonical_request_hash])
    # signer.sign() signs using RSA-SHA256 with PKCS1v15 padding
    signature = binascii.hexlify(
        google_credentials.signer.sign(string_to_sign)
    ).decode()
    scheme_and_host = '{}://{}'.format('https', host)
    signed_url = '{}{}?{}&x-goog-signature={}'.format(
        scheme_and_host, canonical_uri, canonical_query_string, signature)
    return signed_url
```

To be honest, this implementation kind of like inventing the wheel on your own, and does not have 
much difference from the sample 1 , function-wise. And as you can see, we still need to explicitly pass
`service_account_file` to the function. 

It is so hard to favour both of sample 1 & 2, as it is not recommended to "upload" private key onto serverless 
instances such as cloud run / functions. Generally speaking, serverless instances do not have persistent disk storage, so 
you will have to add the private key to your repo, and let CI/CD pipeline deploy the key together with the code to cloud run / functions.
However, it is a big no-no to save private key in the git repo. 

Is it possible to get rid of key file ?
Yes ! 

### 4. Dive deep into the source code

We have to dig deep into the source code of
[generate_signed_url](https://googleapis.dev/python/storage/latest/_modules/google/cloud/storage/blob.html#Blob.generate_signed_url)
It is natural to assume these three parameters are related to the authentication. 
`credentials, service_account_email, access_token`.

I investigated the source code and figured the following logics:
 - `credentials` has to be verified if `service_account_email` or `access_token` are missing. The function `ensure_signed_credentials` verifies if the type of the credential is 
`google.auth.credentials.Signing`. 

[code 1 ](https://github.com/googleapis/python-storage/blob/3664ddebe8746d0395acd86d5efa1ef973eeac3d/google/cloud/storage/_signing.py#L547)
```python
if not access_token or not service_account_email:
    ensure_signed_credentials(credentials)
    client_email = credentials.signer_email
```
[code 2](https://github.com/googleapis/python-storage/blob/3664ddebe8746d0395acd86d5efa1ef973eeac3d/google/cloud/storage/_signing.py#L41)
```python
def ensure_signed_credentials(credentials):
    """Raise AttributeError if the credentials are unsigned.
    :type credentials: :class:`google.auth.credentials.Signing`
    :param credentials: The credentials used to create a private key
                        for signing text.
    :raises: :exc:`AttributeError` if credentials is not an instance
            of :class:`google.auth.credentials.Signing`.
    """
    if not isinstance(credentials, google.auth.credentials.Signing):
        raise AttributeError(
            "you need a private key to sign credentials."
            "the credentials you are currently using {} "
            "just contains a token. see {} for more "
            "details.".format(type(credentials), SERVICE_ACCOUNT_URL)
        )
```

- If `GOOGLE_APPLICATION_CREDENTIALS` is set, the client can retrieve the signed credential from they private key file. 
In the cloud function, the key was not provided, and the client fetches credential from `Metadata Service`, and cloud function throws 
the exception "you need a private key to sign credentials" as the credential is not signed. More details regarding auth [on](https://google-auth.readthedocs.io/en/master/_modules/google/auth/_default.html#default).
- If `service_account_email, access_token` are provided, the client will not verify credentials.


Given the above logic, there are two workarounds:
- pass `service_account_email, access_token`  to the function.
- pass `credentials` which implements the class `Signing` to the function


### 5. Implementation

1. pass `service_account_email, access_token`  to the function. 
It is easy to generate credential instance from  google.auth library.
And furthermore we could retrieve email and token from credential, see details in the code
```python
import datetime
from google import auth
from google.cloud import storage
import google.auth.transport.requests
def generate_download_signed_url_v4(bucket_name, blob_name):
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(blob_name)
    
    ## generate token and email
    credentials, project_id = google.auth.default()   ## fetch credentials instance
    request = google.auth.transport.requests.Request()
    credentials.refresh(request) ## generate token by calling refresh()
    token = credentials.token  ## token 
    service_account_email = credentials.service_account_email ## service account email

    url = blob.generate_signed_url(
        version="v4",
        # This URL is valid for 15 minutes
        expiration=datetime.timedelta(minutes=15),
        # Allow GET requests using this URL.
        method="GET",
        service_account_email=service_account_email,   
        access_token=token
    )
    return url
def hello_world(request):
    return generate_download_signed_url_v4("yifan_example_bucket", "dummy.png")
```

2. pass `credentials` which implements the class `Signing` to the function

It is a bit tricky to convert credential into a new instance which implements `Signing`.
Let's take a look at the exception again. 
```
the credentials you are currently using <class 'google.auth.compute_engine.credentials.Credentials'> just contains a token.
```
jump into the [source code](https://google-auth.readthedocs.io/en/master/reference/google.auth.compute_engine.credentials.html) of `google.auth.compute_engine.credentials`, and we can find the class 
`IDTokenCredentials` implements both `Credentials` and `Signing` class, so we can try to leverage this class.


```python
import datetime
from google import auth
from google.cloud import storage
import google.auth.transport.requests
def generate_download_signed_url_v4(bucket_name, blob_name):
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(blob_name)
    
    credentials, _ = google.auth.default()   ## fetch credentials instance
    request = google.auth.transport.requests.Request()
    service_account_email = credentials.service_account_email ## service account email
    credentials_signed = google.auth.compute_engine.IDTokenCredentials(
        request=request,
        target_audience="",
        service_account_email=service_account_email
    )

    url = blob.generate_signed_url(
        version="v4",
        # This URL is valid for 15 minutes
        expiration=datetime.timedelta(minutes=15),
        # Allow GET requests using this URL.
        method="GET",
        credentials=credentials_signed
    )
    return url
```

Tested the both of code 1 and 2 in cloud function, and it works by adding such little change ! 


The end:
Thanks my colleagues for giving me some insights.

