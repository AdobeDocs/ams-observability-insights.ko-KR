---
title: 적용 범위, 환경 및 데이터 보존
description: AEM Managed Services에서 Observability Insights가 모니터링하는 내용, 애플리케이션이 표시되는 방법 및 모니터링 데이터가 유지되는 기간을 참조하십시오.
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
source-wordcount: '267'
ht-degree: 1%
---

# 적용 범위, 환경 및 데이터 보존 {#coverage-environments-and-data-retention}

이 페이지에서는 AEM Managed Services에 대한 Observability Insights에서 수집되는 데이터와 해당 데이터가 구성되는 방식을 요약합니다.

## 범위 모니터링 {#monitoring-coverage}

Adobe 모니터:

- Observability Insights APM Java 플러그인이 포함된 AEM 작성자 계층
- Observability Insights APM Java 플러그인이 포함된 AEM 게시 계층
- Observability Insights 인프라 에이전트를 사용하여 관리되는 토폴로지에서 호스팅된 서버

사용자 정의 APM 및 인프라 모니터링은 비프로덕션 및 프로덕션 Managed Services 환경 모두에서 활성화됩니다.

## 애플리케이션 표시 방법 {#how-applications-are-represented}

각 AEM Managed Services 환경은 일반적으로 다음을 포함합니다.

- 작성자용 APM 애플리케이션 1개
- 게시용 APM 애플리케이션 1개

Managed Services 계약 보고서의 모든 토폴로지를 하나의 Observability Insights 계정으로 만듭니다.

## 데이터 유지 {#data-retention}

APM 지표, 인프라 지표 및 관련 이벤트는 최대 **30일** 동안 유지됩니다.

## 요약 테이블 {#summary-tables}

| 적용 지역 | 모니터링되는 항목 |
| -------------- | ------------------------------------------ |
| APM | AEM Author 및 Publish 애플리케이션 |
| 인프라 | 관리되는 토폴로지의 모든 호스팅 서버 |

| 항목 | 표시 |
| ------------------------------ | ------------------------------------------------------------- |
| AEM 환경 | 작성자 APM 응용 프로그램 하나와 게시 APM 응용 프로그램 하나 |
| Observability Insights 계정 | Managed Services 고객 범위당 Adobe 관리 계정 1개 |

| 데이터 유형 | 유지 |
| --------------------------------- | ------------- |
| APM 지표 및 이벤트 | 최대 30일 |
| 인프라 지표 및 이벤트 | 최대 30일 |

## 작동 중인 의미 {#what-this-means-operationally}

- Observability Insights는 운영 분석, 활성 인시던트 및 최근 트렌드 비교에 적합합니다.
- 보존 기간을 벗어난 기록 분석은 필요한 경우 다른 보고 또는 보관 프로세스를 통해 처리해야 합니다.
- 반복되는 문제를 조사할 때 데이터가 오래되기 전에 스크린샷이나 내보낸 증거를 캡처합니다.
