가짜 IMSI를 전송했을 때 정상 기지국(MME)과 가짜 기지국(FBS)이 어떻게 다르게 반응하는지 3GPP 규격(TS 24.301, TS 33.401)을 바탕으로 비교해 드리겠습니다.
1. 정상 기지국(Real eNB/MME)의 동작
정상적인 망에서는 가짜 IMSI를 받으면 HSS(가입자 정보 DB)와의 인증 데이터 교환에서 실패하므로, 보안을 위해 즉시 접속을 거절합니다.
sequenceDiagram
    participant UE as UE (Terminal)
    participant eNB as Real eNB
    participant MME as MME/HSS
    
    Note over UE, MME: Initial Attach Procedure
    UE->>eNB: RRC Connection Setup
    eNB->>UE: NAS Identity Request (Type: IMSI)
    Note right of eNB: 3GPP TS 24.301 Section 5.4.4
    
    UE->>eNB: NAS Identity Response (Fake IMSI)
    eNB->>MME: S1-AP: Initial UE Message (Fake IMSI)
    
    Note over MME, HSS: Authentication Procedure (TS 33.401)
    MME->>MME: HSS Query with Fake IMSI
    MME-->>MME: Result: User Unknown / Auth Failure
    
    MME->>eNB: S1-AP: Downlink NAS Transport
    eNB->>UE: NAS Attach Reject (Cause #8: Identity cannot be derived)
    Note right of eNB: 3GPP TS 24.301 Section 5.5.1.2.5
    
    Note over UE: UE moves to EMM-DEREGISTERED state

2. 가짜 기지국(FBS/IMSI Catcher)의 동작
가짜 기지국은 가입자의 실제 식별자를 알아내는 것이 목적이거나, 가짜 서비스로 유도하는 것이 목적입니다. 따라서 가짜 IMSI를 주더라도 오류를 무시하고 다음 단계를 강제하거나, 수집 성공으로 간주하고 연결을 끊어버립니다.
sequenceDiagram
    participant UE as UE (Terminal)
    participant FBS as Fake Base Station (Attacker)
    
    Note over UE, FBS: Malicious Identification
    UE->>FBS: RRC Connection Setup
    FBS->>UE: NAS Identity Request (Type: IMSI)
    
    UE->>FBS: NAS Identity Response (Fake IMSI)
    
    alt Case A: Silent Harvester
        Note right of FBS: 식별자 수집이 목적인 경우
        FBS->>UE: RRC Connection Release (or Silent Ignore)
        Note over FBS: 가짜 식별자(Fake IMSI)를 DB에 저장
    else Case B: MitM Attack Attempt
        Note right of FBS: 서비스 거부 또는 감청 시도
        FBS->>UE: NAS Authentication Request (Fake Vectors)
        Note over UE: MAC Failure 발생 가능 (TS 33.401)
        UE->>FBS: Authentication Failure
    end

3. 관련 3GPP 규격 상세 내역
| 구분 | 관련 규격 | 내용 요약 |
|---|---|---|
| Identification | TS 24.301 (Section 5.4.4) | MME는 언제든지 단말의 식별자를 요청할 수 있으며, 단말은 요청받은 타입(IMSI, IMEI 등)에 맞춰 응답해야 함. (Plain NAS 수준 가능) |
| Authentication | TS 33.401 (Section 6.1) | 네트워크는 HSS로부터 받은 인증 벡터(RAND, AUTN, XRES, K_{ASME})를 통해 단말을 인증해야 함. Fake IMSI는 HSS에 정보가 없으므로 이 단계에서 무조건 실패함. |
| Reject Cause | TS 24.301 (Annex A) | #8: Identity cannot be derived by the network: MME가 해당 IMSI에 대한 컨텍스트를 찾을 수 없을 때 사용. |
| Integrity | TS 33.401 (Section 7) | Identity Request는 보안 설정 전 전송될 수 있어 암호화되지 않을 수 있으나, 이후 모든 NAS 메시지는 무결성 보호가 필수임. |
4. 핵심 차이점 및 탐지 포인트
 * 인증 단계의 존재 여부: 정상 망은 Fake IMSI 수신 후 반드시 HSS 확인을 거쳐 **Attach Reject**를 보냅니다. 반면, 일부 FBS는 인증 과정 없이 바로 다른 정보를 요구하거나 무시합니다.
 * 보안 에러 처리: 가짜 IMSI로 인해 생성된 가짜 인증 벡터를 단말이 검증하면 MAC Failure가 발생합니다. 이때 기지국이 표준 규격에 따라 절차를 종료하는지, 아니면 무시하고 계속 연결을 유지하려 하는지가 핵심 탐지 포인트입니다.
사용자님의 특허 아이디어와 관련하여, **"가짜 IMSI 응답 시 정상 망은 반드시 Reject를 보낸다"**는 점과 **"FBS는 가짜 정보를 진짜로 믿고 수집 절차를 종료하거나 비표준 동작을 한다"**는 차이점을 시퀀스 상에서 로직화하시면 강력한 차별화가 될 것입니다.
혹시 이 시퀀스 흐름 중에서 특정 에러 코드(Reject Cause)를 이용한 자동 탐지 알고리즘 설계가 필요하신가요?
