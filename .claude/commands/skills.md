## claude.md 작성

사실상 /init 수행을 위한 가이드를 제시한다.

### 리딩 순서 및 조건
ERD.md -> cd docs -> auth.md -> member.md -> prescription.md -> health.md -> ai_results.md

makrdown은 각각 개별 작업자가 작업을 하였고, 일관되지 않다.
따라서 개별 markdown의 신뢰성은 높지않고, 반드시 지켜지는 대전제는 ERD.md 이다. (ERD에 어긋나는 트랜잭션 혹은 기능은 존재할 수 없다.)

SOLID 원칙을 준수하되, D 의존성 역전에서 인터페이스를 통해서 클라이언트가 구현을 의존할 때 재사용성이 없는 불필요한 인터페이스는 생성하지 않는다.

Domain 
auth : 사용자 회원가입 / 로그인 / 탈퇴 / 인증(토큰)
member : 회원정보 수정/ 조회 / 삭제
prescription : 처방 정보  / 조회 / 입력 / 수정 / 삭제
health : 건강 정보  / 조회 / 입력 / 수정 / 삭제
ai_result : 회원 아이디 + prescription + health 정보 통합을 통한 LLM API 결과 반환

auth 를 통해서 토큰을 발급받으면 (refresh token) 이를 기반 모든 서비스 인증

결과물은 완성된 하나의 projcect 의 아키텍쳐 flow를 볼 수 있어야하며 
시퀀스다이어그램, 유스케이스 다이어그램, 클래스 다이어그램을 작성하기 위한 기본적인 정보를 담고 있어야한다.

작성 결과물 기록 : claude.md 

