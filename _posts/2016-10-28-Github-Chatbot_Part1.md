---
layout: post
title:  "Building Chatbot for Azure Management - Part 1"
date:   2016-10-28 00:00:00 +0000
categories: project
---

## Background

클라우드 리소스 관리가 가끔 불편할 때가 있습니다. 관리 포탈 또는 CLI 툴을 이용하여 관리가 가능하지만, 
가끔 특정 리소스를 언제 어디서나 제어하고자 할 때 관리 포탈과 CLI 툴로 불가능하거나 또는 불편할 때가 있습니다.

예를 들어, MSDN 구독으로 제공되는 Azure 계정의 경우 월간 제공되는 크레딧이 충분하지 않아 데모 또는 테스트 후에 VM과 같은 리소스를
삭제하거나 중지시켜야 하는데 가끔 잊어 버릴 때가 있습니다. 생각이 나서 정지하려고 할 때 는 외부에 있거나, 모바일로는 여간 불편합니다.
그래서, 꽤 오래전부터 이를 손쉽게 구현하는 방법을 고민했고 심플한 구현 방법/아키텍처를 고민하던 중 챗팅과 연계 또는 챗봇을 이용하는
아이디어가 떠 올랐습니다.

Slack bot을 잠시 고민했지만 일단 손쉽게 접근하기 어려워 포기하던 중, 올 봄 Microsoft BUILD에서 소개된 Bot Framework을 활용하고자 했습니다.
바로 시작은 하지 못했고 최근에 Bot Framework를 이용하여 Azure 관리 챗봇, __AzmanBot__,을 만들었고 간단한 테스트앱을 통해 소개하도록 하겠습니다.

(_아래 이미지 클릭으로 동작 확인_)

[![Watch Demo](https://img.youtube.com/vi/2dUxRE5sy0E/0.jpg)](https://youtu.be/2dUxRE5sy0E)

[Goto source code](https://github.com/iljoong/azmanbot/tree/cli)

## Bot Framework

Microsoft는 Bot Framework를 통해 손쉬운 봇서비스 개발을 가능케하고 있습니다. 일단, 특정 Chat bot 서비스에 종속적이지 않은 프레임워크를 통해서
하나의 봇서비스를 여러 챗서비스(Slack, Skype, Telegram)와 연동할 수 있습니다. 그리고, 자연어 처리와 같은 인텔리전스 서비스를 손쉽게 통합할 수 있는
기능을 제공하고 있습니다.

파트 1에서는 Bot Framework의 기능으로 Azure의 리소스를 제어하는 기능을 소개하고, 파트 2에서는 LUIS(Language Understanding Intelligence Service)를
이용한 챗팅의 자연어 처리(영어) 방법을 소개하도록 하겠습니다.

## Main Feature & Architecture

Azure Management를 핸들링하는 샘플 챗봇의 주요 기능은 아래와 같습니다.

### Features
* __기본 챗서비스 및 Skype & Slack 채널 지원__
    * node.js 기반
    * Basic/Prompt/Waterfall 대화
    * CLI 스타일 챗

* __Azure service 관리__
    * OAuth의 client credential 이용한 Access token 획득 (AAD 구성 필요)
    * Azure Service Management API 호출 (VM 리스트, VM 상태, VM 정지/시작, 사용량)
    * Azure Billing API 호출 (별도 .net core 기반 API App 구성)
    * LINQ를 이용한 사용량 계산 및 표기 (날짜 또는 미터링 SKU)

* __스케쥴링__
    * VM start/stop VM 스케쥴링 설정
    * 일일 또는 주간 스케쥴링 설정

* __다중 사용자 지원__
    * 사용자 등록 페이지
    * 사용자 정보 저장(subscription and etc.)

* __서버 트리거 메시지__

Azure Service Management API를 사용하기 위한 토큰 회득 방법은 [이전 블로그](http://iljoong.github.io/project/2016/08/30/Github-AccessAzure.html)를 참조하시기 바랍니다. 
Billing API의 경우 node.js의 한계(?)로 별도의 .net core API app을 만들어서 처리하였습니다.
그리고, 좀더 유용하게 사용할 수 있도록 스케쥴링 기능 및 다중 사용자를 위한 기능을 추가하였고 테스트 용으로 챗서비스의 API를 노출하여 서버에서 
사용자에게 메시지가 전송될 수 있는 기능을 구현했습니다.

### Service Architecture

![Azmanbot Architecture](/images/azmanbot_arch.png)

> **노트**: 등록 페이지(Usage API)에서 Bot API를 Ajax로 호출하도록 구현되어 있어, Bot API가 호스팅되는 App에서는 CORS 설정(Usage API의 URL)을 해줘야 함 

Bot Framework로 챗봇을 구현하고 퍼블리쉬 하면, 기본적으로 Skype와 자동으로 채널 구성을 해주지만, Slack, Facebook Messenger와 같은 챗서비스는
수동으로 채널 구성을 해야 합니다. 채널 구성은 Azure 배포를 포함하여 아래의 링크를 참조하시기 바랍니다.

[Bot Framework Getting Started](https://docs.botframework.com/en-us/csharp/builder/sdkreference/gettingstarted.html)

챗봇 서비스인 AzmanBot은 2개의 API으로 구성되어 있습니다. Bot API는 node.js 기반의 봇서비스를 위한 API 앱이며, Usage API는 구독의 사용량 정보를 위한 
API 서비스 입니다. 두 API 앱은 Azure Service Management API를 접근하여 필요한 정보 호출 및 제어를 수행합니다.

앞서 언급한 것처럼 자연어 처리를 위해서 LUIS API 서비스를 사용하며, 자세한 내용은 파트 2에서 소개하겠습니다. 챗서비스를 구현하면서 기능이 계속 추가/변경
되고 이에 따른 메시지 형식이 변경이 되기 때문에 먼저 CLI 스타일로 구현하고 이후에 자연어 처리를 하는 것이 개발 생산성을 높일 수 있는 것 같습니다. 

## Key Implementations

Bot Framework를 이용한 챗서비스 구현 및 배포는 이미 홈페이지에 자세히 설명되어 있기 때문에 특별히 설명을 하지 않고 아래의 링크를 참조하시기 바랍니다. 
참고로, 검색을 하면 한글로 설명된 자료도 쉽게 찾을 수 있습니다.

[https://docs.botframework.com/en-us/](https://docs.botframework.com/en-us/)

### Rich Display
텍스트 기반의 대화 표시 뿐만 아니라, 카드와 같은 특별한(Rich) 표현이 가능합니다. 참고로, Prompt 및 카드 표현은 채널(Skype, Slack 등)마다 다릅니다. 
Tsmatz 블로그에 자세한 표현 방법이 소개되어 있으니 참고하시기 바랍니다.

[https://blogs.msdn.microsoft.com/tsmatsuz/2016/08/31/microsoft-bot-framework-messages-howto-image-html-card-button-etc/](https://blogs.msdn.microsoft.com/tsmatsuz/2016/08/31/microsoft-bot-framework-messages-howto-image-html-card-button-etc/)

> **주의:** Skype의 경우 html 태그를 인식하기 때문에 “<>”를 테스트 메시지로 사용하는 경우 내부 오류가 발생함

### User State

특정 구독의 Azure 리소스들을 제어하려면, 사용자의 구독 정보(권한, 스케쥴 추가)를 저장 및 유지해야 합니다. 사용자의 세션정보는 `session.userData` 객체를 
통해서 관리할 수 있지만 서비스의 좀더 persistent한 정보 유지를 위해서 Database와 같은 서비스를 사용할 수 있습니다. 그렇지만, 좀더 가볍게 구현하기 위해서 파일시스템을 
사용하였습니다. 상태 유지 데이터는 Azure App Service 기준으로 `D:\home\site\azmanbotglobal.json` 파일로 저장되며, Bot API App이 업데이트 또는 리부팅 되어도 사용자 정보가 유지됩니다.
모든 상태관련 기능은 [`azstate.js`](https://github.com/iljoong/azmanbot/blob/master/botapi/azstate.js)에서 처리 합니다.

원하는 기능을 구현되었으나, `session.userData`를 좀더 활용하여 빌트인 기능으로 구현하는 방법이 필요해 보입니다.

### Scheduling & server triggered message

별도의 Azure 서비스(Scheduler, WebJob)로 구현할 수도 있지만, 내부 Timer를 이용하여 스케쥴링을 구현했습니다. 즉, VM의 정지 또는 시작은 챗을 통해 바로 
실행하거나 원하는 시간에 실행할 수 있도록 스케쥴링을 설정할 수 있습니다.

스케쥴링 구현은 어렵지 않지만 (설정 커맨들를 파싱하는 것은 좀 복잡), 스케쥴이 동작한 후 서버로부터 챗(또는 메시지)트리거링하는 구현 방법에 대한 문서를 
확인하기 어려웠습니다. BotBuilder Github의 이슈 리스트에서 어느 정도 아이디어를 얻어 구현할 수 있었습니다.

[https://github.com/Microsoft/BotBuilder/issues/659](https://github.com/Microsoft/BotBuilder/issues/659)

최소 한번의 사용자와의 챗팅을 해야 하며, Node.js 기준으로 설명하면, 다이얼로그로 전달되는 [`session.message`](https://docs.botframework.com/en-us/node/builder/chat-reference/classes/_botbuilder_d_.session.html#message) 
객체 정보 값을 활용하여 서버로부터 사용자에게 메시지를 트리거할 수 있습니다. 일단 동작을 하는데, 좀더 나은 방법을 찾고 있는 중입니다.

아래는 스케쥴링에 의해 VM을 시작하고 사용자에게 시작 메시지를 전달하는 코드 일부분입니다.

{% highlight js %}
var msg = JSON.parse(azstate.globalState.UserConf[uid]._message);
if (schedule.cmd == "start") {
       azapi.doStart(uid, schedule.vmid, function (restxt) {
       msg.timestamp = date;
       msg.text = restxt;
     azbot.bot.send(msg);
       });
 ...
{% endhighlight %}

### LINQ

Bot framework는 Node.js로 손쉽게 개발 및 배포가 가능한데 아쉬운 부분이 있습니다. 예를 들어, 이번 달 일별 비용 또는 각 서비스(미터)별 비용을 확인하기 위해서는 
Azure의 [Billing API](https://msdn.microsoft.com/en-us/library/azure/mt218998.aspx)에서 제공되는 두개의 API (Pricing API와 Usage API) 
호출 결과 값을 조인하고 그룹핑해야 합니다. Node.js로 이를 구현하기는 좀 까다롭습니다. 물론 Database에 결과값을 저장하고 필요할 때 쿼리를 하면 되지만 
이를 위해 별도의 서비스를 쓴다는 것은 왠지 비용낭비 같아 다른 방법을 강구했습니다.

그래서 .net의 LINQ를 사용하게 되었습니다. LINQ는 SQL 쿼리처럼 유사하게 조인 및 그룹핑 등의 복잡한 쿼리를 실행할 수 있습니다. 또한, 
Azure App Service(같은 App Service Plan 사용)는 1개의 API 앱으로 모든 기능을 구현하지 않고 2개 이상의 API 앱으로 구현하여도 비용이 동일하기 때문에 
비용효율적 입니다.

다음은 .net core 기반의 API App의 LINQ 쿼리입니다.

{% highlight csharp %}
var query = from x in (from p in pricelist
                 join u in usagelist on p.MeterId equals u.properties.meterId
                 select new { price = p, total = p.MeterRates[0] * u.properties.quantity })
            group x by x.price.MeterName into g
            let gtotal = g.Sum(t => t.total)
            orderby gtotal descending
            select g;
{% endhighlight %}

### Multi-user support

마지막으로, 사용자 구독의 정보, OAuth 접근 정보를 코드에 저장하지 않고 별도의 파일로 관리를 합니다. 또한, 여러 사용자들이 사용할 수 있도록 
구현이 되어 있고 (제대로 테스트는 못했음), 손쉬운 사용자 등록을 위해 별도의 등록 페이지를 제공합니다.

처음 AzmanBot 서비스의 챗을 시작하면 등록 URL이 표시되며 등록을 해야 이후 사용할 수 있습니다.

![Register Page](/images/azmanbot_register.png)

사용자 정보는 파일 시스템으로 저장되며 서비스 업데이트 및 리부트가 되어도 계속 유지됩니다.

다음에는 LUIS를 이용하여 AzmanBot 서비스에 자연어처리 기능을 추가하는 방법에 대해서 소개하도록 하겠습니다.


/iljoong/

