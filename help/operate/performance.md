---
title: 성능
description: Observability Insights에서 쿼리, 차트 및 대시보드 성능을 개선합니다.
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '227'
ht-degree: 3%
---

## 성능 {#performance}

모든 쿼리는 15초 시간 초과와 바이트 읽기 예산으로 실행됩니다. 이 문서에서는 Observability Insights가 쿼리를 자동으로 가속화하는 방법과 언제든지 빠르게 유지되는 대시보드를 디자인하는 방법에 대해 설명합니다.

### 롤업 가속(자동)

`Transaction` 차트는 다음 보류 중 **모두**&#x200B;일 때 미리 집계된 테이블을 읽습니다.

| 조건 | 요구 사항 |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------ |
| 모양 | 쿼리에 `TIMESERIES`이(가) 있습니다. 번호/테이블 합계는 항상 정확성을 위해 원시 상태로 읽습니다. |
| 창 | ≥ 3시간(1분 계층) · ≥ 1d(5분 계층) · ≥ 7d(1시간 계층) |
| 집계 | `count(*)`, `percentage(*, WHERE status = 'Error')`, `apdex(duration, 500)`, `percentile(duration, 50 \| 95 \| 99)`만 |
| `FACET` / `WHERE` | `service`, `appInstance`, `kind`에만 |

`CpuUtilization`, `MemoryUtilization`(`average(value)`) 및 `JvmCpuRecentUtilization`(`average(value)`) 차트(≥ 3시간 동안 `FACET host`/`WHERE host …` 읽기 1분 롤업)

**디자인 팁:** 7~31일 트렌드 위젯의 경우 롤업 친화적 세트(p95, 오류 %, Apdex @ 500, 개수, FACET 서비스)를 사용하십시오. 이러한 위젯은 모든 범위에서 빠르게 유지됩니다. 더 짧은 창 또는 표에 대해 `FACET host`/`name`, `average(duration)` 또는 `urlPath` 필터를 저장합니다.

### 일반 팁

- 패싯된 모든 쿼리(특히 `FACET urlPath`, `traceId`, `jcr.path`(높은 카디널리티)에서 `LIMIT`을(를) 사용합니다.
- 낮은 카디널리티 특성(`service`, `kind`, `status`)에 대해 `WHERE`(으)로 일찍 필터링합니다.
- 인바운드 요청만 필요한 경우 `Span`보다 `Transaction`을(를) 선호합니다(훨씬 적은 행).
- `uniqueCount`은(는) 정확합니다. 긴 기간 동안 카디널리티가 높은 특성에서는 값이 가장 비싼 집계입니다.
- 몇 분 정도의 창을 제외하고 `TIMESERIES MAX`을(를) 사용하지 마십시오.
- 대시보드는 30초 동안 캐시 결과를 위젯합니다. 새로 고침 버튼은 강제로 다시 실행합니다.
- 15초 시간 초과 또는 바이트 읽기 예산 오류를 처리하는 위젯 — 창의 범위를 좁히거나 필터를 추가하거나 패싯을 줄입니다.
