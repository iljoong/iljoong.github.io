---
layout: post
title:  "Azure Web App on Linux with Docker"
date:   2016-11-30 00:00:00 +0900
categories: azure
---

## Software appliance and Container
약 10여년전에 가트너의 어느 발표자료에서 [소프트웨어 어플라이언스](https://en.wikipedia.org/wiki/Software_appliance) 라는 용어를 처음 접했을 때, 
곧 소프트웨어 어플라이언스의 시대가 실현이 될 것이라고 생각했습니다. 참고로, 소프트웨어 어플라이언스는 일반 가정의 TV처럼 플러그만 연결하면 바로 쓸 수 있는 
개념으로 많이 설명되며, 즉, 복잡한 구성과 설정없이 그대로 실행만 하면 되는 소프트웨어 말합니다. 하지만, 가상화 및 클라우드 세상에서도 아직도 많은 소프트웨어들은 
복잡한 구성과 설정이 요구됩니다.

Docker로 인해 주목받은 컨테이너 기술은 어떻게 보면 소프트웨어 어플라이언스를 비로서 실현시킬 수 있을 것 같습니다. Docker로 내 애플리케이션의 모든 소프트웨어들을 
하나의 컨테이너 패키지로 만들면 배포를 위한 설정 및 구성을 할 필요도 없고 변경할 할 필요도 없습니다 (_DB연결문자열과 같은 환경변수는 설정 필요_). 또한, 테스트 환경에서는 문제가 없었으나, 
운영환경에서 문제가 발생하는 배포 또는 릴리즈 이슈들을 방지할 수 있습니다. 이런 변경되지 않는 패키지 배포 방식을 **Immutable Pattern**이라고도 이야기 하고, 
**Package Once Deploy Anywhere**라는 이야기도 하고 있습니다. 

이번 블로그에서는 소프트웨어 개발의 화두인 컨테이너 기술에 대해서 간단하게 소개하고 Azure에서 손쉽게 컨테이너 기반의 웹앱을 서비스할 수 있는 
**App Service on Linux (Preview)** 서비스에 대해서 이야기 하고자 합니다. 

## Azure Web App on Linux

**Azure Web App**은 흔히 PaaS 형 웹앱 서비스로 많이 알려져 있습니다. 서비스 아키텍처를 좀 들여다보면, _서비스 플랜_ 이라는 일종의 VM과 Windows Sandbox 
기반으로 실행되는 여러 앱들로 구성되어 있습니다. 앱들은 Web, API, Mobile Backend, Function 앱으로 목적에 따라 실행할 수 있으며 또한, 앱들은 서로 격리된 
상태로 간섭없이 동시에 _서비스 프랜_ 내에서 실행됩니다. 가장 중요한 부분은 Azure Web App 의 비용은 앱들의 수가 아닌 앞서 언급한 _서비스 플랜_ 의 인스턴스 
수로 계산됩니다. 이런 아키텍처는 어떻게 보면 Docker 컨테이너 기술과 매우 흡사합니다.

매우 괜찮은 서비스 이지만, 아직도 Linux와 Java가 웹개발에서 선호되는 한국에서는 Azure Web App이 사실 주목받기 어려웠습니다. 최근 에서야 Azure Web App on Linux가
발표되었지만, 아쉽게도 빌트인 런타임으로 Node.js와 PHP만을 지원해서 어떻게 보면 반쪽짜리 였습니다. 하지만, Connect 2016에서 .Net Core 런타임이 추가되었고 
기대하지 않았던 Docker 컨테이너 및 Private Docker Registry를 지원한다고 발표했습니다. 이번 발표로, Azure App Service (Web App)은 이제야 최고의 PaaS형 
웹앱으로 알려질 것 같습니다.

## Dockerize First!

Tomcat에서 동작하는 Java Web App을 Web App on Linux에 Docker 이미지로 배포하는 방법을 설명하도록 하겠습니다.

가장 먼저 Docker 이미지를 만들고 테스트를 할 수 있는 Docker 엔진 또는 Docker 머신(Windows 경우)이 설치된 개발 클라이언트가 필요합니다. Docker 엔진 또는 머신을
설치할 수도 있지만, 간단히 Azure에 이미 Docker 엔진이 설치된 VM 이미지(Docker on Ubuntu Server)로 VM을 만들어 사용할 수도 있습니다.

그리고, 개발된 Java App을 준비합니다. 본 블로그에서는 tomcat 사이트에 공개된 sample.war을 사용합니다. Docker 이미지 생성을 위해선 
먼저 Docker Image 패키징을 위한 `Dockerfile`을 작성해야 합니다.

### Dockerfile 작성

`./tomcatsample` 디렉토리를 생성하고, 디렉토리 내에 `Dockerfile`을 생성하여 아래와 같이 작성합니다.

{% highlight docker %}
FROM tomcat:8.0

MAINTAINER "iljoong <iljoong@outlook.com>"

ADD server.xml /usr/local/tomcat/conf/server.xml

RUN mv /usr/local/tomcat/webapps/ROOT /usr/local/tomcat/webapps/ROOT_bak
RUN wget -O /usr/local/tomcat/webapps/ROOT.war http://tomcat.apache.org/tomcat-6.0-doc/appdev/sample/sample.war

EXPOSE 80
{% endhighlight %}

참고로, Docker Hub의 기본 tomcat 이미지는 8080 포트로 설정되어 있기 때문에 80 포트로 설정하는 `server.xml` 파일을 추가합니다. 
(Azure Web App은 80 또는 443 포트만 사용 가능하기 때문에 변경이 필요함. 또한, Startup command에서 `-p 80:8080` 이 적용되지 않음)

### Docker Image 빌드

```
$ docker build -t iljoong/tomcatsample .
```

빌드는 앞서 생성한 디렉토리 안에서 실행하기 때문에 “.” 을 빼먹지 않도록 주의가 필요

### 이미지 테스트

```
$ docker run -p 80:80 -d iljoong/tomcatsample
```

### 이미지 업로드

```
$ docker login
$ docker push iljoong/tomcatsample
```

Docker Hub에 이미지를 올리기 위해서는 Docker에 가입되어야 하고, 로그인이 되어 있어야 합니다. 그리고, Docker Hub의 Tomcat과 JRE가 이미 구성된 이미지를 사용하기 
때문에 업로드가 빠릅니다. 정상적으로, 올라가면 아래와 같이 등록됩니다.
 
![Docker Hub](/images/webapplx_image1.png)

Docker 사용에 대한 상세 설명 및 자료는 웹검색 또는 Youtube를 참고하시기 바랍니다.

## Create App Service with Docker 

Web App on Linux (preview)를 Azure 포탈에서 생성하고, 아래와 같이 Docker Container 구성에 이미지를 입력합니다.

![Azure Web App Configure](/images/webapplx_image2.png)

## Private Registry

기업 또는 조직은 Docker Hub 대신 사설 레지스트리인 Azure Container Registry를 사용할 수 있습니다. Docker Hub와 동일한 레지스트리로 Docker Tool과 완벽하게 
호환되고, 또한 기본적인 스토리지 이외에 별도의 비용을 들지 않습니다.


앞서 생성한 Docker 이미지는 아래와 같이 동일한 방식으로 Azure Container Registry로 등록하면 됩니다.

```
$ docker login -u <login name> -p <password> <registry>-microsoft.azurecr.io
$ docker tag iljoong/tomcatsample <registry>-microsoft.azurecr.io/tomcatsample
$ docker push <registry>-microsoft.azurecr.io/tomcatsample
```

아쉽게도 Docker 레지스트 관리 기능은 Azure 포탈에서 제공되지 않고, 별도의 Preview 사이트 ([http://aka.ms/acr/manage](http://aka.ms/acr/manage))에서 확인 가능합니다. 
(아직 삭제 등의 관리는 불가능!)

![Azure Container Registry](/images/webapplx_image3.png)
 
마지막으로 Web App on Linux에서 Azure Container Registry에 등록된 Docker 이미지를 사용하고자 할 때는 아래와 같이 설정하면 됩니다.
 
![Azure Web App Configure - Private Registry](/images/webapplx_image1.png)

## Automation

Azure 포탈에서 Web App을 만들어 Docker 이미지를 배포할 수 있지만, Azure 파워셀 및 ARM 템플릿으로 이런 과정을 자동화 할 수 있습니다. 아직, 관련 파워셀 커맨드가 
제공되지 않고 또한 문서화되지 않았지만 [https://resources.azure.com](https://resources.azure.com) 에서 이미 배포한 Linux 웹앱의 ARM 템플릿을 비교하여 비교적 손쉽게 구성할 수 있었습니다.

기존 Windows 기반의 App Service와 달리 `Microsoft.Web/serverfarms`의 속성 중 `"kind"`: `"linux"` 를 지정해야만 Web App on Linux로 생성이 됩니다. 그리고, 
Docker 이미지 배포는 `Microsoft.Web/sites` 의 `config.appsettings` 에 `DOCKER_CUSTOM_IMAGE_NAME` 를 추가하고 해당 이미지를 입력하면 앱이 생성됩니다.
사설 레지스트의 Docker 이미지로 앱을 생성하고자 하면, 추가로 아래의 3개 앱설정 값을 추가하여 지정하면 됩니다.

`DOCKER_REGISTRY_SERVER_URL`, `DOCKER_REGISTRY_SERVER_USERNAME`, `DOCKER_REGISTRY_SERVER_PASSWORD`

{% highlight json %}
{ 
 "$schema": "http://schema.management.azure.com/schemas/2014-04-01-preview/deploymentTemplate.json#", 
 "contentVersion": "1.0.0.0", 
 "parameters": { 
...
  }, 
  "variables": {
     "webappSite": "[parameters('siteName')]"
  },
  "resources": [ 
    { 
      "apiVersion": "2015-08-01", 
      "name": "[parameters('appServicePlanName')]", 
      "type": "Microsoft.Web/serverfarms", 
      "location": "[parameters('location')]", 
      "kind": "linux",
      "properties": { 
        "name": "[parameters('appServicePlanName')]",
        "kind": "linux",
        "workerSize": 0,
        "numberOfWorkers": 1,
        "reserved": true
      },
      "sku": {
        "name": "S1",
        "tier": "Standard",
        "size": "S1",
        "family": "S",
        "capacity": 1
        }
    }, 
    { 
      "apiVersion": "2015-08-01", 
      "name": "[variables('webappSite')]", 
      "type": "Microsoft.Web/sites", 
      "location": "[parameters('location')]",
      "dependsOn": [ 
        "[resourceId('Microsoft.Web/serverfarms', parameters('appServicePlanName'))]" 
      ], 
      "properties": { 
        "name": "[variables('webappSite')]", 
        "serverFarmId": "[parameters('appServicePlanName')]" 
      }, 
      "resources": [ 
         {
         ... 
         },
         {
            "apiVersion": "2015-08-01", 
            "name": "appsettings", 
            "type": "config", 
            "location": "[parameters('location')]",
            "dependsOn": [ 
              "[resourceId('Microsoft.Web/Sites', variables('webappSite'))]" 
            ], 
            "properties": { 
                "DOCKER_CUSTOM_IMAGE_NAME": "[parameters('dockerImage')]"    
           }
         }
       ] 
     } 
   ] 
 } 
{% endhighlight %}

위의 템플릿은 아래의 Azure 파워셀을 실행하여 자동화할 수 있습니다.

{% highlight powershell %}
$webAppName = 'mytomcatsample'
$appServicePlan =  'TEST-LINUXPLAN'
$resourceGroupName = 'TEST-RESGROUP'
$location = 'Southeast Asia'

$dockerImage = "iljoong/tomcatsample"

Write-Output  ">>>1.CREATE WEB APP: $webAppName"
	
$parameters = @{"siteName"=$WebAppName; "appServicePlanName" = $appServicePlan; "dockerImage" = $dockerImage; "location" = $location }

New-AzureRmResourceGroup -Name $resourceGroupName -Location $location
New-AzureRmResourceGroupDeployment -ResourceGroupName $resourceGroupName -TemplateFile "dockerjavawebapp.json"  -TemplateParameterObject $parameters -Verbose 

{% endhighlight %}

Azure Web App on Linux에 Docker 이미지로 앱을 생성하는 샘플 ARM 템플릿과 파워셀 스크립트는 아래의 참고하시기 바랍니다.

[Sample Template and PowerShell 확인](https://github.com/iljoong/azuresample-webapplinux)

## But… It’s still in preview

아직은 프리뷰라서 그런지 모든 기능이 다 갖추지는 못했습니다. KUDU의 기능도 매우 제한적이고 (온라인 편집 불가능) Docker 이미지로 배포된 앱은 KUDU를 통한 이미지 
내부의 관리가 불가능합니다. 참고로, Docker에서 아래의 커맨드로 Docker 이미지 내의 bash를 실행하여 내부로 접속할 수 있지만, Web App on Linux에서는 아직은 접근이 
불가능합니다.

```
$ docker exec -it <psid> bash
```

그리고, Azure Container Registry 또한 Admin 사용자만 접근할 수 있고 AAD와 아직 연계가 안되어 개별 사용자들의 접근 및 권한 관리가 불가능합니다. 예를 들어 일반 
사용자는 Pull만되고 Push는 안되게 하는 관리가 현재는 불가능 합니다.

Docker를 고려하는 고객들에겐 서버의 자원을 최적화/밀도 높게 사용할 수 있는 Docker 컨테이너가 매우 매력적이지만 컨테이너 클러스터를 구성하고 관리하는 것은 좀
고민스러웠을 수 있습니다. Web App on Linux는 이런 고민을 해결해 주었고 손쉬운 사용성을 함께 제공합니다.
아직은 프리뷰라서 좀더 개선이 필요하고 가용 DC지역이 더 확대되어야 되어야 겠지만, Linux를 선호하는 국내에서 마이크로사이트 및 소규모 LOB성 웹사이트에 많이 활용될 수 있을 것 같습니다.

/iljoong/
