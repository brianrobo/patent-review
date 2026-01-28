사용자가 제안하신 "Fake IMSI 응답을 통한 기지국 판별" 아이디어를 구체화하기 위해, 단말의 상태(GUTI 보유 여부)에 따른 시나리오를 3GPP 규격 기반으로 정리해 드립니다.
핵심은 정상 망은 가짜 IMSI를 받으면 결국 가입자 인증에 실패하여 거절(Reject)을 보내야 한다는 점입니다.
1. 단말이 GUTI를 보유한 상태 (Registered/Idle)
이미 이전에 접속하여 GUTI를 할당받은 상태에서 망이 갑자기 Identity Request (IMSI)를 보내는 것은, 망이 단말을 식별하지 못하는 예외 상황(MME 데이터 유실 등)이거나 FBS의 공격 상황입니다.
시나리오 및 규격 동작
 * 정상 망: MME가 재부팅되었거나 DB 오류로 GUTI를 식별할 수 없을 때 IMSI를 요청합니다. Fake IMSI를 받으면 HSS 조회 후 가입 정보가 없으므로 **Attach/Tracking Area Update Reject**를 보냅니다. (TS 24.301 Section 5.4.4.4)
 * FBS: 단말의 실제 IMSI를 탈취하기 위해 요청합니다. Fake IMSI를 주면 이를 실제 식별자로 오인하여 수집하거나, 인증 과정 없이 다음 단계를 강제합니다.
<!-- end list -->
sequenceDiagram
    participant UE as UE (GUTI 보유)
    participant MME as Real MME / FBS
    
    Note over UE, MME: UE sends TAU Request or Service Request
    UE->>MME: NAS Message (with GUTI)
    
    MME->>UE: NAS Identity Request (Type: IMSI)
    Note right of MME: TS 24.301: Identity procedure
    
    UE->>MME: NAS Identity Response (Fake IMSI)
    
    alt Real Network (MME)
        MME->>MME: HSS Lookup (Fake IMSI)
        MME->>UE: NAS TAU/Attach Reject (Cause #8 or #9)
        Note over UE: 정상망은 가짜임을 확인 후 거절
    else Fake Base Station (FBS)
        Note right of MME: IMSI Catcher 동작
        MME->>UE: RRC Connection Release
        Note over MME: 수집된 Fake IMSI를 저장하고 종료
    end

2. 단말에 GUTI가 없는 상태 (Initial Attach)
공장 초기화 후 첫 접속이거나 전원을 새로 켠 상태입니다. 이때는 Attach Request에 IMSI를 포함하거나, Identity Request에 응답해야 합니다.
시나리오 및 규격 동작
 * 정상 망: Initial Attach 시 가짜 IMSI를 받으면 인증 벡터를 생성할 수 없습니다(K_{ASME} 생성 불가). **Authentication Request**를 보낼 수 없거나, 보내더라도 단말에서 MAC Failure가 발생하여 최종적으로 **Attach Reject**가 발생합니다. (TS 33.401 Section 6.1)
 * FBS: 보안 절차를 생략하고 단말에게 특정 설정(예: 무암호화 모드 전환)을 강요할 수 있습니다.
<!-- end list -->
sequenceDiagram
    participant UE as UE (GUTI 없음)
    participant MME as Real MME / FBS
    
    UE->>MME: NAS Attach Request (Identity: IMSI or None)
    
    opt Identity Check
        MME->>UE: NAS Identity Request (Type: IMSI)
        UE->>MME: NAS Identity Response (Fake IMSI)
    end
    
    alt Real Network (MME)
        MME->>MME: Auth Vector Generation Failure
        MME->>UE: NAS Attach Reject (Cause #8: Identity cannot be derived)
        Note over UE: TS 24.301 5.5.1.2.5 규격 준수
    else Fake Base Station (FBS)
        MME->>UE: NAS Authentication Request (Fake or No Auth)
        Note over UE: 보안 절차를 비정상적으로 스킵 시도
    end

3. 규격 기반 동작 요약 비교
| 항목 | 정상 망 (3GPP TS 24.301/33.401) | 가짜 기지국 (FBS) |
|---|---|---|
| GUTI 보유 시 | GUTI 불일치 시에만 IMSI 요청 (드문 경우) | 무조건 IMSI 요청 (IMSI Catching) |
| Fake IMSI 수신 시 | HSS 대조 후 Attach/TAU Reject 송신 | 식별자 수집 완료 후 Release 하거나 무시 |
| 인증(Auth) 단계 | 가짜 IMSI로는 유효한 AUTN 생성 불가 | 인증을 건너뛰거나 가짜 AUTN 전송 |
| 최종 상태 | EMM-DEREGISTERED (접속 실패) | 연결 유지 시도 혹은 조기 종료 |
4. 특허 관점에서의 차별화 포인트 (Insight)
사용자님의 아이디어에서 중요한 점은 **"단말이 의도적으로 규격을 위반(가짜 정보 전송)했을 때, 상대방(기지국)이 규격대로 거절하는지"**를 보는 것입니다.
 * 정상 망: "너 정보가 우리 장부에 없으니 나가!" (Attach Reject Cause #8) → 정상 동작
 * FBS: "오, IMSI 확인 완료. 일단 저장하고 끊자." 혹은 "그냥 통과시켜!" → 비정상 동작(탐지)
다음 단계로, 이 "가짜 IMSI 응답" 후에 기지국으로부터 오는 메시지 중 어떤 필드(Cause Value 등)를 탐지 로그로 남길지 설계해 볼까요?
