# AI Doctor 서비스 — 백엔드 CLAUDE.md

## 프로젝트 개요

의료기록의 개인화를 통한 AI 닥터 서비스. 사용자의 건강 데이터(증상, 병력)와 처방 데이터를 LLM에 전달하여 건강 상태 분석, 질병 예측, 식이·운동 추천을 제공하는 Spring Boot 백엔드.

---

## 핵심 원칙

### ERD 우선 원칙
**ERD.md가 유일한 진실(source of truth)이다.** 개별 도메인 명세서(docs/)는 작성자가 다르고 일관성이 없으므로 ERD와 충돌하면 ERD를 따른다.

### SOLID 적용 기준
- **S/O/L/I**: 원칙대로 적용한다.
- **D (의존성 역전)**: 인터페이스를 사용하되, **재사용 가능성이 없는 단일 구현체 전용 인터페이스는 만들지 않는다.**  
  - `HealthService` 인터페이스: 유지 (LLM 연동 등 교체 가능성 있음)  
  - `AiDatasetService`: 인터페이스 없는 구현체 직접 사용 (단일 구현, 재사용 불필요)

---

## ERD (Ground Truth)

```
users
├── user_id     BIGINT PK AUTO_INCREMENT
├── login_id    VARCHAR(50) UNIQUE NOT NULL
├── password    VARCHAR(255) NOT NULL
├── name        VARCHAR(50) NOT NULL
├── email       VARCHAR(100) UNIQUE NOT NULL
├── phone       VARCHAR(20)
├── birth_date  DATE NOT NULL
├── gender      CHAR(1)
├── status      VARCHAR(10) NOT NULL DEFAULT 'ACTIVE'
├── created_at  DATETIME NOT NULL DEFAULT now()
└── updated_at  DATETIME

refresh_token
├── token_id    BIGINT PK AUTO_INCREMENT
├── user_id     BIGINT FK → users.user_id NOT NULL
├── token_hash  VARCHAR(255) UNIQUE NOT NULL
├── expires_at  DATETIME NOT NULL
├── revoked     BOOLEAN NOT NULL DEFAULT false
└── created_at  DATETIME NOT NULL DEFAULT now()

health
├── health_id   BIGINT PK AUTO_INCREMENT
├── user_id     BIGINT FK → users.user_id NOT NULL
├── symptom     TEXT NOT NULL
├── history     TEXT
├── note        TEXT
├── record_date DATE NOT NULL
├── created_at  DATETIME NOT NULL DEFAULT now()
└── updated_at  DATETIME

prescription
├── prescription_id   BIGINT PK AUTO_INCREMENT
├── user_id           BIGINT FK → users.user_id NOT NULL
├── prescription_date DATE NOT NULL
├── hospital_name     VARCHAR(100) NOT NULL
├── status            VARCHAR(10) NOT NULL DEFAULT 'ACTIVE'
├── created_at        DATETIME NOT NULL DEFAULT now()
└── updated_at        DATETIME

prescription_detail
├── detail_id       BIGINT PK AUTO_INCREMENT
├── prescription_id BIGINT FK → prescription.prescription_id NOT NULL
├── medicine_name   VARCHAR(100) NOT NULL
├── dosage          VARCHAR(50) NOT NULL
├── duration        VARCHAR(50) NOT NULL
├── created_at      DATETIME NOT NULL DEFAULT now()
└── updated_at      DATETIME

ai_result
├── result_id             INT PK AUTO_INCREMENT
├── user_id               BIGINT FK → users.user_id NOT NULL
├── health_status         VARCHAR(20) NOT NULL   -- GOOD | CAUTION | WARNING
├── summary_note          TEXT NOT NULL
├── potential_diseases    JSON
├── recommended_foods     JSON
├── recommended_exercises JSON
├── precautions           TEXT
├── analysis_date         DATETIME NOT NULL
└── raw_llm_response      JSON
```

### 관계 요약
- `users` 1:N `health`
- `users` 1:N `refresh_token`
- `users` 1:N `prescription`
- `prescription` 1:N `prescription_detail`
- `users` 1:N `ai_result`

---

## 도메인 구조

### 패키지 레이아웃
```
com.swe.backend
├── domain
│   ├── auth          (인증 — 회원가입/로그인/로그아웃/토큰)
│   ├── member        (회원 정보 수정/조회/탈퇴/아이디·비밀번호 찾기)
│   ├── health        (건강 기록 CRUD)
│   ├── prescription  (처방전 CRUD)
│   └── ai            (LLM 분석 트리거 및 결과 조회)
└── global
    ├── config        (SecurityConfig, JwtConfig 등)
    ├── exception     (GlobalExceptionHandler)
    └── response      (ApiResponse<T>)
```

각 도메인 내부 구조: `controller / service / repository / entity / dto / exception`

---

## 인증 흐름

1. 로그인 성공 → **Access Token** (Authorization Header) + **Refresh Token** (HttpOnly Cookie) 발급
2. Refresh Token 원본은 클라이언트 Cookie에만 존재; DB에는 **해시값**만 저장
3. Access Token 만료 → `POST /api/member/token/refresh` (Refresh Cookie 전달) → 새 Access Token 발급
4. 로그아웃/탈퇴/비밀번호 재설정 → 해당 사용자의 모든 Refresh Token `revoked = true`
5. 모든 보호 API: `Authorization: Bearer {accessToken}` 헤더 필수

### 인증이 불필요한 엔드포인트
`/api/member/signup`, `/api/member/login`, `/api/member/find-id`, `/api/member/reset-password`

---

## API 엔드포인트 목록

### Auth / Member
| 메서드 | URL | 인증 | 설명 |
|--------|-----|------|------|
| POST | /api/member/signup | 불필요 | 회원가입 |
| POST | /api/member/login | 불필요 | 로그인 (AccessToken + RefreshCookie 반환) |
| POST | /api/member/logout | AccessToken | 로그아웃 (RefreshToken 폐기) |
| POST | /api/member/token/refresh | RefreshCookie | AccessToken 재발급 |
| GET | /api/member/me | AccessToken | 내 정보 조회 |
| PUT | /api/member/me | AccessToken | 내 정보 수정 (name, phone, password) |
| POST | /api/member/me/verify-password | AccessToken | 본인 확인 |
| DELETE | /api/member/me | AccessToken | 회원 탈퇴 (status → WITHDRAWN) |
| POST | /api/member/find-id | 불필요 | 아이디 찾기 (name+birthDate) |
| POST | /api/member/reset-password | 불필요 | 비밀번호 재설정 |

### Health
| 메서드 | URL | 설명 |
|--------|-----|------|
| POST | /api/health/create | 건강 기록 생성 |
| GET | /api/health/list | 목록 조회 (record_date DESC) |
| GET | /api/health/{healthId} | 단건 조회 |
| PUT | /api/health/update/{healthId} | 수정 |
| DELETE | /api/health/delete/{healthId} | 삭제 |

### Prescription
| 메서드 | URL | 설명 |
|--------|-----|------|
| POST | /api/prescription/create | 처방전 + 처방약 생성 |
| GET | /api/prescription/list | 목록 조회 |
| GET | /api/prescription/{prescription_id} | 상세 조회 |
| PUT | /api/prescription/update/{prescription_id} | 수정 (처방약 재등록 방식) |
| DELETE | /api/prescription/delete/{prescription_id} | Soft Delete (status → DELETED) |

### AI Result
| 메서드 | URL | 권한 | 설명 |
|--------|-----|------|------|
| POST | /api/ai/trigger | ADMIN | AI 분석 수동 실행 |
| GET | /api/ai/result/{userId} | 본인/ADMIN | 최신 분석 결과 전체 |
| GET | /api/ai/result/{userId}/summary | 본인/ADMIN | health_status + summary_note |
| GET | /api/ai/result/{userId}/diseases | 본인/ADMIN | potential_diseases |
| GET | /api/ai/result/{userId}/foods | 본인/ADMIN | recommended_foods |
| GET | /api/ai/result/{userId}/exercises | 본인/ADMIN | recommended_exercises |
| GET | /api/ai/result/{userId}/precautions | 본인/ADMIN | precautions |

---

## 공통 응답 형식

```json
{ "status": 200, "message": "...", "data": { ... } }
```

Health 도메인은 `"status": "success" | "error"` 문자열을 사용하는 명세가 있으나, ERD 기반 구현 시 전체 통일을 권장한다.

---

## 핵심 비즈니스 규칙

### Member / Auth
- `status` Enum: `ACTIVE` / `LOCKED` / `WITHDRAWN`
- 탈퇴 시 `login_id`, `email`을 마스킹 치환 → UNIQUE 제약 충돌 방지
- 비밀번호는 반드시 해시 저장 (BCrypt)
- 로그인 실패와 존재하지 않는 계정 모두 동일한 예외로 처리 (정보 노출 방지)

### Prescription
- Soft Delete: 실제 삭제 없이 `status = DELETED`
- 처방약 수정은 기존 `prescription_detail` 삭제 후 재삽입
- 조회 시 `status = ACTIVE`만 반환

### Health
- `symptom`, `record_date`는 필수 / `history`, `note`는 선택
- 소유권 검증: `findByIdAndUserId` — 존재하지 않거나 타인 소유면 동일하게 404 반환 (보안)
- 목록 조회: `record_date DESC` 정렬

### AI Result
- 분석 트리거는 동기 처리 (MVP 기준, 비동기 미적용)
- `health`, `prescription` 데이터가 없으면 트리거 시 400 반환
- LLM 호출 실패: Exponential Backoff 재시도 3회 (1s → 2s → 4s), 전부 실패 시 예외 전파
- `raw_llm_response`는 디버깅용, 클라이언트에 미노출
- `health_status` 값: `GOOD` / `CAUTION` / `WARNING`

---

## AI 분석 파이프라인

```
POST /api/ai/trigger (userId)
        │
        ▼
AiDomainServiceImpl.triggerAnalysis(userId)
        │
        ├─1─▶ AiDatasetService.buildDataset(userId)
        │         ├── HealthRepository → health CSV
        │         └── PrescriptionRepository + PrescriptionDetailRepository → prescription CSV
        │         └── 반환: AiResultRequestDTO { userId, healthCsv, prescriptionCsv }
        │
        ├─2─▶ AiResultService.analyze(AiResultRequestDTO)  ← 인터페이스 (LLM 교체 가능)
        │         └── AiResultServiceImpl: 시스템 프롬프트 + 유저 프롬프트 조합 → LLM API 호출
        │         └── 응답 JSON 파싱 → AiResultResponseDTO
        │
        └─3─▶ AiResultRepository.save(AiResultEntity)
                  └── ai_result 테이블 저장
```

### LLM 프롬프트 구조
- **시스템**: 순수 JSON만 출력, 배열 최대 5개, health_status 3가지 값만 허용
- **유저**: `{healthCsv}` + `{prescriptionCsv}` 삽입 후 분석 요청

---

## 클래스 설계 (다이어그램용 기반 정보)

### 레이어 의존 방향
```
Controller → Service(interface) → Repository(interface)
                                        ↓
                                     Entity
```

### 주요 클래스 목록

| 도메인 | Controller | Service Interface | ServiceImpl | Repository |
|--------|-----------|-------------------|-------------|------------|
| auth/member | MemberController | MemberService | MemberServiceImpl | UserRepository, RefreshTokenRepository |
| health | HealthController | HealthService | HealthServiceImpl | HealthRepository |
| prescription | PrescriptionController | PrescriptionService | PrescriptionServiceImpl | PrescriptionRepository, PrescriptionDetailRepository |
| ai | AiController | AiDomainService, AiResultService | AiDomainServiceImpl, AiResultServiceImpl | AiResultRepository |
| ai dataset | — | *(없음)* | AiDatasetService | (HealthRepo, PrescriptionRepo 직접 의존) |

### Entity 목록
`User`, `RefreshToken`, `Health`, `Prescription`, `PrescriptionDetail`, `AiResultEntity`

### 글로벌 컴포넌트
- `JwtTokenProvider` — 토큰 생성/파싱/검증
- `SecurityConfig` — Filter Chain, 인증 불필요 경로 설정
- `GlobalExceptionHandler` (`@RestControllerAdvice`) — 모든 예외를 `ApiResponse` 형태로 통일
- `ApiResponse<T>` — 공통 응답 래퍼

---

## 예외 설계

| 예외 클래스 | HTTP | 발생 조건 |
|------------|------|----------|
| `UserNotFoundException` | 404 | 존재하지 않거나 탈퇴한 사용자 |
| `DuplicateLoginIdException` | 409 | login_id 중복 |
| `DuplicateEmailException` | 409 | email 중복 |
| `InvalidPasswordException` | 401 | 비밀번호 불일치 또는 아이디 미존재 |
| `AccountLockedException` | 423 | 잠금 상태 계정 |
| `HealthNotFoundException` | 404 | 건강 기록 없음 또는 타인 소유 |
| `PrescriptionNotFoundException` | 404 | 처방전 없음 또는 타인 소유 |
| `UnauthorizedException` | 401 | 토큰 없음/만료 |
| `ForbiddenException` | 403 | 권한 없는 접근 |

---

## UserStatus Enum

```java
ACTIVE    // 정상 계정 (기본값)
LOCKED    // 잠금 상태
WITHDRAWN // 탈퇴 상태
```

---

## 시퀀스 다이어그램 핵심 흐름 (작성 참고용)

### 로그인 흐름
```
Client → MemberController.login(loginId, password)
       → MemberServiceImpl: 사용자 조회 → 상태 확인 → 비밀번호 검증
       → JwtTokenProvider: AccessToken 생성 + RefreshToken 생성
       → RefreshTokenRepository: token_hash 저장
       → Response: body(accessToken, name) + Cookie(refreshToken HttpOnly)
```

### AI 분석 트리거 흐름
```
Admin → AiController.trigger(userId)
      → AiDomainServiceImpl.triggerAnalysis(userId)
      → AiDatasetService: health CSV + prescription CSV 생성
      → AiResultServiceImpl: LLM API 호출 (재시도 3회)
      → AiResultRepository: ai_result 저장
      → Response: AiResultResponseDTO
```

### 토큰 재발급 흐름
```
Client → MemberController.refresh (Cookie: refreshToken)
       → MemberServiceImpl: token_hash 조회 → revoked/expires_at 검증
       → JwtTokenProvider: 새 AccessToken 생성
       → Response: body(accessToken) + 새 RefreshCookie
```

---

## 유스케이스 요약 (UC 번호는 member.md 기준)

| UC | 기능 | 관련 API |
|----|------|---------|
| UC-A01 | 로그인/토큰 재발급 | POST /login, POST /token/refresh |
| UC-A02 | 로그아웃 | POST /logout |
| UC-A03 | 회원가입 | POST /signup |
| UC-A04 | 정보 조회 | GET /me |
| UC-A05 | 정보 수정 | POST /me/verify-password → PUT /me |
| UC-A06 | 회원 탈퇴 | POST /me/verify-password → DELETE /me |
| UC-A07 | 아이디/비밀번호 찾기 | POST /find-id, POST /reset-password |
| UC-H01 | 건강 기록 관리 | POST/GET/PUT/DELETE /api/health/* |
| UC-P01 | 처방전 관리 | POST/GET/PUT/DELETE /api/prescription/* |
| UC-AI01 | AI 건강 분석 | POST /api/ai/trigger, GET /api/ai/result/* |
