---
source-git-commit: 4949fcad799260aa73354a8c0b9d79e6394cb257
workflow-type: tm+mt
source-wordcount: '585'
ht-degree: 20%
---
## 데이터 사전 {#data-dictionary}

쿼리할 수 있는 모든 엔티티 및 속성입니다. 설명이 있는 라이브 카탈로그는 [탐색] 및 [위젯 추가] 대화 상자의 **쿼리 안내서** 패널에도 있습니다.

### 엔티티 추적

| 기관 | 다음 포함 | 고정 범위 | 롤업 |
| ------------- | --------------------------------------------------------- | ---------------------------- | ------------------------------------------ |
| `Transaction` | 인바운드 서버 요청 | `kind = Server` | 1m/5m/1h([성능](#performance)) |
| `AppRequest` | 인바운드 서버 **및** 메시지 소비자 요청 | `kind IN (Server, Consumer)` | — |
| `Span` | 모든 범위(서버, 클라이언트, 생산자, 소비자, 내부) | — | — |

<details open>
<summary><strong>추적 특성</strong>(세 엔터티 모두에서 동일함)</summary>

| 속성 | 유형 | 설명 |
| --------------------------------------------------- | ----------- | ----------------------------------------------------------------------------------------------- |
| `timestamp` | timestamp | 시작 시간 범위 지정 |
| `traceId`, `spanId`, `parentSpanId` | 문자열 | 식별자(루트 범위 `parentSpanId = ''`) |
| `name` | 문자열 | 작업 또는 트랜잭션 이름 |
| `kind` | 문자열 | `Server`, `Client`, `Producer`, `Consumer`, `Internal` |
| `service` | 문자열 | 서비스 이름 내보내기 |
| `serviceKey` | 문자열 | 유효한 서비스 ID(`service`이(가) 비어 있거나 조부모 항목일 경우 `appName`(으)로 대체됨) |
| `appName` | 문자열 | Observability Insights 애플리케이션 이름 |
| `appInstance` | 문자열 | 인스턴스당 ID: 작성자/게시 호스트 접두사, 앱 이름, 서비스 |
| `hostTierPrefix` | 문자열 | 호스트 이름이 `author` / `publish` / `dispatcher` 역할 접미사에서 잘렸습니다. |
| `duration` | 숫자(ms) | 범위 기간(밀리초) |
| `status` | 문자열 | `Ok`, `Error`, `Unset` |
| `statusMessage` | 문자열 | 상태 설명 |
| `host` | 문자열 | 호스트 이름 |
| `environment` | 문자열 | 배포 환경 |
| `region` | 문자열 | 클라우드 영역 |
| `cloud.provider` | 문자열 | AWS / AZURE / GCP |
| `http.method` | 문자열 | `GET`, `POST`, ... |
| `http.statusCode` | 숫자 | HTTP 응답 상태 |
| `http.route` | 문자열 | 정규화된 경로 템플릿 |
| `url` | 문자열 | 전체 URL |
| `urlPath` | 문자열 | URL의 경로 부분(`/content/site/en.html`) |
| `externalDomain` | 문자열 | 아웃바운드 URL의 도메인(클라이언트 스팬에 사용) |
| `peer.service` | 문자열 | 다운스트림 서비스 호출됨 |
| `db.system`, `db.operation`, `db.namespace` | 문자열 | 데이터베이스 기술, 작업 및 스키마 |
| `messaging.system`, `messaging.destination` | 문자열 | Kafka, RabbitMQ, ... 및 주제 또는 대기열 |
| `rpc.system`, `rpc.method` | 문자열 | gRPC 및 유사 |
| `error.type` | 문자열 | 예외 또는 오류 유형 |
| `sling.script`, `sling.servlet`, `sling.requestUri` | 문자열 | AEM Sling 세부 정보 |
| `jsp.path` | 문자열 | AEM JSP 실행됨 |
| `jcr.operation`, `jcr.path` | 문자열 | AEM JCR 저장소 작업 및 경로 |
| `graphql.operation` | 문자열 | GraphQL 작업 이름 |

</details>

### 일반 지표 엔티티

| 기관 | 다음 포함 |
| ------------- | -------------------------------------------------- |
| `MetricGauge` | 모든 시점 OpenTelemetry 측정 샘플 |
| `MetricSum` | 모든 누적 또는 델타 OpenTelemetry 합계 샘플 |

**검색** 또는 조정된 엔터티가 없는 지표에 사용합니다. `WHERE metric = 'jvm.gc.duration'`.

<details open>
<summary><strong>지표 특성</strong>(모든 지표 엔터티에서 공유)</summary>

| 속성 | 유형 | 설명 |
| ----------------------------------------------------------------------- | --------- | ------------------------------------------------------- |
| `timestamp` | timestamp | 샘플 시간 |
| `metric` | 문자열 | OpenTelemetry 지표 이름 |
| `value` | 숫자 | 샘플 값(조정된 엔티티에서 변환됨. 아래 참조) |
| `unit` | 문자열 | 지표 단위 |
| `service`, `appName`, `host`, `environment`, `region`, `cloud.provider` | 문자열 | 리소스 ID |
| `state` | 문자열 | CPU 모드 또는 메모리 상태 |
| `cpu`, `device`, `direction`, `mountpoint` | 문자열 | CPU, 디바이스, I/O 방향, 파일 시스템 |
| `pool` | 문자열 | JVM 메모리 풀 또는 연결 풀 |
| `topic` | 문자열 | Kafka 주제 |
| `process.pid`, `process.name`, `process.owner` | 문자열 | 프로세스 ID |

</details>

### 조정된 지표 엔티티

조정된 각 엔터티는 올바른 필터 및 단위 변환으로 하나의 OpenTelemetry 지표를 래핑하므로 `value`은(는) 이미 의미가 있습니다. **카운터** 엔터티(✔)에 누적 합계가 있습니다. `rate(value)` 및 `TIMESERIES`(으)로 쿼리합니다.

| 기관 | `value`은(는) 다음을 의미합니다 | 단위 | 카운터 | 메모 |
| ------------------------------------------------------------------------------ | ----------------------------------- | --------- | ------- | -------------------------------------------- |
| `CpuUtilization` | CPU 바쁨 | % |         | 호스트별 `average(value)`에 대한 1분 롤업 |
| `CpuUserPercent` / `CpuSystemPercent` / `CpuIowaitPercent` / `CpuStealPercent` | 모드별 CPU 시간 | % |         |                                              |
| `MemoryUtilization` | 사용된 메모리 | % |         | `average(value)`에 대한 1분 롤업 |
| `LoadAverage1m` / `LoadAverage5m` / `LoadAverage15m` | 평균 로드 | — |         |                                              |
| `LoadAverageWithFallback` | 5m 로드, 1m로 폴백 | — |         |                                              |
| `DiskUtilization` | 마운트당 디스크 사용 | % |         | `mountpoint`의 Facet |
| `DiskFreeBytes` | 루트 파일 시스템 사용 가능 | 바이트 |         |                                              |
| `DiskReadRate` / `DiskWriteRate` | 디스크 처리량 | 바이트/초 | ✔ |                                              |
| `DiskReadOpsRate` / `DiskWriteOpsRate` | 디스크 IOPS | ops/s | ✔ |                                              |
| `NetworkReceiveRate` / `NetworkTransmitRate` | eth0 처리량 | 바이트/초 | ✔ |                                              |
| `NetworkReceiveRateAllDevices` / `NetworkTransmitRateAllDevices` | 모든 장치 | 바이트/초 | ✔ |                                              |
| `NetworkIoRate` | 모든 방향 및 장치가 결합됨 | 바이트/초 | ✔ |                                              |
| `NetworkReceiveErrorRate` / `NetworkTransmitErrorRate` | 네트워크 오류 | 오류/초 | ✔ |                                              |
| `NetworkReceivePacketRate` / `NetworkTransmitPacketRate` | 패킷 | 패킷/초 | ✔ |                                              |
| `ProcessCpuUtilization` | 프로세스별 CPU | % |         | `process.name`의 Facet |
| `ProcessMemoryUsage` | 프로세스별 메모리 | MB |         |                                              |
| `JvmHeapUsed` / `JvmHeapLimit` | JVM 힙 | MB |         | 메모리 풀별로 보고됨 |
| `JvmThreadCount` | JVM 스레드 | — |         |                                              |
| `JvmCpuRecentUtilization` | JVM 프로세스 CPU | % |         | 1분 롤업 |
| `DispatcherRequestRate` | AEM dispatcher HTTP 요청 | 요청/초 | ✔ |                                              |

&#x200B;> [!NOTE]
> JVM 메모리 지표는 메모리 풀당 **보고되므로 `average(value)`이(가) 풀 간 `JvmHeapUsed` / `JvmHeapLimit`의 평균입니다.**&#x200B;정확한 풀별 보기를 보려면 `FACET host, pool`을(를) 사용하십시오.

### `SpanBreakdown`: 미리 집계된 트랜잭션 구성

`SpanBreakdown`은(는) 각 트랜잭션의 분당 자식 범위 구성을 유지합니다. 수집 시 계산되므로 검색하기가 저렴합니다.

| 속성 | 유형 | 설명 |
| --------------- | --------- | ---------------------------------------------------------------------- |
| `timestamp` | timestamp | 1분 버킷 |
| `service` | 문자열 | 서비스 |
| `rootOperation` | 문자열 | 상위 트랜잭션 이름 |
| `segment` | 문자열 | 하위 범위 이름 |
| `callCount` | 숫자 | 버킷에 있는 이 세그먼트의 호출 |
| `avgDurationMs` | 숫자 | 버킷의 평균 세그먼트 기간(수동으로 단위를 `ms`(으)로 설정) |
