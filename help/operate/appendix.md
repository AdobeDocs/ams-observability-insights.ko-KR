---
title: 부록
description: Observability Insights SQL 쿼리 및 대시보드에 대한 추가 정보를 참조하십시오.
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '145'
ht-degree: 5%
---

## 부록 {#appendix}

**이 문서에서:** [14.1 제한](#141-limits) · [14.2 내보내기 파일 형식](#142-export-file-format) · [14.3 한 페이지 치트 시트](#143-one-page-cheat-sheet)

### 제한

| 항목 | 제한 |
| -------------------------------- | ----------------------------------------------------------------------- |
| 쿼리 텍스트 | 4KB |
| `FACET` 특성 | 5 |
| `LIMIT` | 1 - 5 000(기본값: 숫자 1, 패싯 10, 이벤트 100, 시계열 5 000) |
| `IN (…)`개 값 | 1 - 100 |
| 기간 | 31일 |
| 쿼리 시간 제한 | 15초(바이트 읽기 예산 추가) |
| `TIMESERIES AUTO` 최소 버킷 | 2분 |
| 공식 | 2 - 6 쿼리, 표현식 ≤ 500자, 이름 ≤ 40자 |
| 대시보드 | ≤ 40개 탭 · ≤ 200개 위젯 · ≤ 50개 변수 |
| 위젯 | title ≤ 200 · subtitle ≤ 500 · unit ≤ 40 · ≤ 10 임계값 |
| 텍스트 위젯 | ≤ 4천 자 |
| 변수 | 이름 ≤ 64자 · ≤ 500개의 옵션/값 · 쿼리 ≤ 4KB |

### 파일 형식 내보내기

**Export**&#x200B;을(를) 사용하여 만들고 **Import**&#x200B;을(를) 사용하여 로드하십시오. 버전 제어 대시보드나 테넌트 간 승격에 유용합니다.

```json
{
  "name": "Golden Signals",
  "templateVariables": [
    {
      "id": "var-1",
      "name": "svc",
      "label": "Service",
      "kind": "query",
      "multi": true,
      "query": "SELECT count(*) FROM Transaction FACET service LIMIT 200",
      "value": null
    }
  ],
  "pages": [
    {
      "id": "page-1",
      "name": "Overview",
      "widgets": [
        {
          "id": "sec-kpis",
          "title": "SLO KPIs",
          "queryLanguage": "text",
          "text": "",
          "isSection": true,
          "collapsed": false,
          "layout": { "x": 0, "y": 0, "w": 12, "h": 1 }
        },
        {
          "id": "w-error-rate",
          "title": "Error %",
          "queryLanguage": "SQL",
          "query": "SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction",
          "vizType": "stat",
          "unit": "%",
          "thresholds": [
            { "value": 0, "status": "ok" },
            { "value": 1, "status": "warning" },
            { "value": 5, "status": "error" }
          ],
          "layout": { "x": 0, "y": 1, "w": 3, "h": 2 }
        },
        {
          "id": "w-throughput",
          "title": "Throughput by service",
          "queryLanguage": "SQL",
          "query": "SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 8",
          "vizType": "area",
          "layout": { "x": 3, "y": 1, "w": 9, "h": 4 }
        },
        {
          "id": "w-availability",
          "title": "Availability %",
          "queryLanguage": "SQL-formula",
          "formulaQueries": [
            {
              "name": "ok",
              "query": "SELECT count(*) AS v FROM Transaction WHERE status != 'Error'"
            },
            {
              "name": "total",
              "query": "SELECT count(*) AS v FROM Transaction"
            }
          ],
          "formulaExpression": "(ok / total) * 100",
          "vizType": "stat",
          "unit": "%",
          "layout": { "x": 0, "y": 3, "w": 3, "h": 2 }
        }
      ]
    }
  ],
  "exportedAt": "2026-10-01T00:00:00.000Z"
}
```

`vizType` 값: `timeseries`(줄), `area`, `bar`, `pie`, `stat`(숫자), `table`. 임계값 `status`: `ok`(녹색), `warning`(주황색), `error`(빨간색).

### 1페이지 치트 시트

```text
SHAPE        SELECT agg FROM Entity                         -> Number
             ... FACET a[, b..e] LIMIT n                     -> Bar / Pie / Table
             ... TIMESERIES AUTO                             -> Line / Area
             ... FACET a TIMESERIES AUTO LIMIT n             -> top-n series
             SELECT attr1, attr2 FROM Entity LIMIT n         -> event Table

AGGREGATES   count(*)  average(x)  sum(x)  min(x)  max(x)  uniqueCount(x)
             percentile(x, 95)  stddev(x)  latest(x[, rank])  earliest(x[, rank])
             percentage(*, WHERE c)  apdex(duration, 500)  percentOfTotal(x)
             rate(value)        -- counters, TIMESERIES only
             derivative(value)  -- gauges,   TIMESERIES only
             agg FILTER (WHERE c) AS alias

WHERE        = != < <= > >=   LIKE / NOT LIKE ('%', '_')   IN / NOT IN (...)
             IS [NOT] NULL   AND OR NOT ( )   strings in 'single quotes'
             string attr present?  ->  attr != ''

ORDER/LIMIT  ORDER BY alias [ASC|DESC]   LIMIT 1..5000

VARIABLES    = {{name}}   IN ({{name}})   numeric: > {{name}}

FORMULA      2-6 named queries, one metric each, no FACET, same shape
             expression: + - * / ( ) numbers names
```
