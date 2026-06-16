# 장비 수집 데이터 분석 (정합성 분석) API 요청 명세

원본(NW-A·NW-B)과 감사본(NW-C)을 대조하여 누락 현황을 분석하는 화면에서 사용할 API 명세입니다.
프론트는 서버가 **대조·집계·페이지네이션을 모두 처리**하는 것을 전제로 합니다. (수십만 건 → 클라이언트 집계 불가)

---

## 1. 공통 사항

### 1.1 대조 개념
- **원본(source)**: NW-A, NW-B 가 직접 제공한 데이터
- **감사(audit)**: NW-C 가 별도 수집한 데이터. 단, NW-C 레코드는 내부적으로 `sysCd` 로 **NW-A 또는 NW-B** 를 가리킴
- **대조 키**: `(collDt, sysCd, eqpId)` — 동일 키에 대해 원본 존재여부 ↔ 감사 존재여부 비교
- **모집단**: 수집 대상(`collYn = 'Y'`) 장비만. **수집 제외(`collYn = 'N'`)는 분석에서 제외**

### 1.2 상태 코드 (status)

| code | 라벨 | 의미 |
|------|------|------|
| `MATCH` | 일치 | 원본·감사 양쪽 모두 존재 |
| `SOURCE_MISSING` | 원본 미제공 | 감사(NW-C)에는 있는데 원본(A/B)에는 없음 |
| `AUDIT_MISSING` | 감사 누락 | 원본에는 있는데 감사(NW-C)에는 없음 |
| `BOTH_MISSING` | 양측 누락 | 원본·감사 모두 없음 (수집 대상이나 당일 데이터 없음) |

> 파일 개수 차이는 별도 상태로 분류하지 않습니다. 양쪽 존재 시 `MATCH` 로 보고, 개수 차이는 화면에서 참고 표시 + Diff 도구로 개별 확인합니다.

### 1.3 공통 규칙
- 날짜 파라미터/응답: `YYYYMMDD` 문자열 (예: `20260616`)
- 일시: `YYYYMMDD HH:mm:ss` 문자열
- `sysCd`: `103001`(NW-A), `103002`(NW-B). 미지정 시 전체(A+B)
- 시계열 조회 기간은 **최대 30일** (데이터 보관 기간)

---

## 2. API 목록

| # | Method | Path | 용도 |
|---|--------|------|------|
| 1 | GET | `/collection/reconcile/summary` | 요약(전체 대상 수 + 상태별 건수) |
| 2 | GET | `/collection/reconcile/list` | 대조 결과 목록 (서버 페이지네이션) |
| 3 | GET | `/collection/reconcile/device` | 장비 단건 상세 (원본 vs 감사) |
| 4 | GET | `/collection/reconcile/device/trend` | 장비 30일 시계열 |

---

## 3. API 1 — 비교 요약

KPI 카드 및 전체 수집 대상 수량 표시에 사용.

### Request
```
GET /collection/reconcile/summary?collDt=20260616&sysCd=103001&eqpId=
```

| param | 필수 | 설명 |
|-------|------|------|
| `collDt` | Y | 조회 일자 (YYYYMMDD) |
| `sysCd` | N | NW 코드. 미지정 시 전체 |
| `eqpId` | N | 장비 ID 부분 검색 |

### Response
```jsonc
{
  "collDt": "20260616",
  "total": 2000,              // 대조 모집단(collYn=Y) 수
  "counts": {
    "match": 1890,
    "sourceMissing": 46,
    "auditMissing": 40,
    "bothMissing": 24
  },
  "bySysCd": [                // NW별 분해 (선택)
    { "sysCd": "103001", "sysNm": "NW-A", "match": 1130, "sourceMissing": 28, "auditMissing": 24, "bothMissing": 18 },
    { "sysCd": "103002", "sysNm": "NW-B", "match": 760,  "sourceMissing": 18, "auditMissing": 16, "bothMissing": 6 }
  ]
}
```

---

## 4. API 2 — 비교 결과 목록 (서버 페이지네이션)

상태 카드 클릭 시 해당 카테고리 목록을 페이지 단위로 조회.

### Request
```
GET /collection/reconcile/list?collDt=20260616&sysCd=103001&status=SOURCE_MISSING&eqpId=&page=1&size=50&sortField=eqpId&sortDir=asc
```

| param | 필수 | 설명 |
|-------|------|------|
| `collDt` | Y | 조회 일자 |
| `sysCd` | N | NW 코드 |
| `status` | N | 상태 코드. 미지정 시 전체 카테고리 |
| `eqpId` | N | 장비 ID 부분 검색 |
| `page` | Y | 1-base 페이지 번호 |
| `size` | Y | 페이지 크기 (예: 50) |
| `sortField` | N | 정렬 필드 (기본 `eqpId`) |
| `sortDir` | N | `asc` / `desc` |

### Response
```jsonc
{
  "page": 1,
  "size": 50,
  "totalCount": 46,
  "rows": [
    {
      "sysCd": "103001",
      "sysNm": "NW-A",
      "eqpId": "DEV-A-00012",
      "modelName": "Model-X200",
      "status": "SOURCE_MISSING",
      "srcFileCnt": null,                 // 원본 없음 → null
      "auditFileCnt": 12,                 // 감사(NW-C) 파일 수
      "srcCollDtti": null,
      "auditCollDtti": "20260616 09:30:00"
    }
  ]
}
```

> **성능 권고**
> - 딥 페이징 대비 keyset(cursor) 방식도 옵션 제공 권장: `lastEqpId` 기준 조회
> - 인덱스: `(collDt, sysCd, status, eqpId)`

---

## 5. API 3 — 장비 단건 상세 (원본 vs 감사)

행 클릭 시 상세 Drawer 에서 원본·감사 파일 목록을 나란히 표시.

### Request
```
GET /collection/reconcile/device?collDt=20260616&sysCd=103001&eqpId=DEV-A-00012
```

| param | 필수 | 설명 |
|-------|------|------|
| `collDt` | Y | 조회 일자 |
| `sysCd` | Y | NW 코드 |
| `eqpId` | Y | 장비 ID |

### Response
```jsonc
{
  "sysCd": "103001",
  "sysNm": "NW-A",
  "eqpId": "DEV-A-00012",
  "modelName": "Model-X200",
  "status": "MATCH",
  "source": {
    "exists": true,
    "collDtti": "20260616 09:12:00",
    "files": [
      { "fileNm": "config_main.json", "fileId": "DEV-A-00012-src-F001", "collFileTpNm": "TXT", "fileSizeVal": 512 }
    ]
  },
  "audit": {
    "exists": true,
    "collDtti": "20260616 09:30:00",
    "files": [
      { "fileNm": "config_main.json", "fileId": "DEV-A-00012-aud-F001", "collFileTpNm": "TXT", "fileSizeVal": 540 }
    ]
  }
}
```

### 파일 객체 필드

| 필드 | 설명 |
|------|------|
| `fileNm` | 파일명 |
| `fileId` | 파일 식별자 (내용 조회/다운로드용) |
| `collFileTpNm` | 파일 타입. `BIN` 이면 바이너리(읽기 불가), 그 외(`TXT` 등)는 텍스트로 열람/Diff 가능 |
| `fileSizeVal` | 파일 크기 (bytes) |

> 한쪽이 없는 경우 해당 side 는 `"exists": false`, `"files": []`, `"collDtti": null`.

---

## 6. API 4 — 장비 30일 시계열

상세 Drawer 의 정합성 추이(언제부터 누락이 시작됐는지) 표시.

### Request
```
GET /collection/reconcile/device/trend?sysCd=103001&eqpId=DEV-A-00012&fromDt=20260518&toDt=20260616
```

| param | 필수 | 설명 |
|-------|------|------|
| `sysCd` | Y | NW 코드 |
| `eqpId` | Y | 장비 ID |
| `fromDt` | Y | 시작일 (YYYYMMDD) |
| `toDt` | Y | 종료일 (YYYYMMDD). **fromDt~toDt 최대 30일** |

### Response
```jsonc
{
  "sysCd": "103001",
  "eqpId": "DEV-A-00012",
  "series": [
    { "collDt": "20260614", "srcExists": true,  "srcFileCnt": 12, "auditExists": true,  "auditFileCnt": 12, "status": "MATCH" },
    { "collDt": "20260615", "srcExists": false, "srcFileCnt": 0,  "auditExists": true,  "auditFileCnt": 12, "status": "SOURCE_MISSING" },
    { "collDt": "20260616", "srcExists": false, "srcFileCnt": 0,  "auditExists": false, "auditFileCnt": 0,  "status": "BOTH_MISSING" }
  ]
}
```

---

## 7. 백엔드 구현 권고 (선택)

- **일배치 대조 결과 사전 산출**: 매일 원본↔감사 대조 결과를 집계 테이블에 적재
  `(collDt, sysCd, eqpId, status, srcFileCnt, auditFileCnt, srcCollDtti, auditCollDtti)`
  → 4개 API 모두 이 테이블만 조회하면 되어 조회 부하 최소화. 30일 보관 기준으로 적재량 관리 용이.
- 요약(API1)·목록(API2)은 동일 집계 테이블에서 `GROUP BY` / 페이지네이션으로 처리.
- 상세(API3)·시계열(API4)은 동일 키로 조회.
