# Member (회원 정보 관리) 도메인 — 파트 명세서

> **프로젝트:** 의료기록의 개인화를 통한 AI 닥터 서비스
> **도메인:** Member (회원 정보 관리)

---

## 1. 도메인 개요

회원 가입, 로그인/로그아웃, 회원 정보 조회, 수정, 탈퇴, 아이디/비밀번호 찾기 등 사용자 인증 및 계정 관리 전반을 담당

---

## 2. ERD 기반 테이블 구조

### 2.1 users 테이블

| 컬럼명 | 타입 | 제약조건 | 설명 |
|---|---|---|---|
| user_id | BIGINT | PK, AUTO_INCREMENT | 회원 고유 식별자 |
| login_id | VARCHAR(50) | UNIQUE, NOT NULL | 로그인 아이디 (이메일과 별도) |
| password | VARCHAR(255) | NOT NULL | 비밀번호 (해시 저장) |
| name | VARCHAR(50) | NOT NULL | 이름 |
| email | VARCHAR(100) | UNIQUE, NOT NULL | 이메일 |
| phone | VARCHAR(20) | NULLABLE | 전화번호 |
| birth_date | DATE | NOT NULL | 생년월일 |
| gender | CHAR(1) | NULLABLE | 성별 |
| status | VARCHAR(10) | NOT NULL, DEFAULT 'ACTIVE' | 계정 상태 |
| created_at | DATETIME | NOT NULL, DEFAULT NOW() | 가입일시 |
| updated_at | DATETIME | NULLABLE | 최종 수정일시 |

### 2.2 refresh_token 테이블

| 컬럼명 | 타입 | 제약조건 | 설명 |
|---|---|---|---|
| token_id | BIGINT | PK, AUTO_INCREMENT | 토큰 고유 ID |
| user_id | BIGINT | FK → users.user_id, NOT NULL | 회원 ID |
| token_hash | VARCHAR(255) | UNIQUE, NOT NULL | Refresh Token 해시 |
| expires_at | DATETIME | NOT NULL | 만료 시각 |
| revoked | BOOLEAN | NOT NULL, DEFAULT FALSE | 폐기 여부 |
| created_at | DATETIME | NOT NULL, DEFAULT NOW() | 생성일시 |


---

## 3. Controller 계층

#### MemberController.java

POST /api/member/signup
- `SignupRequest`를 입력받아 회원가입을 수행한다.
- 성공 시 `SignupResponse`를 반환한다.
- 실패 시 `DuplicateLoginIdException` 또는 `DuplicateEmailException`을 던진다.

POST /api/member/login
- `LoginRequest`(loginId, password)를 입력받아 인증을 수행한다.
- 성공 시 Access Token을 body로, Refresh Token을 HttpOnly Cookie로 반환한다.
- 실패 시 `InvalidPasswordException`을 던진다.
- 계정 잠금 상태면 `AccountLockedException`을 던진다.

POST /api/member/logout
- Access Token(Authorization 헤더)과 Refresh Token(Cookie)을 입력받는다.
- Refresh Token을 폐기 처리하고 정상 응답을 반환한다.

POST /api/member/token/refresh
- Refresh Cookie로 새 Access Token을 재발급한다.
- 만료/폐기된 Refresh Token이면 인증 실패 응답을 반환한다.

GET /api/member/me
- 인증된 사용자의 식별자를 기반으로 본인 정보를 조회하여 `MemberInfoResponse`를 반환한다.

PUT /api/member/me
- `MemberUpdateRequest`를 입력받아 회원 정보를 수정한다.
- 변경할 필드만 포함하며, 변경 가능 필드는 name, phone, password이다.

POST /api/member/me/verify-password
- `VerifyPasswordRequest`(현재 비밀번호)를 입력받아 본인 확인을 수행한다.
- 성공 시 정상 응답, 실패 시 `InvalidPasswordException`을 던진다.

DELETE /api/member/me
- `WithdrawRequest`(탈퇴 사유)를 입력받아 회원 탈퇴를 처리한다.
- 탈퇴 후 해당 회원의 모든 Refresh Token을 폐기한다.

POST /api/member/find-id
- `FindIdRequest`(name, email)를 입력받아 아이디 찾기를 수행한다.
- 성공 시 마스킹된 login_id를 `FindIdResponse`로 반환한다.

POST /api/member/find-password
- `FindPasswordRequest`(loginId, email)를 입력받아 비밀번호 재설정 링크를 메일로 발송한다.

POST /api/member/reset-password
- `ResetPasswordRequest`(token, newPassword)를 입력받아 비밀번호를 재설정한다.
- 토큰이 만료/무효하면 실패 응답을 반환한다.

---

## 4. Service 계층

#### MemberService.java (인터페이스)
- 컨트롤러는 이 인터페이스에만 의존한다.

#### MemberServiceImpl.java

**회원가입 — signup(SignupRequest)**
1. login_id 중복 여부를 확인한다. 중복 시 `DuplicateLoginIdException`을 던진다.
2. email 중복 여부를 확인한다. 중복 시 `DuplicateEmailException`을 던진다.
3. 비밀번호를 해시 처리하여 저장한다.
4. `status = ACTIVE`로 회원을 생성하고 `SignupResponse`를 반환한다.

**로그인 — login(LoginRequest)**
1. login_id로 사용자를 조회한다. 존재하지 않으면 `InvalidPasswordException`을 던진다.
2. 계정 상태가 잠금/탈퇴 상태인지 확인한다. 잠금 상태면 `AccountLockedException`을 던진다.
3. 비밀번호 해시를 비교한다. 불일치하면 `InvalidPasswordException`을 던진다.
4. 인증 성공 시 Access Token과 Refresh Token을 생성한다.
5. Refresh Token의 해시를 `refresh_token` 테이블에 저장한다.
6. `LoginResponse`(accessToken, name, role 정보)를 반환한다.

**로그아웃 — logout(accessToken, refreshToken)**
1. 해당 Refresh Token 레코드를 조회하여 `revoked = true`로 폐기 처리한다.

**토큰 재발급 — refresh(refreshToken)**
1. Refresh Token 해시로 DB에서 레코드를 조회한다.
2. 폐기 여부(`revoked`), 만료 시각(`expires_at`)을 검증한다.
3. 유효하면 새 Access Token을 발급하여 `TokenRefreshResponse`로 반환한다.

**내 정보 조회 — getMyInfo(userId)**
1. userId로 사용자를 조회한다. 탈퇴 상태인 경우 `UserNotFoundException`을 던진다.
2. `MemberInfoResponse`를 반환한다.

**내 정보 수정 — updateMyInfo(userId, MemberUpdateRequest)**
1. 변경 요청된 필드만 업데이트한다. (name, phone, password)
2. phone 변경 시 중복 여부를 확인한다.
3. password 변경 시 새 비밀번호를 해시 처리하여 저장한다.
4. `updated_at`을 현재 시각으로 갱신한다.

**본인 확인 — verifyPassword(userId, password)**
1. 현재 비밀번호 해시를 비교하여 일치 여부만 반환한다.
2. 불일치 시 `InvalidPasswordException`을 던진다.

**회원 탈퇴 — withdraw(userId, WithdrawRequest)**
1. 계정 상태를 'WITHDRAWN'으로 전환한다.
2. login_id, email을 마스킹 치환하여 UNIQUE 충돌을 회피한다.
3. 해당 회원의 모든 Refresh Token을 일괄 폐기(`revoked = true`)한다.

**아이디 찾기 — findId(FindIdRequest)**
1. name + email 조합으로 사용자를 조회한다.
2. 매칭되는 회원이 있으면 마스킹된 login_id를 `FindIdResponse`로 반환한다. (예: `user***`)
3. 매칭 실패 시 `UserNotFoundException`을 던진다.

**비밀번호 찾기 — sendPasswordResetLink(FindPasswordRequest)**
1. loginId + email로 사용자를 조회한다.
2. 재설정용 임시 토큰을 생성하고 메일로 발송한다.

**비밀번호 재설정 — resetPassword(ResetPasswordRequest)**
1. 토큰의 유효성과 만료 여부를 검증한다.
2. 새 비밀번호를 해시 처리하여 덮어쓴다.
3. 보안을 위해 해당 회원의 모든 Refresh Token을 일괄 폐기한다.

---

## 5. Entity 계층

#### User.java
- `users` 테이블과 매핑. PK는 `user_id` (IDENTITY 전략).
- `login_id`, `email` 각각 UNIQUE 제약.
- `status`는 Enum(`UserStatus`)으로 관리하며, `@Enumerated(EnumType.STRING)` 적용.
- `@PrePersist`로 `created_at` 자동 세팅 + `status = ACTIVE` 초기화.
- `@PreUpdate`로 `updated_at` 자동 갱신.
- `gender`는 NULLABLE — 선택 입력 항목.
- `phone`도 NULLABLE — ERD 기준 NOT NULL 제약 없음.

#### RefreshToken.java
- `refresh_token` 테이블과 매핑.
- `user_id`는 `User` 엔티티에 대한 FK (`@ManyToOne` 또는 단순 Long 참조).
- `token_hash` UNIQUE — 원본 토큰은 저장하지 않고 해시만 보관.
- `revoked = false`가 기본값. 로그아웃/탈퇴 시 `true`로 전환.

---

## 6. Repository 계층

#### UserRepository.java
- `findByLoginId(loginId)` → 로그인, 아이디 중복 확인
- `findByEmail(email)` → 이메일 중복 확인
- `findByNameAndEmail(name, email)` → 아이디 찾기
- `findByLoginIdAndEmail(loginId, email)` → 비밀번호 찾기
- `existsByLoginId(loginId)` → 가입 시 login_id 중복 검증
- `existsByEmail(email)` → 가입 시 email 중복 검증

#### RefreshTokenRepository.java
- `findByTokenHash(hash)` → 토큰 재발급/검증
- `deleteByUserId(userId)` → 탈퇴/비밀번호 재설정 시 전체 폐기

---

## 7. DTO 계층

#### Request DTO

**SignupRequest**
- loginId(필수), password(필수, 규칙 검증), name(필수), email(필수, 형식 검증), phone(선택), birthDate(필수, 과거 일자), gender(선택)

**LoginRequest**
- loginId(필수), password(필수)

**MemberUpdateRequest**
- name(선택), phone(선택), newPassword(선택) — 변경할 필드만 포함

**VerifyPasswordRequest**
- password(필수) — 본인 확인용 현재 비밀번호

**WithdrawRequest**
- reason(선택) — 탈퇴 사유

**FindIdRequest**
- name(필수), email(필수) — 이름+이메일로 login_id 찾기

**FindPasswordRequest**
- loginId(필수), email(필수) — 본인 확인 후 재설정 링크 발송

**ResetPasswordRequest**
- token(필수), newPassword(필수)

#### Response DTO

**SignupResponse**
- userId, loginId, message

**LoginResponse**
- accessToken, name — Refresh Token은 body가 아닌 Cookie로 전달

**MemberInfoResponse**
- userId, loginId, name, email, phone, birthDate, gender, createdAt, updatedAt

**FindIdResponse**
- maskedLoginId (예: `user***`)

**TokenRefreshResponse**
- accessToken

**ApiResponse\<T\> (공통 래퍼)**
- status(HTTP 상태 코드), message(결과 메시지), data(T)

---

## 8. JWT 토큰 설계

#### Access Token
- 단기 유효 토큰. 요청 시 `Authorization: Bearer {token}` 헤더로 전달.
- Claims에 사용자 식별자와 역할 정보를 포함한다.
- 만료 시 Refresh Token으로 재발급을 시도한다.

#### Refresh Token
- 장기 유효 토큰. HttpOnly + Secure Cookie로 전달 (XSS 방어).
- DB(`refresh_token` 테이블)에 해시만 저장, 원본은 클라이언트 Cookie에만 존재.
- 로그아웃/탈퇴/비밀번호 재설정 시 `revoked = true`로 폐기.

#### JwtTokenProvider.java (util)
- 토큰 생성, 파싱, 서명 검증을 담당한다.
- 비밀키는 설정 파일의 환경변수로 주입한다.

#### SecurityConfig.java (global.config)
- JWT 인증 필터를 Security Filter Chain에 삽입한다.
- 인증 불필요 엔드포인트: `/signup`, `/login`, `/find-id`, `/find-password`, `/reset-password`
- 그 외 엔드포인트는 인증된 사용자만 접근 가능하다.

---

## 9. Exception 계층

**(도메인별 예외를 정의하고, 전역 핸들러에서 `ApiResponse` 포맷으로 일관된 에러 응답을 반환하는 역할.)**

- `UserNotFoundException` → 존재하지 않는 회원 (탈퇴 포함)
- `DuplicateLoginIdException` → login_id 중복
- `DuplicateEmailException` → email 중복
- `InvalidPasswordException` → 비밀번호 불일치
- `AccountLockedException` → 잠금 상태 계정 접근 시도
- `GlobalExceptionHandler` → `@RestControllerAdvice`로 전역 처리

---

## 10. Enum 정의

#### UserStatus
- `ACTIVE` — 활성 (ERD 기본값)
- `LOCKED` — 잠금
- `WITHDRAWN` — 탈퇴

---

# API 명세서

**Base URL:** `http://localhost:8080`
**공통 응답:** `{ "status": int, "message": string, "data": object }`

---

### 1. 회원가입 `POST /api/member/signup`

**인증:** 불필요

**Request Body**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| loginId | String | O | 로그인 아이디 (UNIQUE, 최대 50자) |
| password | String | O | 비밀번호 (영문+숫자+특수문자 8자↑) |
| name | String | O | 이름 (최대 50자) |
| email | String | O | 이메일 (UNIQUE) |
| phone | String | X | 전화번호 |
| birthDate | String | O | 생년월일 (YYYY-MM-DD) |
| gender | String | X | 성별 (M/F) |

```json
// Request
{
  "loginId": "gildongi2003",
  "password": "Abcd1234!",
  "name": "홍길동",
  "email": "test@gachon.ac.kr",
  "phone": "010-1234-5678",
  "birthDate": "2003-05-10",
  "gender": "M"
}

// Response 200
{
  "status": 200,
  "message": "회원가입 완료",
  "data": {
    "userId": 12,
    "loginId": "gildongi2003",
    "message": "가입이 정상적으로 처리되었습니다."
  }
}
```

**Error:** 409 login_id 중복 / 409 email 중복 / 400 Validation 실패

---

### 2. 로그인 `POST /api/member/login`

**인증:** 불필요

**Request Body**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| loginId | String | O | 로그인 아이디 |
| password | String | O | 비밀번호 |

```json
// Request
{ "loginId": "parang2003", "password": "Abcd1234!" }

// Response 200 (+ Set-Cookie: refreshToken=...; HttpOnly; Secure)
{
  "status": 200,
  "message": "로그인 성공",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiJ9...",
    "name": "김팔랑"
  }
}
```

**Error:** 401 비밀번호 불일치 / 423 계정 잠금 / 404 존재하지 않는 아이디

---

### 3. 로그아웃 `POST /api/member/logout`

**인증:** Access Token 필수

**Request Header:** `Authorization: Bearer {accessToken}` + Cookie: `refreshToken=...`

```json
// Response 200 (+ Set-Cookie: refreshToken=; Max-Age=0)
{ "status": 200, "message": "로그아웃 되었습니다.", "data": null }
```

---

### 4. 토큰 재발급 `POST /api/member/token/refresh`

**인증:** Refresh Cookie 필수

```json
// Response 200 (+ 새 Refresh Cookie 발급)
{
  "status": 200,
  "message": "토큰 재발급 성공",
  "data": { "accessToken": "eyJhbGciOiJIUzI1NiJ9..." }
}
```

**Error:** 401 Refresh 만료/폐기

---

### 5. 내 정보 조회 `GET /api/member/me`

**인증:** Access Token 필수

```json
// Response 200
{
  "status": 200,
  "message": "조회 성공",
  "data": {
    "userId": 12,
    "loginId": "parang2003",
    "name": "김팔랑",
    "email": "test@gachon.ac.kr",
    "phone": "010-1234-5678",
    "birthDate": "2003-05-10",
    "gender": "M",
    "createdAt": "2026-04-10T14:22:01",
    "updatedAt": "2026-04-12T09:10:45"
  }
}
```

---

### 6. 본인 확인 `POST /api/member/me/verify-password`

**인증:** Access Token 필수

```json
// Request
{ "password": "Abcd1234!" }

// Response 200
{ "status": 200, "message": "본인 확인 성공", "data": null }
```

**Error:** 401 비밀번호 불일치

---

### 7. 내 정보 수정 `PUT /api/member/me`

**인증:** Access Token 필수

**Request Body** (변경할 필드만 넣어놓았습니다.)
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| name | String | X | 이름 |
| phone | String | X | 전화번호 |
| newPassword | String | X | 새 비밀번호 |

```json
// Request
{ "name": "홍건적(개명)", "newPassword": "Newpass1234!" }

// Response 200
{ "status": 200, "message": "회원 정보가 수정되었습니다.", "data": null }
```

**Error:** 400 Validation 실패

---

### 8. 회원 탈퇴 `DELETE /api/member/me`

**인증:** Access Token 필수

```json
// Request
{ "reason": "서비스를 자주 이용하지 않습니다." }

// Response 200
{ "status": 200, "message": "회원 탈퇴가 완료되었습니다.", "data": null }
```

---

### 9. 아이디 찾기 `POST /api/member/find-id`

**인증:** 불필요

```json
// Request
{ "name": "홍길동", "email": "gildongi@gachon.ac.kr" }

// Response 200
{
  "status": 200,
  "message": "아이디 찾기 성공",
  "data": { "maskedLoginId": "gild*******" }
}
```

**Error:** 404 일치하는 회원 없음

---

### 10. 비밀번호 찾기 (재설정 링크 발송) `POST /api/member/find-password`

**인증:** 불필요

```json
// Request
{ "loginId": "gildong2003", "email": "gildongi@gachon.ac.kr" }

// Response 200
{ "status": 200, "message": "비밀번호 재설정 메일이 발송되었습니다.", "data": null }
```

---

### 11. 비밀번호 재설정 `POST /api/member/reset-password`

**인증:** 불필요

```json
// Request
{ "token": "a1b2c3d4...", "newPassword": "Newpass1234!" }

// Response 200
{ "status": 200, "message": "비밀번호가 재설정되었습니다.", "data": null }
```

**Error:** 400 토큰 만료/무효 / 400 Validation 실패

---

## 인증 헤더 요약

| API 구분 | Access Token | Refresh Cookie |
|---|---|---|
| 공개 (signup, login, find-id, find-password, reset-password) | X | X |
| 인증 (me, verify-password, logout) | O | X |
| 토큰 재발급 (token/refresh) | X | O |

---

## 요구사항 ↔ API 매핑

| 요구사항 | API | UC |
|---|---|---|
| FR-01 회원가입 | POST /api/member/signup | UC-A03 |
| FR-02 로그인 | POST /api/member/login, POST /api/member/token/refresh | UC-A01 |
| FR-03 로그아웃 | POST /api/member/logout | UC-A02 |
| FR-04 정보 조회 | GET /api/member/me | UC-A04 |
| FR-05 정보 수정 | POST /api/member/me/verify-password → PUT /api/member/me | UC-A05 |
| FR-06 회원 탈퇴 | POST /api/member/me/verify-password → DELETE /api/member/me | UC-A06 |
| FR-07 아이디/비밀번호 찾기 | POST /find-id, /find-password, /reset-password | UC-A07 |
