---
layout: post
title:  "Let's make web secure by \"let's encrypt\" - part 2"
date:   2018-03-22 00:00:00 +0900
categories: azure
---

### Part 2

앞서 Let's Encrypt를 Azure의 서비스에 적용하는 방법을 소개했습니다. 한가지 소개하지 못한 부분이 있는데, 바로 Wildcard SSL 인증서를 획득하는 부분입니다. 참고로 Wildcard SSL 인증서는 *.domainname.org와 같이 특정 서브도메인이 지정되지 않은 모든 서브도메인에 사용할 수 있는 SSL 인증서입니다.

이 SSL 인증서를 사용하면 서브도메인을 만들때 마다 인증서를 매번 받아야 하는 불편함이 줄어 듭니다. 물론 일반적으로 가격도 좀도 높습니다. 
또한, wildcard SSL 인증서를 사용하면 이전 블로그 소개한 App Gateway와 같이 native client tool을 지원하지 않는 경우 복잡한 절차/꼼수를 수행할 필요가 없습니다.

얼마전에 wildcard SSL 인증서를 지원하는 Let's Encrypt가 GA가 되었는데, 이번 블로그에서는 추가로 wildcard SSL 인증서 받는 방법과 wildecard SSL 인증서가 필요한 Azure의 App Service Environment 서비스에 적용하는 방법을 소개하도록 하겠습니다. 

### Get Wildcard Certificate

Let's encrypt의 wildcard SSL 인증서는 ACMEv2부터 지원되기 때문에 최신 certbot와 v2 endpoint를 사용해야 합니다. 간단하게 스텝(Linux Ubuntu 기준)을 설명하면 다음과 같습니다. 한가지 다른 부분은 `DNS-01 challenge`가 수행되기 때문에 TXT 형식의 DNS 레코드를 만들어야 합니다.

- 최신 certbot 설치

```
wget https://dl.eff.org/certbot-auto
chmod a+x ./certbot-auto
sudo ./certbot-auto
```

- 스크립트 실행 (DNS-01 challenge)

```
sudo ./certbot-auto certonly --server https://acme-v02.api.letsencrypt.org/directory \
    --manual --preferred-challenges dns -d *.domainname.org
```

- DNS TXT 레코드 추가

스크립트 실행시 DNS와 TXT 레코드의 값이 출력되는데, DNS Zone에 해당 TXT 형식의 DNS (예: `_acme-challenge.domainname.org`) 레코드를 생성하고 TXT 값을 추가합니다.

앞서 소개드린 것 처럼 인증서가 `/etc/letsencrypt/live/domainname.org` 위치에 생성됩니다.

자세한 절차는 제가 참고한 아래를 참고하시기 바랍니다. 

[https://blogs.msdn.microsoft.com/mihansen/2018/03/15/creating-wildcard-ssl-certificates-with-lets-encrypt/](https://blogs.msdn.microsoft.com/mihansen/2018/03/15/creating-wildcard-ssl-certificates-with-lets-encrypt/)

### Get SSL cert for App Service Environment

App Service Environment(ASE)는 Azure의 App Service의 Private/dedicate 버전입니다. 서비스 플랫폼의 리소스(Frontend, publishing, worker VM 등)를 다른 tenant와 공유하지 않고 dedicate 리소스로 사용하는 서비스입니다. 나만의 VNet 안에 생성되어 보안이 좀더 강화되며, 또한 더 많은 인스턴스 사이즈를 가질 수 있습니다.

특히, Internal LB ASE를 사용하게 되면 커스텀 도메인이름과 인증서를 사용해야 합니다. 인증서가 wildcard를 지원해야만 ASE 내의 다수의 web app을 HTTPS로 접근할 때 좀더 관리적으로 편리하고, 특히 KUDU/SCM 사이트는 무조건 HTTPS만 접근할 수 있기 때문에 wildcard 인증서는 필수 입니다.

ASE를 위한 인증서는, 앞서 소개한 스크립트에 다음과 같이 `-d *.scm.domainname.org` 서브도메인을 하나 더 추가합니다.

```
sudo ./certbot-auto certonly --server https://acme-v02.api.letsencrypt.org/directory \
    --manual --preferred-challenges dns -d *.domainname.org -d *.scm.domainname.org
```

도메인 이름이 2개라서 2번의 DNS challenge가 수행됩니다만 (2개의 TXT 형식의 DNS 레코드 필요), 2개의 도메인을 지원하는 wildcard SSL 인증서 1개만 만들어 집니다.

그리고, ASE에서 사용할 PFX 인증서 형식으로 변환 합니다. 즉, VM의 `/etc/letsencrypt/live/domainname.org`으로 이동, 아래의 커맨드 실행합니다.

```
openssl pkcs12 -export -out domainname.pfx -inkey privkey.pem -in cert.pem
```

### Closing

Let's encrypt를 이용한 Wildcard SSL 인증서를 생성하는 방법을 간략하게 소개했습니다. Wildcard SSL 인증서 지원이 얼마 안되서 그런지 자동화된 Tool 들이 아직 보이지 않습니다. Azure에서 native한 Let's Encrypt 지원이 확대되기를 기대해 봅니다. 