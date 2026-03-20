# 선문대학교 스마트 복지 플랫폼 — 상세 설계 명세서

> **Version:** 1.0  
> **Date:** 2026-03-20  
> **Status:** Gemini 초안 비판적 리뷰 + 구체 설계 완료

---

## 1. Gemini 초안 비판 요약 — 반드시 수정할 것

### 1.1 데이터 치명적 오류

| # | 문제 | 심각도 | 해결 방안 |
|---|------|--------|----------|
| 1 | 6개 기관만 추출, 실제 74개 | CRITICAL | 전수 파싱 완료 (73개 레코드, 별첨 JSON) |
| 2 | 용평리조트 주소가 보령시 (PPT 원본 오류 그대로 복사) | HIGH | address_note 필드로 플래그 + 인사총무팀 확인 요청 |
| 3 | target 분류가 4가지뿐 (FACULTY/STUDENT/FAMILY/FOREIGNER) | HIGH | 7가지로 확장: +TRAINEE, ACQUAINTANCE, ALL_UNIVERSITY |
| 4 | 하위 카테고리 없음 (병원을 일괄 처리) | MEDIUM | 18개 세분류 ENUM 설계 완료 |
| 5 | credential을 프론트엔드 JSON에 하드코딩 | CRITICAL | 별도 encrypted 테이블 + 인증 후 API 조회 |
| 6 | 서울메디컬을 단일 기관으로 처리 | MEDIUM | MEDICAL_PLATFORM 카테고리로 별도 취급 |

### 1.2 설계 부재 영역

- **인증 흐름**: SSO 연동 명세 없음 → 본 문서 §4에서 상세 설계
- **암호화 방식**: credential 저장/전송 방법 미정 → §4.2 AES-256-GCM 명세
- **API 에러 핸들링**: 정의 없음 → §3.4 에러 응답 표준
- **데이터 이관 파이프라인**: PPT→DB 자동화 없음 → §5 정규화 파이프라인
- **감사 로그**: 없음 → §4.3 감사 테이블 + 정책

---

## 2. 데이터베이스 스키마 (PostgreSQL 16)

### 2.1 코어 테이블

```sql
-- ================================================
-- ENUM 정의
-- ================================================

CREATE TYPE institution_category AS ENUM (
  'HOSPITAL_GENERAL',     -- 병원/일반 (종합, 내과, 외과)
  'HOSPITAL_DENTAL',      -- 병원/치과
  'HOSPITAL_OPHTH',       -- 병원/안과
  'HOSPITAL_ORIENTAL',    -- 병원/한방
  'HOSPITAL_REHAB',       -- 병원/요양
  'HOSPITAL_PLASTIC',     -- 병원/성형
  'HOSPITAL_MENTAL',      -- 병원/정신건강
  'HOSPITAL_PAIN',        -- 병원/통증의학
  'MEDICAL_PLATFORM',     -- 의료복지플랫폼
  'ACCOMMODATION_RESORT', -- 숙박/리조트·콘도
  'ACCOMMODATION_HOTEL',  -- 숙박/호텔·스테이
  'ACCOMMODATION_CAMPING',-- 숙박/캠핑·글램핑
  'ACCOMMODATION_SPA',    -- 스파/온천
  'FUNERAL',              -- 장례
  'FOOD',                 -- 음식
  'SHOPPING',             -- 쇼핑/몰
  'CULTURE',              -- 문화/레저
  'FITNESS',              -- 운동/피트니스
  'TELECOM'               -- 통신
);

-- UI에서 상위 그룹핑 매핑:
--   병원 = HOSPITAL_* + MEDICAL_PLATFORM
--   숙박 = ACCOMMODATION_*
--   장례 = FUNERAL
--   기타 = FOOD + SHOPPING + CULTURE + FITNESS + TELECOM

CREATE TYPE target_audience AS ENUM (
  'FACULTY',          -- 교직원
  'FACULTY_FAMILY',   -- 교직원 직계가족
  'STUDENT',          -- 재학생
  'FOREIGN_STUDENT',  -- 외국인 유학생
  'TRAINEE',          -- 교육생
  'ACQUAINTANCE',     -- 지인 포함
  'ALL_UNIVERSITY'    -- 선문대학교 전체
);

-- ================================================
-- 핵심 테이블
-- ================================================

CREATE TABLE welfare_institutions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name            VARCHAR(120) NOT NULL,
  category        institution_category NOT NULL,

  -- 소개 / 혜택
  description     TEXT,                        -- 기관 한줄 소개
  benefit_summary TEXT NOT NULL,               -- 할인/혜택 요약
  benefit_detail  TEXT,                        -- 혜택 상세 (줄바꿈 포함)
  reservation_method TEXT,                     -- 예약/이용 방법

  -- 주소 (도로명 기준, 지번은 레거시 보관)
  road_address    VARCHAR(200),
  jibun_address   VARCHAR(200),
  address_detail  VARCHAR(100),               -- 층/호 등 상세
  address_note    TEXT,                        -- 특이사항 (예: "전국 다수", 주소 불일치 등)

  -- 좌표 (카카오맵 Geocoding으로 검증)
  latitude        DECIMAL(10,7),
  longitude       DECIMAL(10,7),
  geo_verified    BOOLEAN DEFAULT FALSE,

  -- 연락처
  phone           VARCHAR(30),
  phone_secondary VARCHAR(30),
  website         VARCHAR(500),

  -- 민감정보 플래그
  has_credentials BOOLEAN DEFAULT FALSE,

  -- 정렬/관리
  sort_order      INT DEFAULT 0,              -- PPT 원본 슬라이드 순서
  is_active       BOOLEAN DEFAULT TRUE,
  ppt_slide_number INT,                        -- 원본 PPT 슬라이드 번호 (추적용)

  -- 타임스탬프
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW(),
  created_by      VARCHAR(50) DEFAULT 'system',
  updated_by      VARCHAR(50) DEFAULT 'system'
);

-- 전문 검색 인덱스
CREATE INDEX idx_inst_name_trgm ON welfare_institutions 
  USING gin (name gin_trgm_ops);
CREATE INDEX idx_inst_benefit_trgm ON welfare_institutions 
  USING gin (benefit_summary gin_trgm_ops);
CREATE INDEX idx_inst_category ON welfare_institutions (category);
CREATE INDEX idx_inst_active ON welfare_institutions (is_active) WHERE is_active = TRUE;

-- PostGIS 위치 인덱스 (위치기반 정렬용)
-- CREATE INDEX idx_inst_geo ON welfare_institutions 
--   USING gist (ST_MakePoint(longitude, latitude));

-- ================================================
-- 대상자 M:N 관계
-- ================================================

CREATE TABLE institution_targets (
  institution_id UUID NOT NULL 
    REFERENCES welfare_institutions(id) ON DELETE CASCADE,
  target         target_audience NOT NULL,
  target_note    VARCHAR(200),                -- "동반 2인까지" 같은 조건
  PRIMARY KEY (institution_id, target)
);

CREATE INDEX idx_targets_target ON institution_targets (target);

-- ================================================
-- 민감 인증정보 (격리 테이블)
-- ================================================

CREATE TABLE institution_credentials (
  institution_id  UUID PRIMARY KEY 
    REFERENCES welfare_institutions(id) ON DELETE CASCADE,
  login_url       VARCHAR(500),               -- 전용 예약사이트 URL
  login_id_enc    BYTEA NOT NULL,             -- AES-256-GCM 암호화
  login_pw_enc    BYTEA NOT NULL,             -- AES-256-GCM 암호화
  iv              BYTEA NOT NULL,             -- 초기화 벡터 (12바이트)
  tag             BYTEA NOT NULL,             -- GCM 인증 태그 (16바이트)
  access_note     TEXT,                       -- 예약 절차 안내
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- ================================================
-- 감사 로그
-- ================================================

CREATE TABLE audit_log (
  id          BIGSERIAL PRIMARY KEY,
  table_name  VARCHAR(50) NOT NULL,
  record_id   UUID,
  action      VARCHAR(10) NOT NULL CHECK (action IN ('INSERT','UPDATE','DELETE','READ')),
  actor_id    VARCHAR(50) NOT NULL,           -- 사용자 ID
  actor_role  VARCHAR(20),                    -- FACULTY, ADMIN 등
  acted_at    TIMESTAMPTZ DEFAULT NOW(),
  ip_address  INET,
  user_agent  VARCHAR(500),
  old_values  JSONB,
  new_values  JSONB,
  detail      TEXT                            -- credential 접근 시 기관명 기록
);

CREATE INDEX idx_audit_actor ON audit_log (actor_id, acted_at DESC);
CREATE INDEX idx_audit_table ON audit_log (table_name, acted_at DESC);
CREATE INDEX idx_audit_cred ON audit_log (table_name, action) 
  WHERE table_name = 'institution_credentials' AND action = 'READ';
```

### 2.2 UI 카테고리 그룹핑 뷰

```sql
CREATE VIEW v_category_groups AS
SELECT 
  CASE 
    WHEN category::text LIKE 'HOSPITAL_%' OR category = 'MEDICAL_PLATFORM'
      THEN '병원'
    WHEN category::text LIKE 'ACCOMMODATION_%'
      THEN '숙박'
    WHEN category = 'FUNERAL' THEN '장례'
    ELSE '기타'
  END AS category_group,
  category AS category_detail,
  COUNT(*) AS count
FROM welfare_institutions
WHERE is_active = TRUE
GROUP BY 1, 2
ORDER BY 1, count DESC;
```

---

## 3. FastAPI 백엔드 API 명세

### 3.1 프로젝트 구조

```
backend/
├── main.py                      # FastAPI app 생성, 미들웨어
├── api/
│   ├── __init__.py
│   ├── deps.py                  # Depends: DB 세션, 현재 사용자
│   └── v1/
│       ├── __init__.py
│       ├── router.py            # APIRouter 통합
│       ├── institutions.py      # 공개 조회 API
│       ├── credentials.py       # 인증 필요 API
│       ├── admin.py             # 관리자 CRUD
│       └── auth.py              # SSO 콜백, JWT 발급
├── services/
│   ├── __init__.py
│   ├── institution_service.py   # 비즈니스 로직
│   ├── credential_service.py    # AES 암복호화
│   ├── geocoding_service.py     # 카카오 Geocoding
│   ├── search_service.py        # pg_trgm 검색
│   └── audit_service.py         # 감사 로그 기록
├── repositories/
│   ├── __init__.py
│   ├── institution_repo.py      # SQL 쿼리 계층
│   └── audit_repo.py
├── models/
│   ├── __init__.py
│   ├── institution.py           # SQLAlchemy ORM
│   ├── audit.py
│   └── enums.py                 # Python Enum ↔ PG Enum 매핑
├── schemas/
│   ├── __init__.py
│   ├── institution.py           # Pydantic 요청/응답 모델
│   ├── auth.py
│   └── common.py                # PaginatedResponse 등
├── core/
│   ├── __init__.py
│   ├── config.py                # pydantic-settings
│   ├── security.py              # JWT + AES-256-GCM
│   ├── database.py              # async SQLAlchemy
│   └── exceptions.py            # 커스텀 예외
└── tests/
    ├── conftest.py              # pytest fixtures
    ├── test_institutions.py
    ├── test_credentials.py
    ├── test_auth.py
    └── test_normalizer.py
```

### 3.2 엔드포인트 상세

#### 공개 API (인증 불필요)

```
GET /api/v1/institutions
  Query Parameters:
    category_group  : string (optional) — "병원" | "숙박" | "장례" | "기타"
    category        : string (optional) — "HOSPITAL_DENTAL" 등 세분류
    target          : string (optional) — "FACULTY" | "STUDENT" | "FOREIGN_STUDENT"
    q               : string (optional) — 기관명/혜택 텍스트 검색 (pg_trgm)
    lat             : float  (optional) — 위치기반 정렬
    lng             : float  (optional) — 위치기반 정렬
    radius_km       : int    (default: 50) — 검색 반경 km
    sort            : string (default: "sort_order")
                      — "sort_order" | "name" | "distance" | "category"
    page            : int    (default: 1, min: 1)
    size            : int    (default: 20, min: 1, max: 50)
  
  Response 200:
    {
      "items": [InstitutionListItem],
      "total": 73,
      "page": 1,
      "size": 20,
      "has_next": true,
      "category_counts": {        // 현재 필터 적용 후 카테고리별 카운트
        "병원": 42,
        "숙박": 18,
        "장례": 4,
        "기타": 9
      }
    }

  InstitutionListItem:
    {
      "id": "uuid",
      "name": "천안충무병원",
      "category": "HOSPITAL_GENERAL",
      "category_group": "병원",
      "category_label": "병원/일반",
      "targets": [
        {"type": "FACULTY", "label": "교직원"},
        {"type": "FACULTY_FAMILY", "label": "직계가족"},
        {"type": "STUDENT", "label": "재학생"}
      ],
      "benefit_summary": "비급여: 10%",
      "phone": "041-570-7555",
      "website": "http://www.cmhos.co.kr/",
      "road_address": "충남 천안시 서북구 다가말3길 8",
      "has_credentials": false,
      "distance_km": 3.2          // lat/lng 제공 시에만 포함
    }

---

GET /api/v1/institutions/{id}
  Response 200:
    {
      ...InstitutionListItem,
      "description": "중부권 최대 종합병원, 13개의 전문화 센터",
      "benefit_detail": null,
      "reservation_method": "전화 또는 홈페이지 ... / 사원증 및 명함 지참",
      "address_detail": null,
      "latitude": 36.xxxx,
      "longitude": 127.xxxx,
      "phone_secondary": null,
      "has_credentials": false
    }
  
  Response 404: { "error": "INSTITUTION_NOT_FOUND", "message": "..." }

---

GET /api/v1/categories
  Response 200:
    {
      "groups": [
        {
          "group": "병원",
          "icon": "heart-pulse",
          "total": 42,
          "subcategories": [
            {"key": "HOSPITAL_DENTAL", "label": "치과", "count": 16},
            {"key": "HOSPITAL_GENERAL", "label": "일반", "count": 10},
            ...
          ]
        },
        ...
      ]
    }
```

#### 인증 필요 API

```
GET /api/v1/institutions/{id}/credentials
  Headers:
    Authorization: Bearer {jwt_access_token}
  
  Pre-conditions:
    - JWT 유효 + role = "FACULTY" 또는 "ADMIN"
    - 해당 institution의 has_credentials = true
    - Rate limit: 10회/시간/사용자
  
  Response 200:
    {
      "login_url": "https://reserve.high1.com/...",
      "login_id": "선문대학교",        // 서버에서 복호화
      "login_pw": "high4444#",         // 서버에서 복호화
      "access_note": "선문대 교직원 전용 예약사이트에서 예약",
      "accessed_at": "2026-03-20T10:30:00Z"
    }
  
  Side Effect:
    - audit_log INSERT (actor, institution_id, IP, timestamp)
    - access_count INCREMENT on institution_credentials
  
  Response 401: { "error": "UNAUTHORIZED", "message": "로그인이 필요합니다" }
  Response 403: { "error": "FORBIDDEN", "message": "교직원만 조회할 수 있습니다" }
  Response 404: { "error": "NO_CREDENTIALS", "message": "..." }
  Response 429: { "error": "RATE_LIMITED", "message": "...", "retry_after": 3600 }
```

#### 관리자 API

```
POST   /api/v1/admin/institutions          — 기관 등록
PUT    /api/v1/admin/institutions/{id}      — 기관 수정
DELETE /api/v1/admin/institutions/{id}      — 기관 비활성화 (soft delete)
POST   /api/v1/admin/institutions/bulk      — CSV/JSON 일괄 등록
GET    /api/v1/admin/audit-log              — 감사 로그 조회
POST   /api/v1/admin/geocode/{id}           — 특정 기관 좌표 재계산

  Headers (모든 관리자 API):
    Authorization: Bearer {jwt_access_token}
    X-Admin-Reason: "2026년 상반기 협약 갱신"  // 변경 사유 (audit_log.detail)
  
  Pre-conditions:
    - JWT role = "ADMIN"
```

### 3.3 핵심 서비스 코드

#### credential_service.py — AES-256-GCM 암복호화

```python
"""
민감 인증정보 암복호화 서비스.
KEY는 환경변수 CREDENTIAL_ENCRYPTION_KEY에서 로드 (32바이트 hex).
프로덕션에서는 AWS KMS / Vault 연동 권장.
"""
import os
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

class CredentialService:
    def __init__(self):
        key_hex = os.environ["CREDENTIAL_ENCRYPTION_KEY"]
        # 반드시 64자 hex = 32바이트 = 256비트
        if len(key_hex) != 64:
            raise ValueError("CREDENTIAL_ENCRYPTION_KEY must be 64 hex chars (256 bits)")
        self._key = bytes.fromhex(key_hex)
        self._aesgcm = AESGCM(self._key)
    
    def encrypt(self, plaintext: str) -> tuple[bytes, bytes, bytes]:
        """
        Returns: (ciphertext, iv, tag)
        - iv: 12바이트 랜덤 nonce
        - tag: GCM이 자동으로 ciphertext에 append (마지막 16바이트)
        """
        iv = os.urandom(12)
        data = plaintext.encode("utf-8")
        # AESGCM.encrypt는 ciphertext + tag(16bytes)를 반환
        ct_with_tag = self._aesgcm.encrypt(iv, data, None)
        ct = ct_with_tag[:-16]
        tag = ct_with_tag[-16:]
        return ct, iv, tag
    
    def decrypt(self, ciphertext: bytes, iv: bytes, tag: bytes) -> str:
        """
        복호화. 실패 시 InvalidTag 예외 발생.
        """
        ct_with_tag = ciphertext + tag
        plaintext = self._aesgcm.decrypt(iv, ct_with_tag, None)
        return plaintext.decode("utf-8")


# !! 절대 하면 안 되는 것들 !!
# 1. 복호화된 credential을 로그에 기록
# 2. 응답에 Cache-Control 없이 반환
# 3. KEY를 코드/DB/Git에 저장
# 4. ECB 모드 사용 (GCM 필수)
# 5. IV 재사용 (매번 os.urandom 필수)
```

#### institution_service.py — 거리 계산 쿼리

```python
from sqlalchemy import text

class InstitutionService:
    
    async def list_with_distance(
        self, 
        db: AsyncSession,
        lat: float | None = None,
        lng: float | None = None,
        radius_km: int = 50,
        **filters
    ):
        """
        위치기반 정렬. PostGIS 없이 Haversine 공식 사용.
        정밀도 요구가 낮으므로 (km 단위) 구면 근사 충분.
        """
        base_query = """
            SELECT wi.*,
              CASE WHEN :lat IS NOT NULL AND wi.latitude IS NOT NULL THEN
                6371 * acos(
                  cos(radians(:lat)) * cos(radians(wi.latitude)) *
                  cos(radians(wi.longitude) - radians(:lng)) +
                  sin(radians(:lat)) * sin(radians(wi.latitude))
                )
              ELSE NULL END AS distance_km
            FROM welfare_institutions wi
            WHERE wi.is_active = TRUE
        """
        
        # 카테고리 필터
        if filters.get("category_group"):
            group_map = {
                "병원": "category::text LIKE 'HOSPITAL_%' OR category = 'MEDICAL_PLATFORM'",
                "숙박": "category::text LIKE 'ACCOMMODATION_%'",
                "장례": "category = 'FUNERAL'",
                "기타": "category NOT LIKE 'HOSPITAL_%' AND category NOT LIKE 'ACCOMMODATION_%' AND category != 'FUNERAL' AND category != 'MEDICAL_PLATFORM'",
            }
            condition = group_map.get(filters["category_group"])
            if condition:
                base_query += f" AND ({condition})"
        
        # 대상자 필터
        if filters.get("target"):
            base_query += """
                AND wi.id IN (
                    SELECT institution_id FROM institution_targets 
                    WHERE target = :target
                )
            """
        
        # 텍스트 검색 (pg_trgm)
        if filters.get("q"):
            base_query += """
                AND (
                    wi.name % :q OR 
                    wi.benefit_summary % :q OR
                    wi.road_address % :q
                )
            """
        
        # 거리 필터
        if lat and lng:
            base_query = f"""
                SELECT * FROM ({base_query}) sub
                WHERE distance_km IS NULL OR distance_km <= :radius_km
                ORDER BY COALESCE(distance_km, 99999)
            """
        else:
            base_query += " ORDER BY wi.sort_order"
        
        return await db.execute(text(base_query), {
            "lat": lat, "lng": lng, "radius_km": radius_km,
            "target": filters.get("target"),
            "q": filters.get("q"),
        })
```

### 3.4 에러 응답 표준

```python
# schemas/common.py

class ErrorResponse(BaseModel):
    error: str          # 기계 판독용 코드
    message: str        # 사람 읽기용 한국어 메시지
    detail: dict | None = None

# 에러 코드 목록
ERROR_CODES = {
    "INSTITUTION_NOT_FOUND":    (404, "요청한 기관을 찾을 수 없습니다"),
    "NO_CREDENTIALS":           (404, "해당 기관의 인증 정보가 없습니다"),
    "UNAUTHORIZED":             (401, "로그인이 필요합니다"),
    "FORBIDDEN":                (403, "접근 권한이 없습니다"),
    "RATE_LIMITED":             (429, "요청 횟수를 초과했습니다. 잠시 후 다시 시도해주세요"),
    "VALIDATION_ERROR":         (422, "입력값이 올바르지 않습니다"),
    "GEOCODING_FAILED":         (502, "주소 좌표 변환에 실패했습니다"),
    "INTERNAL_ERROR":           (500, "서버 내부 오류가 발생했습니다"),
}
```

---

## 4. 보안 설계 상세

### 4.1 인증 흐름

```
사용자 ──→ Next.js ──→ /api/auth/login ──→ 선문대 SSO (SAML 2.0 or OAuth 2.0)
                                                     │
                                                     ▼
                                              IDP 인증 성공
                                                     │
                                                     ▼
                                          Callback: /api/auth/callback
                                                     │
                                          사번/학번 + 역할 수신
                                                     │
                                                     ▼
                                          JWT 발급 (Access + Refresh)
                                                     │
                                          ┌──────────┴──────────┐
                                          │                     │
                                    Access Token           Refresh Token
                                    30분 수명              7일 수명
                                    Authorization          httpOnly cookie
                                    header 전달            Secure, SameSite=Strict

JWT Payload:
{
  "sub": "20260001",          // 사번 또는 학번
  "role": "FACULTY",          // FACULTY | STUDENT | ADMIN
  "name": "홍길동",
  "dept": "인사총무팀",
  "iat": 1710900000,
  "exp": 1710901800           // +30분
}
```

### 4.2 민감정보 보호 정책

```
┌─────────────────────────────────────────────────────┐
│ 절대 하면 안 되는 것 (Security Anti-Patterns)        │
├─────────────────────────────────────────────────────┤
│ ✗ 프론트엔드 코드에 credential 하드코딩              │
│   → Gemini 초안의 welfareData JSON이 정확히 이것     │
│                                                      │
│ ✗ localStorage에 복호화된 credential 캐싱            │
│   → XSS 공격 시 즉시 유출                            │
│                                                      │
│ ✗ API 응답에 Cache-Control 미설정                     │
│   → 브라우저/CDN 캐시에 credential 잔류              │
│                                                      │
│ ✗ AES ECB 모드 사용                                   │
│   → 동일 평문이 동일 암호문 생성 (패턴 노출)         │
│                                                      │
│ ✗ 암호화 키를 DB나 Git에 저장                         │
│   → 환경변수 또는 KMS 필수                           │
│                                                      │
│ ✗ 복호화된 값을 로그에 기록                           │
│   → 로그 수집 시스템에서 credential 유출              │
│                                                      │
│ ✗ 관리자 URL만 숨겨서 보안 처리                       │
│   → API 레벨 role 검증 필수                          │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ 반드시 해야 하는 것 (Best Practices)                 │
├─────────────────────────────────────────────────────┤
│ ✓ credential API 응답 헤더:                          │
│   Cache-Control: no-store, no-cache, must-revalidate │
│   Pragma: no-cache                                   │
│   X-Content-Type-Options: nosniff                    │
│                                                      │
│ ✓ credential 조회마다 audit_log 기록                 │
│   who(사번) + when + what(기관ID) + IP               │
│                                                      │
│ ✓ Rate Limit: 10회/시간/사용자 (Redis 기반)          │
│   초과 시 429 + Retry-After + 관리자 알림            │
│                                                      │
│ ✓ AES-256-GCM (인증 암호화)                          │
│   12바이트 IV + 16바이트 Tag, IV 절대 재사용 금지    │
│                                                      │
│ ✓ 프론트엔드: credential을 state에만 보관,           │
│   컴포넌트 unmount 시 즉시 클리어                    │
│   (useEffect cleanup + state null 설정)              │
│                                                      │
│ ✓ 전송: HTTPS TLS 1.3 only (HSTS 헤더 포함)         │
│                                                      │
│ ✓ DB: credential 테이블에 RLS 적용 (권장)            │
│   또는 application-level role 체크 이중화            │
└─────────────────────────────────────────────────────┘
```

### 4.3 감사 정책

```
감사 대상:
1. institution_credentials READ → 항상 기록
2. welfare_institutions INSERT/UPDATE/DELETE → 항상 기록  
3. JWT 발급 → 항상 기록

보관 기간: 최소 1년

리포트:
- 월 1회 인사총무팀에 credential 접근 통계 발송 (선택)
- 비정상 패턴 감지: 
  - 동일 사용자가 1시간 내 모든 credential 기관 조회 → 알림
  - 근무시간 외(22:00~06:00) credential 접근 → 알림
```

---

## 5. 데이터 정규화 파이프라인

### 5.1 PPT 원본 데이터 품질 리포트

```
총 기관 수: 73개 (PPT 슬라이드 3~75)
정상 레코드: 68개 (93.2%)
문제 레코드: 5개 (6.8%)

문제 유형별:
- 주소 누락: 4건 (YGOON, 선문베네핏몰, 일성콘도, 현대옥)
- 전화번호 누락: 3건 (롯데시네마, 선문베네핏몰, 현대옥)
- 혜택 누락: 2건 (YGOON, 선문베네핏몰)
- 웹사이트 누락: 17건
- 주소에 전화번호 혼입: 18건 → 자동 클리닝 + 수동 보정 완료

특이 데이터:
- 일성콘도: 주소 = "전국 다수" (다중 지점)
- 서울메디컬: 의료복지플랫폼 (160개 병원 중개)
- 강남밝은성모안과: 강남점/부산점 2개 지점 동시 기재
- 천안ON호텔그룹: 3개 지점 통합 기재
- 용평리조트: PPT 원본 주소가 보령시 (실제 평창 → 확인 필요)

카테고리 분포:
  치과: 16  |  일반병원: 10  |  안과: 8  |  호텔: 8
  리조트: 6 |  장례: 4       |  음식: 3  |  기타: 18
```

### 5.2 Geocoding 검증 전략

```python
# services/geocoding_service.py

"""
카카오 로컬 API로 주소 → 좌표 변환.
!! 반드시 검증해야 할 케이스 !!
1. "전국 다수" → geocoding 불가, geo_verified=False 유지
2. "천안시 불당동 (3곳)" → 대표 지점 1개만 geocoding
3. 강남/부산 복수 지점 → 기관 레코드 분리 검토
4. 지번주소만 있는 경우 → 도로명 변환 후 geocoding
"""

KAKAO_GEOCODING_URL = "https://dapi.kakao.com/v2/local/search/address.json"

class GeocodingService:
    def __init__(self, api_key: str):
        self.api_key = api_key
    
    async def geocode(self, address: str) -> dict | None:
        """
        Returns: {"lat": float, "lng": float, "road_address": str} or None
        """
        # 특수 케이스 처리
        if any(kw in address for kw in ["전국", "다수", "확인 필요"]):
            return None
        
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                KAKAO_GEOCODING_URL,
                params={"query": address},
                headers={"Authorization": f"KakaoAK {self.api_key}"}
            )
            data = resp.json()
            
            if not data.get("documents"):
                return None
            
            doc = data["documents"][0]
            return {
                "lat": float(doc["y"]),
                "lng": float(doc["x"]),
                "road_address": doc.get("road_address", {}).get("address_name"),
            }
    
    async def batch_geocode(self, institutions: list) -> dict:
        """
        전수 geocoding + 결과 리포트.
        Rate limit: 카카오 API 30회/초 → 100ms 간격 호출.
        """
        results = {"success": 0, "failed": 0, "skipped": 0, "details": []}
        
        for inst in institutions:
            addr = inst.get("road_address") or inst.get("address", "")
            if not addr or "전국" in addr:
                results["skipped"] += 1
                continue
            
            geo = await self.geocode(addr)
            if geo:
                results["success"] += 1
                results["details"].append({
                    "name": inst["name"],
                    "address": addr,
                    "lat": geo["lat"],
                    "lng": geo["lng"],
                })
            else:
                results["failed"] += 1
                results["details"].append({
                    "name": inst["name"],
                    "address": addr,
                    "error": "geocoding failed",
                })
            
            await asyncio.sleep(0.1)  # Rate limit 준수
        
        return results
```

---

## 6. 프론트엔드 설계

### 6.1 프로젝트 구조

```
frontend/
├── app/
│   ├── (public)/
│   │   ├── page.tsx                    # 메인 대시보드
│   │   ├── institution/[id]/page.tsx   # 상세 페이지
│   │   └── layout.tsx
│   ├── (admin)/
│   │   └── admin/
│   │       ├── layout.tsx              # 사이드바 레이아웃
│   │       ├── institutions/page.tsx   # CRUD 테이블
│   │       └── audit/page.tsx          # 감사 로그
│   ├── api/
│   │   └── auth/[...nextauth]/route.ts
│   ├── layout.tsx                      # Pretendard + 테마
│   └── globals.css
├── components/
│   ├── ui/                             # shadcn/ui
│   ├── benefit-card.tsx
│   ├── category-chips.tsx
│   ├── target-toggle.tsx
│   ├── search-bar.tsx
│   ├── credential-dialog.tsx           # 민감정보 모달
│   ├── distance-badge.tsx
│   └── empty-state.tsx
├── hooks/
│   ├── use-institutions.ts             # SWR/React Query
│   ├── use-debounce.ts
│   └── use-geolocation.ts
├── lib/
│   ├── api.ts                          # fetch 래퍼
│   ├── constants.ts                    # 카테고리/타겟 라벨 매핑
│   ├── kakao-map.ts
│   └── utils.ts
└── tailwind.config.ts
```

### 6.2 Tailwind 컬러 시스템

```typescript
// tailwind.config.ts
const config = {
  theme: {
    extend: {
      colors: {
        // 선문대 브랜드 (PPT 원본 색상 기반)
        brand: {
          navy:    '#1B3A6B',   // 메인 네이비 (PPT 헤더 색)
          gold:    '#D4A843',   // 골드 액센트
          teal:    '#2A7B88',   // 숙박 카테고리
          sage:    '#5A7A5A',   // 장례 카테고리
          coral:   '#C45C3E',   // 경고/삭제
        },
        surface: {
          DEFAULT: '#F7F8FA',
          card:    '#FFFFFF',
          muted:   '#F1F3F5',
        }
      },
      fontFamily: {
        sans: ['Pretendard Variable', 'Pretendard', '-apple-system', 'sans-serif'],
      }
    }
  }
}
```

### 6.3 URL 동기화 (공유 가능한 필터 상태)

```typescript
// app/(public)/page.tsx — 서버 컴포넌트

interface SearchParams {
  category?: string;      // "병원" | "숙박" | "장례" | "기타" | 세부 ENUM
  target?: string;        // "FACULTY" | "STUDENT" | "FOREIGN_STUDENT"
  q?: string;             // 검색어
  sort?: string;          // "distance" | "name" | "sort_order"
  page?: string;
}

export default async function WelfarePage({ 
  searchParams 
}: { 
  searchParams: Promise<SearchParams> 
}) {
  const params = await searchParams;
  
  // URL → API 파라미터 매핑
  // 예: /welfare?category=병원&target=STUDENT&q=임플란트
  // → GET /api/v1/institutions?category_group=병원&target=STUDENT&q=임플란트
}
```

---

## 7. v0.dev 최종 프롬프트

### 프롬프트 1 — 메인 대시보드

```
Build a mobile-first Next.js 15 App Router page for "선문대 복지협약 안내" using 
shadcn/ui components and Tailwind CSS with Lucide React icons. 
The font is Pretendard (Korean) loaded from CDN.

DESIGN SYSTEM:
- Primary navy: #1B3A6B
- Gold accent: #D4A843  
- Background: #F7F8FA
- Cards: white with border-gray-100 and subtle shadow-sm
- Active chips: navy bg, white text
- Inactive chips: white bg, gray-200 border

LAYOUT (mobile-first, max-w-6xl center):

1. STICKY HEADER (bg-white border-b shadow-sm):
   Left: 40x40 rounded-lg navy square with white "S" letter + "선문대 복지협약" 
   text-lg bold + "인사총무팀 | 2026" text-xs gray-500.
   Right: Search icon button. On click, expands to full-width input 
   with 300ms debounce. Placeholder: "기관명, 혜택, 주소 검색..."
   On mobile, search expands below header. On desktop, inline right side.

2. CATEGORY CHIPS (horizontal scroll, no scrollbar, gradient mask edges):
   "전체 (73)", "병원 (42)", "숙박 (18)", "장례 (4)", "기타 (9)"
   Each chip: rounded-full, Lucide icon (Building2, HeartPulse, BedDouble, 
   Flower2, Package). Active = navy bg white text shadow-md. 
   Min touch target 44px height.

3. TARGET TOGGLE (shadcn ToggleGroup variant="outline"):
   "전체 보기" | "학생 혜택" | "교직원 전용"
   Active segment: filled with respective color.

4. RESULTS BAR: 
   "총 {n}개 협약기관" text-sm gray-500, bold blue count number.
   Right side: sort dropdown (기본순 | 이름순 | 거리순).

5. CARD GRID: 1col mobile, 2col md, 3col lg. gap-4 md:gap-6.
   Cards are BenefitCard components (see Prompt 2).

6. EMPTY STATE (when filter returns 0):
   Centered, Search icon 48px gray-300, "검색 결과가 없습니다" text-lg,
   "다른 카테고리나 검색어를 시도해보세요" text-sm gray-500.

STATE MANAGEMENT:
- All filters sync to URL searchParams: ?category=병원&target=STUDENT&q=치과
- Filters update URL without page reload (useRouter + shallow)
- Category counts update dynamically based on current target filter

CRITICAL:
- Chip bar must NOT cause horizontal page scroll
- All interactive elements min 44x44px touch target
- Search filters name + benefit + address simultaneously
- Korean font rendering: Pretendard with system fallback
```

### 프롬프트 2 — BenefitCard 컴포넌트

```
Build a "BenefitCard" component using shadcn/ui Card + Badge + Button 
with Tailwind CSS and Lucide React. TypeScript strict mode.

PROPS:
{
  name: string
  category: string           // "HOSPITAL_DENTAL" enum value
  categoryLabel: string      // "병원/치과" display label
  categoryGroup: string      // "병원" | "숙박" | "장례" | "기타"
  targets: Array<{type: string, label: string}>  
  benefitSummary: string     // may contain newlines
  reservationMethod?: string
  phone: string
  website?: string
  roadAddress?: string
  hasCredentials: boolean
  distanceKm?: number        // only when geolocation active
}

VISUAL STRUCTURE:

Top row (flex justify-between):
  Left: Category badge — 
    병원: bg-blue-50 text-blue-700 with HeartPulse icon
    숙박: bg-teal-50 text-teal-700 with BedDouble icon  
    장례: bg-gray-100 text-gray-600 with Flower2 icon
    기타: bg-amber-50 text-amber-700 with Package icon
  Right: Target badge —
    If includes STUDENT: bg-green-50 text-green-700 "학생포함" with GraduationCap
    If FACULTY only: bg-orange-50 text-orange-700 "교직원전용" with UserCheck
    If FOREIGN_STUDENT: bg-purple-50 text-purple-700 "유학생" with Globe

If distanceKm provided: small "📍 3.2km" badge next to category.

Title: text-lg font-bold text-gray-900 line-clamp-1 with title tooltip on hover.

Benefit box: bg-blue-50/60 border border-blue-100 rounded-lg p-3.
  Text: text-sm font-medium text-blue-800 whitespace-pre-line.

Info section (text-sm text-gray-500, collapsible):
  - MapPin icon + address (line-clamp-1, expand on click)
  - Info icon + reservation method (only show if exists)

If hasCredentials: 
  Banner above buttons: bg-amber-50 border border-amber-200 rounded-lg p-2
  Lock icon + "교직원 로그인 후 예약정보 확인" text-xs amber-700

Action buttons (mt-auto, grid):
  If website exists: grid-cols-3, else grid-cols-2
  Each button: bg-gray-50 hover:bg-gray-100 border border-gray-200 
  rounded-xl py-2.5 flex items-center justify-center gap-1.5
  - "전화" with Phone icon (text-blue-600) → tel:{phone with hyphens removed}
  - "길찾기" with MapPin icon (text-red-500) → kakaomap deep link
  - "웹사이트" with Globe icon (text-green-600) → external link, 
    rel="noopener noreferrer" target="_blank"

CARD CONTAINER:
  bg-white rounded-2xl border border-gray-100 shadow-sm 
  hover:shadow-md transition-shadow
  p-5 flex flex-col h-full
  Benefit area: flex-grow (for grid row alignment)
  Buttons: mt-auto

CRITICAL:
  - h-full + flex-col + flex-grow on benefit area = grid alignment
  - tel: link must strip ALL hyphens from phone
  - External links MUST have rel="noopener noreferrer"
  - Touch targets minimum 44px height
  - line-clamp on name and address to prevent card height explosion
  - hasCredentials banner must NOT push buttons out of view
```

### 프롬프트 3 — Credential 모달 (교직원 전용)

```
Build a "CredentialDialog" using shadcn/ui Dialog + Alert + Button.
This component shows sensitive login credentials for institutional 
booking systems. Security is critical.

TRIGGER: "예약정보 확인" button on BenefitCard (only visible when 
hasCredentials=true AND user is logged in as FACULTY).

DIALOG CONTENT:
  Header: Lock icon + "예약 로그인 정보" title
  
  Alert (variant="warning" bg-amber-50):
    "이 정보는 선문대학교 교직원 전용입니다. 
     외부 유출 시 서비스 이용이 제한될 수 있습니다."
  
  Info card (bg-gray-50 rounded-lg p-4 space-y-3):
    - "예약 사이트" → clickable URL (external link icon)
    - "아이디" → text with copy button (Clipboard icon)
    - "비밀번호" → masked by default (••••••), 
      toggle eye icon to show/hide. Copy button.
    - "이용 방법" → reservation note text
  
  Footer: "확인" close button

SECURITY REQUIREMENTS:
  - Fetch credential from API only when dialog opens (not on page load)
  - Show loading skeleton while fetching
  - On dialog close: immediately clear credential state (set to null)
  - useEffect cleanup: also clear on unmount
  - Never store credential in localStorage/sessionStorage
  - Auto-close dialog after 3 minutes (timeout with countdown text)
  - Show "접근이 기록됩니다" notice at bottom

ERROR STATES:
  - 401: "로그인이 필요합니다" + login redirect button
  - 403: "교직원만 조회할 수 있습니다"
  - 429: "조회 횟수를 초과했습니다. {retry_after}초 후 다시 시도해주세요"
  - Network error: retry button
```

---

## 8. 구현 체크리스트

### Phase 1 — MVP (2주)

- [ ] PPT 73개 기관 전수 데이터 JSON 완성 (본 문서 별첨)
- [ ] PostgreSQL 스키마 생성 + seed data INSERT
- [ ] 카카오 Geocoding으로 좌표 일괄 취득 + 검증 리포트
- [ ] Next.js 프론트엔드: 정적 JSON 기반 카드 그리드
- [ ] 카테고리 필터 + 타겟 토글 + 검색 동작
- [ ] URL searchParams 동기화
- [ ] 모바일 반응형 테스트 (iOS Safari, Android Chrome)

### Phase 2 — 백엔드 연동 (2주)

- [ ] FastAPI 서버 구축 (api/services/repositories 3계층)
- [ ] GET /institutions + /categories 엔드포인트
- [ ] pg_trgm 검색 + Haversine 거리 정렬
- [ ] 선문대 SSO 연동 협의 (IDP 프로토콜 확인)
- [ ] JWT 발급/검증 미들웨어
- [ ] credential 암복호화 (AES-256-GCM)
- [ ] GET /institutions/{id}/credentials + 감사 로그
- [ ] Rate limiting (Redis)

### Phase 3 — 관리자 (1주)

- [ ] 관리자 CRUD 엔드포인트
- [ ] DataTable + Form 관리 페이지
- [ ] CSV 일괄 등록
- [ ] 감사 로그 조회 페이지
- [ ] 변경 사유 기록

### Phase 4 — 고도화 (선택)

- [ ] PWA manifest + service worker (오프라인 캐시)
- [ ] 카카오맵 연동 (길찾기 deep link → 지도 인라인 표시)
- [ ] 위치 기반 "가까운 기관" 자동 정렬
- [ ] 푸시 알림 (신규 협약 추가 시)
- [ ] 다국어 (외국인 유학생용 영어/중국어/베트남어)

---

## 별첨: 데이터 통계

```
카테고리별:
  HOSPITAL_DENTAL:      16개 (21.9%)
  HOSPITAL_GENERAL:     10개 (13.7%)
  HOSPITAL_OPHTH:        8개 (11.0%)
  ACCOMMODATION_HOTEL:   8개 (11.0%)
  ACCOMMODATION_RESORT:  6개 (8.2%)
  FUNERAL:               4개 (5.5%)
  FOOD:                  3개 (4.1%)
  기타(7개 카테고리):    18개 (24.7%)

대상자별 (중복 포함):
  FACULTY:              70개 (95.9%) — 거의 모든 기관
  STUDENT:              17개 (23.3%) — 학생 이용 가능
  FACULTY_FAMILY:       12개 (16.4%)
  ALL_UNIVERSITY:        2개 (2.7%)
  ACQUAINTANCE:          2개 (2.7%)
  TRAINEE:               1개 (1.4%)
  FOREIGN_STUDENT:       1개 (1.4%)

민감정보 보유:
  일성콘도:       법인예약 ID/PW
  하이원리조트:   전용 예약사이트 ID/PW
  파라다이스스파: 전용 예약사이트 URL
```
