---
title: 문제 해결
description: Observability Insights에서 SQL 쿼리, 차트 및 대시보드 문제를 해결합니다.
source-git-commit: 873e7b9f279c3276fc0197969d667d40d022968a
workflow-type: tm+mt
source-wordcount: '335'
ht-degree: 5%
---

## 문제 해결 {#troubleshooting}

쿼리를 **유효성 검사** 또는 **실행**&#x200B;할 때 편집기와 대시보드의 위젯 자체에 오류 메시지가 표시됩니다. 첫 번째 열에서 메시지를 찾습니다.

| 메시지 / 증상 | 원인 | 수정 |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| `Unknown entity 'X'` | 카탈로그 이름 입력 또는 비입력 | 쿼리 가이드 패널에서 [데이터 사전](#data-dictionary)을(를) 확인합니다. |
| `Attribute 'x' is not available on Entity` | 해당 엔터티에 특성이 없습니다(예: `CpuUtilization`의 `duration`). | 추적 특성과 지표 특성 비교 — [데이터 사전](#data-dictionary)을 참조하십시오. |
| `Raw attributes cannot be mixed with aggregates; use FACET to group results` | `SELECT service, count(*)` | `SELECT count(*) … FACET service`. |
| `FACET and TIMESERIES require an aggregate query` | FACET/TIMESERIES가 있는 이벤트 쿼리 | 합계를 사용합니다. |
| `FACET supports at most five attributes` | > 5개 패싯 | 줄이기. |
| `average requires a numeric attribute` | e.g. `average(service)` | `uniqueCount`/`latest` 또는 숫자 특성을 사용하십시오. |
| `rate requires TIMESERIES` / `rate is only available for counter measures` | `rate()`(시계열 또는 계기 없음) | `TIMESERIES AUTO`을(를) 추가합니다. ✔ 카운터 엔터티([조정된 지표 엔터티](../references/data-dictionary.md#curated-metric-entities))를 사용합니다. 게이지의 경우 `derivative()`을(를) 사용합니다. |
| `rate cannot be combined with other aggregates` | `rate()`이(가) 있는 여러 항목 | 위젯당 1개의 `rate()`. 수식과 결합합니다. |
| `percentOfTotal cannot be combined with TIMESERIES` | — | 시계열을 제거합니다. |
| `apdex requires an entity with a status attribute` | 지표 엔터티의 `apdex` | `Transaction`/`Span`/`AppRequest` 사용 |
| `ORDER BY 'x' must reference a selected value or alias` | 선택하지 않은 항목별 순서 지정 | 별칭을 사용하여 합계를 지정하고 별칭을 기준으로 순서를 지정합니다. |
| `IN requires between 1 and 100 values` | — | `LIKE`을(를) 분할하거나 사용합니다. |
| `Use IS NULL or IS NOT NULL for null comparisons` | `attr = NULL` | `IS NULL` 사용. |
| `Unexpected character '{'` | 위젯 추가 미리 보기 또는 저장 시 `{{variable}}` | [자습서 3](../tutorials/tutorial-3.md#make-dashboards-reusable-with-variables) 제한을 참조하십시오. |
| `Unexpected character ';'` / `'"'` | 세미콜론 또는 큰따옴표 | `;`을(를) 제거하십시오. 작은 따옴표를 사용하십시오. |
| `Expected a positive duration such as 30 minutes` | `TIMESERIES 5m` | `TIMESERIES 5 minutes`을(를) 씁니다. |
| `Time range cannot exceed 31 days` | 선택기 > 31d | 창 범위를 좁힙니다. |
| `Select a customer before running a SQL query` | 테넌트/데모 테넌트 없음 | 테넌트 전환기에서 고객을 선택합니다. |
| `All queries in a formula must be the same kind…` | 합계와 시계열의 혼합 또는 FACET | 모든 하위 쿼리를 일반 쿼리 또는 모든 `TIMESERIES AUTO`(으)로 만듭니다. FACET를 제거하십시오. |
| `Query 'x' must select exactly one metric…` | 하위 쿼리에는 두 개의 합계가 있습니다. | 하위 쿼리당 하나의 집계입니다. |
| `Unknown query name 'x' - must be one of: …` | 수식에서 누락된 이름을 참조합니다. | 이름을 정확하게 일치시키십시오(대/소문자 구분). |
| **저장 실패** | 위젯 쿼리가 잘못된 서버측(종종 `{{var}}`)입니다. | 최근에 편집한 각 위젯을 열고 **실행**&#x200B;합니다. 잘못된 위젯을 수정하거나 제거합니다. |
| _값 선택:{{x}}_ | 변수에 선택 항목이 없습니다. | 변수 막대에서 값을 선택합니다. |
| _최근 활동이 없습니다..._ | 창에 행 없음 | 창을 넓히고 검색 쿼리([데이터 검색](../examples/query-cookbook.md#data-discovery))로 필터를 확인합니다. |
| 선이 점으로 산산조각 나옴 | 데이터 케이던스보다 작은 버킷 | `TIMESERIES AUTO`(≥ 2분) 또는 막대 차트를 사용합니다. |
| 패싯에 빈 문자열 그룹이 표시됩니다. | 일부 행에서 속성이 누락됨 | `WHERE attr != ''`을(를) 추가합니다. |
| _이 레거시 SQL 패널은 더 이상 실행되지 않습니다_ | 이전 raw-SQL 빌더의 패널 | SQL에서 편집 → 다시 빌드합니다. |
