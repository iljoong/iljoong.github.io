---
layout: post
title:  "Cloud Native Security Practice"
date:   2018-04-16 00:00:00 +0900
categories: azure
---

퍼블릭 클라우드를 도입할 때 가장 민감한 부분은 **보안** 일 것 입니다. 퍼블릭 클라우드가 외부 해킹에 더 안전하다고 알려져 있지만 퍼블릭 클라우드의 사용이 확대되면서 사용자의 부주의에 의한 보안 사고가 공공연하게 발생되고 있습니다.

일반적인 보안과 관련된 체크리스트 보면 대부분 온프레미스에 기반한 항목이다 보니 애플리케이션/서비스 보다는 DC 인프라에 중점된 항목들로 퍼블릭 클라우드에서 운영될 서비스는 새로운 관점의 보안이 요구됩니다.

클라우드의 기본적인 보안 아키텍처는 아래의 PCI DSS 보안 사항에 대응하는 Azure 보안 아키텍처를 참조하는 것을 권장드립니다. 상세 내용은 [PCI DSS Blueprint](https://docs.microsoft.com/ko-kr/azure/security/blueprints/payment-processing-blueprint)를 참조하시기 바랍니다.

![PCI Security Architecture](https://docs.microsoft.com/ko-kr/azure/security/blueprints/images/pci-architectural-diagram.png)

위의 보안 아키텍처에서 표시된 보안 사항들과 실경험을 토대로 본 블로그에서는 Azure에서 핵심 보안 프랙티스들을 적용하는 내용을 소개하고자 합니다. 

아래는 퍼블릭 클라우드에서 핵심적으로 사용되는 보안 프랙티스이며 좀더 상세하게 소개하겠습니다.

1. Firewall/NSG
2. WAF
3. Jump Box
4. Key Management & IAM
5. Outbound SNAT

### 1. Firewall/NSG

Virtual Network내의 방화벽 목적으로 NSG(Network Security Group)를 사용하여 Inbound/Outbound 트랙픽에 대한 Allow/Deny를 설정할 수 있습니다. Azure에서는 기본적으로 외부 Inbound에는 제한을 두고 Virtual Network 내에서는 모두 열어 두는 룰 정책을 가지고 있습니다. Outbound는 기본적으로 모두 열려 있습니다.

하지만, 이런 정책은 대부분의 기업에서 선호하는 정책이 아니기 때문에 Inbound/Outbound의 모든 트랙픽을 기본적으로 막는 룰로 변경하고 하나씩 여는 정책을 선호 합니다.

이렇게 설정한 경우 흔히 격는 이슈는 정상적인 통신이 안되는 경우를 경험하게 됩니다. 특히, NSG 룰에 Source IP와 Destination IP를 정확하게 입력했고 VM간의 통신은 잘되는데 Internal LB를 통하는 경우 안된다고 어려움을 호소하는 경우을 가끔 봅니다. 원인은 LB의 Health Probing이 제대로 되지 않는 경우이며 ILB를 위한 NSG rule을 추가해야 합니다. 참고로 Azure 모든 LB는 `168.63.129.16` IP 주소를 가지고 있으며 이 주소로부터 오는 Health probing 트래픽을 열어주어야 합니다. (아쉽게도 기본 LB의 metrics가 제공되지 않아 Backendpool의 health를 쉽게 확인하기 어렵습니다. 표준 LB는 metrics가 제공되어 상태확인 가능합니다.)

| Priority | Rule Name      | Source (IP:Port)       | Destination | Action |
|:---:|:---:|:---:|:---:|:---:|
| 4095     | AllowLB        | AzureLoadBalancer:&#42;| &#42;:&#42; | Allow  |
| 4096     | DenyAllInBound | &#42;:&#42;            | &#42;:&#42; | Deny   |


세부 설정은 [Terraform으로 작성한 NSG 설정 예](https://github.com/iljoong/azure-terraform/blob/master/DOC.md#asg) 참고하시기 바랍니다.

복잡한 구성에서는 IP Prefix 주소로 NSG 를을 편집하는 것이 매우 힘들 수 있습니다. 이때, Tag 또는 Label 방식으로 NSG 룰을 편집할 수 있는 ASG(Application Security Group)가 있으며 최근 GA가 되었습니다. ASG를 활용하면 좀더 편리하게 NSG 룰을 편집할 수 있으며, Blog에 소개된 ASG 내용을 참조하시기 바랍니다.

[https://azure.microsoft.com/ko-kr/blog/applicationsecuritygroups/](https://azure.microsoft.com/ko-kr/blog/applicationsecuritygroups/)

NSG가 AWS의 SG(Security Group)와 많이 비교되는데, 유사하지만 실사용 방식은 좀 차이가 있습니다. Azure의 NSG가 좀더 flexible 하지만 반대로 AWS는 심플하기 때문에 적용방법의 호불호가 있습니다. 즉, AWS의 SG는 Allow 설정만 가능하기 때문에 여러 SG 들을 순서없이 복합적으로 적용할 수 있는 반면 Azure는 Deny도 설정할 수 있어 여러 NSG 들을 복합적으로 적용할 수 없습니다. Deny가 존재하는 NSG의 rule들은 우선순위가 있기 때문에 여러 NSG의 복합 적용이 불가능 하지만, 반대로 Deny가 제공되는 Azure에서는 Blacklist 설정할 수 있는 장점도 있습니다.

복잡하다고 좋은 정책은 아니며 Inbound/Outbound 트래픽을 정확하게 이해하고 필요한 부분만 적용하는 것이 향후 관리에도 좋은 것 같습니다.

### 2. WAF

웹방화벽은 외부 노출이 있는 서비스의 필수 보안 항목입니다. 잘 알려진 상용 웹방화벽 솔루션들을 권장하지만, 비용이 부담스럽다면 App Gateway + WAF가 좋은 대안이 될 수 있습니다.

App Gateway는 쉽고 간편하게 구성할 수 있지만 아쉽게도 구성 시간이 상대적으로 많이 소요됩니다. 특히, App Gateway 생성 이후 세부 구성 요소들을 설정할 때에도 느린 설정 속도로 답답하거나 또는 설정이 완료되지 않은 상태에서 추가 설정으로 인해 오류가 발생하는 경우도 있습니다.

CLI 스크립트 또는 Terraform을 이용한 App Gateway + WAF를 생성/설정하면 이런 불편함을 좀더 최소화 시킬 수 있습니다. 아래의 링크를 참조하고, 스크립트로 실행 시 대략 생성과 설정에 대략 25분정도 소요됩니다. 

[https://github.com/iljoong/azure-tf-waf](https://github.com/iljoong/azure-tf-waf)

WAF 설정 시 Detection 또는 Prevention 모드를 선택할 수 있는데, 먼저 Detection 모드로 선택하고 운영한 후 Prevention 모드로 전환하는 것을 권장 드립니다. WAF Monitoring은 Security Center에서 된다고 하지만 Log Analytics와 연동이 필요합니다. ApplicationGatewayFirewallLog를 Log Analytics로 수집하여 모니터링 및 Alert를 설정할 수 있습니다. 아래는 Log Analytics의 모니터링 쿼리입니다.

```
# WAF Detection/Block 확인 쿼리
AzureDiagnostics 
| where Category == "ApplicationGatewayFirewallLog" 
| summarize count() by bin(TimeGenerated, 1h), action_s 
| render barchart

# WAF Detection 쿼리 – 시간별 해킹시도 클라이언트 IP 및 시도횟수
AzureDiagnostics 
| where Category == "ApplicationGatewayFirewallLog" and action_s == "Detected" 
| summarize dcount(ruleId_s) by bin(TimeGenerated, 1h), clientIp_s 
| render barchart

# Top 10 WAF Detection 확인 쿼리
AzureDiagnostics 
| where Category == "ApplicationGatewayFirewallLog" and action_s == "Detected"
| summarize count() by ruleId_s
| top 10 by count_
```

현재 Azure App GW + WAF는 아직 Static Public IP를 지원하지 않습니다. IP 변경이 가능하다지만, 운영 중에는 IP 변경이 거의 발생하지는 않고 App GW를 정지/시작 할 때 IP가 변경됩니다. 하지만, Azure Portal에서는 정지/시작 기능이 보이지는 않고 CLI에서만 지원합니다.

```
az network application-gateway [stop | start] -n <appgw name> -g <resource group>
``` 

일부 기업에서 RFC 1918에서 정의한 영역대가 아닌 IP가 충돌나는 Private IP 주소 사용하는 경우가 간혹 있는데, 이 때 APP GW를 정지/시작 시켜 충돌나지 않는 IP로 변경할 수 있습니다. (여러 번의 시도가 필요할 수도 있음) 

추가로 Web App과 같은 Multi-tenant PaaS 서비스에 WAF를 적용할 경우, DNS/IP가 외부에 직접 노출되어 있기 때문에 WAF로부터의 트래픽만 받을 수 있도록 Web App을 설정해야 합니다. 자세한 내용은 아래를 참조하세요.

[https://azure.microsoft.com/en-us/blog/ip-and-domain-restrictions-for-windows-azure-web-sites/](https://azure.microsoft.com/en-us/blog/ip-and-domain-restrictions-for-windows-azure-web-sites/)

### 3. Jump Box

퍼블릭 클라우드의 IP 주소가 공개되어 있어 요즘은 Public IP가 외부로 노출된 VM 들은 엄청난 RDB/SSH Brute-Forace 공격을 받습니다. 이런 공격을 피하기 위해서는 외부 노출은 최소화하고 외부 접근이 필요한 웹서버와 같은 VM만을 DMZ에 위치하도록 구성합니다. 관리자 또는 운영자들은 VM에 애플리케이션을 설치한다 거나 설정을 하기위한 안전한 접속 방법이 필요한데, [Bastion Host](https://en.wikipedia.org/wiki/Bastion_host) 또는 Jump Box 형태의 스페셜 VM을 두어 외부에서 모든 연결은 여기를 통해서만 접근하도록 구성 합니다.

Jump Box를 연결할 때 잘 알려진 포트번호(SSH 22 / RDP 3389) 대신 임이의 포트번호(예: 50022)를 사용하는 것을 권장합니다. VM에서 리스너 포트를 변경하는 것 보다는 Azure에서는 LB을 생성하고 Port-forwarding 또는 [Inbound NAT rule](https://docs.microsoft.com/en-us/cli/azure/network/lb/inbound-nat-rule?view=azure-cli-latest) 설정하여 사용할 수 있습니다. 좀더 보안을 강화하기 위해서는 지정된 시간에만 접근할 수 있도록 제한하는 Security Center의 [JIT access](https://docs.microsoft.com/ko-kr/azure/security-center/security-center-just-in-time) 기능도 고려할 수 있습니다.

사용자 PC에서 Jump Box에 로그인하여 내부 VM들을 접근할 수도 있으나, Jump Box가 Linux OS인 경우 SSH 터널링을 통해 PC에서 직접 내부 VM을 접근할 수 있도록 터널링 구성을 할 수도 있습니다. SSH 터널링은 PuTTY를 이용하거나 Bash on Ubuntu on Windows 콘솔에서 커맨드로 실행시킬 수 있습니다.

```
ssh  -nNT -L <localport>:<remote ip>:<remoteport> <user>@<ssh server>
```

외부 접근이 불가능한 관리자 콘솔 또는 SQL 서버는 PC에서 SSH 터널링을 통해 PC의 브라우저 또는 클라이언트 도구로 연결하여 편리하게 관리할 수 있습니다. 그리고, 파일 편집을 종종하게 되는데 간단한 파일 편집은 VI editor로 가능하지만 복잡한 파일 편집은 아무래도 VI가 편리하지 않을 수 있습니다.

이런 경우, [VSCode](https://code.visualstudio.com/)와 [RMate](https://github.com/textmate/rmate) 이용한 리모트 에디팅을 고려할 수 있습니다. 리모트 에디팅을 실행하기 위해서는 먼저 VSCode의 Remote VSCode Extension과 리모트 VM에 RMate가 미리 설치되어 있어야 합니다. 그리고, VSCode의 터미널을 bash로 지정 및 Remote VSCode 설정 값을 추가하고 VSCode의 bash 터미널에서 아래의 커맨드를 실행합니다.

```
ssh -R 52698:localhost:52698 user@<remote server>
```

bash 터미널로 연결된 `remore server`에서 아래 커맨드를 실행하면, VSCode의 편집창이 나타나며 편집이 가능해집니다.

```
rmate -p 52698 <filename>
```

상세한 내용은 [블로그](http://thrillfighter.tistory.com/458) 참조하시기 바랍니다. 한가지 아쉬운 점이 있다면, 여러 파일을 동시에 편집은 불가능하며 한번에 하나의 파일만 편집 가능하는 제약이 있습니다.

### 4. Key Management & IAM

애플리케이션 내에서 SQL을 접근할 경우 connection string과 같은 접근 키가 필요합니다. 또한, 클라우드 기반의 서비스를 개발하기 때문에 클라우드 기반의 다양한 서비스 리소스(Blob Storage, Service Bus, Event Grid 등)와 연동하는 코드가 추가되고 여기에도 서비스들의 접근 키가 요구됩니다.

소스코드 내에 이런 접근 키를 하드코딩하는 것은 유출 가능성이 높기 때문에 Production 환경에서는 권장되지 않습니다. 중요한 사용자 Key/Secret/Certificate를 안전하게 저장하고 참조 방법으로 [Azure Key Vault](https://docs.microsoft.com/ko-kr/azure/key-vault/key-vault-whatis)를 사용합니다. 하지만, Key Vault를 통해 사용자 Key를 가져오기 위해서는 Key Vault을 접근하는 AAD Access Token이 필요하고 Access Token을 얻기 위한 [AAD Service Principal](https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/resource-group-create-service-principal-portal)의 Key가 요구되기 때문에 하드코딩을 피하려던 보안적 이슈가 다시 원점으로 돌아갑니다.

이런 이슈를 보완하기 위해서 VM 등의 리소스에 IAM(Identity Access Management) 또는 RBAC(Role-Based Access Control)을 적용합니다. 아쉽게도 Azure에서는 이 부분이 좀 부족했는데, 이제는 AAD SP 방식이 아닌 [MSI(Managed Service Identity)](https://docs.microsoft.com/ko-kr/azure/active-directory/managed-service-identity/overview)를 이용하여 AAD Access Token을 손쉽게 획득할 수 있습니다.

아래는 VM 내에서 access token을 다이나믹하게 얻어 Key Vault내의 `pwdkey` secret을 참조하는 shell script 예 입니다. 이 코드가 정상적으로 실행되기 위해서는 먼저 MSI가 활성화 되어야 하고 코드가 실행되는 VM은 `myvault` Key Vault 엑세스 정책(Access Policy)에 추가되어 있어야 합니다. 

```
access_token=$(curl -s http://localhost:50342/oauth2/token --data "resource=https://vault.azure.net" -H Metadata:true | jq -r ".access_token")

curl https://myvault.vault.azure.net/secrets/pwdkey/?api-version=2016-10-01 -H "Authorization: Bearer $access_token"
```

자세한 내용은 [MSI의 Azure Key Vault Access](https://docs.microsoft.com/ko-kr/azure/active-directory/managed-service-identity/tutorial-linux-vm-access-nonaad)를 참조하시기 바랍니다.

유사한 방법으로 Blob Storage를 비롯하여 다양한 Azure의 리소스를 하드코드된 key 없이 접근할 수 있습니다. 하지만, 아쉽게도 Azure의 리소스들은 통일화된 인증방식(Bearer Token, SAS, ACS 등)을 모두 지원하지 않아서 Blob Storage와 같은 서비스는 추가적으로 Storage Key를 획득하여 접근해야 하는 불편함이 있고, Azure SDK도 MSI가 Preview라서 그런지 아직 잘 지원하고 있지 않는 것 같습니다. 또한, VM Extension 설치 방식으로 인해 지원하는 OS의 제약이 있어 현재 이부분은 [Instance Metadata Service](https://docs.microsoft.com/ko-kr/azure/virtual-machines/windows/instance-metadata-service) 방식으로 개선될 예정이라고 합니다. 

### 5. Outbound SNAT

마지막으로, 대부분의 서비스들은 외부의 다양한 서비스와 연동을 필요로 합니다. 온프레미스 API 서비스일 수도 있으며, 외부 3rd-party API 서비스도 될 수 있습니다. 이런 외부 서비스 대부분 방화벽 뒤에 위치하기 때문에 상대 방화벽에 Source IP가 등록되어야 접근이 가능합니다.

Azure에서 Outbound Source IP를 고정하기 위해서는 몇 가지 옵션이 있으나 가장 손쉽게 구현하는 방법은 LB를 활용하는 것입니다. Azure의 LB는 Outbound SNAT를 지원하여 LB의 백앤드풀에 연결된 VM은 기본적으로 LB의 Public IP로 masquerade(가장)되어 외부로 나갑니다. 자세한 설명은 [Azure Load Balancer문서](https://docs.microsoft.com/ko-kr/azure/load-balancer/load-balancer-outbound-connections)를 참조하시기 바랍니다.

Azure LB로 구성하기 어려운 경우에는 NAT Instance를 구성하고 UDR을 설정하여 VM의 모든 트랙픽을 NAT Instance로 라우팅하여 NAT Instance의 Public IP로 나가도록 합니다. 

![NAT Instance](https://raw.githubusercontent.com/iljoong/azure-terraform/master/images/terraform_azure2.png)

NAT Instance를 구성하는 방법은 Terraform 예제에서 소개한 [NAT Instance](https://github.com/iljoong/azure-terraform)를 참조하시기 바랍니다.

아쉽게도 Azure는 Managed NAT Gateway 서비스가 없지만 이 서비스도 조만간 출시될 것으로 예상하고 있습니다.

### Closing...

모든 보안 프랙티스를 소개하지는 못했지만 실제 클라우드 환경에서 사용되는 핵심 보안 프랙티스를 간략하게 소개하였습니다. 퍼블릭 클라우드는 안전하지만 잘 관리할 때 안전한 것이지 관리를 소홀이 하면 오히려 더 위험할 수 있습니다. 본 내용이 퍼블릭 클라우드 보안 때문에 고민하거나 또는 현 시스템의 보안을 강화하는데 도움이 되기를 기대해 봅니다.
