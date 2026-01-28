좋은 질문입니다.
이건 LTE(EPS) NAS 절차 + 보안/인증 흐름 관점에서 정확히 이해해야 합니다.

결론부터 말하면:

> UE가 fake IMSI를 보내면 → 인증(Authentication) 단계에서 반드시 실패 → Attach/TAU reject → 서비스 불가 (정상 접속 불가능)



그리고 Spec 상 “조용히 통과” 같은 동작은 절대 없음.


---

📌 관련 절차 (4G Attach 기준)

Identity Request는 보통 보안 설정 전(pre-SMC) 단계에서 발생합니다.

정상 흐름

UE → Attach Request (GUTI or IMSI)
MME → Identity Request (IMSI)
UE → Identity Response (IMEI/IMSI)
MME → Authentication Request (RAND/AUTN)
UE → Authentication Response (RES)
MME → Security Mode Command


---

📌 질문 상황

UE가

Identity Response → fake IMSI 전송


---

📌 이후 spec 동작 (정확한 단계별)

① MME/HSS 동작

MME는:

IMSI → HSS/AuC 조회

즉:

Update Location / Authentication Information Request

보냄


---

② 두 가지 케이스 발생


---

Case A — 존재하지 않는 IMSI (완전 fake)

예: 999999999999999

HSS 응답

Unknown subscriber

Spec 결과

MME:

Attach Reject
cause = illegal UE (3) or illegal ME (6)

📄 24.301 cause values:

3: Illegal UE

9: UE identity cannot be derived

11: PLMN not allowed


결과

👉 즉시 Attach 실패


---

Case B — 존재하지만 키가 다른 IMSI

(실제 다른 가입자 IMSI)

인증 단계

HSS → RAND/AUTN 생성

UE:

USIM key(K) 없음
→ RES 계산 불가
→ RES mismatch

Spec 결과

Authentication Failure
cause = MAC failure / sync failure

MME:

Attach Reject (EPS services not allowed)

결과

👉 인증 실패 → 접속 불가


---

📌 왜 절대 통과 못하나?

LTE 인증은:

AKA 구조

RES = f(K, RAND)

K = USIM secret key

HSS만 동일 K 보유


👉 IMSI만 맞는다고 통과 불가 👉 반드시 K 필요

즉:

> IMSI spoofing만으로는 LTE attach 불가능



(2G와 결정적으로 다른 점)


---

📌 Spec 근거

3GPP TS 24.301

Authentication 실패 시

Network shall abort attach procedure

Identity mismatch 시

Attach reject shall be sent


---

📌 실무/보안 관점 추가 설명

fake IMSI 보내면?

HSS signaling 낭비

rate limit / DoS 탐지

blacklist 가능


일부 코어망:

repeated failure → temporary barring



---

📌 예외 케이스 (중요)

1️⃣ Emergency attach

인증 생략 가능

IMSI 없어도 허용


2️⃣ Fake BTS/IMSI catcher

네트워크가 가짜라면? → UE는 속을 수 있음 → 하지만 "진짜 LTE 코어"에서는 통과 불가



---

📌 한 줄 결론

> LTE에서 Identity Response에 fake IMSI를 보내면, 인증 단계(AKA)에서 반드시 실패하며 Attach/TAU는 Reject 된다. 정상 접속이나 권한 획득은 구조적으로 불가능하다.




---

원하면 추가로: ✅ Wireshark NAS 메시지 실제 로그 예 ✅ Attach Reject cause code 표 정리 ✅ 2G/3G와 비교 (왜 2G는 IMSI spoof 가능했는지) ✅ IMSI catcher 동작 원리

어느 쪽 더 궁금하세요?
