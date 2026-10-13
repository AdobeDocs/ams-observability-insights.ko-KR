---
source-git-commit: d3cd77e213674587152d6d33efa65328f420a547
workflow-type: tm+mt
source-wordcount: '295'
ht-degree: 10%
---
### SQL 쿼리 예제

다음은 보다 완벽한 예입니다. 이 쿼리의 경우:

- `Transaction`을(를) 엔터티로 선택합니다.
- `SELECT`을(를) 사용하여 95번째 백분위수 기간을 계산합니다.
- `FACET`을(를) 사용하여 서비스별로 결과를 그룹화합니다.
- `TIMESERIES AUTO`을(를) 사용하여 자동으로 크기가 조정된 간격으로 데이터를 그립니다.

```sql
FROM Transaction SELECT percentile(duration, 95) AS p95 FACET service TIMESERIES AUTO
```

이렇게 하면 기본적으로 10개의 가장 바쁜 서비스가 표시된 라인 차트에서 서비스당 한 개의 라인이 생성됩니다.

아래의 각 예를 클릭하여 확장합니다.

<details>
<summary><strong>인바운드 요청의 기본 쿼리</strong></summary>

모든 인바운드 요청의 평균 응답 시간(밀리초):

```sql
SELECT average(duration) FROM Transaction
```

</details>

<details>
<summary><strong>점이 포함된 특성 이름</strong></summary>

점선 이름은 따옴표를 사용하지 않아도 됩니다.

```sql
SELECT count(*) FROM Transaction WHERE http.statusCode >= 500 FACET http.route
```

</details>

<details>
<summary><strong>여러 열을 반환하는 쿼리</strong></summary>

한 쿼리의 여러 합계가 테이블의 열이 됩니다.

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, percentile(duration, 95) AS p95, max(duration) AS maxMs FROM Transaction FACET service
```

</details>

<details>
<summary><strong>한 번의 패스로 조건부 집계</strong></summary>

합계, 오류 및 5xx 나란히:

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, count(*) FILTER (WHERE http.statusCode >= 500) AS http5xx FROM Transaction FACET service
```

</details>

<details>
<summary><strong>큐레이션된 지표 쿼리</strong></summary>

시간 경과에 따른 호스트당 평균 CPU:

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO
```

</details>

<details>
<summary><strong>이름별로 OpenTelemetry 지표를 쿼리하는 중</strong></summary>

조정된 엔터티가 없는 지표에 `MetricGauge` / `MetricSum` 사용:

```sql
SELECT latest(value) AS bytesUsed FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool
```

</details>

<a id="what-sql-doesnt-do"></a>

### SQL에서 수행할 수 없는 작업

SQL은 의도적으로 작습니다. 이렇게 하면 쿼리가 안전하고 신속하며 테넌트가 격리됩니다. 이러한 기능 중 하나가 필요한 경우 대체 기능을 사용하십시오.

| 기능 | 상태 | 대체 |
| ------------------------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------ |
| `SELECT *` | 지원되지 않음 | 원하는 특성을 나열합니다. `SELECT timestamp, name, duration, traceId …` |
| 원시 속성 및 집계 혼합 | 지원되지 않음 | `FACET attr` 또는 `latest(attr)` / `earliest(attr)`을(를) 사용합니다. |
| `SELECT`의 계산(`a - b`, `count(*) / 60`) | 지원되지 않음 | **수식** 위젯(대시보드). |
| 조인 및 하위 쿼리 | 지원되지 않음 | **수식** 위젯 또는 두 개의 위젯이 나란히 표시됩니다. |
| `HAVING` | 지원되지 않음 | `ORDER BY alias` + `LIMIT`. |
| `COMPARE WITH` | 지원되지 않음 | 시간 선택기를 변경하거나 두 개의 위젯을 다른 탭에 배치합니다. |
| `FACET CASES` | 지원되지 않음 | 케이스당 하나의 `count(*) FILTER (WHERE …)` 열([자습서 3](#83-group-values-into-cases-with-filter) 참조). |
| `histogram()` | 지원되지 않음 | `FILTER`개 열이 그룹화되었습니다([자습서 3](#84-build-a-latency-histogram) 참조). |
| `uniques()` / `keyset()` | 지원되지 않음 | `FACET attr LIMIT n`은(는) 고유한 값을 나열합니다. 검색은 지표 및 채워진 속성을 나열합니다. |
| `RLIKE`(정규 표현식) | 지원되지 않음 | `%` 및 `_` 와일드카드가 있는 `LIKE` / `NOT LIKE` 또는 `IN (…)`입니다. |
| 댓글(`--`, `//`, `/* */`) | 지원되지 않음 | 위젯 제목 또는 **텍스트** 위젯에 메모를 넣습니다. |
| `SINCE`의 절대 날짜 | 지원되지 않음 | 시간 선택기의 절대 범위를 사용합니다. |
| 형식 전환(`numeric()`, `string()`) | 필요 없음 | 카탈로그에 특성이 입력됩니다. |
