## 도메인 기능 소개

본 spec에서 정의되는 기능은 LLM API 를 이용하여 사용자의 처방정보 / 건강정보를 수집하여 AI로
건강상태를 요약해서 반환한다.

### 상세 설계
Spring으로 개발되며 SOLID 원칙을 준수한다.
데이터 취합에 필요한 데이터 수집은 health / prescription / prescription_detail 테이블의 데이터를 수집한다,
사용자의 user_id 기반으로 user_id에 해당하는 위 테이블의 데이터들을 csv 같은 데이터 구조로 LLM api 호출하여 분석하고
그 결과를 다시 ai_result 테이블에 맞게끔 집어넣어야한다.

### AiController.java
post /api/ai/trigger 
->관리자 웹페이지에서 AI 추론을 위한 데이터 취합의 트리거

GET api 들
ai_result 테이블에 저장된 데이터를 프론트에 보내주기 위함
user_id               bigint   [not null, ref: > users.user_id]
health_status         varchar(20) [not null]
summary_note          text     [not null]
potential_diseases    json
recommended_foods     json
recommended_exercises json
precautions           text
analysis_date         datetime [not null]
이거 다 각각 프론트에 띄워줄 것임

### AiDomainService.java
인터페이스 / Controller가 직접 의존 
### AiResultService.java
인터페이스 OPenAPI, claude 등 다양한 AI 로 교체될 가능성
### AiDatasetService.java
원하는 데이터셋의 생성, 구현체(인터페이스 없음, 교체 가능성 없음)
userid 기반으로  health / prescription / prescription_detail 각각의 레포지토리에서취합
### AiResultServiceImpl.java
실제 AI API 호출해서 프롬프트 기반 요청전송 및 결과 json 수신
csv 첨부하고 프롬프트로 ai_results 테이블 형식에 맞는 json 데이터 반환하는 프롬프트 엔지니어링
raw_llm_response 는 AI 응답 원문.
### AiResultRepository.java
AI 결과를 조회해서 보여주기 위함
### AiResultEntity.java
ai_result 테이블 매핑
### AiResultResponseDTO.java
돌아온 요청결과 json의 직렬화를 위함 
### AiResultRequestDTO.java
취합한 데이터를 AI가 추론할 수 있는 csv로 직렬화를 위함

### 오류 처리 전략
- LLM API 호출 실패 시: 최대 3회 재시도 (exponential backoff 적용), 전부 실패 시 예외를 상위로 전파하고 해당 user_id의 분석 결과는 저장하지 않는다.
- AI 응답 파싱 실패 시: raw_llm_response에 원본 응답을 저장한 뒤 예외를 던져 트랜잭션을 롤백한다.
- ai_result 저장 실패 시: 트랜잭션 롤백으로 부분 저장을 방지한다.


비동기 처리 같은 복잡한 시스템 구성은 제외. MVP 개발 