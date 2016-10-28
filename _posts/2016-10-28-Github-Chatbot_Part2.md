---
layout: post
title:  "Building Chatbot for Azure Management - Part 2"
date:   2016-10-28 00:00:00 +0000
categories: project
---

# Building Chat bot for Azure Management – Part 2

## Continue ...

앞서 AzmanBot의 기본 챗봇 서비스 기능을 소개하였습니다. 좀더 휴먼 친화적인 대화형 인터페이스를 위해서는 CLI 스타일의 채팅보다는 자연어 처리가 되는 
채팅이 보다 선호될 것입니다. (하지만, 솔직히 CLI가 익숙해지면 자연어처리 보다 간결하고 편리하기도 합니다.)

그리고, 요즘 IT의 화두는 AI (인공지능) 입니다. 자연어처리는 오래된 인공지능의 한분야이지만 최근 들어 놀라운 성과가 나오기 시작한 것 같습니다. 
최근에 Apple Siri, Amazon Alexa, Microsoft Cortana와 같은 자연어를 처리가 적용된 상용 서비스들이 봇물처럼 출시되고 있고, 사용자들이 경험이 증가되고 
있어 이런 서비스는 다양한 곳에 더욱더 확대될 것으로 봅니다.

(_아래 이미지 클릭으로 동작 확인_)

[![Watch Demo](https://img.youtube.com/vi/pgbrDQFqMDc/0.jpg)](https://youtu.be/pgbrDQFqMDc)

[Goto source code](https://github.com/iljoong/azmanbot/tree/luis)

## LUIS

자연어 처리는 어려운 구현이지만, Azure의 [Cognitive Service](https://azure.microsoft.com/ko-kr/services/cognitive-services/)의 하나인 
LUIS(Language Understanding Intelligence Service)를 이용하면 정말로 손쉽게 구현이 가능합니다. LUIS의 웹도구와 API를 통해서 어려운 
자연어 처리를 마치 엑셀로 복잡한 회계를 관리할 수 있는 것처럼 도구화해 주었습니다. 
아래의 10분짜리 동영상만 시청하면 바로 즉시 LUIS를 기본 기능을 사용할 수 있습니다.

[LUIS Tutorial](https://www.luis.ai/Help) 

LUIS는 Cognitive Computing의 강자로 익숙한 IBM의 Watson의 유사 서비스인 [Natural Language Classifier](https://www.ibm.com/watson/developercloud/nl-classifier.html)
보다 더 나은 기능과 도구를 제공하는 것 같습니다.

아쉽게도 아직 한국어는 지원하지 않지만 현재 중국어와 일본어를 지원하기 때문에 한국어도 곧 지원하기를 기대해 봅니다. 본 샘플 앱은 영어기반으로 구현했습니다.

## Modeling with LUIS

LUIS 사용을 간단히 설명을 하면, 애플리케이션 도메인 또는 컨텍스트에 해당하는 대화를 먼저 모델링(학습) 시킨 후 이를 API로 퍼블리슁 합니다. 그리고, 
이 API를 소비하는 애플리케이션을 구현합니다. 

모델링은 아래와 같이 수행합니다.

1.	새로운 LUIS 애플리케이션 생성

2.	Intent(질문의 의도)를 생성하고 각 Intent(질문의 의도)에 해당하는 Utterance(실제 대화 내용/문장)를 5개 정도 학습시켜 모델링

    * 예: `stop (azurevm)`, `shutdown (azurevm)`, `turn off (azurevm)`

3.	대화 내용에서 Entity(개체)를 모델링 도구를 이용하여 식별.

4.	참고로, 숫자 및 날짜와 같은 빌트인 엔티티를 사용하거나 또는 사용자 정의 엔티티를 추가하여 식별

    * 예: 제품이름과 같이 패턴이 있는 이름의 경우 정규식 엔티티(Regex Entity)로 적용. 참고로 VM명을 인식하기 위해서 `( )`를 패턴으로 사용하였으며, LUIS는 `( )`를 제외하고 VM명을 추출해 줌

5.	모델링 또는 학습이 완료된 후 퍼블리싱하며, 필요시, 오류가 발생하지 않도록 재학습

    * __주의__: 재학습이 되어도 퍼블리싱이 해야만 API에 최종적으로 적용됨 

LUIS의 자세한 내용은 [LUIS Documentation](https://www.microsoft.com/cognitive-services/en-us/LUIS-api/documentation/home)를 참조하시기 바랍니다. 

그리고, Github에 공개한 본 [샘플앱의 LUIS app](https://github.com/iljoong/azmanbot/tree/luis/botapi/luisapp)을 참조하세요.

## Integration LUIS into App

Node.js기반 챗봇 앱의 LUIS 통합은 아래의 문서링크를 참조하시기 바랍니다.

[Bot Framwork - Understanding Natural Language](https://docs.botframework.com/en-us/node/builder/guides/understanding-natural-language/#navtitle)

그리고, LUIS를 적용한다고 해서 기존 코드의 변화는 크지 않습니다. 마치 Bot Framework를 염두해두고 LUIS가 개발된 것처럼 일부분만 수정하면 자연어 처리를 
구현할 수 있습니다.

다음과 같이 순서로 LUIS를 통합 시킵니다.

1.	퍼블리싱된 LUIS 애플리케이션 API 연결 코그 추가

2.	모델링된 Intent 핸들링 구현

3.	Intent 내의 엔티티 처리. 미리 정의된 엔티티(예: 날짜 엔티티) 자동 인식 처리됨.

좀더 자세히 설명하면, LUIS 모델링이 완료되면 기존 다이얼로그에서 정규식으로 이식했던 `intents`는 `LuisRecognizer` 방식으로 변경합니다.

{% highlight js %}
var model = 'https://api.projectoxford.ai/luis/v1/application?id=<app>&subscription-key=<key>';
var recognizer = new builder.LuisRecognizer(model);
var intents = new builder.IntentDialog({ recognizers: [recognizer] });
bot.dialog('/', intents);
{% endhighlight %}

그리고, `intents`는 기존 정규식(예: `intents.matches(^/list/I, `로 매칭하던 방식에서 Luis intent (예: `intents.matches('List', `로 변경합니다.

기존 CLI 방식에서 패러미터를 식별하는 방식에서 자연어로 엔티티를 식별하는 방식으로 변경되었기 때문에 이 부분의 코드 변경은 필요합니다. 
간단한 Intent 처리는 아래와 같이 손쉽게 처리할 수 있습니다. 참고로 Intent의 score가 50% 미만이면 해당 Intent를 실행하지 않거나 또는 다른 선택을 유도합니다.

{% highlight js %}
intents.matches('List', [
    (session, args, next) => {
        if (args.score < 0.5) {
            session.beginDialog('/dontknow');
            return;
        }

        var uid = session.message.user.id;

        azapi.getList(uid, function (result) {

            session.userData.list = {};

            for (var i = 0; i < result.value.length; i++) {
                session.userData.list[result.value[i].name] = { 'name': result.value[i].name, 'id': result.value[i].id };
            }

            session.beginDialog('/list');
        });
    }
]);
{% endhighlight %}

스케쥴링과 같은 Intent는 날짜와 관련된 속성을 인식하여 처리하는 부분이 필요합니다. 좀 어렵게 보이기는 하지만 복잡한 날짜/시간의 자연어 처리에 대해서
LUIS의 장점이 확실히 발휘됩니다. 예를 들어 기존 CLI 방식은 `sch set start vmname 8:55am`
와 같이 자연스럽지 못하고 설정이 유연하지 못했는데, LUIS를 이용하여 `set schedule to start (vmname) at 8:55 am` 또는 
`set schedule at 8:55 am to start (vmname)`와 같이 좀더 자연스럽고 엔티티 입력 순서에 얽매이지 않으며 `at 8:55 am on every Monday`,
`at 8:55 am on 2016-10-31`와 같은 다양한 표현이 가능합니다.

{% highlight js %}
if (args.entities[0]) {
            args.entities.forEach(function (element) {

                switch (element.type) {
                    case 'vmname':
                        _vmname = element.entity;
                        break;
                    case 'cmdtype':
                        _cmd = element.entity
                        break;
                    case 'builtin.datetime.time':
                        _time = azbot.parseTime(element.resolution.time);
                        break;
                    case 'builtin.datetime.set':
                        if (element.resolution.set.match(/XXXX-WXX-/)) {
                            //XXXX-WXX-2
                            _offsetday = getOffsetday(element.resolution.set.match(/\d/));
                            _rep = 86400000 * 7;
                        } else {
                            _rep = (element.resolution.set === 'XXXX-XX-XX') ? 86400000 : 86400000 * 7;   //XXXX-XX-WXX
                        }
                        break;
                    case 'builtin.datetime.date':
                        //element.resolution.date = XXXX-WXX-5
                        _offsetday = getOffsetday(element.resolution.date.match(/\d/));
                        break;

                    default:
                        var _log = util.format("%s", element.type);
                        console.log(_log);
                }

            }, this);
{% endhighlight %}

참고로, `at 8:55 am on every Monday`와 같은 날짜/시간 문장에 대해서 아래와 같은 엔티티 정보가 전달되며 처리합니다.

* `at 8:55 am` : `Builtin.datetime.time` 속성에 값은 `T08:55`

* `on every Monday` : `builtin.datetime.set` 속성에 값은 `XXXX-WXX-1`

미리 정의된 엔티티에 대해서는 아래의 문서를 참조하기 바랍니다.

[https://www.luis.ai/help#PreBuiltEntities](https://www.luis.ai/help#PreBuiltEntities)


## Conclusion

AzmanBot 샘플 앱을 통해서 Bot Framework를 이용한 챗봇 서비스 구현과 LUIS를 이용한 자연어 처리 구현 방법에 대해서 소개 드렸습니다. 저도 실력이 부족하고
깊이 알고 있지 못해 많은 내용을 담지 못한 짧은 소개였지만, 챗봇 기반의 대화형 서비스 개발을 생각하시는 분들께 도움이 되었으면 합니다.

/iljoong/

