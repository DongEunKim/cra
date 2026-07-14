# eSync 기반 독립 OTA 플랫폼 — 기획 노트

| 항목 | 내용 |
|------|------|
| **성격** | **기획 노트 (비확정 · 탐색 단계)** — 확정 사양 아님 |
| **상태** | Draft |
| **관계** | 확정 사양은 [DOC-10](../docs/10-시스템장비-보안아키텍처/시스템장비-보안아키텍처명세서.md)·[DOC-20](../docs/20-업데이트사양/업데이트-보안명세서.md). 본 노트는 그 위의 **플랫폼 기획**이며, 결정이 서면 해당 사양으로 승격한다. |
| **용어 기준** | [DOC-00 용어집](../docs/00-공통/용어집.md) |

> 본 노트는 **확정 사양이 아니라 기획 정리**다. 정식 문서 지도(스파인)에 포함하지 않는다. substrate 연동 등 미확정 사항은 TBD로 둔다.

---

## 1. 배경

- **eSync(Excelfore) 분석·번역:** [End_to_end_Security_in_eSync_KO.md](End_to_end_Security_in_eSync_KO.md)
- **조사 결론(요지):**
  - eSync는 **전송 독립(transport-agnostic) 설계**다(특허 US 10,708,360 · 브로커 버스 WO2019168907A1). eSync 메시지(JSON)를 그때의 메시징 프로토콜에 캡슐화한다.
  - AWS 연동은 **MQTT 토픽("pipe") 매핑**으로 이뤄지고(Hero MotoCorp 양산), **Azure(MCVP)와도 연동** → 특정 클라우드 비종속.
  - 펌웨어 **bulk는 MQTT가 아니라 Secure CDN/HTTPS(아웃오브밴드)**로 전달.

## 2. 핵심 정의 — 독립 OTA(eSync) 플랫폼

> 연결 substrate와 **논리적으로 분리된 독립 플랫폼**이자 **OTA 오케스트레이션의 단일 권위**.

substrate(AWS IoT Core·TMS 백엔드 등)는 **전송·디바이스 연결·1홉 인증·도달성**을 제공하는 하위 계층일 뿐이고, **판단·신원 권위는 플랫폼**에 있다.

## 3. 플랫폼이 소유하는 것 (우리 설계 범위)

- 배포 **캠페인·타겟팅** (호환성 관문 포함 → DOC-20 §3.4)
- **중앙집중 서명** (KMS 서명 서비스 · CS-IC 단일 서명) → DOC-20 §1.2
- **패키지·컴포넌트 DB**, 패키지 포맷(매니페스트·서명·Target HW ID) → DOC-20 §2
- **종단간 디바이스 신원 검증** — 디바이스 Dev-IC 신원을 **OEM Root CA로 플랫폼이 직접 검증**
- **감사·추적성**(CRA — 업데이트 기록·SBOM 등)
- **Secure CDN** (펌웨어 bulk 배포)

## 4. 범위 밖 — substrate 연동 (별도 팀 소관 · TBD)

기존 **AWS IoT Core**·**TMS 백엔드**는 각각 **다른 팀이 운영**하며(다른 AWS 계정 가능성 포함), 실제 연동은 **플랫폼 기획 이후 그 팀들과 별도 논의**한다. 아래는 현 설계 범위 **밖**이다.

- 디바이스 물리 연결·전송(MQTT/TLS, CoAP/DTLS), 1홉 전송 인증, 네트워크 도달성
- AWS IoT Core / TMS 백엔드와의 **연동 방식·메시지 라우팅·크로스계정 구성·브리지 여부**

## 5. 설계 원칙

1. **전송 독립** — 어떤 substrate가 실어오든 무관하게 동작한다.
2. **종단간 신원** — 플랫폼이 디바이스 Dev-IC 신원을 **독립 검증**한다(중간 계층·계정을 전적으로 신뢰하지 않음 = zero-trust, CRA 추적성에도 부합).
3. **substrate = pluggable, 플랫폼 = 권위.**

## 6. 플랫폼 ↔ substrate 계약 (인터페이스만; 구현 TBD)

| 방향 | 내용 |
|------|------|
| substrate → 플랫폼 | ① 전송독립 메시지 전달, ② 디바이스 연결·1홉 전송 인증, ③ 네트워크 도달성 |
| 메시지에 실려야 | 디바이스의 **종단간 Dev-IC 신원**(플랫폼이 OEM Root CA로 검증) |
| bulk | **Secure CDN(HTTPS)** — 플랫폼 소유, 디바이스가 직접 다운로드 |

## 7. 이미 확정되어 사양에 반영된 관련 결정

- **중앙집중 서명**(개발자·벤더 키 미보유, CS-IC 단일 서명) → DOC-20 §1.2 · DOC-10 §3.1 원칙6
- **서명·호환성 분리 + 호환성 관문** → DOC-20 §3.4 · DOC-10 §3.1 원칙5
- **인증서 수명·폐기(경계별)** → DOC-20 §1.4

## 8. 열린 과제 (플랫폼 기획 시 결정)

- **디바이스 종단간 신원 표현** — 인증서 제시(mTLS/DTLS) vs 서명 토큰(JWT 등) 중 무엇으로 플랫폼에 전달·검증할지
- **OCSP 범위 최종 확정** — 권고안(연결 경계=OCSP 스테이플링, 오프라인=만료+스테이플/CRL) 승인 여부
- **단수명 인증서 실제 기간** — 폐기 부담이 줄었으므로 여유 있게 설정
- **TMS 프로파일** — 전송이 TLS인지 DTLS인지, 상호 인증 여부, 버전(≥TLS/DTLS 1.2) 확인 필요
- **CoAP 바인딩** — eSync 기성 지원 확인 vs 게이트웨이/어댑터 제작
- **UDS DID 규약**(HW버전·품번 읽기) → DOC-30 후속
- **TGU ↔ 타겟 제어기(UDS/CAN) 디바이스 인증** — substrate가 대신 못 하는 구간, 열린 과제

## 개정 이력

| 버전 | 일자 | 내용 |
|------|------|------|
| v0.1 | 2026-07-15 | 초안: 독립 OTA 플랫폼 정의·소유 범위·범위 밖(substrate 연동 TBD)·설계 원칙·계약·열린 과제 정리. eSync 조사 결론 반영. |

## 참고

- eSync 원문 번역: [End_to_end_Security_in_eSync_KO.md](End_to_end_Security_in_eSync_KO.md)
- Integrating eSync with AWS IoT Core — Excelfore (blog)
- US Patent 10,708,360 — Transport agnostic communication between IoT client and broker (Excelfore)
- WO2019168907A1 — Broker-based bus protocol and multi-client architecture
- Excelfore × Microsoft Azure MCVP (multi-backend 근거)
