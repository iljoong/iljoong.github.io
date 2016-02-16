---
layout: post
title:  "First Github!"
date:   2016-02-16 03:06:56 +0000
categories: project
---

#First Github

첫 Blog에 이어 Github에 첫 소스코드를 공유해 보았습니다.

[https://github.com/iljoong/sqlperfmon](https://github.com/iljoong/sqlperfmon)

SQLPerfMon은 Web App으로 Azure SQL Database의 성능카운트를 보여줍니다.

3개의 대쉬보드로 구성되어 있으며, 다음과 같습니다.

* _Stat_: 최근 1시간의 성능지표
* _Perf_: 일일 성능지표
* _Mon_: 일일 SLO(Service Level Objective) 지표 및 스케일 권고

Azure SQL Database의 성능 관련해서는 [https://azure.microsoft.com/en-us/documentation/articles/sql-database-performance-guidance/](https://azure.microsoft.com/en-us/documentation/articles/sql-database-performance-guidance/)참고하면 됩니다.

![Screenshot](https://github.com/iljoong/sqlperfmon/raw/master/doc/pix/azsqlmonweb01.png)

SQLPerMon은 다양한 Azure의 서비스로 구성되어 있습니다. _Stat_ 데이터를 제외하고, _Perf_ 와 _Mon_ 데이터는 Azure Automation의 스케쥴에 의해서 매일 실행되고 Database에 저장됩니다.
성능지표 데이터는 Database에서 직접 가져오지 않고, API app을 통해서 가져옵니다. API app은 Azure AD로 보호되어 인증된 사용자만 데이터를 접근할 수 있습니다.

자세한 구성은 SQLPerMon의 아키텍처 구성도를 참고하시기 바랍니다.

![Architecture](https://github.com/iljoong/sqlperfmon/raw/master/doc/pix/architecture.png)

Azure Portal에서 제공되는 Performance 지표와 비교해서 특별하게 나은 점은 없으나, 내 입맛에 맞게 대쉬보드를 구현할 수 있습니다.
또한, Azure Portal에서 제공하지 않는 SLO 지표 및 스케일 권고를 확인할 수 있습니다.

소스코드와 세부 설정/배포 방법은 Github를 참고하시기 바랍니다.

/iljoong/

