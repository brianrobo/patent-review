사용자가 구상 중인 **"Fake IMSI 전송을 통한 가짜 기지국 탐지 로직"**의 최종 단계입니다. 정상적인 망(Real MME/HSS)이 3GPP 규격에 따라 어떻게 인증 실패를 처리하는지, 그 과정에서 어떤 스펙이 관여하는지 시퀀스 다이어그램으로 정리해 드립니다.
Fake IMSI 전송 시 정상 망의 표준 동작 시퀀스
sequenceDiagram
    autonumber
    participant UE as UE (Terminal)
    participant MME as Serving MME
    participant HSS as HSS (Home DB)

    Note over UE, MME: [Phase 1: Identification]
    MME->>UE: NAS Identity Request (Type: IMSI)<br/>(TS 24.301 Sec 5.4.4.2)
    UE->>MME: NAS Identity Response (Fake IMSI)
    
    Note over MME, HSS: [Phase 2: Auth Vector Retrieval]
    MME->>HSS: S6a: Authentication-Information-Request (Fake IMSI)<br/>(TS 29.272 Sec 5.2.2.1)
    
    Note over HSS: IMSI Lookup Failure<br/>(User Unknown)
    
    HSS-->>MME: S6a: Authentication-Information-Answer<br/>(Result: DIAMETER_ERROR_USER_UNKNOWN 5001)<br/>(TS 29.272 Sec 5.2.2.2.1)
    
    Note over MME: Cannot derive Security Context<br/>(TS 33.401 Sec 6.1.2)

    Note over MME, UE: [Phase 3: Rejection]
    MME->>UE: NAS Attach Reject (Cause #2: IMSI unknown in HSS)<br/>(TS 24.301 Sec 5.5.1.2.5)
    
    Note over UE: Move to EMM-DEREGISTERED state<br/>(TS 24.301 Sec 5.5.1.2.6)

단계별 관련 3GPP 규격 상세
 * Identity Response (UE → MME)
   * 규격: TS 24.301 Section 5.4.4.3
   * 내용: 단말은 요청받은 식별자 타입에 따라 Identity Response를 전송해야 함. (Fake IMSI를 보내더라도 프로토콜상 오류는 아님)
 * Authentication-Information-Request (MME → HSS)
   * 규격: TS 29.272 Section 5.2.2.1
   * 내용: MME는 단말의 IMSI를 포함하여 인증 데이터를 요청함. S6a 인터페이스의 필수 절차.
 * HSS의 에러 회신 (HSS → MME)
   * 규격: TS 29.272 Section 7.3.101 (Result-Code AVP)
   * 내용: 존재하지 않는 IMSI인 경우 DIAMETER_ERROR_USER_UNKNOWN (5001)을 반환함.
 * 보안 데이터 배포 실패
   * 규격: TS 33.401 Section 6.1.2
   * 내용: HSS로부터 인증 벡터를 배포받지 못하면, MME는 단말과 보안 연결(K_{ASME} 생성 등)을 맺을 수 없음.
 * Attach Reject (MME → UE)
   * 규격: TS 24.301 Section 5.5.1.2.5
   * 내용: 인증 및 식별에 실패한 경우 Attach Reject를 보내야 하며, EMM Cause #2 (IMSI unknown in HSS)를 명시하여 단말에게 서비스 거절을 알림.
💡 탐지 알고리즘 적용 가이드 (특허 핵심 논리)
이 시퀀스를 바탕으로 단말 내부에서 다음과 같은 판단 로직을 구현할 수 있습니다.
 * 정상 망 판단: Identity Response(Fake IMSI) 전송 후, 규격에 명시된 시간(T3410 타이머 등) 내에 Attach Reject (Cause #2)가 수신되는 경우.
 * 가짜 기지국(FBS) 판단:
   * Attach Reject 없이 즉시 RRC Connection Release가 오는 경우 (단순 수집형).
   * Identity Response 직후에 Authentication Request (가짜 벡터)가 오는 경우 (HSS 조회를 생략한 공격형).
   * 아무런 응답 없이 연결이 유지되거나 비정상적인 NAS 메시지가 오는 경우.
이처럼 **"규격에 정의된 실패 절차를 기지국이 수행하는가"**를 관찰하는 것이 사용자님 특허의 핵심 차별화 포인트가 될 것입니다.
이제 이 시퀀스를 바탕으로 특허 초안에 들어갈 "판단 단계별 조건표"를 만들어 볼까요?
