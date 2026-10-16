---
title: 대시보드 블루프린트
description: 애플리케이션 및 인프라 모니터링용 블루프린트를 사용하여 가시성 인사이트 대시보드를 작성합니다.
source-git-commit: 873e7b9f279c3276fc0197969d667d40d022968a
workflow-type: tm+mt
source-wordcount: '1324'
ht-degree: 1%
---

## 대시보드 블루프린트 {#dashboard-blueprints}

각 블루프린트는 완벽하고 복잡한 대시보드입니다. 여기에는 목표, 탭 및 섹션 레이아웃, 모든 위젯(차트 유형, 단위, 임계값, 크기 및 쿼리 포함)이 나열됩니다. 크기가 12열 눈금의 `width×height`입니다. 위에서 아래로 각 하나를 빌드한 다음 드래그하여 조정합니다.

| 블루프린트 | 대상자 | 강조 표시 |
| ------------------------------------------------------------------------------------ | --------------------- | ---------------------------------------------------------------------------------- |
| [11.1 골든 시그널 / SLO](#111-golden-signals--slo-board) | SRE, 호출 시 | 임계값, Apdex, 오류 %, 포화가 있는 KPI 타일 |
| [11.2 오류 등급](#112-error-triage-board) | 호출 시 | 오류 클래스, 5xx 핫스팟, 추적 ID로 요청 실패 |
| [11.3 지연 심층 분석](#113-latency-deep-dive-board) | 성능 엔지니어 | 백분위수, 지연 버킷, 가변성, 트랜잭션 구성, 가장 느린 추적 |
| [11.4 다운스트림 종속성](#114-downstream-dependencies-board) | 서비스 소유자 | 아웃바운드 HTTP, DB, 메시징, RPC |
| [11.5 AEM 작성자/게시/Dispatcher](#115-aem-author--publish--dispatcher-board) | AEM 팀 | 계층별 섹션, Sling/JSP/JCR, Dispatcher 로드 |
| [11.6 인프라 용량](#116-infrastructure-capacity-board) | 플랫폼 / 용량 | CPU 모드, 메모리, 디스크, 네트워크 속도, 공식 |
| [11.7 JVM 상태](#117-jvm-health-board) | Java 팀 | 힙 % 공식, 호스트당 힙 및 풀, 스레드, JVM CPU |
| [11.8 경영진/NOC 담벼락](#118-executive--noc-wall-board) | NOC, 리더십 | KPI 타일, 가용성 공식, 계층별 트래픽, SLO 참고 사항 |
| [11.9 비율 및 여유 공간](#119-ratios--headroom-board-formula-showcase) | 모든 사용자 | 공식 위젯 쇼케이스 |
| [11.10 서비스 탐색기](#1110-service-explorer-variable-driven) | 모든 팀 | 변수 중심, 재사용 가능 |

**차트 범례:** **N** = 숫자, **L** = 줄, **A** = 영역, **B** = 막대, **P** = 원형, **T** = 테이블.



### 골든 시그널 / SLO 보드

**목표:** &quot;현재 플랫폼이 정상입니까?&quot;에 대한 하나의 화면 — 트래픽, 오류, 지연, 채도
**시간 선택기:** 지난 1시간(문제) 또는 24시간(일일 검토).

```text
┌──────────── Section: SLO KPIs ────────────────────────────────┐
│ N Requests │ N Error % │ N p95 │ N Apdex │ N Slow % │ N Hosts  │  (6 × 2w×2h)
├──────────── Section: Traffic & Errors ────────────────────────┤
│ A Throughput by service (6w) │ L Error % by service (6w)       │
├──────────── Section: Latency ─────────────────────────────────┤
│ L p50/p95/p99 (6w)           │ L Apdex by service (6w)         │
├──────────── Section: Saturation ──────────────────────────────┤
│ L CPU by host (4w) │ L Memory by host (4w) │ L Load 5m (4w)    │
└───────────────────────────────────────────────────────────────┘
```

**W1 · 요청** — N, 2×2

```sql
SELECT count(*) AS requests FROM Transaction
```

**W2 · 오류 %** — N, 단위 `%`, 2×2. 임계값: `0 → Green`, `1 → Amber`, `5 → Red`.

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction
```

**W3 · p95 지연** — N, 단위 `ms`, 2×2. 임계값: `0 → Green`, `800 → Amber`, `2000 → Red`.

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction
```

**W4 · Apdex** — N, 2×2. 임계값: `0 → Red`, `0.7 → Amber`, `0.85 → Green`(일치하는 가장 높은 밴드가 성공하므로 낮은 점수는 빨간색으로 유지됨).

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

**W5 · 요청 > 2초** — N, 단위 `%`, 2×2. 임계값: `0 → Green`, `2 → Amber`, `10 → Red`.

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowPct FROM Transaction
```

**W6 · 활성 호스트** — N, 2×2

```sql
SELECT uniqueCount(host) AS hosts FROM Transaction
```

**W7 · 서비스별 처리량** — A, 6×4

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**W8 · 서비스별 오류 비율** — L, 단위 `%`, 6×4

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**W9 · 대기 시간 백분위수** — L, 단위 `ms`, 6×4

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

**W10 · 서비스별 Apdex** — L, 6×4

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**W11 · 호스트별 CPU** — L, 4×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO LIMIT 10
```

**W12 · 호스트별 메모리** — L, 4×4

```sql
SELECT average(value) AS memory FROM MemoryUtilization FACET host TIMESERIES AUTO LIMIT 10
```

**W13 · 5m 로드** — L, 4×4

```sql
SELECT average(value) AS load5m FROM LoadAverage5m FACET host TIMESERIES AUTO LIMIT 10
```



### 오류 등급 보드

**목표:** &quot;오류율이 사용 중&quot;에서 &quot;이 끝점, 이 예외, 이 추적&quot;까지 세 번의 클릭으로 계속됩니다.

```text
Tab "Overview"
│ N Errors │ N Error % │ N Affected traces │ N 5xx count │                 (4 × 3w×2h)
│ L Errors by service (8w) │ P Status code mix (4w)                       │
│ T Errors vs total by service (12w)                                       │
Tab "Drill-down"
│ T 5xx hot spots: service × path × code (12w)                             │
│ T Exception types (6w) │ T Downstream failures (6w)                      │
│ T Latest failing requests – event list (12w×6h)                          │
```

**개요 탭**

**W1 · 오류** — N, 임계값 `0 → Green`, `1 → Amber`, `100 → Red`

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error'
```

**W2 · 오류 %** — N, 단위 `%`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction
```

**W3 · 영향을 받는 추적** — N

```sql
SELECT uniqueCount(traceId) AS traces FROM Span WHERE status = 'Error'
```

**W4 · HTTP 5xx** — N, 임계값 `0 → Green`, `1 → Red`

```sql
SELECT count(*) AS http5xx FROM Transaction WHERE http.statusCode >= 500
```

**W5 · 서비스별 오류** — L, 8×4

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error' FACET service TIMESERIES AUTO LIMIT 10
```

**W6 · 상태 코드 혼합** — P, 4×4

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.statusCode >= 400 FACET http.statusCode LIMIT 10
```

**W7 · 오류와 서비스별 합계** — T, 12×4

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, percentage(*, WHERE status = 'Error') AS errorPct, count(*) FILTER (WHERE http.statusCode >= 500) AS http5xx, count(*) FILTER (WHERE http.statusCode >= 400 AND http.statusCode < 500) AS http4xx FROM Transaction FACET service ORDER BY errors DESC LIMIT 30
```

**드릴다운 탭**

**W8 · 5xx 핫스팟** — T, 12×5

```sql
SELECT count(*) AS errors, percentile(duration, 95) AS p95, latest(traceId) AS lastTraceId FROM Transaction WHERE http.statusCode >= 500 FACET service, urlPath, http.statusCode ORDER BY errors DESC LIMIT 100
```

**W9 · 예외 유형** — T, 6×5

```sql
SELECT count(*) AS errors, latest(statusMessage) AS lastMessage FROM Span WHERE status = 'Error' AND error.type != '' FACET service, error.type ORDER BY errors DESC LIMIT 50
```

**W10 · 다운스트림 실패** — T, 6×5

```sql
SELECT count(*) AS failures, percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client' AND status = 'Error' FACET service, peer.service, externalDomain ORDER BY failures DESC LIMIT 50
```

**W11 · 최근 실패 요청** — T, 12×6

```sql
SELECT timestamp, service, host, name, urlPath, http.statusCode, duration, statusMessage, traceId FROM Transaction WHERE status = 'Error' LIMIT 200
```



### 지연 심층 분석 보드

**목표:** 찾기 _위치_ 시간 경과 — 작업, 세그먼트, 일관성

```text
│ L p50/p95/p99 (8w)               │ N p99 (4w)                │
│ B Time share by operation (6w)   │ B Slowest ops by p95 (6w) │
│ T Latency profile per operation (12w)                         │
│ T Latency buckets per service (6w) │ B Variability (6w)       │
│ T Transaction composition – SpanBreakdown (12w)               │
│ T Slowest trace per operation (12w)                           │
```

**W1 · 백분위수** — L, `ms`, 8×4

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

**W2 · p99** — N, `ms`, 임계값 `0 → Green`, `1500 → Amber`, `4000 → Red`, 4×4

```sql
SELECT percentile(duration, 99) AS p99 FROM Transaction
```

**W3 · 작업별 시간 공유** — B, `%`, 6×4

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Transaction FACET name LIMIT 15
```

**W4 · 가장 느린 작업(p95)** — B, `ms`, 6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 15
```

**W5 · 지연 프로필** — T, 12×5

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99, max(duration) AS maxMs FROM Transaction FACET service, name ORDER BY p95 DESC LIMIT 100
```

**W6 · 지연 버킷** — T, 6×4

```sql
SELECT count(*) FILTER (WHERE duration <= 100) AS fast, count(*) FILTER (WHERE duration > 100 AND duration <= 500) AS ok, count(*) FILTER (WHERE duration > 500 AND duration <= 2000) AS slow, count(*) FILTER (WHERE duration > 2000) AS verySlow FROM Transaction FACET service LIMIT 20
```

**W7 · 지연 가변성** — B, `ms`, 6×4

```sql
SELECT stddev(duration) AS stddevMs FROM Transaction FACET service ORDER BY stddevMs DESC LIMIT 15
```

**W8 · 트랜잭션 구성** — T, 단위 `ms`, 12×5

```sql
SELECT sum(callCount) AS calls, average(avgDurationMs) AS avgMs FROM SpanBreakdown FACET service, rootOperation, segment ORDER BY calls DESC LIMIT 100
```

**W9 · 작업당 가장 느린 추적** — T, 12×5

```sql
SELECT count(*) AS calls, max(duration) AS slowestMs, latest(traceId, duration) AS slowestTraceId, latest(host, duration) AS slowestHost FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 50
```



### 다운스트림 종속성 보드

**목표:**&#x200B;이(가) 당사입니까, 아니면 해당 사용자입니까? 호출하는 모든 데이터베이스, API, 큐 및 RPC의 상태입니다.

```text
│ N Outbound calls │ N Outbound error % │ N Outbound p95 │            (3 × 4w×2h)
│ T Downstream services (12w)                                       │
│ L External domain p95 (6w) │ T External domains (6w)              │
│ B DB p95 by system/op (6w) │ T Slow DB namespaces (6w)            │
│ L Messages per destination (6w) │ T Producer vs consumer (6w)     │
│ T RPC methods (12w)                                               │
```

**W1 · 아웃바운드 호출** — N

```sql
SELECT count(*) AS calls FROM Span WHERE kind = 'Client'
```

**W2 · 아웃바운드 오류 %** — N, `%`, 임계값 `0 → Green`, `2 → Amber`, `10 → Red`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE kind = 'Client'
```

**W3 · 아웃바운드 p95** — N, `ms`

```sql
SELECT percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client'
```

**W4 · 다운스트림 서비스** — T, 12×5

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE kind = 'Client' FACET service, peer.service ORDER BY calls DESC LIMIT 100
```

**W5 · 외부 도메인 p95** — L, `ms`, 6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain TIMESERIES AUTO LIMIT 8
```

**W6 · 외부 도메인** — T, 6×4

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain ORDER BY calls DESC LIMIT 50
```

**W7 · DB p95** — B, `ms`, 6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Span WHERE db.system != '' FACET db.system, db.operation ORDER BY p95 DESC LIMIT 20
```

**W8 · 느린 DB 네임스페이스** — T, 6×4

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 99) AS p99, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE db.system != '' FACET service, db.system, db.namespace ORDER BY p99 DESC LIMIT 50
```

**W9 · 대상당 메시지** — L, 6×4

```sql
SELECT count(*) AS messages FROM Span WHERE messaging.system != '' FACET messaging.destination TIMESERIES AUTO LIMIT 10
```

**W10 · 생산자와 소비자 비교** — T, 6×4

```sql
SELECT count(*) FILTER (WHERE kind = 'Producer') AS produced, count(*) FILTER (WHERE kind = 'Consumer') AS consumed, percentile(duration, 95) AS p95 FROM Span WHERE messaging.system != '' FACET messaging.system, messaging.destination LIMIT 50
```

**W11 · RPC 메서드** — T, 12×4

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE rpc.system != '' FACET service, rpc.system, rpc.method ORDER BY calls DESC LIMIT 50
```



### AEM Author / Publish / Dispatcher 보드

**목표:** 계층별로 그룹화된 AEM 작업 보드를 사용하여 클래식 New Relic AEM 레이아웃을 미러링합니다.
이름이 `PUBLISH`, `AUTHOR`, `DISPATCHER`인 **섹션 헤더** 및 `- PUBLISH`/`- AUTHOR`인 종료 위젯 제목을 사용하여 색상 큐를 가져옵니다.

```text
Tab "Tiers"
├─ Section: OVERVIEW
│ A Traffic by tier (8w) │ T Content mix by tier (4w)
├─ Section: PUBLISH
│ N Requests - PUBLISH │ N Error % - PUBLISH │ N p95 - PUBLISH │ N Apdex - PUBLISH
│ L p95 by host - PUBLISH (6w) │ L CPU - PUBLISH (6w)
│ L Heap - PUBLISH (6w) │ T Slowest /bin servlets - PUBLISH (6w)
├─ Section: AUTHOR
│ N Requests - AUTHOR │ N Error % - AUTHOR │ N p95 - AUTHOR │ N Active authors' hosts - AUTHOR
│ L p95 by host - AUTHOR (6w) │ L Heap - AUTHOR (6w)
├─ Section: DISPATCHER
│ L Dispatcher req/s (8w) │ L CPU - DISPATCHER (4w)
Tab "Code hot spots"
│ T Slow Sling scripts (6w) │ T Slow JSPs (6w)
│ T Hottest JCR paths (6w)  │ B JCR operation mix (6w)
│ T Sling request URIs (6w) │ T GraphQL operations (6w)
│ B Sling servlet time share (12w)
```

**개요**

**W1 · 계층별 트래픽** — A, 8×4

```sql
SELECT count(*) AS requests FROM Transaction WHERE hostTierPrefix != '' FACET hostTierPrefix TIMESERIES AUTO LIMIT 10
```

**W2 · 계층별 컨텐츠 혼합** — T, 4×4

```sql
SELECT count(*) FILTER (WHERE urlPath LIKE '/content/%') AS pages, count(*) FILTER (WHERE urlPath LIKE '/bin/%') AS servlets, count(*) FILTER (WHERE urlPath LIKE '/content/dam/%') AS assets FROM Transaction FACET hostTierPrefix LIMIT 20
```

**섹션 게시**

**W3 · 요청 - 게시** — N

```sql
SELECT count(*) AS requests FROM Transaction WHERE host LIKE '%publish%'
```

**W4 · 오류 % - 게시** — N, `%`, 임계값 `0 → Green`, `1 → Amber`, `5 → Red`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction WHERE host LIKE '%publish%'
```

**W5 · p95 - 게시** — N, `ms`, 임계값 `0 → Green`, `1000 → Amber`, `3000 → Red`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%publish%'
```

**W6 · Apdex - PUBLISH** — N, 임계값 `0 → Red`, `0.7 → Amber`, `0.85 → Green`

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction WHERE host LIKE '%publish%'
```

호스트별 **W7 · p95 - PUBLISH** — L, `ms`, 6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO LIMIT 10
```

**W8 · CPU - 게시** — L, `%`, 6×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO
```

**W9 · 힙 - PUBLISH** — L, `MB`, 6×4

```sql
SELECT average(value) AS heapMb FROM JvmHeapUsed WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO
```

**W10 · 가장 느린 /bin 서블릿 - PUBLISH** — T, 6×4

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction WHERE host LIKE '%publish%' AND urlPath LIKE '/bin/%' FACET urlPath ORDER BY p95 DESC LIMIT 30
```

**작성자 섹션** — W3-W9를 복제하고 `'%publish%'`을(를) `'%author%'`(으)로 변경하고 제목 접미사를 `- AUTHOR`(으)로 변경합니다. 추가 위젯:

**W11 · 작성자가 활성 상태 - 작성자** — N

```sql
SELECT uniqueCount(host) AS hosts FROM Transaction WHERE host LIKE '%author%'
```

**Dispatcher 섹션**

**W12 · Dispatcher req/s** — L, `req/s`, 8×4

```sql
SELECT rate(value) AS requestsPerSecond FROM DispatcherRequestRate FACET host TIMESERIES AUTO
```

**W13 · CPU - DISPATCHER** — L, `%`, 4×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization WHERE host LIKE '%dispatcher%' FACET host TIMESERIES AUTO
```

**코드 핫스팟 탭**

**W14 · 느린 슬링 스크립트** — T

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE sling.script != '' FACET sling.script ORDER BY totalMs DESC LIMIT 50
```

**W15 · 느린 JSP** — T

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95 FROM Span WHERE jsp.path != '' FACET jsp.path ORDER BY p95 DESC LIMIT 50
```

**W16 · 최신 JCR 경로** — T

```sql
SELECT count(*) AS operations, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE jcr.path != '' FACET jcr.operation, jcr.path ORDER BY totalMs DESC LIMIT 50
```

**W17 · JCR 작업 혼합** — B

```sql
SELECT count(*) AS operations FROM Span WHERE jcr.operation != '' FACET jcr.operation LIMIT 20
```

**W18 · Sling 요청 URI** — T

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction WHERE sling.requestUri != '' FACET sling.requestUri ORDER BY requests DESC LIMIT 100
```

**W19 · GraphQL 작업** — T

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE graphql.operation != '' FACET graphql.operation ORDER BY calls DESC LIMIT 50
```

**W20 · Sling 서블릿 시간 공유** — B, `%`, 12×4

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Span WHERE sling.servlet != '' FACET sling.servlet LIMIT 20
```



### 인프라 수용작업량 보드

**목표:** 호스트 간 용량 계획 및 채도 사냥을 설정합니다.
**시간 선택기:** 트렌드(롤업은 속도를 높임)는 7일, 인시던트는 1시간.

```text
Tab "Compute"
│ N Hosts reporting │ N Peak CPU % │ N Peak memory % │ N Max load 5m │
│ L CPU by host (6w) │ L Memory by host (6w)                         │
│ L CPU user / system / iowait / steal (4 × 3w)                     │
│ T Host snapshot (12w)                                              │
│ T Top processes by CPU (6w) │ T Top processes by memory (6w)       │
Tab "Storage & Network"
│ T Disk used per mount (6w) │ N Lowest free root disk (6w)          │
│ L Disk read (6w) │ L Disk write (6w)                               │
│ L Network rx (6w) │ L Network tx (6w)                              │
│ L Total network I/O for one host – formula (12w)                   │
```

**W1 · 호스트 보고** — N

```sql
SELECT uniqueCount(host) AS hosts FROM CpuUtilization
```

**W2 · 최대 CPU** — N, `%`, 임계값 `0 → Green`, `75 → Amber`, `90 → Red`

```sql
SELECT max(value) AS peakCpu FROM CpuUtilization
```

**W3 · 최대 메모리** — N, `%`, 임계값 `0 → Green`, `80 → Amber`, `92 → Red`

```sql
SELECT max(value) AS peakMemory FROM MemoryUtilization
```

**W4 · 최대 로드 5m** — N

```sql
SELECT max(value) AS maxLoad FROM LoadAverage5m
```

호스트별 **W5 · CPU** — L, `%`, 6×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO
```

**W6 · 호스트별 메모리** — L, `%`, 6×4

```sql
SELECT average(value) AS memory FROM MemoryUtilization FACET host TIMESERIES AUTO
```

모드별 **W7-W10 · CPU** — L, `%`, 3×4

```sql
SELECT average(value) AS userPct FROM CpuUserPercent FACET host TIMESERIES AUTO LIMIT 10
```

```sql
SELECT average(value) AS systemPct FROM CpuSystemPercent FACET host TIMESERIES AUTO LIMIT 10
```

```sql
SELECT average(value) AS iowaitPct FROM CpuIowaitPercent FACET host TIMESERIES AUTO LIMIT 10
```

```sql
SELECT average(value) AS stealPct FROM CpuStealPercent FACET host TIMESERIES AUTO LIMIT 10
```

**W11 · 호스트 스냅숏** — T, 12×5

```sql
SELECT average(value) AS avgCpu, percentile(value, 95) AS p95Cpu, max(value) AS peakCpu, latest(value) AS nowCpu FROM CpuUtilization FACET host ORDER BY p95Cpu DESC LIMIT 100
```

**W12 · CPU의 주요 프로세스** — T, 6×5

```sql
SELECT average(value) AS cpuPct, max(value) AS peakPct FROM ProcessCpuUtilization FACET host, process.name ORDER BY cpuPct DESC LIMIT 30
```

**W13 · 메모리별 상위 프로세스** — T, `MB`, 6×5

```sql
SELECT average(value) AS memMb, max(value) AS peakMb FROM ProcessMemoryUsage FACET host, process.name ORDER BY memMb DESC LIMIT 30
```

**저장소 및 네트워크 탭**

**W14 · 마운트당 사용된 디스크** — T, `%`

```sql
SELECT latest(value) AS usedPct, max(value) AS peakPct FROM DiskUtilization FACET host, mountpoint ORDER BY usedPct DESC LIMIT 50
```

**W15 · 사용 가능한 가장 낮은 루트 디스크** — N, 단위 `bytes`, 임계값 `0 → Red`, `5000000000 → Amber`, `20000000000 → Green`

```sql
SELECT min(value) AS freeBytes FROM DiskFreeBytes
```

**W16/W17 · 디스크 처리량** — L, `bytes/s`

```sql
SELECT rate(value) AS readBytesPerSec FROM DiskReadRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS writeBytesPerSec FROM DiskWriteRate FACET host TIMESERIES AUTO
```

**W18/W19 · 네트워크 처리량** — L, `bytes/s`

```sql
SELECT rate(value) AS rxBytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS txBytesPerSec FROM NetworkTransmitRate FACET host TIMESERIES AUTO
```

**W20 · 한 호스트(수식)에 대한 총 네트워크 I/O** — 수식 위젯, L, `bytes/s`, 12×4. 수식은 `FACET`할 수 없으며 `rate()`의 범위를 하나의 호스트로 지정해야 합니다([조정된 지표 엔터티](../references/data-dictionary.md#curated-metric-entities) 참조). 따라서 호스트를 고정합니다. `web-publish-01`을(를) W5에서 실제 호스트 이름으로 바꿉니다.

`netRx` 쿼리:

```sql
SELECT rate(value) AS v FROM NetworkReceiveRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

`netTx` 쿼리:

```sql
SELECT rate(value) AS v FROM NetworkTransmitRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

공식:

```text
netRx + netTx
```



### JVM 상태 게시판

**목표:** 힙 압력, 스레드 누수 및 JVM CPU 급등이 중지되기 전에 이를 확인합니다.

```text
│ N Heap used % (formula) │ N Peak threads │ N JVM CPU % │ N Max heap MB │
│ L Heap used % over time – formula (12w)                                │
│ L Heap used by host (6w) │ T Heap limit by host (6w)                    │
│ L Threads by host (6w)   │ L JVM CPU by host (6w)                       │
│ T Memory pools (12w)                                                   │
```

**W1 · 사용된 더미 %(수식)** — 수식, N, `%`, 임계값 `0 → Green`, `75 → Amber`, `90 → Red`

`used` 쿼리:

```sql
SELECT average(value) AS v FROM JvmHeapUsed
```

`limit` 쿼리:

```sql
SELECT average(value) AS v FROM JvmHeapLimit
```

공식:

```text
(used / limit) * 100
```

**W2 · 최대 스레드** — N

```sql
SELECT max(value) AS peakThreads FROM JvmThreadCount
```

**W3 · JVM CPU** — N, `%`

```sql
SELECT average(value) AS jvmCpu FROM JvmCpuRecentUtilization
```

**W4 · 사용된 최대 더미** — N, `MB`

```sql
SELECT max(value) AS maxHeapMb FROM JvmHeapUsed
```

**W5 · 시간에 따른 힙 사용률(%)** — 공식, L, `%`, 12×4

`used` 쿼리:

```sql
SELECT average(value) AS v FROM JvmHeapUsed TIMESERIES AUTO
```

`limit` 쿼리:

```sql
SELECT average(value) AS v FROM JvmHeapLimit TIMESERIES AUTO
```

공식:

```text
(used / limit) * 100
```

>[!NOTE]
>
> 수식을 하나의 호스트로 범위를 지정하려면 **둘 다**&#x200B;개의 쿼리에 `WHERE host = 'my-publish-01'`을(를) 추가하십시오.

**W6 · 호스트에서 사용하는 힙** — L, `MB`, 6×4

```sql
SELECT average(value) AS heapMb FROM JvmHeapUsed FACET host TIMESERIES AUTO
```

**W7 · 호스트별 힙 제한** — T, `MB`, 6×4

```sql
SELECT latest(value) AS heapLimitMb FROM JvmHeapLimit FACET host LIMIT 50
```

**W8 · 호스트별 Threads** — L, 6×4

```sql
SELECT latest(value) AS threads FROM JvmThreadCount FACET host TIMESERIES AUTO
```

**W9 · 호스트별 JVM CPU** — L, `%`, 6×4

```sql
SELECT average(value) AS jvmCpu FROM JvmCpuRecentUtilization FACET host TIMESERIES AUTO
```

**W10 · 메모리 풀** — T, 12×5

```sql
SELECT latest(value) AS bytesUsed, max(value) AS peakBytes FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool ORDER BY peakBytes DESC LIMIT 100
```



### 11.8 경영진 / NOC 담판

**목표:** 투명하고 색상으로 구분된 벽 화면입니다. 대부분 임계값이 있는 숫자 위젯으로, 행당 하나의 트렌드입니다.

```text
│ N Requests │ N Availability % (formula) │ N Apdex │ N p95 │ N 5xx │ N Hosts │
│ L Throughput (12w)                                                          │
│ B Requests by service (6w) │ P Traffic by tier (6w)                         │
│ Text: "SLO: 99.9 % availability, p95 < 800 ms. Owner: platform-sre" (12w×1h) │
```

**W1 · 요청** — N

```sql
SELECT count(*) AS requests FROM Transaction
```

**W2 · 가용성 %(수식)** — 수식, N, `%`, 임계값 `0 → Red`, `99 → Amber`, `99.9 → Green`

`ok` 쿼리:

```sql
SELECT count(*) AS v FROM Transaction WHERE status != 'Error'
```

`total` 쿼리:

```sql
SELECT count(*) AS v FROM Transaction
```

공식:

```text
(ok / total) * 100
```

**W3 · Apdex** — N, 임계값 `0 → Red`, `0.7 → Amber`, `0.85 → Green`

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

**W4 · p95** — N, `ms`, 임계값 `0 → Green`, `800 → Amber`, `2000 → Red`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction
```

**W5 · 5xx** — N, 임계값 `0 → Green`, `1 → Amber`, `50 → Red`

```sql
SELECT count(*) AS http5xx FROM Transaction WHERE http.statusCode >= 500
```

**W6 · 호스트** — N

```sql
SELECT uniqueCount(host) AS hosts FROM Transaction
```

**W7 · 처리량** — L, 12×4

```sql
SELECT count(*) AS requests FROM Transaction TIMESERIES AUTO
```

**W8 · 서비스별 요청** — B, 6×4

```sql
SELECT count(*) AS requests FROM Transaction FACET service LIMIT 10
```

**W9 · 계층별 트래픽** — P, 6×4

```sql
SELECT count(*) AS requests FROM Transaction WHERE hostTierPrefix != '' FACET hostTierPrefix LIMIT 10
```

**W10 · SLO 메모** — 텍스트 위젯, 12×1.



### 비율 및 헤드룸 보드(공식 쇼케이스)

**목표:**&#x200B;에서 파생된 KPI에 두 개 이상의 쿼리가 필요합니다.

**W1 · 호스트당 요청** — 공식, L, `req/host`

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

**W2 · 5xx가 모든 오류를 공유** — 수식, N, `%`

`e5xx` 쿼리:

```sql
SELECT count(*) AS v FROM Transaction WHERE http.statusCode >= 500
```

`errors` 쿼리:

```sql
SELECT count(*) AS v FROM Transaction WHERE http.statusCode >= 400
```

공식:

```text
(e5xx / errors) * 100
```

**W3 · 데이터베이스의 시간과 총 서버 시간** — 공식, L, `%`

`db` 쿼리:

```sql
SELECT sum(duration) AS v FROM Span WHERE db.system != '' TIMESERIES AUTO
```

`server` 쿼리:

```sql
SELECT sum(duration) AS v FROM Transaction TIMESERIES AUTO
```

공식:

```text
(db / server) * 100
```

**W4 · 힙 헤드룸(MB)** — 공식, L, `MB`

`limit` 쿼리:

```sql
SELECT average(value) AS v FROM JvmHeapLimit TIMESERIES AUTO
```

`used` 쿼리:

```sql
SELECT average(value) AS v FROM JvmHeapUsed TIMESERIES AUTO
```

공식:

```text
limit - used
```

**W5 · 한 호스트에 대한 네트워크 출력/입력 비율** — 수식, L(호스트를 고정합니다. [조정된 지표 엔터티](../references/data-dictionary.md#curated-metric-entities) 참조)

`rx` 쿼리:

```sql
SELECT rate(value) AS v FROM NetworkReceiveRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

`tx` 쿼리:

```sql
SELECT rate(value) AS v FROM NetworkTransmitRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

공식:

```text
tx / rx
```

**W6 · 게시와 작성자 트래픽 비율** — 공식, N

`publish` 쿼리:

```sql
SELECT count(*) AS v FROM Transaction WHERE host LIKE '%publish%'
```

`author` 쿼리:

```sql
SELECT count(*) AS v FROM Transaction WHERE host LIKE '%author%'
```

공식:

```text
publish / author
```

**W7 · 가중 오류 예산 굽기(SLO 99.9 %)** — 공식, L

`errors` 쿼리:

```sql
SELECT count(*) AS v FROM Transaction WHERE status = 'Error' TIMESERIES AUTO
```

`total` 쿼리:

```sql
SELECT count(*) AS v FROM Transaction TIMESERIES AUTO
```

공식(소진율: 1.0 = 일정에 따라 정확히 예산 소비):

```text
(errors / total) / 0.001
```



### 서비스 탐색기(변수 기반)

**목표:** 팀이 _서비스 및 환경을 지정할 수 있는 하나의 보드._

>[!WARNING]
>
> [자습서 3, 단계 8.7](#87-make-dashboards-reusable-with-variables)에서 **현재 빌드 제한**&#x200B;을(를) 봅니다. `{{variables}}` 위젯 SQL 내의 은(는) 아직 미리 보거나 저장할 수 없습니다. 이 문제가 해결될 때까지 각 `{{…}}`을(를) 리터럴(예: `'checkout'`)로 바꾸고 _복제_&#x200B;를 사용하여 서비스별로 보드를 복제합니다. 아래 디자인은 대체가 종단간 지원될 때 사용할 준비가 되었습니다.

**변수**

| 이름 | 레이블 | 유형 | 다중 | 숫자 | 정의 |
| -------- | ------------------- | ------------ | ----- | ------- | ---------------------------------------------------------- |
| `svc` | 서비스 | 쿼리에서 | ✔ |         | `SELECT count(*) FROM Transaction FACET service LIMIT 200` |
| `env` | 환경 | 쿼리에서 | ✔ |         | `SELECT count(*) FROM Span FACET environment LIMIT 50` |
| `slowMs` | 느린 임계값(ms) | 텍스트 상자 |       | ✔ | 기본 `2000` |

```text
Section: "{{svc}} in {{env}}"
│ N Requests │ N Error % │ N p95 │ N Slow requests │
│ L Throughput by service (6w) │ L p95 by service (6w)   │
│ T Operations (12w)                                     │
│ T Slow requests over {{slowMs}} ms – events (12w)       │
│ T Downstream calls (12w)                               │
```

**W1 · 요청 —{{svc}}** — N

```sql
SELECT count(*) AS requests FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}})
```

**W2 · 오류 %** — N, `%`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}})
```

**W3 · p95** — N, `ms`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}})
```

**W4 · 느린 요청** — N

```sql
SELECT count(*) AS slow FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) AND duration > {{slowMs}}
```

**W5 · 서비스별 처리량** — L

```sql
SELECT count(*) AS requests FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) FACET service TIMESERIES AUTO
```

서비스별 **W6 · p95** — L, `ms`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) FACET service TIMESERIES AUTO
```

**W7 · 작업** — T

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) FACET service, name ORDER BY calls DESC LIMIT 100
```

**W8 · 느린 요청(이벤트)** — T

```sql
SELECT timestamp, service, host, name, urlPath, duration, traceId FROM Transaction WHERE service IN ({{svc}}) AND duration > {{slowMs}} ORDER BY duration DESC LIMIT 200
```

**W9 · 다운스트림 호출** — T

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE kind = 'Client' AND service IN ({{svc}}) FACET peer.service, externalDomain, db.system ORDER BY calls DESC LIMIT 50
```
