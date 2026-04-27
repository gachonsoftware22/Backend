Prescription Domain Specification

#### Overview
사용자가 병원에서 받은 처방전 정보를 입력, 조회, 수정, 삭제하여 이력을 관리한다.

#### PrescriptionController.java
(클라이언트 요청 처리)
POST /api/prescription/create -> 처방전 및 처방약 정보를 입력받아 생성 요청을 처리한다.
GET /api/prescription/list -> 로그인된 사용자의 처방전 목록을 조회한다.
GET /api/prescription/{prescription_id} -> 특정 처방전의 상세 정보를 조회한다.
PUT /api/prescription/update/{prescription_id} -> 기존 처방전 정보를 수정한다.
DELETE /api/prescription/delete/{prescription_id} -> 처방전 정보를 삭제 상태로 변경한다.

#### PrescriptionService.java
(기능 정의)
- createPrescription() → 처방전 생성
- getPrescriptionList() → 처방전 목록 조회
- getPrescriptionDetail() → 처방전 상세 조회
- updatePrescription() → 처방전 수정
- deletePrescription() → 처방전 삭제

#### PrescriptionServiceImpl.java
(실제 로직 처리)
- 처방전 생성 요청 시 필수 입력값(처방일, 병원명, 약 정보 등)을 검증한다.
- 사용자 ID를 기반으로 처방전 데이터를 생성하고 저장한다.
- 처방약 정보가 여러 개인 경우 반복 처리하여 각각 저장한다.
- 처방전 조회 시 사용자 ID 기준으로 데이터 필터링을 수행한다.
- 처방전 목록 조회 시 최신순으로 정렬하여 반환한다.
- 특정 처방전 조회 시 처방전 정보와 처방약 목록을 함께 조회한다.
- 처방전 수정 시 기존 데이터를 조회한 후 입력값으로 업데이트한다.
- 처방약 정보 수정 시 기존 데이터를 제거 후 재등록한다.
- 삭제 요청 시 실제 삭제가 아닌 상태값(status)을 DELETED로 변경한다.
- 존재하지 않는 처방전 요청 시 예외를 처리한다.
- 사용자 권한이 없는 데이터 접근 시 예외를 처리한다.

#### PrescriptionRepository.java
(DB 접근)
- prescription 테이블에 대한 CRUD 수행
- 사용자 ID 기반 처방전 조회
- 상태값 기준 필터링 조회

#### PrescriptionDetailRepository.java
(DB 접근)
- prescription_detail 테이블에 대한 CRUD 수행
- prescription_id 기준 처방약 목록 조회

#### Entity
- Prescription
- prescription_id
- user_id
- prescription_date
- hospital_name
- status
- created_at
- updated_at
- PrescriptionDetail
- detail_id
- prescription_id
- medicine_name
- dosage
- duration
- created_at
- updated_at

#### DTO
- PrescriptionRequestDto
- prescriptionDate
- hospitalName
- medicineList (약 정보 리스트)
- PrescriptionResponseDto
- prescriptionId
- prescriptionDate
- hospitalName
- medicineList

#### Exception Handling
- 필수값 누락 시 예외 처리
- 존재하지 않는 처방전 조회 시 예외 처리
- 권한 없는 접근 시 예외 처리
- 잘못된 입력값 형식에 대한 예외 처리

#### Notes
- 처방전과 처방약은 1 관계로 관리한다.
- 삭제는 Soft Delete 방식으로 처리한다.
- 모든 데이터는 사용자 기준으로 관리된다.

#### Prescription API 명세서
 1. 처방전 생성
- URL: POST /api/prescription/create
(설명: 처방전 및 처방약 정보를 등록한다.)

- Request
{
  "prescriptionDate": "2026-04-26",
  "hospitalName": "서울병원",
  "medicineList": [
    {
      "medicineName": "타이레놀",
      "dosage": "1일 2회",
      "duration": "5일"
    }
  ]
}

- Response
{
  "prescriptionId": 1,
  "message": "처방전이 생성되었습니다."
}

(예외)
- 필수값 누락 시 요청 실패
- 잘못된 데이터 형식 입력 시 실패

2. 처방전 목록 조회
- URL: GET /api/prescription/list
(설명: 로그인된 사용자의 처방전 목록을 조회한다.)

- Response
[
  {
    "prescriptionId": 1,
    "prescriptionDate": "2026-04-26",
    "hospitalName": "서울병원"
  }
]

(예외)
- 인증되지 않은 사용자 접근 시 실패

3. 처방전 상세 조회
- URL: GET /api/prescription/{prescription_id}
(설명: 특정 처방전의 상세 정보를 조회한다.)

- Response
{
  "prescriptionId": 1,
  "prescriptionDate": "2026-04-26",
  "hospitalName": "서울병원",
  "medicineList": [
    {
      "medicineName": "타이레놀",
      "dosage": "1일 2회",
      "duration": "5일"
    }
  ]
}

(예외)
- 존재하지 않는 처방전 조회 시 실패

4. 처방전 수정
- URL: PUT /api/prescription/update/{prescription_id}
(설명: 기존 처방전 정보를 수정한다.)

- Request
{
  "prescriptionDate": "2026-04-27",
  "hospitalName": "강남병원",
  "medicineList": [
    {
      "medicineName": "항생제",
      "dosage": "1일 3회",
      "duration": "7일"
    }
  ]
}

- Response
{
  "message": "처방전이 수정되었습니다."
}

(예외)
- 존재하지 않는 처방전 수정 시 실패
- 권한 없는 사용자 접근 시 실패

5. 처방전 삭제
- URL: DELETE /api/prescription/delete/{prescription_id}
(설명: 처방전을 삭제 상태로 변경한다.)

- Response
{
  "message": "처방전이 삭제되었습니다."
}

(예외)
- 존재하지 않는 처방전 삭제 시 실패
- 권한 없는 사용자 접근 시 실패