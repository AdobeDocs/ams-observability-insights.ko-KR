---
title: Observability Insights를 사용하여 AEM Managed Services 환경 모니터링
description: 여기에서 AEM Managed Services의 Observability Insights가 다루는 내용, 대상 및 이 안내서의 나머지 부분을 탐색하는 방법을 이해할 수 있습니다.
feature: Operations
role: Admin
product_v2:
  - id: fd1f54a9-f50c-467d-8956-cebbaf4f3eb8
    internal-label: Experience Manager
feature_v2:
  - id: 8a70d214-ab7b-58c1-b001-2ed2e5d6303d
    internal-label: Operations
role_v2:
  - id: c66ffd68-0f65-42bb-aa23-b4020f12e0bd
    internal-label: Admin
source-git-commit: e0cc17c9d725cad021ba99da4332bca176eae6db
workflow-type: tm+mt
source-wordcount: '741'
ht-degree: 0%
---

# Observability Insights를 사용하여 AEM Managed Services 환경 모니터링 {#observability-insights-monitoring}

**Observability Insights**&#x200B;에서는 별도의 모니터링 플랫폼을 사용하지 않고도 AEM Managed Services의 애플리케이션 성능, 인프라 상태 및 서비스 동작에 대한 가시성을 제공합니다.

서비스 안정성, 인시던트 응답 또는 성능 분석에 대한 책임이 있는 경우 **가시성 인사이트**&#x200B;를 사용하면 증상에서 증거로 신속하게 이동할 수 있습니다. 애플리케이션 원격 분석과 호스트 수준 상태 신호를 결합하여 고객 팀과 Adobe이 공유된 운영 보기에서 문제를 조사할 수 있도록 합니다.

## Observability Insights 백서

[Observability Insights 백서 다운로드](v2-assets/Observability_Insights_Overview.pdf)

## 팀이 Observability Insights를 사용하는 이유 {#why-teams-use-observability-insights}

Observability Insights를 사용하여 다음과 같은 운영 질문에 답변합니다.

- 이 문제가 작성자, 게시 또는 둘 다에 영향을 줍니까?
- 애플리케이션 동작, 호스트 리소스 압력 또는 이 둘의 조합으로 인해 발생하는 문제입니까?
- 오류 또는 지연 급증을 설명하는 트랜잭션, 엔드포인트 또는 상태 그룹은 무엇입니까?
- 문제가 한 환경으로 분리됩니까? 아니면 더 광범위한 토폴로지에서 표시됩니까?

Observability Insights는 최근 비헤이비어의 작동 분석을 위해 설계되었습니다. 에스컬레이션 또는 수정 조치 전에 변경된 사항, 변경된 사항 및 가장 관련성이 높은 신호를 파악하는 데 도움이 됩니다.

## 가시성 인사이트를 통해 얻을 수 있는 이점은 무엇입니까? {#what-observability-insights-helps-you-do}

가시성 인사이트를 사용하여 다음을 수행할 수 있습니다.

- 작성자 및 게시 계층이 실제 트래픽에서 어떻게 작동하는지 파악합니다.
- 애플리케이션 지연 시간, 오류율 및 JVM 상태와 호스트 수준 신호의 상관 관계를 파악할 수 있습니다.
- 문제가 하나의 환경, 하나의 계층 또는 하나의 호스트로 분리되었는지 확인합니다.
- 조사 중에 Adobe Managed Services 및 내부 팀에 공유된 운영 보기를 제공합니다.

가시성 인사이트는 AEM Managed Services에 포함되어 있습니다. Adobe은 계정을 프로비저닝하고 관리하며, 지원되는 환경을 계측하고 결과 대시보드를 읽기 전용 운영 도구로 팀에 노출합니다.

Adobe은 플랫폼 설정 및 계측을 관리하므로 에이전트 배포, 계정 관리 또는 대시보드 어셈블리보다 조사와 해석에 집중할 수 있습니다.

## 개요 {#at-a-glance}

AEM Managed Services의 일부로 다음을 받을 수 있습니다.

- **전용 Observability Insights 계정** — 팀에 대한 읽기 전용 액세스 권한으로 Adobe Managed Services에서 프로비저닝하고 감독합니다.
- **심층 AEM 트랜잭션 모니터링** — Observability Insights APM 에이전트는 의미 있는 트랜잭션을 메서드 호출(줄 번호 포함), 외부 종속성 및 저장소 작업까지 추적합니다.
- **통합 응용 프로그램 및 호스트 보기** - 응용 프로그램과 호스트 수준 지표를 결합하여 전체적으로 성능을 최적화합니다.

## 이 설명서의 대상 {#who-this-documentation-is-for}

이 설명서는 주로 다음을 위해 설계되었습니다.

- 모니터링되는 환경에 대한 가시성이 필요한 AEM Managed Services 관리자
- 사고, 트렌드 분석 및 서비스 검토를 처리하는 운영 및 지원 팀
- 조사 기간 동안 Adobe Managed Services과 제휴한 고객 엔지니어링 팀
- 모니터링 범위 및 운영 책임을 이해해야 하는 이해 당사자

## 가시성 통찰력을 통해 Adobe에서 모니터링하는 사항 {#what-we-monitor}

Adobe은 Observability Insights APM Java 플러그인을 사용하여 AEM **작성자** 및 **게시** 계층을 모니터링합니다. 토폴로지의 모든 호스팅 서버는 Observability Insights 인프라 에이전트를 통해 모니터링됩니다. 사용자 지정 APM 및 인프라 모니터링은 비프로덕션 및 프로덕션 Managed Services 환경 모두에서 활성화됩니다.

![AEM 작성자, 게시 및 호스팅된 서버 간의 Observability Insights APM 및 인프라 모니터링을 보여 주는 다이어그램](v2-assets/login-screen.png)

### 계정의 애플리케이션 {#applications-in-your-account}

Observability Insights 계정은 단일 Adobe 마스터 계정에 연결되어 있으며 다음을 포함하여 여러 애플리케이션에서 데이터를 받을 수 있습니다.

- AEM Managed Services 환경당 **작성자** 계층용 APM 응용 프로그램 1개
- AEM Managed Services 환경당 **게시** 계층에 대한 APM 응용 프로그램 1개

각 응용 프로그램에는 자체 라이선스 키가 있습니다. Managed Services 계약 보고서의 모든 토폴로지를 하나의 Observability Insights 계정으로 만듭니다. APM 및 인프라 지표와 이벤트는 최대 **30일** 동안 유지됩니다.

## 계정에 액세스 {#access}

모니터링 데이터는 Adobe에서 프로비저닝하고 관리하는 Observability Insights 계정에 통합됩니다. 고객 사용자는 에이전트에서 수집한 APM 및 인프라 데이터에 대해 **읽기 전용 액세스**&#x200B;를 받습니다. Adobe Managed Services은 계정 소유권 및 관리 권한을 유지합니다.

### 사전 요구 사항 {#access-prerequisites}

로그인하기 전에 다음을 확인하십시오.

- 조직에 활성 **AEM Managed Services** 구독이 있습니다. Observability Insights는 추가 비용 없이 포함됩니다.
- CSE(고객 성공 엔지니어)가 Adobe IMS 계정을 프로비저닝하고 조직의 Observability Insights 계정에 대한 액세스 권한을 부여했습니다.

>[!NOTE]
>
> **Observability Insights에 대한 액세스:** 액세스에는 Adobe IMS 프로비저닝이 필요합니다. CSE(고객 성공 엔지니어)에게 연락하여 조직의 사용자 액세스를 프로비저닝하고 관리합니다.

CSE가 계정을 프로비저닝하면 [insights.adobecqms.net](https://insights.adobecqms.net)에 로그인합니다. 이 URL은 모든 AEM Managed Services 고객에 대해 동일합니다. 조직의 환경 및 대시보드의 범위는 프로비저닝된 계정으로 설정됩니다.
