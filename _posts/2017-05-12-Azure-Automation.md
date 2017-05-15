---
layout: post
title:  "Automating Service Provisioning on Azure"
date:   2017-05-15 00:00:00 +0900
categories: azure
---

## Automation

**Infrastructure as a Code**라는 용어를 들어 보셨을 것입니다. DevOps 자동화 툴로 잘 알려진 [Chef](https://www.chef.io/chef/)및 [Puppet](https://puppet.com/), [Ansible](https://www.ansible.com/)에 더 나아가 멀티 벤더의 클라우드 서비스를 자동화해 주는 [Terraform](https://www.terraform.io/)과 같은 툴들이 주목받고 있습니다. 즉, 간단한 웹서버와 같은 IaaS VM 생성과 설치를 자동화하는 것을 넘어 이제는 매우 복잡한 서비스 프로비저닝(생성, 설치 및 구성) 자동화 요구가 커지고 있습니다.

이전 블로그에서 Azure 상에서 PaaS 기반 서비스의 BCDR을 구성하는 방법을 소개했습니다. BCDR과 같은 데모 서비스를 잘 구성하고 그대로 두자니 비용이 부담되고, 
삭제하자니 다시 잘 구성할 수 있을까 걱정이 되기도 합니다. 사실, 다시 데모를 구성하려면 단 번에 잘 안되고 오류가 나서 고생한 경험도 몇 번 있었습니다. 

이런 서비스를 직접 수작업으로 프로비저닝하는 것은 초보자에겐 어려울 수도 있고, 숙련자에게는 동일한 작업을 주기적으로 수행하는 것은 귀찮을 수도 있습니다. 그래서 *‘필요는 발명의 어머니’*라는 말처럼 자동화하는 스크립트를 고민하게 되었고, 수작업 구성 없은 완벽한 자동화 스크립트 작성을 시작하게 되었습니다.

## Automation Consideration

PowerShell 또는 CLI로 많은 자동화 스크립트를 작성은 해봤지만, 복잡한 서비스 구성을 하는데 한계가 있습니다. 파워풀한 스크립트 구성과 자동화 서비스의 확장성을 고민한 끝에 Azure SDK for node.js을 기본으로 작성하게 되었습니다. 하지만, Azure SDK만으로 스크립트 작성은 어려웠으며, ARM Template과 REST API 호출을 적절하게 사용할 수 밖에 없었습니다.

참고로, Node.js 기반이라서 작성된 스크립트는 클라이언트 콘솔에서는 물론 서버 사이드(웹 앱) 및 데스크탑 앱(Electron 기반)까지도 확장이 가능한 장점이 있습니다.

## Demo & Automation Code

이전 블로그에서 소개한 BCDR 서비스 프로비저닝을 자동화하는 스크립트에 대해서 소개하도록 하겠습니다.

서비스 프로비저닝을 위해서는 각 서비스들의 패러미터 설정(name, app package url, arm template url 등) 및 구독에 대한 로그인이 수행되어야 하며, 아래와 같이 3단계로 순차적으로 진행 됩니다. 

* Primary 사이트의 리소스 생성 및 설정
* Secondary 사이트의 리소스 생성 및 설정
* DR을 위한 추가 Search 설정 및 Traffic Manager 생성 및 설정

아래는 Primary 사이트의 리소스 생성 코드의 일부 입니다.

```
 [
     function (callback) {
        createResourceGroup(resourceClient, params, function (err, result, res, req) {
                if (err) {
                    return callback(err);
                }
                cb('createResourceGroup created');
                callback(null, result);
            });
        },
        function (callback) {
            fstorage.provisionStorage(storageClient, params, function (err, result, res, req) {

                params.storageConnString = res;

                if (err && params.storageConnString) {
                    return callback(err);
                }
                cb('provisionStorage created');
                callback(null, result);
            });
        },
        function (callback) {
            fsearch.provisionSearch(token, params, function (err, result, res, req) {

                if (err) {
                    return callback(err);
                }

                params.searchAPIKey = res;

                cb('provisionSearch created');
                callback(null, result);
            });
        },
        function (callback) {
            ffx.provisionFunction(webSiteClient, token, params, function (err, result, res, req) {
                if (err) {
                    return callback(err);
                }

                cb('provisionFunction created');
                callback(null, result);
            });
        },
        function (callback) {
            ffx.getFunctionUrl(token, params, function (err, result) {

                if (err) {
                    return callback(err);
                }

                cb('getFunctionUrl');
                params.funcApiAppUrl = result;
                callback(null, result);
            });
        },
        function (callback) {
            fwebapp.provisionWebApp(resourceClient, webSiteClient, token, params, function (err, result, res, req) {

                if (err) {
                    return callback(err);
                }

                cb('provisionWebApp created');
                callback(null, result);
            });
        }
    ]
```

리소스를 생성하고 각 해당 리소스의 설정값을 세팅하며, 일부 리소스에 대해서는 특정 설정 값을 참조(스토리지 문자열, 검색 API Key, Functions URL)하여 다른 리소스의 설정값으로 세팅합니다. 

자세한 데모 스크립트의 소스코드는 [github/iljoong/azure-automation](https://github.com/iljoong/azure-automation)를 참조하시기 바라며, 아래는 콘솔에서 실행한 스크립트의 실행 화면 입니다. (클릭하여 확인)


[![Watch Demo](https://img.youtube.com/vi/58i5M0AzDgY/0.jpg)](https://youtu.be/58i5M0AzDgY)

참고로, Azure의 서비스 리소스(Storage, Search 등)는 총 11개가 생성이 되고 세부설정이 이뤄지며, 대략 10분 정도에서 완료됩니다.
 
## Challenges

자동화 스크립트를 작성이 쉬울 줄 알았지만 그렇지 않았습니다. 스크립트를 작성하면서 얻은 몇가지 경험과 노하우를 공유 드리고자 합니다.

### Handling Asynchronous Process

Node.js가 확장성은 좋지만, 언어적 특성으로 기본적으로 비동기 처리해야하고, 수많은 callback 함수를 핸들링해야 하는 것이 매우 불편했습니다. 다행이도 [Async 모듈](https://github.com/caolan/async)로 손쉽게 비동기 호출을 순차적으로 실행 할 수 있었습니다. 물론 최근 업데이트된 [Azure SDK for node.js 2.0](https://azure.microsoft.com/en-us/blog/announcing-azure-sdk-node-2-preview/)는 Promises를 지원하여 비동기 처리가 비교적 개선되기는 했습니다. 참고로 JavaScript에도 C#의 async/await 와 같은 기능이 제공될 예정이라고 합니다.

Node.js의 비동기뿐만 아니라 다른 곳에도 비동기 요소가 있습니다. 일부 Azure 서비스(예: web app/functions app)는 SDK 또는 Template으로 서비스를 생성을 요청하면 비동기로 생성이 됩니다. 즉 완료 상태는 주기적으로 체크해야 합니다. 예를 들어 functions app의 functions url을 확인하기 위해서는 functions app 생성이 완료된 후 functions url을 가져오는 REST API를 호출해야 합니다. (참고로, functions url은 Web app에서 사용되면 appsettings에 functions url 값을 설정함)

상태를 주기적으로 호출하여 완료 상태를 확인하는 Retry/Loop 코드가 필요했고, node.js에서 retry 처리를 하는 방법은 아래를 참고하여 구현했습니다. (최근에야 async 모듈에 retry 기능이 있다는 것을 알게되었지만)

[http://stackoverflow.com/questions/35185800/node-js-request-loop-until-status-code-200](http://stackoverflow.com/questions/35185800/node-js-request-loop-until-status-code-200)

또한, 다수의 서비스를 생성하고 설정하기 때문에 간혹 일부 서비스 생성/설정 시 실패가 발생하곤 했습니다. 스크립트를 다시 재실행할 수도 있으나 (대부분의 서비스 생성방법이 create-or-update 또는 incremental deployment라서 재실행해도 크게 문제는 안됨) 스크립트 내에서 retry logic으로 처리하는데 위의 코드가 활용 되었습니다.

### Preview Azure SDK

Azure SDK가 preview라서 그런지 아직 완벽하지 않습니다. 버그도 종종 있어 SDK의 함수대신 다른 방법을 사용해야 하는 경우도 있었으며, SDK에서 모든 기능을 제공하지 않아 직접 REST API로 구현할 수 밖에 없는 경우도 있었습니다. (예: Search Service 및 Traffic Manager)

SDK를 사용할 수도 있으나, 어떤 경우에는 Azure Template으로 더 편리하게 처리할 수 있습니다. 예를 들어 Web/Functions App의 패키지를 배포하는 것은 Template이 가장 편리한 것 같습니다. (사실 SDK에는 관련 함수가 존재하지 않는 것 같음)

그리고, 최근 SDK 업데이트 되면서 azure-arm-website의 함수명이 변경되어 기존 스크립트가 갑자기 정상적으로 동작하지 않는 경험도 했습니다. 현재, azure-arm-website는 [sdk 문서]( http://azure.github.io/azure-sdk-for-node/)에서도 누락되어 있습니다.

### Unexpected Error

초기에 생각하지 못했던 에러가 발생하기도 했습니다. 이런 에러는 항상이 아닌 가끔 발생하기 때문에 원인을 확인하기가 어려웠습니다. 특히, ARM Template으로 Web app의 appsettings 설정과 패키지를 msdeploy로 설치하는 경우 Race condition issue로 deployment가 간혹 실패됩니다. 원인은 appsettings 설정이 적용되면 Web app이 재시작되기 때문에 패키지 설치 중간에 재시작이 되면서 이때 실패가 발생하는 것이었습니다.
 
[https://blogs.msdn.microsoft.com/hosamshobak/2016/05/26/arm-template-msdeploy-race-condition-issue/](https://blogs.msdn.microsoft.com/hosamshobak/2016/05/26/arm-template-msdeploy-race-condition-issue/)

Web app의 경우 msdeploy 리소스의 종속성(dependsOn) 설정을 통해 race condition issue가 발생하지 않았지만, Functions app은 약간 다른 현상이 발생했습니다. Web app과 같이 종속성 설정을 할 경우 설치된 패키지가 사라져버렸습니다. 이 문제 해결을 위해 정말로 많은 시간으로 소요되었고, 결국엔 두번의 각기 다른 Template 배포(appsettings와 msdeploy)로 해결했습니다.

마지막으로, 해결하지 못한 오류가 하나 있습니다. 막 생성된 Azure search의 도메인을 간혹 찾지 못하는 DNS 오류(ENOTFOUND)가 발생합니다. 아무래도 node.js의 내부 DNS 모듈의 버그(?) 같은데 DNS Flush(ipconfig /flushdns)을 실행하고 다시 스크립트를 실행하면 정상적으로 동작합니다.

### 100% Automation

Azure의 서비스 생성은 상대적으로 쉬었지만, 모든 서비스의 세부적인 설정을 자동화 하는 것은 사실 매우 어려웠습니다. 문서화가 잘되어 있으면 다행이지만 문서화가 잘 안되어 있거나 쉽게 찾을 수 없는 경우가 있습니다.

특히, Azure functions 의 HTTPTrigger functions url을 프로그래밍적으로 가져오는 방법을 몰라서 포기하고 유일하게 수작업으로 설정하는 것으로 했습니다. 다행히 Azure functions의 담당 PM을 소개 받아 그의 도움으로 문서를 확인할 수 있었습니다. (azure.com 문서 사이트에는 해당 내용이 없고, github의 projectkudu에 관련 문서가 존재!)

[https://github.com/projectkudu/kudu/wiki/Functions-API#getting-a-functions-secrets](https://github.com/projectkudu/kudu/wiki/Functions-API#getting-a-functions-secrets)

## Next...

Azure에서 복잡한 PaaS 기반의 서비스 프로비저닝을 자동화하는 방법에 대해서 소개했습니다. 이런 스크립트를 통해서 SaaS 형태의 서비스를 제공할 수 있고, 더 나아가 서비스를 카탈로그/마케플레이스 형태로 제공할 수도 있을 것입니다. 

스크립트의 최적화도 필요하고, 또한 Primary 사이트 생성과 Secondary 사이트를 병렬로 수행되도록 하면 프로비저닝 시간을 좀더 단축 시킬 수도 있으나 숙제로 남깁니다. 

아직 끝나지 않았습니다. 다음에는 이런 자동화 스크립트를 데스크탑 앱으로 패키징하여 관리할 수 있는 방법에 대해서 소개하도록 하겠습니다. 

샘플 데모 스크립트는 아래를 참조하시기 바랍니다. [Azure Automation Demo script](https://github.com/iljoong/azure-automation)

/iljoong/

