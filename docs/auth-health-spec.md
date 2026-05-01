# Auth & Health Spec

담당 도메인: 인증(Auth), 건강 정보(Health)

---

## 1. Overview
인증(회원가입/로그인/로그아웃) 및 사용자 건강 정보 관리 기능을 담당한다.

---

## 2. DB Schema

### User Table
| Column | Type | Constraint | Description |
|--------|------|-----------|------------|
| user_id | BIGINT | PK, AUTO_INCREMENT | 사용자 ID |
| login_id | VARCHAR(50) | UNIQUE, NOT NULL | 로그인 ID |
| email | VARCHAR(100) | UNIQUE, NOT NULL | 이메일 |
| password | VARCHAR(255) | NOT NULL | 비밀번호 |
| name | VARCHAR(50) | NOT NULL | 이름 |
| phone | VARCHAR(20) |  | 전화번호 |
| birth_date | DATE | NOT NULL | 생년월일 |
| gender | CHAR(1) |  | 성별 |
| status | VARCHAR(10) | NOT NULL, DEFAULT 'ACTIVE' | 상태 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일 |
| updated_at | DATETIME |  | 수정일 |

### Refresh Token Table
| Column | Type | Constraint | Description |
|--------|------|-----------|------------|
| token_id | BIGINT | PK, AUTO_INCREMENT | 토큰 ID |
| user_id | BIGINT | FK(users.user_id), NOT NULL | 사용자 ID |
| token_hash | VARCHAR(255) | UNIQUE, NOT NULL | 리프레시 토큰 해시 |
| expires_at | DATETIME | NOT NULL | 만료일 |
| revoked | BOOLEAN | NOT NULL, DEFAULT false | 폐기 여부 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일 |

### Health Table
| Column | Type | Constraint | Description |
|--------|------|-----------|------------|
| health_id | BIGINT | PK, AUTO_INCREMENT | 건강 정보 ID |
| user_id | BIGINT | FK(users.user_id), NOT NULL | 사용자 ID |
| symptom | TEXT | NOT NULL | 증상 |
| history | TEXT |  | 병력 |
| note | TEXT |  | 메모 |
| record_date | DATE | NOT NULL | 기록 날짜 |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP | 생성일 |
| updated_at | DATETIME |  | 수정일 |

---

## 3. ERD
- User (1) : (N) Health
- User (1) : (N) RefreshToken

---

## 4. API

### Auth
- POST /api/auth/signup → 회원가입
- POST /api/auth/login → 로그인
- POST /api/auth/logout → 로그아웃
- GET /api/auth/me → 사용자 정보 조회

### Health
- POST /api/health → 건강 정보 등록
- GET /api/health → 건강 정보 목록 조회
- GET /api/health/{id} → 건강 정보 상세 조회
- PUT /api/health/{id} → 건강 정보 수정
- DELETE /api/health/{id} → 건강 정보 삭제

---

## 5. Flow Specification

### Signup Flow
- Input: SignupRequestDto(loginId, email, password, name, birthDate)
- Success: UserResponseDto 반환
- Fail:
  - 중복된 loginId 또는 email이면 DuplicateUserException 발생
  - 필수 입력값이 비어있으면 InvalidRequestException 발생

### Login Flow
- Input: LoginRequestDto(loginId 또는 email, password)
- Success: AuthToken(accessToken, refreshToken) 반환
- Fail:
  - 아이디/비밀번호 불일치 시 InvalidCredentialException 발생
  - 비활성 계정이면 AccountInactiveException 발생

### Logout Flow
- Input: LogoutRequestDto(refreshToken)
- Success: 해당 refreshToken 폐기 처리
- Fail:
  - 유효하지 않은 토큰이면 InvalidTokenException 발생

### Get Current User Flow
- Input: Authorization Header의 accessToken
- Success: UserResponseDto 반환
- Fail:
  - 토큰이 없거나 만료되면 UnauthorizedException 발생

### Health Create Flow
- Input: HealthRequestDto(symptom, history, note, recordDate)
- Success: HealthResponseDto 반환
- Fail:
  - 로그인하지 않은 사용자는 UnauthorizedException 발생
  - symptom 또는 recordDate가 비어있으면 InvalidHealthDataException 발생

### Health List Flow
- Input: Authorization Header의 accessToken
- Success: 사용자의 HealthResponseDto 목록 반환
- Fail:
  - 로그인하지 않은 사용자는 UnauthorizedException 발생

### Health Update Flow
- Input: healthId, HealthRequestDto
- Success: 수정된 HealthResponseDto 반환
- Fail:
  - 존재하지 않는 건강 정보이면 HealthNotFoundException 발생
  - 본인의 건강 정보가 아니면 ForbiddenException 발생

### Health Delete Flow
- Input: healthId
- Success: 건강 정보 삭제
- Fail:
  - 존재하지 않는 건강 정보이면 HealthNotFoundException 발생
  - 본인의 건강 정보가 아니면 ForbiddenException 발생

---

## 6. Structure

- Controller: 클라이언트 API 요청 처리
- Service: 비즈니스 로직 처리
- Repository: DB 접근 및 쿼리 수행
- Entity: DB 테이블과 매핑
- DTO: Request / Response 데이터 전달
- JWT: 인증 토큰 발급 및 검증

---

## 7. Controller / Service Detail

### AuthController.java
- signup()
- login()
- logout()
- getCurrentUser()

### AuthService.java
- signup()
- login()
- logout()
- getCurrentUser()

### AuthServiceImpl.java
- AuthService 인터페이스 구현체
- 인증 관련 실제 비즈니스 로직 처리

### HealthController.java
- createHealth()
- getHealthList()
- getHealthDetail()
- updateHealth()
- deleteHealth()

### HealthService.java
- createHealth()
- getHealthList()
- getHealthDetail()
- updateHealth()
- deleteHealth()

### HealthServiceImpl.java
- HealthService 인터페이스 구현체
- 건강 정보 관련 실제 비즈니스 로직 처리

---

## 8. Repository

- UserRepository
- RefreshTokenRepository
- HealthRepository

---

## 9. Entity

- User
- RefreshToken
- Health

---

## 10. DTO

### Auth DTO
- SignupRequestDto
- LoginRequestDto
- LoginResponseDto
- LogoutRequestDto
- UserResponseDto

### Health DTO
- HealthRequestDto
- HealthResponseDto

---

## 11. JWT

### Header
Authorization: Bearer {accessToken}

### Description
- 로그인 성공 시 accessToken과 refreshToken을 발급한다.
- accessToken은 인증이 필요한 API 요청 시 Authorization Header에 포함한다.
- refreshToken은 accessToken 재발급 또는 로그아웃 처리 시 사용한다.