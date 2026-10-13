---
title: '자습서 2: 데이터 제어'
description: Observability Insights에서 쿼리 결과를 필터링, 그룹화 및 제어하는 방법을 알아봅니다.
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '361'
ht-degree: 0%
---

## 자습서 2: 데이터 제어 {#tutorial-2-control-your-data}

이 튜토리얼에서는 원시 카운트, 즉 고유 카운트, 최신 값, 백분율, 백분위수, 와일드카드, 정렬 및 다차원 그룹화를 응답으로 바꾸는 함수 및 절에 대해 알아봅니다.

**이 자습서에서:** [7.1 uniqueCount](#71-count-distinct-values-with-uniquecount) · [7.2 latest/earliest](#72-find-the-latest-and-earliest-values) · [7.3 percentage](#73-calculate-percentages) · [7.4 percentage](#74-understand-latency-with-percentile) · [7.5 레이블](#75-make-results-readable-with-labels) · [7.6 와일드카드 및 목록](#76-match-patterns-and-lists) · [7.7 정렬](#77-sort-and-rank-with-order-by) · [7.8 다중 패싯](#78-group-by-several-attributes) · [7.9 이전 Windows](#79-look-at-a-past-window)

### `uniqueCount`을(를) 사용하여 고유 값 계산

각 서비스에는 몇 개의 호스트와 고유한 추적이 있습니까?

```sql
SELECT uniqueCount(host) AS hosts, uniqueCount(traceId) AS traces FROM Transaction FACET service
```

`uniqueCount`은(는) 정확하지만 예상이 아닙니다.

고유한 값 자체(NRQL이 `uniques()`을(를) **목록**&#x200B;하려면 특성에 패싯을 사용하십시오.

```sql
SELECT count(*) AS requests FROM Transaction FACET host LIMIT 100
```

### 가장 최근 및 가장 이른 값 찾기

`latest(attr)`은(는) 가장 최근 값을 반환하고 `earliest(attr)`은(는) 가장 오래된 값을 반환합니다. 현재 상태 테이블에 유용합니다.

```sql
SELECT latest(value) AS currentCpu FROM CpuUtilization FACET host
```

```sql
SELECT earliest(timestamp) AS firstSeen, latest(timestamp) AS lastSeen FROM Transaction FACET service
```

### 백분율 계산

`percentage(*, WHERE condition)`은(는) 조건과 일치하는 행의 공유를 반환합니다. 다음은 오류율입니다.

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction
```

서비스별 트렌드 표시:

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

느린 요청의 공유를 추적합니다.

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowPct FROM Transaction FACET service
```

### `percentile`(으)로 지연 이해

평균은 이상치를 숨깁니다. 백분위수는 대부분의 사용자가 경험하는 것(p50)과 불운한 사용자가 경험하는 것(p95, p99)을 보여줍니다.

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

>[!TIP]
>
> `Transaction.duration`의 p50, p95 및 p99는 긴 창에 걸쳐 롤업에서 제공되므로 주별 보기를 빠르게 볼 수 있습니다.

### 레이블을 사용하여 결과를 읽을 수 있도록 만들기

공백이 포함된 레이블에 작은 따옴표로 `AS`을(를) 사용합니다.

```sql
SELECT count(*) AS 'Requests', percentile(duration, 95) AS 'p95 (ms)' FROM Transaction FACET service
```

>[!NOTE]
>
> 별칭은 `SELECT`개 항목에만 적용됩니다. Facet 열에는 항상 속성 이름이 레이블이 지정됩니다.

### 패턴 및 목록 일치

`LIKE`은(는) 문자 수에 관계없이 `%`을(를) 사용하고 정확히 하나의 문자에 대해서는 `_`을(를) 사용합니다.

```sql
SELECT count(*) AS requests FROM Transaction WHERE urlPath LIKE '/content/%.html' FACET urlPath LIMIT 20
```

`NOT LIKE`을(를) 사용하여 정적 자산 제외:

```sql
SELECT count(*) AS requests FROM Transaction WHERE urlPath NOT LIKE '%.js' AND urlPath NOT LIKE '%.css' AND urlPath NOT LIKE '%.png' FACET urlPath LIMIT 20
```

`IN` 및 `NOT IN`(최대 100개 값)이 있는 목록과 일치:

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.method IN ('POST', 'PUT', 'DELETE') FACET service
```

>[!NOTE]
>
> SQL에 정규식(`RLIKE`)이 없습니다. `LIKE` 패턴을 `OR`과(와) 결합하거나 `IN`을(를) 사용합니다.

### `ORDER BY`(으)로 정렬 및 순위 지정

기본적으로 패싯된 결과는 첫 번째 집계별로 정렬됩니다. `ORDER BY`을(를) 사용하여 p95까지 가장 느린 작업과 같이 다른 항목으로 순위를 매깁니다.

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 20
```

또는 가장 적게 사용되는 서비스를 먼저 선택합니다.

```sql
SELECT count(*) AS requests FROM Transaction FACET service ORDER BY requests ASC LIMIT 10
```

>[!TIP]
>
> `HAVING`이(가) 없습니다. &quot;사용 중&quot; 작업만 찾으려면 `requests` 순위를 매기고 상위 N을 유지하거나 노이즈를 제거하는 `WHERE`을(를) 추가하십시오(예: `WHERE http.method != 'OPTIONS'`).

### 여러 속성으로 그룹화

`FACET`은(는) 최대 5개의 특성을 허용합니다. 각 조합은 하나의 행 또는 시리즈가 됩니다.

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.statusCode >= 400 FACET service, http.method, http.statusCode LIMIT 30
```

### 지난 창 보기

`COMPARE WITH`이(가) 없습니다. 대시보드 및 탐색에서 **시간 선택기**&#x200B;를 변경합니다. 명확한 범위 없이 API를 호출하면 `SINCE … UNTIL …`에 이전 창이 표시됩니다.

```sql
SELECT count(*) AS lastWeekSameHour FROM Transaction SINCE 169 hours ago UNTIL 168 hours ago
```
