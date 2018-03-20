---
layout: post
title:  "Let's make web secure by \"let's encrypt\""
date:   2018-03-20 00:00:00 +0900
categories: azure
---

### Intro

안전한 웹사이트 구축 또는 인터넷 사용을 위해서 SSL/TLS 통신이 요구되지만 일반 개발자/운영자가 SSL 인증서를 구매하기에는 비용이 부담스러운 것이 사실입니다. 고맙게도 [Let's encrypt](https://letsencrypt.org/)라는 비영리재단에서 Domain Name 소유자에게 무료로 SSL 인증서를 제공하고 있습니다.

SSL 인증서를 메일과 같은 일반적인 방식으로 받는 것은 아니고 ACME protocol을 이용하여 인증서를 획득하는데, certbot 또는 다양한 지원 client tool을 이용하여 손쉽게 인증서를 받고 적용할 수 있습니다. 인증서 유효기간이 3개월이라 3개월 마다 renewal을 수행해야 하지만 많은 client tool들이 자동 renewal을 지원하기 때문에 큰 이슈는 아닙니다. 저는 주로 테스트 용도로 사용해서 3개월 기간은 충분하기도 합니다.

본 블로그에서는 웹사이트 운영에 사용되는 Azure의 다양한 서비스에 대해서 SSL 인증서를 적용하는 방법을 아래에서 상세하게 설명합니다.

### Buy domain name

웹사이트에 SSL 인증서를 적용하기 위해서는 당연하게도 자신의 도메인이 필요합니다. 도메인 구매는 국내외 Domain Register를 통해서 구매할 수 있습니다. 참고로 [가비아](https://domain.gabia.com/) 기준 __.xyz__ 도메인은 커피 한잔 값도 안되는 2천원 정도라서 큰 부담이 안됩니다. 하지만, 이 가격은 Promotion 가격이라서 1년 후에는 일반 가격으로 높아집니다. 저는 주로 테스트 용도로만 사용할 도메인 이라서 매년 새로운 도메인 (언제까지 Promotion이 되는지 모르지만)으로 옮기는 방법을 사용합니다.

### Delegate domain to Azure DNS

도메인을 구매한 사이트의 관리툴을 이용할 수도 있지만, 좀더 편리한 관리를 위해 Azure의 도메인 위임을 미리 구성하는 것을 권장합니다.
자세한 정보는 아래 링크 참조 하시기 바랍니다.

[https://docs.microsoft.com/ko-kr/azure/dns/dns-delegate-domain-azure-dns](https://docs.microsoft.com/ko-kr/azure/dns/dns-delegate-domain-azure-dns)

간략하게 설명하면, Domain Register(예: 가비아)의 도메인 관리 설정에 있는 네임서버 1~4를 Azure의 네임서버 1~4로 변경해주면 됩니다. 주의할 사항은 위임은 바로 적용되는 않고 변경 후 약 1시간 정도 지나야 위임이 완료됩니다.

위임이 완료되었는지는 `nslookup -type=SOA domainname.xzy`으로 확인이 가능합니다.

### Get SSL cert for VM(Nginx/Linux) & App Service

VM과 App Service에 대해서 let's encrypt을 이용하여 SSL 인증서를 획득하는 방법은 검색하면 쉽게 찾을 수 있어 여기서는 자세한 설명은 하지 않습니다. 참고로 VM과 App Service는 아래를 참고하시기 바랍니다.

- [VM(Nginx/linux)](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04)

- [App Service](https://gooroo.io/GoorooTHINK/Article/16420/Lets-Encrypt-Azure-Web-Apps-the-Free-and-Easy-Way/20073#.WqCm5ujFKU)

그리고, Client Tool은 아래의 링크를 참조하시기 바랍니다.

[https://letsencrypt.org/docs/client-options/](https://letsencrypt.org/docs/client-options/)

한 가지 주의할 부분은 인증서를 획득하는 과정에서 `http-01 challenge`가 실행되는데, 해당 VM/App Service에서 challenge가 정상적으로 수행될 수 있도록 80 port 및 challenge URL(예: `/.well-known/acme-validation/XYZ`)이 정상으로 접근되어야 합니다.

### Get SSL cert for App Gateway

App Gateway는 아쉽게도 Native한 client tool이 아직 제공되지 않으므로 좀 귀찮은 수작업 또는 꼼수(?)가 필요합니다.즉, VM(Nginx/linux) 방법을 활용하여 App Gateway에 사용할 도메인으로 SSL 인증서 미리 획득하고, App Gateway에 적용합니다. 참고로 VM의 SSL 인증서는 App Gateway에서 바로 사용할 수 있는 인증서 형식이 아니라서 변환(pfx 인증서)이 필요합니다.  

상세 스텝는 아래와 같습니다.

- VM(nginx/linux) 생성 및 DNS 등록 (임시)

- SSL 인증서 획득

- PFX 인증서 형식으로 변환

VM의 `/etc/letsencrypt/live/domainname`으로 이동, 아래의 커맨드 실행

```
openssl pkcs12 -export -out domainname.pfx -inkey privkey.pem -in cert.pem
```

PFX 인증서는 안전하게 옮길 수 있는 인증서 형식이라서 암호를 입력해야 하며, PFX 인증서를 import할 때 이 암호를 입력해야합니다.

- App Gateway에 SSL 인증서 추가

App Gateway가 미리 만들어 졌다면, Listener에 HTTPS를 설정하면서 PFX 형식의 SSL 인증서를 추가하거나 또는 App Gateway를 생성할 때 HTTPS Listener에 추가하면 됩니다.

[https://docs.microsoft.com/ko-kr/azure/application-gateway/application-gateway-ssl-portal](https://docs.microsoft.com/ko-kr/azure/application-gateway/application-gateway-ssl-portal)

- DNS 재등록

Azure의 DNS에서 VM(A record 형식)으로 등록된 도메인은 App Gateway(CNAME record 형식)으로 변경되지 않기 때문에 DNS를 삭제후 다시 등록 합니다.

- 완료

정상적으로 SSL 인증서가 적용되면 HTTPS 접속시 아래와 같이 표시됩니다.

![SSL Web](/images/azure-letsencrypt-sslweb.png)

### Closing

Let's encrypt를 이용한 Azure Service에 SSL 인증서를 적용하는 방법에 대해서 알아 보았습니다. 아직 자세히 확인하지 못했지만, 몇 일전부터 [Wildcard SSL 인증서를 지원](https://community.letsencrypt.org/t/acme-v2-and-wildcard-certificate-support-is-live/55579)하기 시작했습니다. Let's encrypt의 활용도는 보다 확대될 것 같습니다.