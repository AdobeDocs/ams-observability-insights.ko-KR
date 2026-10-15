---
title: '자습서 1: 첫 번째 쿼리'
description: Observability Insights 원격 분석에 대해 첫 번째 SQL 쿼리를 작성하십시오.
source-git-commit: 52cc24792c47727700428be58b5bbb2a5c7223d2
workflow-type: tm+mt
source-wordcount: '552'
ht-degree: 0%
---

## 자습서 1: 첫 번째 쿼리 {#tutorial-1-your-first-queries}

SQL을 시작합니다. 이 튜토리얼에서는 첫 번째 쿼리를 작성하고, 시간 창을 제어하고, 트렌드를 표시하고, 결과를 필터링하고, 그룹화하고, 대시보드에 첫 번째 차트를 배치합니다. 각 단계는 이전 단계를 기반으로 합니다.

### 시작하기에 앞서

- Observability Insights에 액세스하고 추적 또는 지표를 보내는 하나 이상의 고객 테넌트에 액세스해야 합니다.
- 테넌트 전환기에서 **customer**&#x200B;을(를) 선택하십시오. **모든 계정**&#x200B;을 선택하는 동안에는 SQL이 실행되지 않습니다.
- **탐색**&#x200B;을 열고 **SQL** 탭으로 전환합니다. 각 쿼리 뒤에 **실행**(⌘/Ctrl + Enter)을 사용하십시오.

### `SELECT`, `FROM` 및 `LIMIT`을(를) 사용하여 이벤트 쿼리

모든 쿼리에는 두 개의 절이 필요합니다. `SELECT`은(는) 반환할 항목을 나타내며 `FROM`은(는) 읽을 엔터티를 나타냅니다. 인바운드 요청으로 시작:

```sql
SELECT timestamp, service, name, duration FROM Transaction
```

**이벤트 쿼리**&#x200B;입니다. 각 행은 하나의 요청입니다. `LIMIT`이(가) 없으면 최대 100개의 행이 제공됩니다. 간단히 요청하십시오.

```sql
SELECT timestamp, service, name, duration FROM Transaction LIMIT 10
```

행은 첫 번째 열을 기준으로 내림차순으로 정렬되므로 `timestamp`을(를) 먼저 놓으면 맨 위에 최신 요청이 표시됩니다.

> **참고**
>
> SQL에 `SELECT *`이(가) 없습니다. 속성을 나열하면 결과를 작고 읽을 수 있습니다. [데이터 사전](#data-dictionary) 및 편집기의 자동 완성이 사용 가능한 항목을 표시합니다.

### 데이터 집계

대부분의 대시보드에는 개별 이벤트가 표시되지 않습니다. 요약됩니다. 특성을 **집계 함수**(으)로 바꾸기:

```sql
SELECT count(*) FROM Transaction
```

창의 인바운드 요청 수입니다. 몇 가지 더 시도해 보십시오.

```sql
SELECT average(duration) FROM Transaction
```

```sql
SELECT max(duration) FROM Transaction
```

```sql
SELECT uniqueCount(host) FROM Transaction
```

### 한 번에 여러 개의 집계 반환

합계를 쉼표로 구분하고 각각 이름을 `AS`(으)로 지정합니다.

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, max(duration) AS maxMs FROM Transaction
```

> **팁**
>
> 별칭은 열 머리글과 범례 레이블이 되며 `ORDER BY`에서 참조하는 별칭입니다. 모든 집계에 이름을 붙이는 습관을 들여라.

### 시간 창 설정

기본적으로 쿼리는 **지난 시간**&#x200B;에 적용됩니다. `SINCE`을(를) 사용하여 더 자세히 되돌아보기:

```sql
SELECT count(*) AS requests FROM Transaction SINCE 1 day ago
```

`UNTIL`을(를) 사용하여 과거 창을 종료합니다. 이는 어제의 트래픽을 48시간에서 24시간 전으로 계산합니다.

```sql
SELECT count(*) AS requests FROM Transaction SINCE 2 days ago UNTIL 1 day ago
```

단위는 `second`, `minute`, `hour`, `day` 또는 `week`일 수 있습니다. 기간은 최대 31일이 될 수 있습니다.

> **중요**
>
> **탐색** 및 **대시보드**&#x200B;에서 페이지 상단의 **시간 선택기**&#x200B;가 창을 제어하고 `SINCE`/`UNTIL`을(를) 재정의합니다. 다른 기간을 보려면 선택기를 변경합니다. 이 안내서의 나머지 부분에서는 이러한 이유로 `SINCE`개의 대시보드 쿼리를 제외합니다.

### `TIMESERIES` 플롯 트렌드

`TIMESERIES`을(를) 추가하여 시간이 지남에 따라 단일 숫자를 행으로 전환합니다.

```sql
SELECT count(*) AS requests FROM Transaction TIMESERIES AUTO
```

`AUTO`이(가) 버킷 크기를 선택하므로 차트가 약 120포인트(1시간 동안 2분, 24시간 동안 15분)입니다. 버킷을 명시적으로 설정할 수도 있습니다.

```sql
SELECT average(duration) AS avgMs FROM Transaction TIMESERIES 5 minutes
```

### `WHERE`(으)로 필터링

`WHERE`은(는) 관심 있는 행만 유지합니다. 실패한 요청만 계산:

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error'
```

조건을 `AND`, `OR` 및 괄호로 묶습니다.

```sql
SELECT count(*) AS slowOrFailed FROM Transaction WHERE service = 'checkout' AND (status = 'Error' OR duration > 2000)
```

부분 일치에 `%`과(와) 함께 `LIKE` 사용:

```sql
SELECT count(*) AS contentRequests FROM Transaction WHERE urlPath LIKE '/content/%'
```

> **참고**
>
> 문자열 값은 대/소문자를 구분하며 `"Error"` 또는 `'error'`이(가) 아닌 작은 따옴표 `'Error'`을(를) 사용합니다. `checkout`을(를) 테넌트의 서비스로 바꾸십시오. 다음 단계에서는 이러한 항목을 나열하는 방법을 보여 줍니다.

### `FACET`이(가) 있는 그룹

`FACET`개의 그룹 결과가 SQL의 `GROUP BY`과(와) 같습니다. 서비스 및 각 서비스의 사용 빈도를 나열합니다.

```sql
SELECT count(*) AS requests FROM Transaction FACET service
```

패싯된 쿼리는 기본적으로 상위 10개 그룹을 반환합니다. `LIMIT`을(를) 사용하여 더 보기:

```sql
SELECT count(*) AS requests, average(duration) AS avgMs FROM Transaction FACET service LIMIT 50
```

`FACET`과(와) `TIMESERIES`을(를) 결합하여 그룹당 한 줄을 가져옵니다.

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 5
```

여기에서 `LIMIT 5`은(는) 가장 바쁜 5개의 시리즈를 유지합니다.

### 쿼리를 대시보드에 저장

1. **대시보드→ 열고 대시보드 만들기 → 사용자 지정 대시보드를 빌드합니다**. 이름을 _내 첫 번째 대시보드_&#x200B;로 지정합니다.
2. **위젯 추가**&#x200B;를 클릭하고 **단일 쿼리**&#x200B;를 선택한 다음 토글을 **SQL**(으)로 전환하세요.
3. 6.7단계의 쿼리를 붙여 넣고 **쿼리 실행**&#x200B;을 클릭합니다.
4. **제목**&#x200B;을(를) _서비스별 요청_(으)로 설정하고 **줄** 차트 유형을 선택하십시오.
5. 상단 막대에서 **위젯 저장**&#x200B;을 클릭한 다음 **저장**&#x200B;을 클릭합니다.

첫 번째 대시보드를 빌드했습니다.
