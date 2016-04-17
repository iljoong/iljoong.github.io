---
layout: post
title:  "Azure Notification Hub"
date:   2016-04-16 09:00:00 +0000
categories: project
---

## Azure NH

최근 NH 사용에 어려움을 호소하거나 또는 오해(?)를 하고 있는 분들을 보았습니다. 다시 말하면, NH를 SaaS 형태의 푸쉬 서비스로 생각하신 분들에게는 사용법이 꽤 어렵게 느껴질 수 있고, 문서가 좀 어렵다 보니 일부 플랫폼(예: [Cordova](http://cordova.apache.org/))은 지원하지 않는다고 생각하시는 분들이 있습니다.

사실 시장에는 이미 많은 SaaS 형태의 서비스가 있어 그렇게 생각할 수도 있지만, NH는 좀 차별화된 강력한 PaaS 서비스로 백앤드 서비스를 구축 또는 연계해서 사용할 수 있습니다. 이번 블로그는 [**Azure Notification Hub(NH)**](https://azure.microsoft.com/services/notification-hubs/) 의 백앤드 서비스를 구현하는 방법에 대해 이야기 해보고자 합니다.

## 푸쉬 서비스 매커니즘

먼저 푸쉬 서비스 매커니즘을 잘 모르시는 분들을 위해서 간단하게 매커니즘을 설명 드립니다.

푸쉬를 활성화한 모바일 앱이 처음 실행하게 되면 PNS(Push Notification Service)에 등록이 되고, 디바이스 고유의 토큰을 받게 됩니다. (참고로, APNs는 Device Token, WNS는 ChannelUri, GCM은 GCM Registration Id)

이 토큰으로 디바이스는 푸쉬를 받게 됩니다. 즉, 토큰과 푸쉬 애플리케이션 키만 있으면 바로 해당 디바이스에 푸쉬를 할 수 있습니다. GCM으로 예를 들면, 아래와 같은 PowerShell 스크립트로 해당 디바이스 토큰으로 푸쉬 전송을 할 수 있습니다.

{% highlight ps linenos %}
# https://developers.google.com/cloud-messaging/http#auth 
$uri = 'https://gcm-http.googleapis.com/gcm/send'

$api_key = 'key=AIza…nc_c'
$token = 'euRZVl06dHI:…Tcq'
$body = "{""to"": ""$token"", ""data"": { ""message"": ""HELLO"" } }"

Invoke-RestMethod -Headers @{"Authorization"=$api_key; "Content-Type"="application/json"} -Uri $uri -Body $body -Method Post
{% endhighlight %}

하지만, 토큰은 디바이스에서만 받아 지기때문에 토큰은 NH와 같은 백앤드 서비스에 등록 되어야만 백앤드 서비스에서 여러 디바이스로 푸쉬 전송을 할 수 있습니다.

## NH와 Azure Mobile App으로 푸쉬 서비스 구현

참고로 Azure Portal 내의 NH 또는 Visual Studio의 Test Send 창에서 푸쉬를 손쉽게 테스트 전송할 수 있습니다. 말그대로 Test Send에서는 최대 10대 디바이스에서 대해서만 테스트 전송됩니다. 그래서, 실제 프로덕션으로 사용하려면 Test Send 만으로는 안되고 추가 앱 백앤드 서비스를 구현해야 합니다.

앱 백앤드 서비스는 앞서 언급했듯이, Node 모듈로 제공하는 [`azure-mobile-apps`](https://www.npmjs.com/package/generator-azure-mobile-apps) (Azure Mobile App의 프레임워크)를 이용하여 구현하는 방법을 설명합니다. 먼저, [`Express`](http://expressjs.com/) 와 관련된 패키지를 설치하고 `azure-mobile-apps` 패키지를 설치합니다. NH의 Hub Name 및 연결문자열이 정상적으로 설정된 환경에서 Windows Universal App의 푸쉬는 아래와 같이 구현할 수 있습니다.

{% highlight js linenos %}
router.post('/push/wns', function(req, res, next) {
  
   var msessage = 'hello';
   var payload = '<toast><visual><binding template="ToastText01"><text id="1">' + message + '</text></binding></visual></toast>';

   req.azureMobile.push.wns.send(null, payload, 'wns/toast', function(error, response) {
   	if (!error){
            // notification sent
            res.status(200).send(response);
          }
          else {
            res.status(500).send(error);
          }
   });
}
{% endhighlight linenos %}

GCM, APNs는 `push.wns.send` 대신, 각각 `push.gcm.send`와 `push.apns.send`를 사용하면 됩니다. `push` 객체의 문서는 아래의 URL을 참고하시기 바랍니다. Azure Mobile Services의 문서이지만 Azure Mobile Apps와 동일한 것으로 추정됩니다.

[https://msdn.microsoft.com/en-us/library/azure/jj554217.aspx](https://msdn.microsoft.com/en-us/library/azure/jj554217.aspx)

아래는 디바이스 등록과 푸쉬 전송의 다이어그램입니다.
![Push Flow](https://i-msdn.sec.s-msft.com/dynimg/IC702622.png)

## 템플릿과 태그 사용

템플릿을 간단하게 설명하면, 각 PNS에 푸쉬 메시지를 전송하는 하는 것이 아니라, 각 PNS 고유의 메시지 형식을 미리 맞춰 저장하고 단일화된 메시지 형식으로 전송하는 방법입니다. 단일화된 메시지 형식뿐만 아니라 개인화된 메시지를 전송하는데 사용할 수 있습니다. 태그는 말 그대로 해당 태그가 지정된 디바이스만 전송하는 일종의 라우팅 방법입니다.

![Template](https://i-msdn.sec.s-msft.com/dynimg/IC702621.png)

NH을 사용하는 이유가 단순히 푸쉬 전송이라면 다른 서비스를 사용하는 것이 나을 수도 있습니다. 템플릿과 태그를 사용하는 것이 진정한 NH 사용의 목적이지만, 생각만큼 쉽지가 않았습니다. 일단 문서가 기능 중심 설명이 대부분이고, 구현 방법의 내용은 전무하다 싶더군요. 일단, 푸쉬 서비스의 이해가 없이 단순하게 접근했다가, 시간이 걸리긴 했지만, 덕분에 각 플랫폼 별 푸쉬 서비스에 이해를 하게 되는 소득(?)을 얻었습니다. 

앞서 푸쉬 서비스 매커니즘을 설명 드린 것처럼 모바일 앱들은 PNS에 연결 시 받은 토큰을 NH에 SDK나 REST API로 등록하게 되며, 등록된 디바이스는 고유의 `RegistrationId`로 관리됩니다. 물론 등록, 템플릿 및 태그 설정은 SDK로 클라이언트 사이드에서 할 수 있지만, 백앤드 사이드에서 실행하는 것을 보안적으로 관리적으로 권장하고 있습니다. 

NH에 등록된 디바이스 리스트를 표시하는 방법은 아래와 같이 구현할 수 있습니다.

{% highlight js linenos %}
router.get('/home', function(req, res, next) {
      
    req.azureMobile.push.listRegistrations(function (err, response){
        if (!err) {
            res.render('index', { title: "Portal", entries: response });
        }
        
    });
});
{% endhighlight %}

`azure-mobile-apps` 패키지는 등록 리스트 표시와 같은 API를 제공하지만 아쉽게도 템플릿 및 태그 설정 API는 제공하지는 않아 직접 REST API로 구현해야 합니다. 

[https://msdn.microsoft.com/en-us/library/azure/dn495827.aspx](https://msdn.microsoft.com/en-us/library/azure/dn495827.aspx)

하지만, RegistrationId로 템플릿을 등록하려면 XML 구성을 이해해야 하고 좀 복잡해 보입니다. 좀더 자세히 들여다 보면 쉬운 방법의 REST API(Update Installation by Installation ID:)도 제공 되는데 `InstallationId`가 필요합니다. 그런데, 어이 없게 `InstallationId`에 대한 설명은 어디에도 없습니다.

[https://msdn.microsoft.com/en-us/library/azure/mt621169.aspx](https://msdn.microsoft.com/en-us/library/azure/mt621169.aspx)

`InstallationId`를 찾아 보면 Registration 정보의 Tags 속성에 하나의 태그 값으로 저장이 됩니다. 아마도 NH가 초기 개발될 때 만들어진 방식이 아니고 이후에 만들어진 방식인 것 같습니다. 아쉽게도 `InstallationId`를 가져오기 위해서는 태그 값을 읽고 파싱하는 코드가 필요합니다. 

또 하나의 이슈는 `InstallationId`를 이용하는 REST API는 `azure-mobile-apps` 패키지가 아직 지원을 하지 않고 별도의 구현을 해야 합니다. 즉, NH 연결문자열 접근, SAS 토큰키 생성 및 원시적인 REST API 호출(예를 들어 [`request`](https://www.npmjs.com/package/request) 패키지를 이용)을 구현해야 합니다. 그래서, `azure-mobile-apps` 패키지 소스 코드를 뒤져보고 분석하여 (물론 수많은 시행오차는 덤으로) 간단한 방법을 찾았습니다.

`WebResource`와 `_executeRequest`를 통해 REST API의 헤더 생성 및 호출을 간단하게 실행시킬 수 있습니다. `_executeRequest`의 응답이 기본으로 xml 형식이기 때문에 `webResource.rawResponse = true` 로 해야만 `InstallationId`가 사용하는 JSON 형식의 응답을 정상 처리할 수 있습니다.

{% highlight js %}
var azureCommon = require('azure-common');
var WebResource = azureCommon.WebResource;
...
router.post('/tag/', function(req, res, next) {
    
    var iid = req.body.iid;
    var tag = req.body.tag; 

    var push = req.azureMobile.push;
    var webResource = WebResource.patch(push.hubName + '/installations/' + iid);
    webResource.rawResponse = true;
    var payload = "[{\"op\": \"add\", \"path\": \"/tags\", \"value\": '" + tag +  "' }]";

    push._executeRequest(webResource, payload, InstallationResult, null, function (err, response) {
        
        if (err) {
            console.log(err);
            res.render('log', { title: iid, content: err });
        } else {
            res.redirect("/home");
        }        
    });
});
{% endhighlight %}

템플릿 및 태그는 물론 Telemetry도 유용한 기능 중 하나입니다. Telemetry의 기능 구현을 포함한 자세한 구현 방법은 Github에 공개한 Azure Notification Demo App 소스 코드를 참조하시기 바랍니다.

[https://github.com/iljoong/nhubdemo](https://github.com/iljoong/nhubdemo)

Demo App의 동작은 아래를 클릭하여 확인하세요
[![Watch NHubDemo](https://img.youtube.com/vi/one4LxEUAQU/0.jpg)](https://youtu.be/one4LxEUAQU)


/iljoong/

