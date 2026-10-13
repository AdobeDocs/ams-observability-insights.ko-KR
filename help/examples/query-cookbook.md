---
source-git-commit: 263ba9e9b3eb1c6e4dd253ca03bd0ce8954c588b
workflow-type: tm+mt
source-wordcount: '565'
ht-degree: 0%
---
## 쿼리 Cookbook {#query-cookbook}

Cookbook은 주제별로 그룹화된 바로 사용 가능한 쿼리 라이브러리입니다. 각 쿼리 앞에는 **bold**&#x200B;의 가장 적합한 차트 종류가 표시됩니다. 시간 선택기가 창을 제어하기 때문에 Time 절이 제외됩니다.

**이 문서에서:** [10.1 데이터 검색](#101-data-discovery) · [10.2 처리량](#102-throughput) · [10.3 대기 시간](#103-latency) · [10.4 오류](#104-errors) · [10.5 Apdex](#105-apdex--user-experience) · [10.6 서비스 및 호스트](#106-services-hosts--instances) · [10.7 종속성](#107-downstream-dependencies) · [10.8 AEM](#108-aem-adobe-experience-manager) · [10.9 인프라](#109-infrastructure) · [10.10 JVM](#1010-jvm--runtime)

&#x200B;> [!TIP]
> `'checkout'` 또는 `'web-publish-01'`과(와) 같은 예제 값을 테넌트의 실제 값으로 바꿉니다. 아래의 데이터 검색 쿼리에는 이러한 쿼리가 나열됩니다.

### 데이터 검색

**테이블** — 트래픽을 보고하는 서비스:

```sql
SELECT count(*) AS requests, uniqueCount(host) AS hosts FROM Transaction FACET service LIMIT 200
```

**테이블** — 호스트/계층이 있는 항목:

```sql
SELECT count(*) AS requests FROM Transaction FACET hostTierPrefix, host LIMIT 200
```

**테이블** — 환경 및 지역:

```sql
SELECT count(*) AS spans FROM Span FACET environment, region LIMIT 50
```

**테이블** — 모든 측정 지표 보고와 최신 값:

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricGauge FACET metric, unit LIMIT 200
```

**테이블** — 모든 합계/카운터 지표:

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricSum FACET metric, unit LIMIT 200
```

**Table** - 채워지는 범위 특성(AEM 예):

```sql
SELECT count(*) FILTER (WHERE sling.script != '') AS slingScripts, count(*) FILTER (WHERE jcr.operation != '') AS jcrOps, count(*) FILTER (WHERE db.system != '') AS dbCalls, count(*) FILTER (WHERE graphql.operation != '') AS graphql FROM Span FACET service LIMIT 50
```

### 처리량

**숫자** — 총 요청 수:

```sql
SELECT count(*) AS requests FROM Transaction
```

**줄** — 처리량 트렌드:

```sql
SELECT count(*) AS requests FROM Transaction TIMESERIES AUTO
```

**영역** — 서비스별 처리량, 상위 8개:

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**막대** — HTTP 메서드별 처리량:

```sql
SELECT count(*) AS requests FROM Transaction FACET http.method LIMIT 10
```

**테이블** — 가장 바쁜 URL 경로:

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET urlPath ORDER BY requests DESC LIMIT 50
```

**숫자** — 개별 추적:

```sql
SELECT uniqueCount(traceId) AS traces FROM Span
```

### 지연

**숫자** — p95 응답 시간(단위 자동 = ms):

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction
```

**줄** — 한 차트의 p50/p95/p99:

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

**줄** — 서비스당 p95, 가장 바쁜 상위 10개:

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

**막대** — p95의 작업 속도가 가장 느림:

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 20
```

**테이블** — 작업당 대기 시간 프로필:

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99, max(duration) AS maxMs, stddev(duration) AS stddevMs FROM Transaction FACET name ORDER BY p95 DESC LIMIT 50
```

**막대** — 작업당 총 체류 시간(시간 예산 위치) 공유:

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Transaction FACET name LIMIT 20
```

**테이블** — 작업당 가장 느린 단일 요청의 추적 ID:

```sql
SELECT count(*) AS calls, max(duration) AS slowestMs, latest(traceId, duration) AS slowestTraceId FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 50
```

**숫자** — 2초보다 느린 요청 공유:

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowShare FROM Transaction
```

**테이블** — 필터링된 수를 통한 지연 버킷(히스토그램 대체):

```sql
SELECT count(*) FILTER (WHERE duration <= 100) AS under100ms, count(*) FILTER (WHERE duration > 100 AND duration <= 500) AS to500ms, count(*) FILTER (WHERE duration > 500 AND duration <= 2000) AS to2s, count(*) FILTER (WHERE duration > 2000) AS over2s FROM Transaction FACET service LIMIT 20
```

### 오류수

**숫자** — 오류율 %:

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction
```

**줄** — 서비스당 오류율 %:

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

**줄** — 오류 개수 트렌드:

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error' TIMESERIES AUTO
```

**원형** — HTTP 상태 클래스 혼합:

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.statusCode IS NOT NULL FACET http.statusCode LIMIT 15
```

**테이블** — 5xx 핫스팟:

```sql
SELECT count(*) AS errors5xx FROM Transaction WHERE http.statusCode >= 500 FACET service, urlPath, http.statusCode LIMIT 50
```

**테이블** — 예외 유형별 오류:

```sql
SELECT count(*) AS errors, uniqueCount(traceId) AS traces, latest(statusMessage) AS lastMessage FROM Span WHERE status = 'Error' FACET service, error.type LIMIT 50
```

**테이블** — 오류와 합계 비교 나란히:

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction FACET service ORDER BY errors DESC LIMIT 20
```

**테이블** — 최근 실패 요청(이벤트 목록):

```sql
SELECT timestamp, service, name, urlPath, http.statusCode, duration, traceId FROM Transaction WHERE status = 'Error' LIMIT 100
```

### Apdex 및 사용자 경험

**숫자** — Apdex(T = 500ms):

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

**줄** — 서비스당 Apdex:

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

**막대** — URL 경로별 최악의 Apdex(300ms 대상 더 엄격함):

```sql
SELECT apdex(duration, 300) AS apdex, count(*) AS requests FROM Transaction WHERE urlPath LIKE '/content/%' FACET urlPath ORDER BY apdex ASC LIMIT 20
```

### 서비스, 호스트 및 인스턴스

**테이블** — 서비스 스코어카드:

```sql
SELECT count(*) AS requests, percentage(*, WHERE status = 'Error') AS errorPct, percentile(duration, 95) AS p95, apdex(duration, 500) AS apdex, uniqueCount(host) AS hosts FROM Transaction FACET serviceKey ORDER BY requests DESC LIMIT 50
```

**테이블** — 인스턴스당(AppRequest에 소비자가 포함됨):

```sql
SELECT count(*) AS requests, percentage(*, WHERE status = 'Error') AS errorPct, percentile(duration, 95) AS p95 FROM AppRequest FACET appInstance ORDER BY requests DESC LIMIT 50
```

**줄** — 호스트당 처리량:

```sql
SELECT count(*) AS requests FROM Transaction FACET host TIMESERIES AUTO LIMIT 10
```

**막대** — 부하 분산 확인(호스트당 요청 공유):

```sql
SELECT count(*) AS requests FROM Transaction FACET host LIMIT 30
```

### 다운스트림 종속성

**테이블** — 호출된 다운스트림 서비스:

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE kind = 'Client' AND peer.service != '' FACET service, peer.service LIMIT 50
```

**테이블** — 호출된 외부 도메인:

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain ORDER BY calls DESC LIMIT 50
```

**막대** — 시스템 및 작업에 의한 데이터베이스 대기 시간:

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95 FROM Span WHERE db.system != '' FACET db.system, db.operation LIMIT 30
```

**테이블** — 느린 데이터베이스 네임스페이스:

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 99) AS p99 FROM Span WHERE db.system != '' FACET db.system, db.namespace ORDER BY p99 DESC LIMIT 30
```

**줄** — 대상당 메시지 처리량:

```sql
SELECT count(*) AS messages FROM Span WHERE messaging.system != '' FACET messaging.destination TIMESERIES AUTO LIMIT 10
```

**테이블** — 주제당 생산자와 소비자 비교:

```sql
SELECT count(*) FILTER (WHERE kind = 'Producer') AS produced, count(*) FILTER (WHERE kind = 'Consumer') AS consumed, percentile(duration, 95) AS p95 FROM Span WHERE messaging.system != '' FACET messaging.system, messaging.destination LIMIT 30
```

**테이블** — RPC 메서드:

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE rpc.system != '' FACET rpc.system, rpc.method LIMIT 30
```

### AEM(Adobe Experience Manager)

**영역** — AEM 계층별 트래픽(작성자/게시/Dispatcher 접두사):

```sql
SELECT count(*) AS requests FROM Transaction WHERE hostTierPrefix != '' FACET hostTierPrefix TIMESERIES AUTO LIMIT 10
```

**줄** — 게시 계층 p95:

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO LIMIT 10
```

**줄** — 작성자 계층 p95:

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%author%' FACET host TIMESERIES AUTO LIMIT 10
```

**테이블** — 페이지 요청(`/content`)과 서블릿 호출(`/bin`) 비교:

```sql
SELECT count(*) FILTER (WHERE urlPath LIKE '/content/%') AS pages, count(*) FILTER (WHERE urlPath LIKE '/bin/%') AS servlets, count(*) FILTER (WHERE urlPath LIKE '/etc.clientlibs/%') AS clientlibs, count(*) FILTER (WHERE urlPath LIKE '/content/dam/%') AS assets FROM Transaction FACET hostTierPrefix LIMIT 20
```

**테이블** — 가장 느린 `/bin` 서블릿:

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction WHERE urlPath LIKE '/bin/%' FACET urlPath ORDER BY p95 DESC LIMIT 30
```

**테이블** — Sling 요청 URI:

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction WHERE sling.requestUri != '' FACET sling.requestUri ORDER BY requests DESC LIMIT 100
```

**테이블** — 느린 슬링 스크립트:

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE sling.script != '' FACET sling.script ORDER BY totalMs DESC LIMIT 50
```

**막대** — 시간 공유별 Sling 서블릿:

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Span WHERE sling.servlet != '' FACET sling.servlet LIMIT 20
```

**테이블** — 느린 JSP:

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95 FROM Span WHERE jsp.path != '' FACET jsp.path ORDER BY p95 DESC LIMIT 50
```

**막대** — JCR 작업 조합:

```sql
SELECT count(*) AS operations FROM Span WHERE jcr.operation != '' FACET jcr.operation LIMIT 20
```

**테이블** — 가장 인기 있는 JCR 경로:

```sql
SELECT count(*) AS operations, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE jcr.path != '' FACET jcr.operation, jcr.path ORDER BY totalMs DESC LIMIT 50
```

**테이블** — GraphQL(Headless) 작업:

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE graphql.operation != '' FACET graphql.operation ORDER BY calls DESC LIMIT 50
```

**줄** — 호스트당 Dispatcher 요청 속도:

```sql
SELECT rate(value) AS requestsPerSecond FROM DispatcherRequestRate FACET host TIMESERIES AUTO
```

**테이블** — 트랜잭션의 구성 요소(세그먼트 분류, 집합 단위 `ms`):

```sql
SELECT sum(callCount) AS calls, average(avgDurationMs) AS avgMs FROM SpanBreakdown WHERE rootOperation LIKE 'GET /content%' FACET rootOperation, segment ORDER BY calls DESC LIMIT 50
```

### 인프라

**줄** — 호스트당 CPU:

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO
```

**줄** — CPU 사용자 시간:

```sql
SELECT average(value) AS userPct FROM CpuUserPercent TIMESERIES AUTO
```

&#x200B;> [!NOTE]
> SQL은 쿼리당 하나의 엔터티를 읽으므로 사용자/시스템/iowait/도용에는 네 개의 나란히 위젯이 필요합니다([블루프린트 11.6](#116-infrastructure-capacity-board) W7-W10 참조). 공식은 그것들을 네 개의 급수가 아닌 하나의 수로 축소할 것이다.

**막대** — 호스트별 CPU p95:

```sql
SELECT percentile(value, 95) AS cpuP95 FROM CpuUtilization FACET host LIMIT 30
```

**테이블** — 호스트 상태 스냅숏:

```sql
SELECT average(value) AS avgCpu, max(value) AS peakCpu, latest(value) AS nowCpu FROM CpuUtilization FACET host ORDER BY peakCpu DESC LIMIT 50
```

**줄** — 사용된 메모리 %:

```sql
SELECT average(value) AS memory FROM MemoryUtilization FACET host TIMESERIES AUTO
```

**줄** — 메모리 증가율(누출 사냥):

```sql
SELECT derivative(value) AS memoryTrend FROM MemoryUtilization FACET host TIMESERIES AUTO
```

**테이블** — 마운트당 사용되는 디스크:

```sql
SELECT latest(value) AS usedPct FROM DiskUtilization FACET host, mountpoint ORDER BY usedPct DESC LIMIT 50
```

**숫자** — 호스트 간 사용 가능한 가장 낮은 루트 디스크(값(바이트), 집합 단위 `bytes`):

```sql
SELECT min(value) AS freeBytes FROM DiskFreeBytes
```

**줄** — 디스크 읽기 및 쓰기 처리량(각각 한 개):

```sql
SELECT rate(value) AS readBytesPerSec FROM DiskReadRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS writeBytesPerSec FROM DiskWriteRate FACET host TIMESERIES AUTO
```

**줄** — 네트워크 수신/전송:

```sql
SELECT rate(value) AS rxBytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS txBytesPerSec FROM NetworkTransmitRate FACET host TIMESERIES AUTO
```

**줄** — 네트워크 오류:

```sql
SELECT rate(value) AS rxErrors FROM NetworkReceiveErrorRate FACET host TIMESERIES AUTO
```

**줄** — 부하 평균:

```sql
SELECT average(value) AS load5m FROM LoadAverage5m FACET host TIMESERIES AUTO
```

**테이블** — CPU 및 메모리의 주요 프로세스:

```sql
SELECT average(value) AS cpuPct, max(value) AS peakCpuPct FROM ProcessCpuUtilization FACET host, process.name ORDER BY cpuPct DESC LIMIT 30
```

```sql
SELECT average(value) AS memMb FROM ProcessMemoryUsage FACET host, process.name ORDER BY memMb DESC LIMIT 30
```

**테이블** — 이름별 원시 게이지(조정된 엔터티 필요 없음):

```sql
SELECT average(value) AS avgValue, max(value) AS maxValue FROM MetricGauge WHERE metric = 'system.cpu.utilization' FACET host, state LIMIT 50
```

### JVM 및 런타임

**줄** — 호스트당 사용된 더미(MB):

```sql
SELECT average(value) AS heapMb FROM JvmHeapUsed FACET host TIMESERIES AUTO
```

**테이블** — 사용된 힙과 제한 비교(두 개의 위젯 또는 수식 — [블루프린트 11.7](#117-jvm-health-board) 참조):

```sql
SELECT latest(value) AS heapLimitMb FROM JvmHeapLimit FACET host LIMIT 50
```

**줄** — JVM 스레드:

```sql
SELECT latest(value) AS threads FROM JvmThreadCount FACET host TIMESERIES AUTO
```

**줄** — JVM 프로세스 CPU:

```sql
SELECT average(value) AS jvmCpu FROM JvmCpuRecentUtilization FACET host TIMESERIES AUTO
```

**테이블** — JVM 메모리 풀(일반 합계 지표):

```sql
SELECT latest(value) AS bytesUsed FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool LIMIT 100
```
