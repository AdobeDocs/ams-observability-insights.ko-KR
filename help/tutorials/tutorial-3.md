---
title: '자습서 3: 대시보딩 진행'
description: 가시성 인사이트에서 변수, 공식 및 여러 위젯을 사용하여 대시보드를 작성합니다.
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '902'
ht-degree: 3%
---

## 자습서 3: 대시보딩 진행 {#tutorial-3-advance-your-dashboarding}

이 자습서에서는 사용자 경험 점수, 카운터 비율, 사례 그룹화, 히스토그램, 임계값이 있는 KPI 타일, 섹션, 공식 위젯 및 변수 등의 프로덕션 등급 대시보드의 구성 요소를 빌드합니다.

**이 자습서에서:** [8.1 apdex](#81-score-user-experience-with-apdex) · [8.2 비율](#82-turn-counters-into-rates-with-rate) · [8.3 사례](#83-group-values-into-cases-with-filter) · [8.4 히스토그램](#84-build-a-latency-histogram) · [8.5 KPI 타일 및 섹션](#85-build-kpi-tiles-and-sections) · [8.6 공식](#86-combine-queries-with-formula-widgets) · [8.7 변수](#87-make-dashboards-reusable-with-variables) · [8.8 종합](#88-put-it-together)

### `apdex`(으)로 사용자 경험 점수 매기기

Apdex는 지연 시간 및 오류를 0에서 1까지 하나의 점수로 전환합니다. 500ms 임계값을 사용하는 경우 요청은 _충족됨_(500ms 이하), _허용_(최대 2,000ms), _불만족_(해당 **이상 또는 실패한 경우**)입니다.

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

서비스별 트렌드 표시:

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

임계값이 `0 → Red`, `0.7 → Amber`, `0.85 → Green`인 **숫자** 위젯에 넣습니다. 가장 일치하는 밴드가 승리하므로 낮은 점수는 빨간색으로 유지됩니다.

### 8.2 카운터를 `rate`(으)로 전환합니다.

카운터 엔터티([데이터 사전](#53-curated-metric-entities)의 ✔)는 계속 증가하는 합계를 저장합니다. `rate(value)`이(가) 초당 값으로 변환합니다. `TIMESERIES`이(가) 필요하며 유일한 집계여야 합니다.

```sql
SELECT rate(value) AS rxBytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS writeBytesPerSec FROM DiskWriteRate FACET host, device TIMESERIES AUTO LIMIT 10
```

>[!WARNING]
>
> `host`(및 디스크의 경우 `device`)별로 패싯 `rate()`을(를) 항상 지정하거나 하나의 호스트로 필터링하십시오. 그렇지 않으면 서로 다른 숙주의 판독값이 한데 뒤섞여 결과가 무의미해진다.

추적에서 **요청** 비율을 가져오려면 대신 고정 버킷에서 요청을 계산하십시오. 이는 분당 요청을 제공합니다.

```sql
SELECT count(*) AS requestsPerMinute FROM Transaction TIMESERIES 1 minute
```

### `FILTER`을(를) 사용하여 값을 대/소문자로 그룹화

NRQL의 `FACET CASES`은(는) 행을 명명된 버킷으로 그룹화합니다. SQL에서 대/소문자당 `FILTER` 열을 하나 작성하십시오. 다음은 서비스별 상태 클래스입니다.

```sql
SELECT count(*) FILTER (WHERE http.statusCode < 400) AS ok, count(*) FILTER (WHERE http.statusCode >= 400 AND http.statusCode < 500) AS clientErrors, count(*) FILTER (WHERE http.statusCode >= 500) AS serverErrors FROM Transaction FACET service LIMIT 20
```

동일한 아이디어는 시간이 지남에 따라 작동하며, 케이스당 한 줄을 제공합니다.

```sql
SELECT count(*) FILTER (WHERE urlPath LIKE '/content/%') AS pages, count(*) FILTER (WHERE urlPath LIKE '/bin/%') AS servlets, count(*) FILTER (WHERE urlPath LIKE '/graphql/%') AS graphql FROM Transaction TIMESERIES AUTO
```

### 지연 히스토그램 작성

SQL에 `histogram()` 함수가 없습니다. `FILTER`을(를) 사용하여 버킷을 직접 만들고 결과를 **막대** 또는 **테이블** 위젯으로 렌더링합니다.

```sql
SELECT count(*) FILTER (WHERE duration <= 100) AS 'le 100ms', count(*) FILTER (WHERE duration > 100 AND duration <= 250) AS '100-250ms', count(*) FILTER (WHERE duration > 250 AND duration <= 500) AS '250-500ms', count(*) FILTER (WHERE duration > 500 AND duration <= 1000) AS '0.5-1s', count(*) FILTER (WHERE duration > 1000 AND duration <= 2000) AS '1-2s', count(*) FILTER (WHERE duration > 2000) AS 'over 2s' FROM Transaction
```

`FACET service`을(를) 추가하여 여러 서비스의 분포를 나란히 비교합니다.

### KPI 타일 및 섹션 작성

좋은 대시보드는 KPI 타일 행으로 시작한 다음 섹션에서 세부 정보 차트를 그룹화합니다.

1. 각각 3열 너비의 **숫자** 위젯 4개 추가:

   | 제목 | 쿼리 | 단위 | 임계값 |
   | -------- | --------------------------------------------------------------------------- | ---- | ---------------------------------------- |
   | 요청 | `SELECT count(*) AS requests FROM Transaction` |      |                                          |
   | 오류 % | `SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction` | `%` | `0 → Green`, `1 → Amber`, `5 → Red` |
   | p95 | `SELECT percentile(duration, 95) AS p95 FROM Transaction` | `ms` | `0 → Green`, `800 → Amber`, `2000 → Red` |
   | Apdex | `SELECT apdex(duration, 500) AS apdex FROM Transaction` |      | `0 → Red`, `0.7 → Amber`, `0.85 → Green` |

2. **텍스트** 위젯을 추가하고 _섹션 머리글로 만들기_&#x200B;를 선택한 다음 **지연**&#x200B;이라고 합니다. 지연 차트를 그 아래에 추가합니다.
3. 다른 섹션 헤더 **오류**&#x200B;를 추가한 다음 오류 차트를 추가합니다. V자형 화살표로 섹션을 축소하여 다른 부분에 초점을 맞춥니다.
4. 다른 대상에 대해 **탭**&#x200B;을 사용하십시오(예: _개요_, _AEM_, _인프라_). 표시되는 탭만 쿼리를 실행합니다.

### 쿼리를 수식 위젯과 결합

SQL에서 `SELECT`을(를) 계산할 수 없으며 하위 쿼리가 없습니다. **수식 위젯**&#x200B;이 이 간격을 채웁니다. 이 위젯은 이름이 지정된 2-6개의 쿼리를 실행하고 그 결과를 `+ - * / ( )`과(와) 결합합니다.

**예: 시간 경과에 따른 호스트당 요청**(수식, 줄)

1. **수식→ 위젯 추가**.
2. 첫 번째 쿼리의 이름을 `requests`(으)로 지정하고 두 번째 `hosts`(으)로 지정합니다.
3. 쿼리와 수식을 입력한 다음 **수식 실행**&#x200B;을 클릭합니다.

`requests` 쿼리:

```sql
SELECT count(*) AS v FROM Transaction TIMESERIES AUTO
```

`hosts` 쿼리:

```sql
SELECT uniqueCount(host) AS v FROM Transaction TIMESERIES AUTO
```

공식:

```text
requests / hosts
```

**예: 데이터베이스에서 보낸 시간 공유**(수식, 숫자, 단위 `%`)

`db` 쿼리:

```sql
SELECT sum(duration) AS v FROM Span WHERE db.system != ''
```

`server` 쿼리:

```sql
SELECT sum(duration) AS v FROM Transaction
```

공식:

```text
(db / server) * 100
```

공식 쿼리 규칙:

| 규칙 | 세부 사항 |
| --------------- | -------------------------------------------------------------------------------------------------------------------- |
| 이름 | 문자, 숫자 및 `_`(문자 또는 `_`(으)로 시작), 최대 40자, 고유 |
| 표현식 | `+ - * / ( )`, 숫자 및 쿼리 이름, 최대 500자 |
| 동일한 모양 | 모든 쿼리는 일반 합계(숫자)이고, **또는**&#x200B;은(는) 모두 **동일한 버킷 크기**&#x200B;을(를) 가진 `TIMESERIES`입니다(선/영역/막대). |
| 각각 지표 1개 | 각 쿼리는 정확히 하나의 집계를 반환하며 `FACET`은(는) 반환하지 않습니다. |
| 데이터 누락 | 값이 없거나 0으로 나누면 빈 점이 생성되며 `Infinity` 또는 `NaN`이(가) 되지 않습니다. |

>[!TIP]
>
> 시계열 수식의 모든 쿼리에 `TIMESERIES AUTO`을(를) 사용합니다. 쿼리는 하나의 창을 공유하므로 항상 동일한 버킷 크기로 확인됩니다.

### 변수를 사용하여 대시보드를 재사용할 수 있도록 설정

변수를 사용하면 하나의 대시보드에서 많은 서비스, 계층 또는 임계값을 제공할 수 있습니다. 각 변수는 변수 모음에서 선택기입니다. 위젯 제목 및 텍스트 위젯에서 `{{name}}`(그리고 아래 제한이 수정되면 위젯 SQL에서)로 참조합니다.

**변수 만들기**(_조직 및 슈퍼 관리자_): **+ 변수**&#x200B;를 클릭하십시오.

| 필드 | 메모 |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **이름** | 문자, 숫자 및 `_`(문자 또는 `_`(으)로 시작) 고유해야 합니다. `{{name}}`(으)로 사용되었습니다. |
| **레이블** | 선택기에 표시되는 내용(선택 사항). |
| **유형** | **사용자 지정 목록**(한 줄에 하나의 값), **쿼리에서**(한 개의 `FACET`이(가) 있는 SQL 쿼리, 고유한 값이 옵션이 됨) 또는 **텍스트 상자**(자유 텍스트). |
| **여러 값 선택 허용** | 쉼표로 구분된 목록으로 확장합니다. `IN (…)`에 사용합니다. |
| **값이 숫자입니다** | 따옴표가 없는 값을 삽입합니다(`'500'` 대신 `500`). |

**대체 작동 방식**

| 선택 항목 | `{{svc}}`이(가) |
| ---------------------------------- | ------------------------------------------------------- |
| 단일 문자열 `checkout` | `'checkout'` |
| 여러 값 `checkout`, `cart` | `'checkout', 'cart'` |
| 번호 `500` | `500` |
| `'`을(를) 포함하는 값 | `''`(으)로 이스케이프 처리됨 |
| 선택한 항목 없음 | 위젯이 _값을 선택하여{{svc}}_&#x200B;을(를) 표시합니다. |

```text
WHERE service = {{svc}}            single select
WHERE service IN ({{svc}})         multiple select
WHERE duration > {{slowMs}}        numeric text box
```

쿼리 기반 변수의 경우 **항상`LIMIT`**&#x200B;을(를) 추가합니다. 패싯된 쿼리는 기본적으로 상위 10개 값만 반환합니다. 최대 200개의 서비스가 나열됩니다.

```sql
SELECT count(*) FROM Transaction FACET service LIMIT 200
```

테넌트 또는 시간 선택기가 변경되고 옵션이 사전순으로 정렬되면 새로 고침됩니다. 현재 선택이 더 이상 유효하지 않으면 첫 번째 값(단일 선택) 또는 모든 값(다중 선택)이 자동으로 선택됩니다. 변수의 쿼리는 다른 변수를 참조할 수 없습니다.

>[!WARNING]
>
> **현재 빌드 제한** 대체는 대시보드 그리드에서 발생하지만 **위젯 추가** 미리 보기 및 **서버측 저장 확인**&#x200B;이(가) 원시 `{{name}}` 토큰을 받고 `Unexpected character '{'`(으)로 거부합니다. 실제로:
>
> - **위젯 제목** 및 **텍스트 위젯**&#x200B;의 `{{variables}}`이(가) 끝까지 작동합니다.
> - **가져오기** 후에 **위젯 SQL** 내의 `{{variables}}`이(가) 올바르게 렌더링되지만 해당 위젯이 있는 동안에는 대시보드가 **저장**&#x200B;될 수 없습니다.
>
> 수정되기 전까지 리터럴 필터(예: `WHERE service = 'checkout'`)로 컨텍스트별 위젯을 빌드하고 **복제**&#x200B;를 사용하여 복제합니다.

### 합쳐

이제 [대시보드 블루프린트](#dashboard-blueprints)에 사용된 모든 빌딩 블록이 있습니다. 임계값, 섹션, 백분위수, Apdex, 비율, 대/소문자 열, 히스토그램, 수식 및 변수가 있는 KPI 타일입니다. 블루프린트는 이를 완벽한 골든 시그널, 오류 등급 지정, AEM, 인프라 및 JVM 대시보드로 결합하는 방법을 보여 줍니다.
