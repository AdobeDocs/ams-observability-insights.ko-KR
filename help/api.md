---
title: Observability Insights 공용 API
description: Observability Insights Public API를 사용하면 요청 개요, 서비스 카탈로그, 추적 및 지표와 같은 고유한 관찰 가능한 데이터를 고유한 도구, 스크립트 및 대시보드로 직접 가져올 수 있습니다.
source-git-commit: f9361af48539ab50bcde6265963247714621c0ce
workflow-type: tm+mt
source-wordcount: '1135'
ht-degree: 7%
---

# Observability Insights 공용 API

Observability Insights Public API를 사용하면 요청 개요, 서비스 카탈로그, 추적 및 지표와 같은 고유한 관찰 가능한 데이터를 고유한 도구, 스크립트 및 대시보드로 직접 가져올 수 있습니다.

- **API 기본 URL(API_BASE_URL):** `https://insights.adobecqms.net/`
- HTTPS를 통한 **형식:** JSON
- **인증:** API 키(전달자 토큰)

> 이 문서 전체에서 `{{API_BASE_URL}}`을(를) Observability Insights 인스턴스의 API 호스트(예: `https://insights.adobecqms.net/`)로 바꾸십시오.

## &#x200B;1. API 키 가져오기

API 키는 계정에 연결되어 있고 단일 조직에 범위가 지정된 개인 자격 증명입니다. 키는 자신이 생성된 조직에 속한 테넌트에 대한 데이터만 읽을 수 있습니다. 다른 조직의 데이터는 볼 수 없습니다.

### 키 생성

1. [Observability Insights 대시보드](https://insights.adobecqms.net/)에 로그인합니다.
2. **API 키**→ 프로필 메뉴(오른쪽 상단)를 엽니다.
   ![API 키 메뉴](v2-assets/api-key.png)
3. **API 키** 탭에서 **키 생성**&#x200B;을 클릭합니다.
   ![API 키 생성](v2-assets/api-key-gen.png)
4. 수사적 이름(예: `CI pipeline`, `Grafana datasource`)을 지정하고, 범위를 지정할 조직을 선택한 다음 선택적으로 만료 날짜를 설정하십시오.
5. **키 생성**&#x200B;을 클릭합니다. 키가 **once** 형식으로 표시됩니다.

   ```
   synx_9pQ2v6f1WYbLZk3n0aRtEo4jXcHsVmDgUiPq7B8l1yc
   ```

   **즉시 복사하고 안전한 곳에 저장하십시오**(비밀 관리자, CI 비밀 저장소 등) — 대시보드에서 다시 표시할 수 없습니다. 분실한 경우 취소하고 새 파일을 생성합니다.

### 기존 키 관리

API 키 섹션에는 조직, 생성 날짜, 만료 및 마지막으로 사용한 타임스탬프를 포함하여 생성한 모든 키가 나열됩니다. **해지**&#x200B;를 위한 키 옆에 있는 휴지통 아이콘을 클릭합니다. 해지 즉시 실행 취소할 수 없습니다.

### 주요 보안

- API 키를 암호와 동일하게 취급합니다. 키가 있는 사람은 해지되거나 만료되기 전까지 범위가 지정된 조직의 모든 테넌트에 대한 모든 가시성 데이터를 읽을 수 있습니다.
- 소스 제어에 키를 커밋하거나 일반 텍스트(채팅, 이메일, 티켓)로 공유하지 마십시오.
- 키를 주기적으로 회전하고 더 이상 사용되지 않는 키를 취소합니다.
- 키가 손상된 경우 **조직 설정 → API 키**&#x200B;에서 즉시 취소하고 대체 키를 생성하십시오.


## &#x200B;2. 요청 인증

공개 API에 대한 모든 요청에는 `Authorization` 헤더에 키가 포함되어야 합니다.

```
Authorization: Bearer synx_9pQ2v6f1WYbLZk3n0aRtEo4jXcHsVmDgUiPq7B8l1yc
```

유효한 키가 없거나 만료/해지된 키가 있는 요청은 `401 Unauthorized`을(를) 받습니다. 세션 로그인(브라우저 쿠키/토큰)이 이 API에서 허용되는 **not**&#x200B;입니다.

## &#x200B;3. 기본 개념

### 임차인

모든 끝점에는 읽을 테넌트의 데이터를 식별하는 `tenant_id` 쿼리 매개 변수가 필요합니다. 키는 만든 조직에 속한 테넌트만 쿼리할 수 있습니다. 해당 조직 외부의 테넌트를 요청하면 `403 Forbidden`이(가) 반환됩니다. 이 API에는 &quot;모든 테넌트&quot; 모드가 없습니다. 항상 특정 `tenant_id`을(를) 전달하십시오.

키가 사용할 수 있는 `tenant_id` 값을 확인하지 못했습니까? [`GET /public/v1/tenants`](#get-publicv1tenants)을(를) 호출합니다. 키에 쿼리할 권한이 있는 테넌트가 정확히 나열됩니다.

### 시간 범위

`from`/`to` 매개 변수를 허용하는 끝점은 Unix 타임스탬프(초), 밀리초 타임스탬프 또는 ISO 8601 날짜/시간 문자열을 사용합니다. 예:

```
from=1735689600
from=2025-01-01T00:00:00Z
```

생략하면 대부분의 엔드포인트가 최근 롤링 창으로 기본 설정됩니다(아래 각 엔드포인트 참조).

### 비율 제한

요청은 API 키당 속도가 제한됩니다. 한도를 초과하면 다음을 받게 됩니다.

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60

{ "error": "Too Many Requests", "message": "Rate limit of 300 requests/60s exceeded" }
```

`Retry-After` 헤더에서 시간(초)이 지난 후 다시 시도하세요. 사용 사례에 더 높은 제한이 필요한 경우 지원 센터에 문의하십시오.

### 오류수

오류는 `error` 필드와 일반적으로 사람이 읽을 수 있는 `message`을(를) 사용하여 JSON으로 반환됩니다.

```json
{ "error": "Bad Request", "message": "tenant_id is required" }
```

| 상태 | 의미 |
| ------------------------- | ------------------------------------------------------------------ |
| `400 Bad Request` | 누락되었거나 잘못된 매개 변수(예: `tenant_id` 없음, 잘못된 시간 범위) |
| `401 Unauthorized` | API 키 누락, 잘못됨, 만료됨 또는 해지됨 |
| `403 Forbidden` | 요청한 테넌트에 대해 키가 승인되지 않았습니다. |
| `429 Too Many Requests` | 속도 제한 초과 — `Retry-After` 참조 |
| `502 Bad Gateway` | 업스트림 쿼리 실패 — 안전하게 다시 시도 |
| `503 Service Unavailable` | 일시적으로 데이터 백엔드를 사용할 수 없음 |

## &#x200B;4. 엔드포인트

### `GET /public/v1/tenants`

키가 쿼리할 권한이 있는 테넌트 ID를 나열합니다. 먼저 이를 호출하십시오. 다른 모든 끝점에는 이러한 값 중 하나가 `tenant_id`(으)로 필요합니다.

```bash
curl -s "{{API_BASE_URL}}/public/v1/tenants" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{ "tenants": ["tenant1", "tenant2"] }
```

### `GET /public/v1/overview`

일정 기간 동안의 테넌트에 대한 높은 수준 상태 KPI: 요청 볼륨, 오류율 및 지연 백분위수.

| 매개 변수 | 필수 | 설명 |
| ------------ | -------- | ------------------------------------------------------------------------- |
| `tenant_id` | 예 | 쿼리할 테넌트 |
| `from`, `to` | 아니요 | 시간 범위([시간 범위](#time-ranges) 참조) |
| `minutes` | 아니요 | `from`/`to`이(가) 제공되지 않은 경우 &quot;마지막 N분&quot;에 대한 축약(기본값 `15`) |

```bash
curl -s "{{API_BASE_URL}}/public/v1/overview?tenant_id=<tenant_id>&minutes=30" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "tenant_id": "<tenant_id>",
  "from": 1735689000,
  "to": 1735690800,
  "total_spans": 48213,
  "errors": 112,
  "error_rate_pct": 0.23,
  "p50_ms": 34,
  "p95_ms": 210,
  "p99_ms": 480,
  "service_count": 12,
  "trace_count": 9021
}
```

### `GET /public/v1/services`

테넌트에 대한 개별 서비스 이름 보고를 나열합니다.

| 매개 변수 | 필수 | 설명 |
| ------------ | -------- | --------------------------------------------------------------------- |
| `tenant_id` | 예 | 쿼리할 테넌트 |
| `from`, `to` | 아니요 | 이 창에 표시되는 서비스로 제한합니다. 기본값은 최근 7일로 설정됩니다. |

```bash
curl -s "{{API_BASE_URL}}/public/v1/services?tenant_id=<tenant_id>" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "tenant_id": "<tenant_id>",
  "services": ["checkout-api", "payments-worker", "web-frontend"]
}
```

### `GET /public/v1/traces`

선택적 필터를 사용하여 테넌트에 대한 최근 추적을 검색합니다.

| 매개 변수 | 필수 | 설명 |
| ----------------- | -------- | ------------------------------------------------- |
| `tenant_id` | 예 | 쿼리할 테넌트 |
| `from`, `to` | 아니요 | 시간 범위. 기본값은 최근 24시간입니다. |
| `limit` | 아니요 | 반환할 최대 행 수(1-200, 기본값 100) |
| `offset` | 아니요 | 페이지 매김 오프셋(기본값 0) |
| `service` | 아니요 | 서비스 이름으로 필터링 |
| `app_name` | 아니요 | 애플리케이션/인스턴스 이름으로 필터링 |
| `status` | 아니요 | 추적 상태별로 필터링: `ok`, `error` 또는 `unset` |
| `search` | 아니요 | 범위/작업 이름에서 자유 텍스트 검색 |
| `min_duration_ms` | 아니요 | 이 기간 이상에서만 추적합니다. |

```bash
curl -s "{{API_BASE_URL}}/public/v1/traces?tenant_id=<tenant_id>&status=error&limit=25" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "data": [
    {
      "TraceId": "4bf92f3577b34da6a3ce929d0e0e4736",
      "ServiceName": "checkout-api",
      "DurationMs": 812,
      "StatusCode": "Error",
      "Timestamp": "2026-08-30T09:12:44Z"
    }
  ],
  "rows": 137,
  "limit": 25,
  "offset": 0
}
```

`rows`(일치하는 총 개수)을(를) `limit`/`offset`과(와) 함께 사용하여 결과를 페이징합니다.

### `GET /public/v1/traces/:traceId`

단일 추적에 대한 전체 스팬 폭포를 반환합니다.

| 매개 변수 | 필수 | 설명 |
| ----------- | -------- | ---------------------------------------- |
| `tenant_id` | 예 | 추적이 속한 테넌트 |
| `limit` | 아니요 | 반환에 대한 최대 범위(1-500, 기본값 500) |
| `offset` | 아니요 | 매우 큰 트레이스에 대한 페이지 매김 오프셋 |

```bash
curl -s "{{API_BASE_URL}}/public/v1/traces/4bf92f3577b34da6a3ce929d0e0e4736?tenant_id=<tenant_id>" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "spans": [
    {
      "SpanId": "00f067aa0ba902b7",
      "Name": "POST /checkout",
      "DurationMs": 812,
      "children": []
    }
  ],
  "totalDurationMs": 812,
  "spanCount": 14,
  "limit": 500,
  "offset": 0
}
```

### `GET /public/v1/metrics`

테넌트에 대한 원시 지표 데이터 포인트를 반환합니다.

| 매개 변수 | 필수 | 설명 |
| ---------------------------------- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `tenant_id` | 예 | 쿼리할 테넌트 |
| `metric` | `metric`/`like` 중 하나 | 정확한 지표 이름 |
| `like` | `metric`/`like` 중 하나 | 여러 지표 이름과 일치하는 SQL `LIKE` 패턴 |
| `type` | 아니요 | `gauge`(기본값) 또는 `sum` |
| `from`, `to` | 아니요 | 시간 범위. 기본값은 최근 24시간입니다. |
| `service` | 아니요 | 서비스 이름으로 필터링 |
| `host` | 아니요 | 호스트 이름별로 필터링합니다. 아래의 인프라 호스트 지표에 필요 — 이 지표가 없으면 테넌트에 있는 모든 호스트의 결과가 함께 혼합됩니다. |
| `attribute_key`, `attribute_value` | 아니요 | 특정 지표 속성으로 필터링(함께 사용해야 함) |

```bash
curl -s "{{API_BASE_URL}}/public/v1/metrics?tenant_id=<tenant_id>&metric=jvm.memory.used&type=gauge" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "data": [
    {
      "TimeUnix": "2026-08-30T09:00:00Z",
      "MetricName": "jvm.memory.used",
      "Value": 512482816,
      "ServiceName": "checkout-api",
      "host": ""
    }
  ],
  "rows": 1
}
```

#### 인프라 호스트 지표

또한 동일한 끝점은 인프라 대시보드에 표시된 호스트 수준 지표(CPU, 메모리, 로드 평균, 디스크 I/O, 네트워크 I/O)를 제공합니다. 항상 `host`을(를) 사용하여 다음과 같은 정확한 `metric`/`attribute_key`/`attribute_value` 조합을 사용합니다.

| 대시보드 위젯 | `metric` | `attribute_key` | `attribute_value` |
| --------------------- | ------------------------------------- | --------------- | ------------------------------------------------------------------------------------------- |
| CPU % | `system.cpu.utilization` | `state` | `idle`(&quot;사용 중&quot;인 경우 1에서 빼기) 또는 쿼리 `user`/`system`/`iowait` 개별 및 합계 |
| 메모리 사용량 % | `system.memory.utilization` | `state` | `used` |
| 평균 로드(1m) | `system.cpu.load_average.1m` | — | — |
| 디스크 읽기 I/O | `system.disk.io` (`type=sum`) | `direction` | `read` |
| 디스크 쓰기 I/O | `system.disk.io` (`type=sum`) | `direction` | `write` |
| 디스크 읽기 작업 | `system.disk.operations` (`type=sum`) | `direction` | `read` |
| 디스크 쓰기 작업 | `system.disk.operations` (`type=sum`) | `direction` | `write` |
| 네트워크 위치 | `system.network.io` (`type=sum`) | `direction` | `receive` |
| 네트워크 출력 | `system.network.io` (`type=sum`) | `direction` | `transmit` |

```bash
curl -s "{{API_BASE_URL}}/public/v1/metrics?tenant_id=<tenant_id>&metric=system.cpu.utilization&type=gauge&attribute_key=state&attribute_value=idle&host=<host_name>" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

**중요 — 디스크 및 네트워크 값은 속도가 아니라 원시 값이며 지속적으로 증가하는 카운터입니다.** 대시보드의 &quot;bytes/sec&quot; 및 &quot;operations/sec&quot; 차트는 2개의 연속 카운터 판독값을 취하고 경과 시간으로 나누어서 계산됩니다.

```
rate = (value_at_t2 - value_at_t1) / (t2 - t1_in_seconds)
```

### `GET /public/v1/pages`

Dispatcher 인스턴스당 요청된 상위 콘텐츠 페이지(`.html`)이며, 요청 개수로 순위가 매겨졌습니다. `dispatcher.httpd.requests` 지표가 지원하는 끝점입니다. 이 끝점은 일반적인 페이지 분석 도구가 아니라 AEM Dispatcher/CDN 스타일 액세스 로그에만 해당됩니다.

| 매개 변수 | 필수 | 설명 |
| ------------ | -------- | -------------------------------------- |
| `tenant_id` | 예 | 쿼리할 테넌트 |
| `from`, `to` | 아니요 | 시간 범위. 기본값은 최근 24시간입니다. |
| `limit` | 아니요 | 반환할 최대 행 수(1-500, 기본값 50) |

```bash
curl -s "{{API_BASE_URL}}/public/v1/pages?tenant_id=<tenant_id>&limit=50" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "tenant_id": "<tenant_id>",
  "from": 1735689000,
  "to": 1735690800,
  "data": [
    {
      "instance": "<instance_name>",
      "domain": "www.abc.com",
      "path": "/join-us/insights.html",
      "full_url": "https://www.abc.com/join-us/insights.html",
      "requests": 7
    }
  ],
  "rows": 1
}
```

## &#x200B;5. 이 API가 수행하지 않는 작업

- **원시 SQL 액세스 권한이 없습니다.** 모든 끝점은 조정된 특별히 빌드된 데이터 모양을 반환합니다. 기본 데이터 저장소를 직접 쿼리할 수는 없습니다.
- **상호 테넌트 쿼리가 없습니다.** 모든 요청의 범위가 정확히 하나의 `tenant_id`(으)로 지정되었습니다.
- **쓰기 액세스 권한이 없습니다.** 공개 API는 읽기 전용입니다.

## &#x200B;6. 지원

예기치 않은 오류가 발생하거나 이러한 엔드포인트에서 다루지 않는 사용 사례가 있는 경우 고객 성공/지원 엔지니어에게 추가 지원을 요청하십시오.
