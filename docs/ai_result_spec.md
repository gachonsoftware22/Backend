## AI Result Domain Specification

#### Overview
사용자가 특정 사용자의 건강정보 및 처방전 데이터를 LLM API에 전달하여 AI 건강 분석 결과를 생성하고,
그 결과를 저장·조회하는 기능을 제공한다.
분석 결과는 health_status / summary_note / potential_diseases / recommended_foods / recommended_exercises / precautions 항목으로 구성되며,
각 항목은 프론트엔드 화면에 개별 위젯으로 표시된다.

---

#### 클래스 구조

##### AiController.java
(클라이언트 요청 처리)
- POST /api/ai/trigger         → 사용자가 특정 사용자의 AI 분석을 수동 실행한다.
- GET  /api/ai/result/me → 해당 사용자의 최신 AI 분석 결과 전체를 반환한다.
- GET  /api/ai/result/me/summary       → 요약 항목(health_status, summary_note, analysis_date)만 반환한다.
- GET  /api/ai/result/me/diseases      → potential_diseases(JSON 배열) 반환한다.
- GET  /api/ai/result/me/foods         → recommended_foods(JSON 배열) 반환한다.
- GET  /api/ai/result/me/exercises     → recommended_exercises(JSON 배열) 반환한다.
- GET  /api/ai/result/me/precautions   → precautions(text) 반환한다.

##### AiDomainService.java (인터페이스)
Controller가 직접 의존하는 파사드 서비스.
- triggerAnalysis(Long userId) → 분석 전체 파이프라인 조율
- getLatestResult(Long userId) → 최신 결과 단건 조회
- getSummary / getDiseases / getFoods / getExercises / getPrecautions → 개별 항목 조회

##### AiDomainServiceImpl.java
AiDomainService의 구현체.
- AiDatasetService, AiResultService, AiResultRepository에 의존한다.
- triggerAnalysis 흐름:
  1. AiDatasetService로 CSV 데이터셋 생성
  2. AiResultService로 LLM 호출 및 파싱
  3. AiResultRepository로 ai_result 테이블에 저장

##### AiResultService.java (인터페이스)
LLM 공급자(Claude, OpenAI 등) 교체 가능성을 고려한 추상화 레이어.
- analyze(AiResultRequestDTO request) → AiResultResponseDTO

##### AiResultServiceImpl.java
AiResultService의 구현체. 실제 LLM API 호출 및 응답 파싱 담당.
- CSV 데이터와 프롬프트를 조합하여 LLM API에 요청을 전송한다.
- 응답 JSON을 AiResultResponseDTO로 역직렬화한다.
- raw_llm_response에 LLM 원본 응답을 보존한다.
- LLM 호출 실패 시 최대 3회 재시도(Exponential Backoff: 1s → 2s → 4s).

##### AiDatasetService.java (구현체, 인터페이스 없음)
userId 기반으로 health / prescription / prescription_detail 레포지토리에서 데이터를 수집하고
LLM에 전달할 CSV 형식의 AiResultRequestDTO를 생성한다.

```
[health]
health_id,symptom,history,note,record_date
1,"두통,피로감","고혈압 가족력","최근 수면 부족","2026-04-01"

[prescription]
prescription_id,prescription_date,hospital_name
10,"2026-03-20","서울내과"

[prescription_detail]
detail_id,prescription_id,medicine_name,dosage,duration
20,10,"암로디핀","1일 1회 1정","30일"
```

##### AiResultRepository.java
ai_result 테이블에 대한 CRUD 수행.
- findTopByUserIdOrderByAnalysisDateDesc(Long userId) → 최신 결과 단건 조회

##### AiResultEntity.java
ai_result 테이블 매핑.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| result_id | int (PK) | 자동 증가 |
| user_id | bigint | FK → users.user_id |
| health_status | varchar(20) | GOOD / CAUTION / WARNING |
| summary_note | text | 종합 건강 요약 문장 |
| potential_diseases | json | 추정 질환 목록 (String 배열) |
| recommended_foods | json | 권장 식품 목록 (String 배열) |
| recommended_exercises | json | 권장 운동 목록 (String 배열) |
| precautions | text | 주의사항 서술 |
| analysis_date | datetime | 분석 실행 일시 |
| raw_llm_response | json | LLM 원본 응답 보존 |

##### AiResultResponseDTO.java
LLM 응답 JSON 역직렬화 및 클라이언트 응답에 사용.
```java
Long    resultId
Long    userId
String  healthStatus        // "GOOD" | "CAUTION" | "WARNING"
String  summaryNote
List<String> potentialDiseases
List<String> recommendedFoods
List<String> recommendedExercises
String  precautions
String  analysisDate        // ISO 8601
```

##### AiResultRequestDTO.java
AiDatasetService가 생성하는 CSV 래퍼 DTO.
```java
Long   userId
String healthCsv            // health 테이블 CSV 문자열
String prescriptionCsv      // prescription + prescription_detail 조인 CSV
```

---

#### 프롬프트 엔지니어링 (AiResultServiceImpl)

**시스템 프롬프트**
```
당신은 환자의 건강 데이터를 분석하는 의료 AI 어시스턴트입니다.
아래 지침을 엄격히 따르십시오.
1. 응답은 반드시 순수 JSON 객체 하나만 출력하십시오. 설명 문장, 마크다운 코드블록, 전후 공백을 절대 포함하지 마십시오.
2. 필드 이름과 타입은 아래 스키마를 정확히 따르십시오.
3. 배열 필드는 최대 5개 항목으로 제한하십시오.
4. health_status 값은 GOOD / CAUTION / WARNING 중 하나만 사용하십시오.
```

**유저 프롬프트 (템플릿)**
```
다음은 환자의 최근 건강 기록 및 처방 데이터입니다.

[건강 기록 CSV]
{healthCsv}

[처방 데이터 CSV]
{prescriptionCsv}

위 데이터를 분석하여 아래 JSON 스키마에 맞는 결과를 반환하십시오.

{
  "health_status": "GOOD | CAUTION | WARNING",
  "summary_note": "한국어 종합 건강 요약 (2~4문장)",
  "potential_diseases": ["질환명1", "질환명2"],
  "recommended_foods": ["식품1", "식품2"],
  "recommended_exercises": ["운동1", "운동2"],
  "precautions": "한국어 주의사항 (1~3문장)"
}
```

---

#### 오류 처리 전략

| 상황 | 처리 방식 |
|------|----------|
| LLM API 호출 실패 | 최대 3회 재시도 (Exponential Backoff: 1s → 2s → 4s). 전부 실패 시 예외 전파, 해당 user_id 결과 저장하지 않음 |
| AI 응답 JSON 파싱 실패 | raw_llm_response에 원본 저장 후 예외를 던져 트랜잭션 롤백 |
| ai_result 저장 실패 | 트랜잭션 롤백으로 부분 저장 방지 |
| 존재하지 않는 userId 트리거 | 404 예외 반환 |
| 권한 없는 접근 (일반 사용자가 타인 결과 조회) | 403 예외 반환 |
| 분석 결과 없음 | 204 No Content 반환 |

---

#### AI Result API 명세서

---

##### 1. AI 분석 트리거 (관리자 전용)

- **URL**: `POST /api/ai/trigger`
- **설명**: 사용자가 자신 최신 건강·처방 데이터를 LLM에 전달하여 AI 분석을 실행하고 결과를 저장한다. 동기 처리.


**Request Header**
```
Authorization: Bearer {accessToken}
Content-Type: application/json
```

**Request Body**
```json
{
  "userId": 42
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| userId | Long | Y | 분석 대상 사용자 ID |

**Response (200 OK)**
```json
{
  "resultId": 15,
  "userId": 42,
  "healthStatus": "CAUTION",
  "summaryNote": "최근 두통과 피로감이 지속되고 있으며, 고혈압 관련 약물을 복용 중입니다. 생활 습관 개선과 정기 검진이 권장됩니다.",
  "potentialDiseases": ["고혈압", "만성피로증후군"],
  "recommendedFoods": ["바나나", "견과류", "브로콜리"],
  "recommendedExercises": ["걷기", "스트레칭"],
  "precautions": "카페인 섭취를 줄이고 충분한 수면을 취하십시오. 혈압 변동이 심할 경우 즉시 병원을 방문하십시오.",
  "analysisDate": "2026-04-28T10:30:00"
}
```

**예외**

| 상황 | HTTP 코드 | 메시지 |
|------|----------|--------|
| 존재하지 않는 userId | 404 | "해당 사용자를 찾을 수 없습니다." |
| LLM API 3회 재시도 실패 | 502 | "AI 분석 서비스에 일시적인 오류가 발생했습니다. 잠시 후 다시 시도해주세요." |
| AI 응답 파싱 실패 | 500 | "AI 응답 처리 중 오류가 발생했습니다." |
| 권한 없음 (비관리자) | 403 | "접근 권한이 없습니다." |

---

##### 2. AI 분석 결과 전체 조회

- **URL**: `GET /api/ai/result/me`
- **설명**: 해당 사용자의 가장 최근 AI 분석 결과 전체를 반환한다.
- **권한**: 본인 또는 ADMIN

**Request Header**
```
Authorization: Bearer {accessToken}
```

**Path Parameter**

| 파라미터 | 타입 | 설명 |
|---------|------|------|
| userId | Long | 조회 대상 사용자 ID |

**Response (200 OK)**
```json
{
  "resultId": 15,
  "userId": 42,
  "healthStatus": "CAUTION",
  "summaryNote": "최근 두통과 피로감이 지속되고 있으며, 고혈압 관련 약물을 복용 중입니다. 생활 습관 개선과 정기 검진이 권장됩니다.",
  "potentialDiseases": ["고혈압", "만성피로증후군"],
  "recommendedFoods": ["바나나", "견과류", "브로콜리"],
  "recommendedExercises": ["걷기", "스트레칭"],
  "precautions": "카페인 섭취를 줄이고 충분한 수면을 취하십시오. 혈압 변동이 심할 경우 즉시 병원을 방문하십시오.",
  "analysisDate": "2026-04-28T10:30:00"
}
```

**예외**

| 상황 | HTTP 코드 | 메시지 |
|------|----------|--------|
| 분석 결과 없음 | 204 | (body 없음) |
| 존재하지 않는 userId | 404 | "해당 사용자를 찾을 수 없습니다." |
| 타인 데이터 접근 | 403 | "접근 권한이 없습니다." |

---

##### 3. 건강 요약 조회

- **URL**: `GET /api/ai/result/me/summary`
- **설명**: health_status, summary_note, analysis_date만 반환한다. 메인 대시보드 카드 위젯에 사용.


**Response (200 OK)**
```json
{
  "healthStatus": "CAUTION",
  "summaryNote": "최근 두통과 피로감이 지속되고 있으며, 고혈압 관련 약물을 복용 중입니다. 생활 습관 개선과 정기 검진이 권장됩니다.",
  "analysisDate": "2026-04-28T10:30:00"
}
```

**healthStatus 값 정의**

| 값 | 의미 |
|----|------|
| GOOD | 건강 상태 양호 |
| CAUTION | 주의 필요 |
| WARNING | 위험, 즉각 조치 권장 |

---

##### 4. 추정 질환 목록 조회

- **URL**: `GET /api/ai/result/me/diseases`
- **설명**: potential_diseases 배열만 반환한다.


**Response (200 OK)**
```json
{
  "potentialDiseases": ["고혈압", "만성피로증후군"]
}
```

---

##### 5. 권장 식품 목록 조회

- **URL**: `GET /api/ai/result/me/foods`
- **설명**: recommended_foods 배열만 반환한다.


**Response (200 OK)**
```json
{
  "recommendedFoods": ["바나나", "견과류", "브로콜리"]
}
```

---

##### 6. 권장 운동 목록 조회

- **URL**: `GET /api/ai/result/me/exercises`
- **설명**: recommended_exercises 배열만 반환한다.


**Response (200 OK)**
```json
{
  "recommendedExercises": ["걷기", "스트레칭"]
}
```

---

##### 7. 주의사항 조회

- **URL**: `GET /api/ai/result/me/precautions`
- **설명**: precautions 텍스트만 반환한다.


**Response (200 OK)**
```json
{
  "precautions": "카페인 섭취를 줄이고 충분한 수면을 취하십시오. 혈압 변동이 심할 경우 즉시 병원을 방문하십시오."
}
```

---

#### Notes
- 비동기 처리는 MVP 범위 외. POST /api/ai/trigger는 동기 처리로 LLM 응답이 완료된 후 200 반환한다.
- LLM 공급자(Claude, OpenAI 등)는 AiResultService 인터페이스를 통해 교체 가능하도록 설계한다.
- health, prescription 데이터가 전혀 없는 사용자에 대해 트리거 호출 시 빈 CSV로 분석을 진행하지 않고 400을 반환한다.
- raw_llm_response는 디버깅 및 감사 목적으로 보존하며 클라이언트에 노출하지 않는다.
