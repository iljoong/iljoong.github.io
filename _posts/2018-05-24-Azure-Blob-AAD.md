---
layout: post
title:  "How to securely access Azure Blob Storage"
date:   2018-05-24 00:00:00 +0900
categories: azure
---

지난 블로그에서 Azure 보안 프랙티스를 소개했었는데, Azure Blob Storage를 MSI로 접근하는 방식이 불편하다고 언급했었습니다.

한국시간으로 오늘, [Azure AD Authentication for Azure Storage가 preview로 announce](https://azure.microsoft.com/en-us/blog/announcing-the-preview-of-aad-authentication-for-storage/) 되었습니다.

지금까지는 blob storage 접근을 하기 위해서는 storage key/SAS key로 접근만 가능했는데, 앞으로 AAD를 통해 획득한 `bearer token`으로 접근 가능 해졌습니다. 

IaaS 환경의 애플리케이션에서 blob storage를 접근하기 위해서는 소스코드에 storage key를 하드코딩해야 하는 보안 이슈(개발자에 의한 key 유출 등)가 있었고, 대부분의 엔터프라이즈 기업에서 허용되지 않는 프랙티스입니다.

[MSI](https://docs.microsoft.com/en-us/azure/active-directory/managed-service-identity/tutorial-windows-vm-access-storage)를 이용하여 이런 보안 이슈를 해결할 수 있지만, 아직까지는 만족스럽다고 이야기하기 어려웠습니다.

- Service Management API를 통해 Blob Storage Key를 획득할 수 있었지만, Blob Storage의 subscription/resource name이 포함되어 있는 Resource URI를 알아야해서 간결하지 않음.

- 대안으로 Key Vault에 Blob Storage Key를 저장하는 방법도 있지만, 역시 Storage Key가 rotate된 경우 Key Vault 역시 업데이트해야 하는 관리 오버헤드가 발생. 아직까지는 이방법이 최선의 방안임.


### Comparison

아래는 linux환경에서 curl을 이용한 사용 비교입니다. 참고로, MSI는 IMDS(Instance Metadata service) endpoint를 사용하여 access_token을 획득합니다.

#### Using Service Management API

MSI의 resource로 `https://management.azure.com/` 지정

```
access_token=$(curl -H 'Metadata:true' -s 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/' | jq -r ".access_token")

storagekey=$(curl 'https://management.azure.com/subscriptions/{subs}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/myblob/listKeys?api-version=2016-12-01' -s -X POST -d "" -H "Authorization: Bearer $access_token" | jq -r ".keys[0].value")

sharedkey=genkey(storagekey, …)

curl https://myblob.blob.core.windows.net/images/test/IMG_0000.JPG -H "Authorization: SharedKey myblob:$sharedkey " -H "x-ms-version: 2017-11-09" -o IMG_0000.JPG
```

참고로, REST API 예로 sharedkey를 사용했으나 SDK를 사용하는 일반적인 경우에는 storagekey를 직접 활용 가능

Sharedkey 생성은 [https://tsmatz.wordpress.com/2016/07/06/how-to-get-azure-storage-rest-api-authorization-header/](https://tsmatz.wordpress.com/2016/07/06/how-to-get-azure-storage-rest-api-authorization-header/) 참조

#### Using Key Vault

MSI의 resource로 `https://vault.azure.net` 지정

```
access_token=$(curl -H 'Metadata:true' -s 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://vault.azure.net' | jq -r ".access_token")

curl https://myvault.vault.azure.net/secrets/storagekey/?api-version=2016-10-01 -H "Authorization: Bearer $access_token"

sharedkey=genkey(storagekey, …)

curl https://myblob.blob.core.windows.net/images/test/IMG_0000.JPG -H "Authorization: SharedKey myblob:$sharedkey " -H "x-ms-version: 2017-11-09" -o IMG_0000.JPG

```

#### Using AAD

blob storage의 IAM에서 접근을 하는 VM에 `Storage Blob Data Contributor` 역할 추가하고, MSI의 resource로 `https://storage.azure.com` 지정

```
access_token=$(curl -H 'Metadata:true' -s 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://storage.azure.com/' | jq -r ".access_token")

curl https://myblob.blob.core.windows.net/images/test/IMG_0000.JPG -H "Authorization: Bearer $access_token" -H "x-ms-version: Blob" -o IMG_0000.JPG
```

추가로 PUT API를 이용한 업로드 예입니다.

```
f='IMG_0000.JPG'
currdate=$(date +"%a, %d %b %Y %H:%M:%S GMT")
length=$(ls -nl $f  | awk '{print $5}')

curl -i -X PUT https://myblob.blob.core.windows.net/images/upload/IMG_0000.JPG \
    -H "Authorization: Bearer $access_token" \
    -H "x-ms-date: $currdate" \
    -H "x-ms-version: 2017-11-09" \
    -H "x-ms-blob-type: BlockBlob" \
    -H "Content-Type: image/jpeg" \
    -H "Content-Length: $length" \
    --data-binary "@$f"
```

### Closing...

AAD 방식이 앞선 방식에 비해서 많이 간결해진 것을 확인할 수 있을 것 입니다. 늦었지만 애플리케이션에서 Azure Blob Storage를 안전하고 편리하게 접근할 수 있게 되었습니다.

Revised: 2018-05-25