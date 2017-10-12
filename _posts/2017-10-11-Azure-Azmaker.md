---
layout: post
title:  "Building Desktop Azure Console App"
date:   2017-10-11 00:00:00 +0900
categories: azure
---

# Building Desktop Azure Console App

오랜만에 블로그를 업데이트 합니다...

앞선 블로그에서 Azure SDK for node.js를 이용하여 Azure 서비스를 프로비저닝 자동화하는 방법에 대해서 소개했었습니다.

커맨드 형태의 콘솔앱으로 손쉽게 실행할 수 있지만, Azure 구독정보와 서비스에 필요한 여러 패러미터를 입력하는 데는 좀 불편합니다. 그래서 좀더 편리한 UI 기반의 데스크탑 앱을 생각하게 되었습니다.

데스크탑 앱을 위해서 처음부터 새로 개발하는 것보다는 기존 node.js 코드를 그대로 활용할 수 있으면 매우 좋을 것이라고 생각했습니다. [Electron](https://electron.atom.io/) 이라는 node.js 기반의 프레임워크가 매우 활발히 이런 용도로 사용되고 있다는 것을 알고 Electron 기반으로 UI 콘솔앱인 AzMaker를만들어 보았습니다.

## AzMaker

먼저, [소스코드](https://github.com/iljoong/azure-azmaker)를 다운을 받고 npm 패키지를 설치하고 실행 시킵니다.

```
npm install
npm start
```

![azmaker](/images/azmaker-subs.png)

앱을 처음 실행하면 먼저 메인 화면의 기어 아이콘(&#9881;)을 클릭하고 `Edit Subscription`을 선택하여 구독정보를 추가합니다. 구독 정보에 Azure 리소스를 생성할 수 있는 권한을 가지 __service principal__ 정보와 동일 합니다.
 
구독 정보 파일은 사용자 홈디렉토리(예 `C:\Users\name\`)에 `azmaker.json` 이라는 파일명으로 저장됩니다.

![azmaker](/images/azmaker-main.png)

그리고, `Demo->Azure PaaS/DR(Fotos)`를 선택합니다.

이전 블로그에서 소개한 iFOTOS BC/DR 데모앱을 구성하는 샘플입니다. 데모앱 생성을 위한 기본(Basic)과 앱(Apps) 정보를 입력하거나 `Load`를 클릭하여 `sample\demo_foto.json` 샘플 설정 파일을 선택합니다.

`Create` 버튼을 클릭하여 데모앱을 생성합니다. 데모앱을 생성하는데는 시간이 소요됩니다. 기어아이콘(&#9881;), `Show devtool`을 선택하여 Chrome의 log 창으로 디버깅 로그를 확인할 수 있습니다.

Node 패키지의 종속성 없이 독립 실행 파일로 실행하고자 하면, `electron-packerger` 툴로 exe 파일을 빌드할 수 있습니다.

```
npm run build
```
/iljoong/
