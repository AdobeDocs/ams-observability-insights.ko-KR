---
title: Observability Insights 시작
description: Observability Insights에 액세스하는 방법, Adobe이 귀하를 대신하여 모니터링하는 사항 및 이 안내서에서 필요한 것을 찾을 수 있는 위치를 알아봅니다.
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
source-wordcount: '533'
ht-degree: 0%
---

# Observability Insights 시작 {#get-started}

이 섹션에서는 신규 사용자를 위한 필수 사항, 즉 Observability Insights 계정에 액세스하는 방법, Adobe이 사용자를 대신하여 모니터링하는 환경 및 데이터, 이 설명서의 나머지 부분을 탐색하는 방법을 다룹니다.

## Observability Insights 인터페이스 {#observability-insights-interface}

[insights.adobecqms.net](https://insights.adobecqms.net)에 로그인하면 시작 화면이 AEM Managed Services 환경의 모든 모니터링 영역에 대한 진입점을 제공합니다.

![APM 및 인프라 모니터링 시작 지점을 표시하는 Observability Insights 시작 화면](../v2-assets/observability-catalog-listing.png)

인터페이스는 두 개의 핵심 모니터링 영역으로 구성됩니다.

- **응용 프로그램** — 작성자 및 게시 계층에 대한 응용 프로그램 성능 데이터를 표시합니다. 이를 사용하여 요청 처리량, 오류율, 지연, JVM 동작 및 추적 수준 실행 세부 정보를 조사할 수 있습니다. [응용 프로그램](../applications.md)을 참조하세요.
- **호스트** — 관리되는 토폴로지에 호스트 수준 상태 데이터를 표시합니다. 이를 사용하여 개별 서버의 CPU, 메모리, 디스크, 네트워크 및 저장소 신호를 평가합니다. [호스트](../hosts.md)를 참조하십시오.

두 영역 모두 고객 사용자의 경우 읽기 전용입니다. Adobe Managed Services은 계정 프로비저닝, 계측 및 관리 제어를 관리합니다.

## 액세스 및 계정 관리 {#access-overview}

가시성 인사이트 액세스는 Adobe IMS를 통해 관리됩니다. Adobe은 조직의 계정을 프로비저닝하고 관리합니다. 고객 팀은 모니터링되는 모든 데이터에 대한 읽기 전용 액세스 권한을 받습니다.

주요 사항:

- 조직의 가시성 인사이트 계정이 단일 Adobe 마스터 계정에 연결되어 있습니다.
- Managed Services 계약의 모든 환경(작성자 및 게시, 프로덕션 및 비프로덕션)이 이 계정에 보고합니다.
- 사용자 액세스는 CSE(Customer Success Engineer)가 프로비저닝하고 관리합니다.

프로비저닝 단계, 사용자 역할, 고객 사용자가 수행할 수 있는 작업 및 수행할 수 없는 작업에 대해서는 [액세스 및 계정 관리](access-and-accounts.md)를 참조하십시오.

## 적용 범위, 환경 및 데이터 보존 {#coverage-overview}

Adobe은 Observability Insights APM Java 플러그인을 사용하여 AEM 작성자 및 게시 계층을 모니터링하고, Observability Insights 인프라 에이전트를 사용하여 호스팅된 모든 서버를 모니터링합니다. 모니터링은 비프로덕션 환경과 프로덕션 환경 모두에서 활성화됩니다.

주요 사항:

- 각 AEM Managed Services 환경에는 작성자용 APM 애플리케이션과 게시용 APM 애플리케이션이 각각 하나씩 포함되어 있습니다.
- APM 지표, 인프라 지표 및 이벤트는 최대 **30일** 동안 유지됩니다.
- Observability Insights는 운영 분석 및 최근 트렌드 비교에 적합하며 아카이브 또는 장기 보고 도구가 아닙니다. 데이터가 오래되기 전에 스크린샷 또는 내보낸 증거를 캡처합니다.

계정에 애플리케이션이 표시되는 방식 및 보존 기간의 운영상의 의미 등 전체 적용 범위에 대한 자세한 내용은 [적용 범위, 환경 및 데이터 보존](coverage-and-data.md)을 참조하십시오.

## 이 안내서의 구성 방식 {#how-this-guide-is-structured}

설명서는 네 가지 영역으로 구성되어 있습니다. 아래 설명을 사용하여 필요한 항목으로 바로 이동하십시오.

**시작** — 이 섹션. 액세스, 계정 프로비저닝, 모니터링 범위 및 데이터 보존에 대해 다룹니다.

**[가시성 인사이트 사용](../use-observability-insights.md)** — 일상적인 조사를 위한 작업 중심 지침입니다. 느린 페이지, 오류 스파이크 또는 불안정한 트랜잭션과 같이 응용 프로그램에 대한 증상이 나타날 때는 [응용 프로그램](../applications.md)을 사용하십시오. 호스트 수준 리소스 압력(CPU, 메모리, 디스크 또는 네트워크)이 응용 프로그램에 표시되는 내용을 설명하는지 여부를 결정해야 하는 경우 [호스트](../hosts.md)를 사용하십시오. [응용 프로그램 문제 조사](../use-cases/investigate-application-issues.md) 및 [인프라 문제 조사](../use-cases/investigate-infrastructure-issues.md)에서 단계별 조사 흐름을 사용할 수 있습니다.

**[자주 묻는 질문](../troubleshooting/common-questions.md)** — 시작 위치를 잘 모르거나 활성 문제 중에 빠른 응답이 필요한 경우에 대한 일반적인 질문 및 지원 중심의 진입점입니다.
