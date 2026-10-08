---
title: 액세스 및 계정 관리
description: Observability Insights 계정이 프로비저닝되는 방법과 액세스 관리 담당자 및 고객 팀이 제어할 수 있는 수준을 이해합니다.
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
source-wordcount: '232'
ht-degree: 0%
---

# 액세스 및 계정 관리 {#access-and-account-management}

Adobe은 AEM Managed Services 조직에 대한 Observability Insights 계정을 프로비저닝하고 관리합니다. 고객 팀은 Adobe IMS를 사용하여 로그인하고 모니터링되는 데이터에 대한 읽기 전용 가시성을 받습니다.

## 계정 소유권 모델 {#account-ownership-model}

- Adobe Managed Services은 Observability Insights 계정을 소유합니다.
- 고객 팀에는 읽기 전용 액세스 권한이 부여됩니다.
- 관리 변경, 프로비저닝 및 액세스 업데이트는 Adobe을 통해 관리됩니다.

## 사용자가 액세스하는 방법 {#how-users-get-access}

Observability Insights에 액세스하려면 Adobe IMS 프로비저닝이 필요합니다.

액세스 권한을 요청하거나 갱신하려면

1. 고객 성공 엔지니어(CSE)에게 문의하십시오.
2. Adobe IMS 프로비저닝에 필요한 사용자 세부 정보를 제공합니다.
3. 올바른 조직 및 액세스 범위가 할당되었는지 확인합니다.

프로비저닝이 완료되면 [insights.adobecqms.net](https://insights.adobecqms.net)에 로그인하십시오.

## 사용자가 수행할 수 있는 작업 {#what-users-can-do}

고객 사용자는 일반적으로 다음과 같은 작업을 수행할 수 있습니다.

- APM 대시보드 보기
- 인프라 대시보드 보기
- 모니터링되는 애플리케이션 및 호스트 지표 검사
- 공유 대시보드 및 추적을 사용하여 조사에 참여

## 사용자가 수행할 수 없는 작업 {#what-users-cannot-do}

고객 사용자는 Adobe이 달리 명시적으로 문서를 작성하지 않는 한 관리 제어가 Adobe Managed Services에 남아 있다고 간주해야 합니다.

일반적인 예는 다음과 같습니다.

- 계정 소유권 관리
- 플랫폼 수준 프로비저닝 변경
- 관리 계측 동작 변경

## 나중에 추가할 정보 {#information-to-add-later}

전체 프로세스 세부 정보를 사용할 수 있게 되면 이 섹션을 사용합니다.

- Adobe IMS 사전 요구 사항
- 예상 프로비저닝 반환 시간
- 에스컬레이션 연락처 및 경로
- 가입자 및 이탈자를 위한 사용자 라이프사이클 관리
