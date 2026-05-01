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
| email | VARCHAR(100) | UNIQUE, NOT NULL | 이메일 |
| password | VARCHAR(255) | NOT NULL | 비밀번호 |
| name | VARCHAR(50) | NOT NULL | 이름 |
| phone | VARCHAR(20) |  | 전화번호 |
| birth_date | DATE |  | 생년월일 |
| gender | CHAR(1) |  | 성별 |
| status | VARCHAR(10) | DEFAULT 'ACTIVE' | 상태 |
| created_at | DATETIME | DEFAULT CURRENT_TIMESTAMP | 생성일 |
| updated_at | DATETIME |  | 수정일 |

---

### Health Table
| Column | Type | Constraint | Description |
|--------|------|-----------|------------|
| health_id | BIGINT | PK, AUTO_INCREMENT | 건강 ID |
| user_id | BIGINT | FK(user.user_id), NOT NULL | 사용자 ID |
| symptom | TEXT | NOT NULL | 증상 |
| history | TEXT |  | 병력 |
| note | TEXT |  | 메모 |
| created_at | DATETIME | DEFAULT CURRENT_TIMESTAMP | 생성일 |
| updated_at | DATETIME |  | 수정일 |

---

## 3. ERD
User (1) : (N) Health

---

## 4. API

### Auth
- POST /api/auth/signup → 회원가입
- POST /api/auth/login → 로그인 (JWT 발급)
- POST /api/auth/logout → 로그아웃
- GET /api/auth/me → 사용자 정보 조회

### Health
- POST /api/health → 건강 정보 등록
- GET /api/health → 목록 조회
- GET /api/health/{id} → 상세 조회
- PUT /api/health/{id} → 수정
- DELETE /api/health/{id} → 삭제

---

## 5. Structure (MVC)

- Controller: API 요청 처리
- Service: 비즈니스 로직 처리
- Repository: DB 접근
- Entity: DB 테이블 매핑
- DTO: Request / Response 데이터 전달

---

## 6. Controller / Service Detail

### AuthController.java
- POST /api/auth/signup → 회원가입
- POST /api/auth/login → 로그인
- POST /api/auth/logout → 로그아웃
- GET /api/auth/me → 사용자 조회

### AuthService.java
- signup() → 사용자 생성 (중복 체크, 암호화)
- login() → 로그인 및 JWT 발급
- logout() → 로그아웃 처리
- getCurrentUser() → 사용자 조회

---

### HealthController.java
- POST /api/health → 등록
- GET /api/health → 목록 조회
- GET /api/health/{id} → 상세 조회
- PUT /api/health/{id} → 수정
- DELETE /api/health/{id} → 삭제

### HealthService.java
- createHealth()
- getHealthList()
- getHealthDetail()
- updateHealth()
- deleteHealth()

---

## 7. Repository

- UserRepository
- HealthRepository

---

## 8. Entity

- User
- Health

---

## 9. DTO

- SignupRequestDto
- LoginRequestDto
- LoginResponseDto
- LogoutRequestDto
- UserResponseDto
- HealthRequestDto
- HealthResponseDto

---

## 10. JWT

### Header
Authorization: Bearer {accessToken}

### Description
- 로그인 성공 시 JWT 토큰 발급
- 이후 API 요청 시 Authorization 헤더에 포함