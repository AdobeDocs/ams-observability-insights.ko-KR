---
title: SQL 참조
description: Observability Insights에서 지원하는 SQL 구문, 절 및 함수를 참조하십시오.
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '1492'
ht-degree: 9%
---

## SQL 참조 {#sql-reference}

이 참조는 SQL 구문, 절, 함수 및 연산자를 다룹니다. 세부 정보 및 예를 보려면 각 항목을 확장합니다.

**이 문서에서:** [구성 요소 쿼리](#41-query-components) · [필수 절](#42-required-clauses) · [선택적 절](#43-optional-clauses) · [집계 함수](#44-aggregator-functions) · [연산자 및 조건](#45-operators-and-conditions) · [유형 처리](#46-type-handling-blank-vs-null) · [SQL을 SQL로 가져오기](#47-translate-sql-to-sql) · [기존 대시보드 가져오기](#48-importing-existing-dashboards-to-insights)

### 쿼리 구성 요소

```text
SELECT <select-item> [, <select-item> ...]
FROM   <Entity>
[WHERE      <condition>]
[FACET      <attribute> [, <attribute> ...]]          -- up to 5
[SINCE      <n> <unit> [AGO]]
[UNTIL      NOW | <n> <unit> [AGO]]
[TIMESERIES AUTO | MAX | <n> <unit>]
[ORDER BY   <alias-or-selected-name> [ASC | DESC]]
[LIMIT      <1..5000>]

<select-item> := <function>(<args>) [FILTER (WHERE <condition>)] [AS <alias>]
               | <attribute> [AS <alias>]
<unit>        := second | seconds | minute | minutes | hour | hours | day | days | week | weeks
```

>[!NOTE]
>
> 위의 `--` 주석은 설명서에만 해당됩니다. SQL 자체는 주석을 허용하지 않습니다.

### 필수 절

<details>
<summary><code>SELECT</code> — 반환할 항목</summary>

`SELECT`은(는) **원시 특성**(이벤트 쿼리) 또는 **집계 함수**(집계 쿼리)를 나열합니다. 둘을 섞으면 안 됩니다.

| 양식 | 예 | 결과 |
| ------------ | -------------------------------------------------- | ------------------------------------------------- |
| 속성 | `SELECT timestamp, name, duration` | 개별 이벤트 행(기본값 100) |
| 집계 | `SELECT count(*), average(duration)` | 집계당 하나의 값(Facet/시간 버킷당) |
| 별칭 사용 | `SELECT count(*) AS requests` | 열 레이블 `requests` |
| 따옴표 붙은 별칭 | `SELECT percentile(duration, 95) AS 'p95 latency'` | 공백이 있는 레이블 |

이벤트 목록:

```sql
SELECT timestamp, service, name, duration, status, traceId FROM Transaction WHERE status = 'Error' LIMIT 50
```

여러 집계:

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, max(duration) AS maxMs FROM Transaction
```

규칙:

- `SELECT *`은(는) 지원되지 않습니다. 원하는 속성을 나열합니다.
- 특성 또는 집계(`sum(a) / count(*)`) 간의 계산이 지원되지 않습니다. [공식 위젯](#86-combine-queries-with-formula-widgets)을(를) 사용합니다.
- _원시 특성을 집계와 혼합할 수 없습니다. FACET을 사용하여 결과를 그룹화하십시오_.

</details>

<details>
<summary><code>FROM</code> — 쿼리할 엔터티</summary>

`FROM`은(는) [데이터 사전](#data-dictionary)에서 정확히 **one** 엔터티를 지정합니다. 엔티티 이름은 대소문자를 구분하지 않습니다.

```sql
SELECT count(*) FROM Transaction
```

```sql
FROM Span SELECT count(*) FACET kind
```

각 엔티티는 고정된 범위를 갖습니다. 예를 들어 `Transaction`은(는) 항상 서버 단위이며 `CpuUtilization`은(는) 항상 `value`을(를) 백분율로 변환한 CPU 사용 중 지표입니다. 단일 쿼리에서 두 개 이상의 엔티티를 쿼리할 수 없습니다. 엔터티를 결합하려면 [수식 위젯](#86-combine-queries-with-formula-widgets)을 사용하세요.

</details>

### 선택적 절

<details>
<summary><code>AS</code> — 열에 레이블 지정</summary>

`AS`을(를) 사용하여 집계 또는 특성에 읽을 수 있는 이름을 지정하십시오. 별칭은 열 머리글, 범례 레이블 및 `ORDER BY`에서 참조하는 이름이 됩니다.

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET service ORDER BY p95 DESC LIMIT 10
```

공백이 있는 레이블에 작은따옴표 문자열 사용:

```sql
SELECT percentile(duration, 95) AS 'p95 latency (ms)' FROM Transaction TIMESERIES AUTO
```

</details>

<details>
<summary><code>FACET</code> — 그룹 결과</summary>

`FACET`개의 그룹이 SQL `GROUP BY`과(와) 같이 최대 **5** 특성으로 결과를 그룹화합니다.

```sql
SELECT count(*) FROM Transaction FACET service, http.method, http.statusCode LIMIT 30
```

| 쿼리 | `LIMIT n`의 의미 | 기본값 |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| `TIMESERIES` 없이 `FACET` | 행 수 | 10 |
| `FACET`(`TIMESERIES` 포함) | **Top n 시리즈**, 전체 창의 첫 번째 집계 합계로 순위가 매겨졌습니다. 해당 시리즈의 버킷이 보관될 때마다. | 모든 시리즈(5,000행 제한) |

패싯된 시계열, 상위 5개:

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 5
```

Facet 값은 문자열 또는 숫자일 수 있습니다. 빈 문자열은 고유한 그룹을 형성합니다. `WHERE attr != ''`(으)로 제외합니다.

</details>

<details>
<summary><code>FILTER</code> — 조건부 집계</summary>

행의 하위 집합을 통해 계산하려면 모든 집계에 `FILTER (WHERE condition)`을(를) 추가하십시오. 이를 통해 한 번의 클릭으로 여러 조건을 비교할 수 있습니다.

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, percentile(duration, 95) FILTER (WHERE http.method = 'GET') AS getP95 FROM Transaction FACET service LIMIT 20
```

`FILTER`은(는) `apdex`, `rate` 또는 `derivative`에 사용할 수 없습니다. `percentage`은(는) 이미 자체 조건을 사용합니다.

</details>

<details>
<summary><code>LIMIT</code> — 행 또는 계열 수</summary>

`LIMIT`은(는) 양의 정수를 사용합니다. 5,000을 초과하는 값은 5,000으로 고정됩니다.

| 결과 모양 | 기본값 |
| ------------ | ---------- |
| 게시판 | 1 |
| Facet | 10 |
| 시계열 | 5,000행 |
| 이벤트 | 100 |

```sql
SELECT count(*) FROM Transaction FACET urlPath LIMIT 50
```

>[!TIP]
>
> 쿼리 기반 변수와 카디널리티 패싯(`urlPath`, `traceId`, `jcr.path`, `url`)에 대해 항상 `LIMIT`을(를) 설정하십시오.

</details>

<details>
<summary><code>ORDER BY</code> — 결과 정렬</summary>

`ORDER BY`은(는) **선택한 별칭 또는 선택한 특성**&#x200B;을 기준으로 정렬합니다. 기본 방향은 `DESC`입니다.

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 20
```

| 결과 모양 | 기본 순서 |
| ----------------- | --------------------------------------------------------------------------- |
| 빌보드/패싯 | 첫 번째 집계, `DESC` |
| 시계열 | 시간, `ASC` |
| 이벤트 | 처음 선택한 열 `DESC`(&quot;가장 최근 항목&quot;의 경우 `timestamp`을(를) 먼저 지정) |

규칙:

- 선택하지 않은 항목을 참조하지 못하면 _ORDER BY &#39;x&#39;에서 선택한 값 또는 별칭을 참조해야 합니다_.
- `TIMESERIES` 쿼리에 `ORDER BY`을(를) 추가하지 마십시오. 차트에 필요한 시간 순서를 재정의합니다.

</details>

<details>
<summary><code>SINCE</code> — 창 시작</summary>

`SINCE n unit [AGO]`은(는) 창이 시작되는 거리를 설정합니다. 단위: `second`, `minute`, `hour`, `day`, `week`(단일 또는 복수). `AGO`은(는) 선택 사항입니다.

```sql
SELECT count(*) FROM Transaction SINCE 6 hours ago
```

- 기본값은 `SINCE 1 hour ago`입니다.
- 창은 **31일**&#x200B;을(를) 초과할 수 없습니다(_시간 범위는 31일_&#x200B;을(를) 초과할 수 없음).
- 절대 날짜(`SINCE '2026-01-01'`) 및 `TODAY`/`YESTERDAY`은(는) 지원되지 않습니다.

>[!IMPORTANT]
>
> 대시보드 및 탐색에서 **시간 선택기가 재정의됨** `SINCE` 및 `UNTIL`. [기간 우선 순위](#time-window-precedence)를 참조하십시오.

</details>

<details>
<summary><code>TIMESERIES</code> — 시간에 따른 플롯</summary>

`TIMESERIES`은(는) 선, 영역 또는 막대 차트에 대해 준비된 시간 버킷당(패싯 값당) 한 행을 반환합니다.

| 양식 | 버킷 크기 |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `TIMESERIES 5 minutes` | 정확히 원하는 내용(모든 단위) |
| `TIMESERIES AUTO` | 창 ÷ 120 정보, 1s / 5s / 10s / 15s / 30s / 1m / 5m / 10m / 15m / 30m / 1h / 3h / 6h / 12h / 1d로 반올림됨. **2분 미만은 안 됩니다.** |
| `TIMESERIES MAX` | 1초 버킷. 몇 분 동안만 사용하십시오. |

일반적인 `AUTO` 버킷:

| 창 | 1시간 | 6시간 | 24시간 | 7일 | 30일 |
| ------ | ----- | ----- | ------ | --- | ---- |
| 버킷 | 2분 | 5분 | 15분 | 3시간 | 6시간 |

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction TIMESERIES AUTO
```

```sql
SELECT count(*) FROM Transaction TIMESERIES 1 hour SINCE 1 day ago
```

>[!TIP]
>
> 대시보드에서 `AUTO`을(를) 선호합니다. 시간 선택기가 무엇을 말하든 약 120점을 유지한다. 또한 호스트 지표는 약 90초마다 스크랩되므로 호스트당 &quot;깨진&quot; 행을 방지합니다.

</details>

<details>
<summary><code>UNTIL</code> — 창 끝</summary>

`UNTIL NOW`(기본값) 또는 `UNTIL n unit [AGO]`. `SINCE`과(와) 결합하여 이전 창을 봅니다.

```sql
SELECT count(*) AS yesterday FROM Transaction SINCE 2 days ago UNTIL 1 day ago
```

`SINCE`은(는) `UNTIL`보다 이전이어야 합니다. 그렇지 않으면 _SINCE가 UNTIL_&#x200B;보다 이전이어야 하므로 쿼리가 실패합니다.

</details>

<details>
<summary><code>WHERE</code> — 행 필터링</summary>

`WHERE`은(는) 조건과 일치하는 행만 유지합니다. 조건을 `AND`, `OR`, `NOT` 및 괄호로 묶습니다. [연산자 및 조건](#45-operators-and-conditions)을 참조하세요.

```sql
SELECT count(*) FROM Transaction WHERE service = 'checkout' AND (status = 'Error' OR http.statusCode >= 500)
```

`WHERE`은(는) 한 번만 사용할 수 있습니다. 모든 조건을 넣으세요.

</details>

### 집계 함수

| 함수 | 인수 | 반환 | 작업 시간 |
| -------------------------------------- | ----------------------------------- | ---------------------- | ------------------------------------- |
| [`apdex`](#fn-apdex) | 숫자 기간, 임계값 `T` | 0-1 | 엔티티 추적 |
| [`average` / `avg`](#fn-average) | 숫자 | 입력과 동일한 단위 | 임의 |
| [`count`](#fn-count) | `*` 또는 특성 | 행 수 | 임의 |
| [`derivative`](#fn-derivative) | 숫자 | 초당 변경 | 모두(필요: `TIMESERIES`) |
| [`earliest`](#fn-latest) | 모든 [, 순위] | 속성 값 | 임의 |
| [`latest`](#fn-latest) | 모든 [, 순위] | 속성 값 | 임의 |
| [`max` / `min`](#fn-minmax) | 임의 | 입력과 동일한 단위 | 임의 |
| [`percentage`](#fn-percentage) | `*` 또는 특성, `WHERE` 조건 | % | 임의 |
| [`percentile`](#fn-percentile) | 숫자, 수준 `p` | 입력과 동일한 단위 | 임의 |
| [`percentOfTotal`](#fn-percentoftotal) | 숫자 | % | 모두(`TIMESERIES` 없음) |
| [`rate`](#fn-rate) | 숫자 카운터 | 초당 | 카운터 엔터티(`TIMESERIES` 필요) |
| [`stddev`](#fn-stddev) | 숫자 | 입력과 동일한 단위 | 임의 |
| [`sum`](#fn-sum) | 숫자 | 입력과 동일한 단위 | 임의 |
| [`uniqueCount`](#fn-uniquecount) | 임의 | 고유 값 | 임의 |

<a id="fn-apdex"></a>

<details>
<summary><code>apdex(attribute, T)</code></summary>

특성 단위(`duration`에 대한 밀리초)에서 임계값 `T`을(를) 사용하여 [Apdex](https://www.apdex.org/) 점수를 0(불만족)에서 1(만족)으로 계산합니다.

| 버킷 | 조건 | 두께 |
| ---------- | ------------------------------------ | ------ |
| 충족됨 | `duration ≤ T`(오류 아님) | 1 |
| 허용 | `T < duration ≤ 4T`(오류 아님) | 0.5 |
| 불만족 | `duration > 4T`, **또는 모든 오류** | 0 |

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service
```

- 추적 엔터티(`Transaction`, `AppRequest`, `Span`)에서만 작동합니다.
- `T`은(는) 0보다 커야 합니다. `FILTER`은(는) 허용되지 않습니다.
- `Transaction`의 `apdex(duration, 500)`은(는) 긴 기간에 걸쳐 롤업 가속화되었습니다.

</details>

<a id="fn-average"></a>

<details>
<summary><code>average(attribute)</code> / <code>avg(attribute)</code></summary>

숫자 속성의 산술 평균입니다.

```sql
SELECT average(duration) AS avgMs FROM Transaction FACET service TIMESERIES AUTO
```

</details>

<a id="fn-count"></a>

<details>
<summary><code>count(*)</code> / <code>count(attribute)</code></summary>

- `count(*)`이(가) 행을 계산합니다.
- `count(attribute)`은(는) 특성이 **NULL이 아님**&#x200B;인 행을 계산합니다. 문자열 특성의 경우 누락된 값이 NULL이 아닌 공백(`''`)이므로 계속 계산됩니다([형식 처리](#46-type-handling-blank-vs-null) 참조).

```sql
SELECT count(*) AS requests, count(http.statusCode) AS withStatusCode FROM Transaction
```

</details>

<a id="fn-derivative"></a>

<details>
<summary><code>derivative(attribute)</code></summary>

연속 버킷 간 `average(attribute)`의 초당 변경: (이 버킷− 이전 버킷) ÷ 버킷 초. 결과는 음수일 수 있습니다. 게이지에 그것을 사용하여 값이 커지고 있는지 또는 작아지고 있는지 확인하십시오.

```sql
SELECT derivative(value) AS heapGrowthPerSec FROM JvmHeapUsed WHERE host = 'publish-1' TIMESERIES 5 minutes
```

- `TIMESERIES`이(가) 필요하며 **only** 집계여야 합니다. `FILTER`은(는) 허용되지 않습니다.
- 첫 번째 버킷에는 전임 작업이 없습니다.

</details>

<a id="fn-latest"></a>

<details>
<summary><code>latest(attribute [, rankAttribute])</code> / <code>earliest(…)</code></summary>

- `latest(attr)`은(는) 가장 최근 행에서 값을 반환합니다. `earliest(attr)`은(는) 가장 오래된 행에서 값을 반환합니다.
- 두 번째 인수를 사용하면, 행이 시간 대신 해당 속성에 의해 선택됩니다. `latest(traceId, duration)`이(가) **가장 느림** 요청의 추적 ID를 반환합니다. `earliest(traceId, duration)`이(가) 가장 빠른 값을 반환합니다.

```sql
SELECT latest(value) AS current FROM MemoryUtilization FACET host
```

```sql
SELECT max(duration) AS slowestMs, latest(traceId, duration) AS slowestTrace FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 10
```

</details>

<a id="fn-minmax"></a>

<details>
<summary><code>max(attribute)</code> / <code>min(attribute)</code></summary>

가장 큰 값 또는 가장 작은 값입니다. 숫자, 문자열 및 타임스탬프에 작동합니다.

```sql
SELECT min(duration) AS fastestMs, max(duration) AS slowestMs, max(timestamp) AS lastSeen FROM Transaction FACET service
```

</details>

<a id="fn-percentage"></a>

<details>
<summary><code>percentage(*, WHERE condition)</code></summary>

`condition`과(와) 일치하는 0에서 100 사이의 행 공유입니다.

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction FACET service TIMESERIES AUTO
```

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowPct FROM Transaction
```

`percentage`에는 자체 조건이 있으므로 `FILTER`이(가) 필요하지 않습니다.

</details>

<a id="fn-percentile"></a>

<details>
<summary><code>percentile(attribute, p)</code></summary>

대략적인 p 백분위수입니다. 여기서 `0 < p ≤ 100`입니다. 수준당 하나의 `percentile`을(를) 사용합니다.

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

`Transaction`의 p50, p95 및 p99는 긴 기간에 걸쳐 롤업 가속화되었습니다.

</details>

<a id="fn-percentoftotal"></a>

<details>
<summary><code>percentOfTotal(attribute)</code></summary>

숫자 특성의 **총 합계**&#x200B;에 대한 각 패싯 행의 공유입니다. 행이 최대 100%까지 추가됩니다.

```sql
SELECT sum(duration) AS totalMs, percentOfTotal(duration) AS shareOfTime FROM Transaction FACET name LIMIT 15
```

`TIMESERIES`에는 사용할 수 없습니다.

</details>

<a id="fn-rate"></a>

<details>
<summary><code>rate(attribute)</code></summary>

**누적 카운터**&#x200B;의 초당 비율입니다. 각 버킷에서 `rate`은(는) 계열당 가장 높은 카운터 읽기를 사용하여 이전 버킷과 비교합니다. 카운터 재설정은 0에서 플러시되고, 첫 번째 버킷은 0입니다.

```sql
SELECT rate(value) AS bytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

- 카운터 엔터티([데이터 사전](#53-curated-metric-entities)에서 ✔(으)로 표시됨)에서만 작동합니다.
- `TIMESERIES`이(가) 필요하며 **only** 집계여야 합니다. `FILTER`은(는) 허용되지 않습니다.

>[!WARNING]
>
> 항상 `FACET host`을(를) 선택하거나 단일 호스트로 필터링하십시오. 많은 호스트에서 패싯되지 않은 `rate()`은(는) 서로 다른 호스트의 카운터를 비교하고 의미 없는 숫자를 생성합니다. 디스크 카운터는 장치마다 있으므로 `FACET host, device`이(가) 가장 정확합니다.

</details>

<a id="fn-stddev"></a>

<details>
<summary><code>stddev(attribute)</code></summary>

샘플 표준 편차. `average`과(와) 함께 사용하면 지연 시간이 얼마나 일관되는지 확인할 수 있습니다.

```sql
SELECT average(duration) AS avgMs, stddev(duration) AS stddevMs FROM Transaction FACET name LIMIT 20
```

</details>

<a id="fn-sum"></a>

<details>
<summary><code>sum(attribute)</code></summary>

숫자 속성의 합계입니다.

```sql
SELECT sum(callCount) AS calls FROM SpanBreakdown FACET segment LIMIT 20
```

</details>

<a id="fn-uniquecount"></a>

<details>
<summary><code>uniqueCount(attribute)</code></summary>

고유 값의 **exact** 수입니다.

```sql
SELECT uniqueCount(host) AS hosts, uniqueCount(traceId) AS traces FROM Transaction FACET service
```

</details>

기능 제한 개요:

| 제한 | 함수 |
| ------------------------------------- | ----------------------------------------------------------------------------------------- |
| 숫자 속성 필요 | `average`, `sum`, `percentile`, `stddev`, `percentOfTotal`, `rate`, `derivative`, `apdex` |
| `FILTER (WHERE …)` 없음 | `apdex`, `rate`, `derivative`, `percentage` |
| `TIMESERIES`이(가) 필요하며 집계만 필요합니다. | `rate`, `derivative` |
| `TIMESERIES`과(와) 함께 있지 않음 | `percentOfTotal` |
| 엔티티만 추적 | `apdex` |
| 카운터 엔티티만 | `rate` |

### 연산자 및 조건

| 연산자 | 의미 | 예 |
| ------------------------- | -------------------------------------------------------------------------- | ---------------------------------------- |
| `=` | 등호 | `status = 'Error'` |
| `!=`, `<>` | 같지 않음 | `kind != 'Internal'` |
| `<`, `<=`, `>`, `>=` | 비교 | `http.statusCode >= 500` |
| `LIKE` | 와일드카드 일치: `%` = 모든 문자, `_` = 한 문자 대/소문자를 구분합니다. | `urlPath LIKE '/content/%'` |
| `NOT LIKE` | 무효화된 와일드카드 | `urlPath NOT LIKE '%.css'` |
| `IN (…)` | 1-100 값 중 하나와 일치 | `service IN ('checkout', 'cart')` |
| `NOT IN (…)` | 어느 값과도 일치하지 않습니다. | `http.method NOT IN ('OPTIONS', 'HEAD')` |
| `IS NULL` / `IS NOT NULL` | Null 확인 | `http.statusCode IS NOT NULL` |
| `AND`, `OR`, `NOT` | 논리 | `NOT (status = 'Ok')` |
| `( … )` | 그룹화 | `(a = 1 OR b = 2) AND c = 3` |

```sql
SELECT count(*) FROM Transaction WHERE (status = 'Error' OR duration > 2000) AND NOT host LIKE '%dispatcher%' AND http.method IN ('GET', 'POST')
```

>[!WARNING]
>
> `attr = NULL`은(는) 오류입니다. `attr IS NULL` 사용.

### 유형 처리: 공백과 NULL

모든 특성은 카탈로그에 유형이 있습니다. `string`, `number`, `boolean` 또는 `timestamp`. SQL 형식 검사 기능이 이에 대해 작동합니다(예: `average(service)`은(는) 거부되었습니다). 형식 변환 함수가 필요하지 않거나 사용할 수 없습니다.

>[!IMPORTANT]
>
> **공백과 NULL 비교** 대부분의 문자열 특성은 _누락_ 키가 `NULL`이(가) 아닌 빈 문자열 `''`로 읽히는 OpenTelemetry 특성 맵에서 가져옵니다. 특성을 포함하는 범위만 유지하려면 `IS NOT NULL` 대신 **`attr != ''`**&#x200B;을(를) 사용합니다. `http.statusCode`과(와) 같은 숫자 특성은 없을 때 `NULL`이므로 `IS NOT NULL`이(가) 작동합니다.

```sql
SELECT count(*) AS jcrCalls FROM Span WHERE jcr.operation != '' FACET jcr.operation
```

### SQL을 SQL로 변환

| SQL | SQL |
| ------------------------------------------------------------ | ------------------------------------------------------------------- |
| `WHERE ts > now() - INTERVAL 1 HOUR` | 없음, 시간 선택기(또는 `SINCE 1 hour ago`)가 창을 설정합니다. |
| `GROUP BY service` | `FACET service` |
| `GROUP BY toStartOfInterval(ts, INTERVAL 5 MINUTE), service` | `FACET service TIMESERIES 5 minutes` |
| `quantile(0.95)(duration)` | `percentile(duration, 95)` |
| `countIf(status = 'Error')` | `count(*) FILTER (WHERE status = 'Error')` |
| `100 * countIf(x) / count()` | `percentage(*, WHERE x)` |
| `count(DISTINCT host)` | `uniqueCount(host)` |
| `argMax(traceId, duration)` | `latest(traceId, duration)` |
| `x / sum(x) OVER ()` | `percentOfTotal(x)` |
| `ORDER BY p95 DESC LIMIT 20` | `ORDER BY p95 DESC LIMIT 20` |
| `HAVING count() > 100` | _지원되지 않음_. 대신 정렬 및 `LIMIT`을(를) 합니다. |
| `JOIN` / 하위 쿼리 | _지원되지 않음_. **수식** 위젯을 사용합니다. |
| `CASE WHEN` | _지원되지 않음_. `FILTER`개 열 또는 `WHERE … LIKE` + `FACET`을(를) 사용합니다. |

### 기존 대시보드를 Insights로 가져오기

기존 쿼리를 알고 있는 경우 다음은 마이그레이션 세부 정보입니다

| 기존 쿼리 | SQL | 메모 |
| ------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------ |
| `FROM Transaction SELECT count(*)` | 동일 | `FROM`-첫 번째 순서가 지원됨 |
| `SINCE 1 day ago` | 동일 | 대시보드에서 무시됨, 선택기가 승리함 |
| `TIMESERIES AUTO` / `MAX` / `5 minutes` | 동일 | `AUTO`층은 2분 |
| `FACET appName` | `FACET appName` / `FACET service` |                                                              |
| `percentage(count(*), WHERE error IS true)` | `percentage(*, WHERE status = 'Error')` |                                                              |
| `filter(count(*), WHERE …)` | `count(*) FILTER (WHERE …)` | SQL 표준 양식 |
| `apdex(duration, t: 0.5)` | `apdex(duration, 500)` | `duration`이(가) **ms**&#x200B;에 있습니다. |
| `latest(x)` / `earliest(x)` | 동일 | 등급 양식 추가: `latest(x, rankAttr)` |
| `uniqueCount(x)` | 동일 | 정확 |
| `rate(count(*), 1 minute)` | `count(*)`(`TIMESERIES 1 minute` 포함) | SQL `rate()`은(는) 카운터 전용입니다. |
| `derivative(x, 1 minute)` | `derivative(x)` + `TIMESERIES` | 초당 |
| `uniques(x)` | `count(*) … FACET x LIMIT n` |                                                              |
| `keyset()` / `SHOW EVENT TYPES` | [검색 쿼리](#101-data-discovery) 또는 쿼리 가이드 패널 |                                                              |
| `histogram(duration, …)` | `FILTER`개 버킷 | [자습서 3](#84-build-a-latency-histogram) 보기 |
| `FACET CASES (…)` | `FILTER`열 | [자습서 3](#83-group-values-into-cases-with-filter) 보기 |
| `COMPARE WITH 1 week ago` | 지원되지 않음 | 시간 선택기를 변경하거나 API를 통해 `SINCE … UNTIL …`을(를) 사용하십시오. |
| `RLIKE` | `LIKE` / `IN` |                                                              |
| `SELECT`/중첩된 집계의 수학 | **수식** 위젯 |                                                              |
| `WITH TIMEZONE`, `EXTRAPOLATE`, `SLIDE BY` | 지원되지 않음 |                                                              |
