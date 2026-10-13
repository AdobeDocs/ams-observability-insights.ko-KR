---
source-git-commit: 263ba9e9b3eb1c6e4dd253ca03bd0ce8954c588b
workflow-type: tm+mt
source-wordcount: '491'
ht-degree: 1%
---
## 자습서 4: 고급 함수 {#tutorial-4-advanced-functions}

이 자습서에서는 변동성, 성장, &quot;최악의 예&quot; 조회, 합계의 공유, 트랜잭션 구성 및 임의의 OpenTelemetry 지표와 같은 심층적인 조사를 수행하는 함수 및 엔터티를 다룹니다.

**이 자습서에서:** [9.1 stddev](#91-measure-consistency-with-stddev) · [9.2 파생물](#92-spot-growth-with-derivative) · [9.3 Ranked Latest](#93-find-the-worst-example-with-ranked-latest) · [9.4 percentOfTotal](#94-find-where-time-goes-with-percentoftotal) · [9.5 SpanBreakdown](#95-break-down-transactions-with-spanbreakdown) · [9.6 모든 지표 선택](#96-discover-and-query-any-metric) · [9.7 엔티티 선택](#97-choose-the-right-trace-entity) · [9.8 중첩 집계](#98-work-around-nested-aggregation)

### `stddev`과의 일관성 측정

두 작업은 평균을 공유할 수 있으며 여전히 매우 다르게 동작합니다. 표준 편차는 지연 시간이 얼마나 달라지는지를 보여줍니다.

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, stddev(duration) AS stddevMs FROM Transaction FACET name ORDER BY stddevMs DESC LIMIT 20
```

평균보다 큰 표준 편차는 요청이 매우 느린 작업을 가리킵니다. 다음에 p99를 보세요.

### `derivative`(으)로 별색 증가

`derivative(attr)`은(는) 버킷 간 특성 평균의 초당 변경 사항을 반환합니다. 누출 전 힙 사용과 같이, 상승하는 게이지를 찾는 데 유용합니다.

```sql
SELECT derivative(value) AS heapMBPerSec FROM JvmHeapUsed FACET host TIMESERIES 5 minutes
```

지속적인 양의 선은 성장을 의미한다. 음수 값은 측정값이 떨어지고 있음을 의미합니다(예: 가비지 수집 후). `rate`과(와) 마찬가지로 `derivative`은(는) `TIMESERIES`이(가) 필요하며 유일한 집계여야 합니다.

### 순위가 `latest`인 최악의 예 찾기

`latest` 및 `earliest`의 두 번째 인수가 &quot;가장 최근&quot;을 &quot;이 특성에 의해 가장 높음&quot;으로 변경합니다. `latest(traceId, duration)`이(가) 가장 느린 요청의 추적 ID를 반환하므로 추적 검색에 붙여 넣을 수 있습니다.

```sql
SELECT count(*) AS requests, max(duration) AS slowestMs, latest(traceId, duration) AS slowestTrace, latest(urlPath, duration) AS slowestPath FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 15
```

비교를 위해 가장 빠른 예제를 가져오려면 `earliest(traceId, duration)`을(를) 사용하십시오.

### 시간이 `percentOfTotal`과(와) 일치하는 위치 찾기

`percentOfTotal(attr)`은(는) 전체 중 각 그룹의 몫을 표시합니다. `duration`에 적용되며 서버 시간이 가장 많이 소모되는 작업을 표시합니다. 이 기능은 작업이 가장 느린 경우보다 더 유용한 경우가 많습니다.

```sql
SELECT count(*) AS requests, sum(duration) AS totalMs, percentOfTotal(duration) AS shareOfServerTime FROM Transaction FACET name LIMIT 15
```

`percentOfTotal`은(는) `TIMESERIES`과(와) 결합할 수 없습니다.

### `SpanBreakdown`(으)로 트랜잭션 분류

`SpanBreakdown`은(는) 수집 시 미리 집계됩니다. 각 트랜잭션(`rootOperation`)에서 소요하는 자식 작업(`segment`)을 분 단위로 기록합니다. &quot;이 엔드포인트가 무엇을 하고 있습니까?&quot;에 답합니다. 저렴한 가격:

```sql
SELECT sum(callCount) AS calls, average(avgDurationMs) AS avgMs FROM SpanBreakdown WHERE rootOperation LIKE '%GET%' FACET rootOperation, segment LIMIT 25
```

시간에 따른 한 세그먼트의 트렌드를 표시합니다.

```sql
SELECT average(avgDurationMs) AS avgMs FROM SpanBreakdown FACET segment TIMESERIES AUTO LIMIT 8
```

> [!NOTE]
> `avgDurationMs`은(는) 단위로 자동 태그가 지정되지 않았습니다. 위젯 **Unit**&#x200B;을(를) `ms`(으)로 설정합니다.

### 모든 지표 검색 및 쿼리

SQL에 `keyset()` 또는 `SHOW EVENT TYPES`이(가) 없습니다. 대신 보고되는 모든 지표를 해당 단위 및 최신 값과 함께 나열합니다.

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricGauge FACET metric, unit LIMIT 200
```

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricSum FACET metric, unit LIMIT 200
```

그런 다음 모든 지표를 이름으로 쿼리합니다. 예를 들어 풀당 사용된 JVM 메모리는 다음과 같습니다.

```sql
SELECT latest(value) AS bytesUsed FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool LIMIT 50
```

각 서비스에 대해 채워지는 **span 특성**&#x200B;을(를) 보려면 공백이 아닌 값을 계산하십시오.

```sql
SELECT count(*) FILTER (WHERE http.route != '') AS withRoute, count(*) FILTER (WHERE db.system != '') AS dbCalls, count(*) FILTER (WHERE peer.service != '') AS withPeer, count(*) FILTER (WHERE error.type != '') AS withErrorType FROM Span FACET service LIMIT 50
```

> [!TIP]
> 조정된 엔터티가 있으면(예: `CpuUtilization`) `MetricGauge`보다 선호합니다. 올바른 필터와 단위 변환을 적용하고 롤업 가속화될 수 있습니다.

### 올바른 추적 엔터티 선택

| 사용 | 원하는 경우 | 예 |
| ------------- | ------------------------------------------------------------------------------------------- | --------------------------------- |
| `Transaction` | 인바운드 HTTP/RPC 요청 KPI(처리량, 지연, 오류, Apdex) 롤업 가속화. | 골든 시그널 |
| `AppRequest` | 동일한 KPI, **메시지 소비자 포함**(Kafka 등) | 이벤트 기반 서비스 |
| `Span` | 요청 아래에 있는 모든 것: 아웃바운드 HTTP, DB, 메시징, AEM Sling/JCR/JSP, 내부 범위 | 종속성 및 코드 경로 분석 |

각 범위 종류를 카운트하여 계측된 내용을 확인합니다.

```sql
SELECT count(*) AS spans, percentile(duration, 95) AS p95 FROM Span FACET kind
```

외부 도메인별 아웃바운드 호출:

```sql
SELECT count(*) AS calls, percentage(*, WHERE status = 'Error') AS errorPct, percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain LIMIT 20
```

### 중첩 집계 작업

NRQL을 사용하면 &quot;호스트당 최대 수의 평균&quot;과 같은 하위 쿼리로 집계를 집계할 수 있습니다. SQL은 그렇지 않습니다. 다음은 선택 사항입니다.

| 목표 | SQL 방식 |
| --------------------------------------- | ---------------------------------------------------------- |
| 그룹당 값, 그룹 등급 지정 | `FACET group` + `ORDER BY alias` + `LIMIT` |
| 두 개의 집계 비율 | [공식 위젯](#86-combine-queries-with-formula-widgets) |
| 호스트당 값, 그 다음 &quot;최악의 호스트&quot; | `FACET host ORDER BY alias DESC LIMIT 1` |
| 시간 경과에 따른 호스트당 평균 | `FACET host TIMESERIES AUTO` 및 시리즈 읽기 |

예: p95에 의한 최악의 호스트:

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET host ORDER BY p95 DESC LIMIT 1
```
