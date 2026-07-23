# eSync End-to-end Security — 문답 정리

> 대상 문서: [End_to_end_Security_in_eSync.pdf](../End_to_end_Security_in_eSync.pdf) (Excelfore, 29쪽)
> 목적: PDF 내용에 대한 질의응답을 간결·명확하게 기록.

## 근거 표기 규약

| 표기 | 의미 |
|------|------|
| 📄 **[문서]** | PDF 본문·다이어그램에 실제로 있는 내용 |
| 🌐 **[외부확인]** | 웹서치 등 문서 밖에서 검증·보충한 내용 |
| 💬 **[추론]** | 위 둘로 확정 안 되는 판단·해석 (근거 명시) |

---

<!-- 여기서부터 문답을 누적한다. 새 문답은 아래에 이어서 추가. -->

## Q1. eSync 보안 레벨에서 "Software Components"는 무엇이고 어떻게 구성되는가?

**정의 (📄문서)** — 차량 ECU에 최종 설치될 실제 SW/펌웨어 **바이너리 페이로드**. OTA 파이프라인의 최초 입력물이며, "Security Levels across eSync" 다이어그램(4쪽) 맨 왼쪽 큐브 아이콘.
- 출처: ECU 벤더/OEM 제공, **벤더 개인키로 서명된 바이너리**로 업로드 (13쪽).
- 무결성: 컴포넌트마다 승인 시 부여된 **Secure Hash**(SHA-256 권장)가 평생 따라붙음 (13쪽).

**업로드·저장 흐름 (📄문서)**
```
Software Components → Authorised User → USER INTERFACE → Component Uploader
   → [Components DB] → [Packages DB] → [Campaigns DB]
```
서명 처리: 벤더 서명 업로드 → 서버 검증 → Test Binary → 승인 시 **서버 개인키로 재서명** → Signed Component Database.
원칙: 차량은 **서버 서명 콘텐츠만 신뢰**, 개발자(벤더) 서명은 인정하지 않음 (8쪽).

**3계층 구성 (🌐외부확인 — eSync 백엔드 API 스펙 6.8.1 스키마)**
PDF는 3개 DB의 포함관계를 본문에 명시하지 않음. API 스키마로 확인:

| 계층 | 스키마 핵심 필드 | 역할 |
|------|------|------|
| Component | `name, nodeName, versions` | 버전 단위 SW 요소, ECU(노드) 대상 |
| Package | `components, componentSequence, componentRules, atomicRollback` | Component 묶음 + 순서·의존성·원자적 롤백 |
| Campaign | 클라이언트/디바이스 연결·배포·상태 | 대상 차량군 실제 배포 실행 단위 |

→ **Component → Package → Campaign** 조립 구조.

**요약 (💬추론)** — Software Component = "ECU 하나에 들어갈 서명된 버전형 바이너리"라는 최소 무결성·배포 원자. 보안 핵심은 컴포넌트별 고정 Secure Hash + 이중 서명(벤더=검증용 / 서버=신뢰용).

## Q2. Linux OS 기반 AP가 대상이면 Software Component가 바이너리가 아닐 수도 있나? (구형 OTA의 "앱+설치스크립트" 대비)

**결론** — 반드시 raw 펌웨어 바이너리일 필요 없음. **페이로드 형식·의미는 이를 처리하는 Update Agent가 정의**하며, Linux AP는 "앱+설치 로직/패키지" 형태 컴포넌트가 정상.

**문서 근거·한계 (📄문서)** — 서술은 "binary/flash" 중심이나 비(非)플래시를 열어둠:
- OS 불문 전제: *"devices ... running different operating systems"* (3쪽)
- *"flash instructions or commands may be part of the update ... Post flash procedure ... **is not part of the eSync Alliance Technical Specification**"* (18쪽)
- 18쪽 다이어그램 *"Do ECU Vendor specific post-install operations"*
- Agent 내부에 `Delta Reconstruction / Update Controller / Device Programming Interface` (5·26쪽)
- → **설치 방식은 표준 밖·Agent/벤더 정의.** 단, 백서에 Linux AP 앱/스크립트 사례는 **명시 없음**.

**외부확인 (🌐)**
- `Component.type` = 파일포맷이 아니라 **"담당 update agent 주소"** (API 스펙 6.8.1). 페이로드 의미는 Agent 소관.
- eSync Agent는 Android/Linux/QNX/AUTOSAR/Integrity/Erika/FreeRTOS 등에 구현, 스펙은 "어떤 OS에서도 구현 가능"하도록 인터페이스만 규정.
- Agent SDK: edge device별 Agent를 직접 구현 → **적용 방식은 Agent 구현체가 정의.**
- 표준이 강제하는 불변식: **설치 전·후 root of trust까지 bit-level 검증.**

**정리 (💬추론)** — eSync가 고정하는 것은 *페이로드 형식*이 아니라 *신뢰(서명/해시)+전달 경로 보안*. 따라서 Linux AP에서 "앱 패키지 + 설치/후처리 지시"를 컴포넌트로 구성하는 것은 eSync 모델과 부합. 프로젝트 반영 시 eSync v2.x 스펙 원문의 payload 정의 재확인 권장.

## Q3. Agent가 TGU에 있고, 대상이 내부 이더넷망의 Linux/Android 앱이면?

**핵심** — "업데이트의 위·변조 검사를 **실제로 프로그램이 실행되는 제어기(끝단)까지** 끌고 가느냐"의 문제. 서명을 검사하는 Agent가 있는 지점이 곧 **검사가 끝나는 지점**.

**문서 근거 (📄문서)**
- 게이트웨이 = eSync Client 자리 *(끝단: HU, T-Box, ECU)* (3쪽).
- *"payload is secured to the last secure hop node ... **if the ECU is resource-constrained**"* (3쪽) → **검사를 중간에서 끝내는 건 "제어기가 암호 연산을 못 할 때"의 예외**.
- *"eSync Agent also verifies the software signature ... **up to the last mile**"* (25·26쪽) → **검사를 수행하는 Agent가 있는 곳까지만 위·변조가 걸러진다**.

**두 가지 구성 (💬추론)** — Linux/Android 제어기는 암호 연산이 가능 → 위 "예외"에 해당 안 됨:

| | ① 권장: 대상 제어기가 자체 Agent 탑재 | ② 대안: Agent를 TGU에 두고 끝냄 |
|---|---|---|
| 역할 | TGU=Client/중계, 대상 제어기=Agent | TGU=Client+Agent |
| 서명 검사 | 실제 실행 제어기에서 수행 → **끝단까지 검사됨** | TGU에서 검사가 끝남 |
| TGU→앱 구간 | Client↔Agent 간 TLS(표준 안) | **eSync 표준 밖 → 구현자가 별도로 보안 책임** |
| 평가 | 26쪽 모델과 일치 | 암호 가능한 제어기엔 **권장되지 않음**(보증 축소) |

**구간별 보안 장치 (📄문서)** — Server↔Client: TLS+X.509 인증서+OCSP(24쪽) / 차량 내부: X.509 TLS 버스 + 차량 내 CA(OEM·Tier1·Excelfore) + Message Broker 등록(26·27쪽).
**여러 제어기 동시 관리 (🌐)** — Alliance: "Agent 하나가 여러 대(several devices)를 동시 업데이트 관리" → **이더넷 하위 여러 제어기를 한 곳에서 관리하는 것 자체는 정상.** 관건은 각 제어기의 Agent를 지휘하느냐(①), TGU에서 검사를 끝내느냐(②).

**결론** — 대상이 암호 가능한 Linux/Android 제어기면 **①(그 제어기에 Agent를 두어 서명 검사를 앱까지)**이 정석. ②는 "대상이 Agent를 못 올릴 때"의 대안이며, 이때 TGU→앱 구간은 별도 보안이 필요.
⚠️ 한계: 백서에 존/도메인 컨트롤러·자동차 이더넷(SOME/IP) 위 Agent 배치는 미명시. eSync v2.x 스펙 원문 확인 권장.

## Q4. Agent를 전부 게이트웨이(Client 위치)에 집중하고 CAN RTOS ECU + Ethernet Linux + Ethernet Android를 모두 담으려면 Software Components를 어떻게 구성?

**전제 검증 (🌐외부확인)** — 게이트웨이 집중 Agent는 eSync 1급 패턴:
- Agent는 "device 내부 or **인근 네트워크 노드(도메인 컨트롤러)**"에 상주 가능.
- Spec 1.0: 자원부족 ECU는 "**Agent를 가장 가까운 도메인 컨트롤러에 배치**"하는 메커니즘 제공.
- "**Updates through the gateway ECU using UDS/DoIP over CAN or Ethernet.**"

**핵심 메커니즘 (📄문서/API)** — `Component.type` = "담당 update agent 주소". 게이트웨이에 **여러 Agent 핸들러**를 올리고 컴포넌트 `type`으로 라우팅. 서버 서명 → 게이트웨이 Agent가 설치 전 위·변조 검사.

**대상 3종 Component 설계 (💬추론)**

| 대상 | type(핸들러) | 페이로드 | 전달 | 위·변조 검사가 끝나는 지점 / 끝단 검사 |
|------|------|------|------|------|
| CAN RTOS 엔진 ECU(비암호) | `uds-can-agent` | 펌웨어(+델타) | UDS/DoIP over CAN(ISO14229) | 게이트웨이에서 종료 → **정상**(3쪽 예외). UDS SecurityAccess/SecOC 보강 |
| Ethernet Linux(암호가능) | `linux-agent` | 패키지/컨테이너/A-B+스크립트 | TLS over Ethernet | 게이트웨이 검사 + **대상서 dm-verity/보안부팅으로 끝단까지 자체 확인** |
| Ethernet Android(암호가능) | `android-agent` | APK/OTA zip | TLS over Ethernet | 게이트웨이 검사 + **APK 서명/Verified Boot로 끝단까지 확인** |

조립(API 스키마): 3 Component → Package(`componentSequence` 순서, `packageDependencies` 의존, `atomicRollback` 롤백) → Campaign 배포.

**감수/보완점 (📄+💬)**
1. 검사가 끝나는 지점 이동 — CAN ECU는 게이트웨이에서 끝내는 게 정석이나, Linux/Android는 암호 가능 제어기라 게이트웨이에서 끝내면 그 뒤 구간이 표준 밖 → **대상의 자체 보안기능(Verified Boot/dm-verity/APK 서명)으로 끝단까지 필수 보완**(제어기에 풀 Agent 없이도 끝단 검사 확보).
2. 게이트웨이=단일 고가치 표적 → **HSM 키관리, 핸들러 간 격리, 보호 저장소** 필수.
3. 품질 우려는 "위치"가 아니라 "전달 로직"에 잔존 — 집중으로 중앙 관리는 개선되나 UDS스택/Android인스톨러/Linux패키지 적용 복잡도는 게이트웨이로 옮겨올 뿐, 핸들러별 검사 책임 유지.

**권장** — CAN ECU: 게이트웨이 UDS 플래시(표준 그대로). Linux/Android: 게이트웨이가 전달·지휘, **끝단 검사는 대상의 보안부팅/서명검사에 맡기는 혼합 방식**.
⚠️ 한계: 백서는 UDS/DoIP·존아키텍처·대리 Agent 미규정. eSync v2.x 스펙 원문 확인 권장.

## Q5. Software Components ↔ Authorised User 사이의 "열쇠"는 뭔가?

**아이콘 자체 (📄문서)** — 다이어그램(4쪽)의 빨간 열쇠(🔑)는 특정 키가 아니라 **"이 인터페이스에 보안이 걸려 있다"는 반복 표식**. 모든 화살표에 동일하게 붙음. 그림 아래 문장: *"security is present across all interfaces."*

**이 지점의 실제 보안 = 인가된 사용자의 인증서/개인키 (📄문서)** — "Important notes"(4쪽): *"certificates for eSync users (for **logging in, component signing, component upload**, campaign deployment)."* 이 키의 역할:
1. 로그인 인증 — "인가된 사용자"임 증명
2. 컴포넌트 서명(code signing) — 개인키로 SW 서명
3. 업로드 권한

**코드 서명 인증서 요건 (📄문서, 9·11쪽)** — 서명 사용자명=업로더 일치 / Developers 그룹 소속 / "sign" 권한 보유.

**정리 (💬추론)** — 그 "열쇠" = **"인가된 사람이 자기 개인키(코드 서명 인증서)로 이 SW에 서명한다"**는 의미. 이후 서버가 업로더를 확인하고 검증 후 **서버 키로 재서명**해 차량에 배포(Q1 이중 서명과 연결).

## Q6. TSP / PKI / Vehicle Identity Provider / OKTA / Custom Login / SSO 요약

모두 Top Level Architecture(5·19·23쪽) 왼쪽 **어댑터**로 eSync Server에 연결되는 외부 서비스·인증 연동.

**외부 서비스**
- **TSP** (📄) — Telematics Service Provider. 9쪽 명시, eSync Server에 SSL 인증서 제시. 🌐 차량-클라우드 통신·텔레매틱스 백엔드 운영 사업자.
- **PKI** (📄) — Public Key Infrastructure. 인증서 발급·서명·갱신·폐기(OCSP) 외부 제공자(7쪽). 구체 구성은 구현자 책임(스펙 범위 밖).
- **Vehicle Identity Provider** (📄언급/💬추론) — 다이어그램만 등장, 상세 미기술. 차량 신원(VIN/UID) 제공·확인 소스로 해석. 15쪽 Secure UID Verification의 신원 기준.

**사용자 로그인 연동 (OTA 도구 접근)**
- **OKTA** (🌐/📄언급) — 상용 클라우드 ID·접근관리(IAM)/SSO 서비스. 로그인 어댑터 예시(22쪽 OEM SSO+MFA 통합).
- **Custom Login** (📄) — OEM 자체(맞춤형) 로그인 방식 연동 선택지.
- **SSO** (📄) — Single Sign-On(통합 로그인). OEM 기존 SSO를 OTA 앱에 연동, MFA 기반 접근(22쪽).

**정리 (💬추론)** — TSP·PKI·Vehicle Identity=서버가 의존하는 외부 서비스(통신/인증서/차량신원). OKTA·Custom Login·SSO=사람이 관리도구에 로그인하는 방식(상용 IAM/자체/통합).

## Q7. `core-backend-appshack.ui-6.8.1-oas-3.1.json` API는 어느 경계의 API?

**결론** — 차량 쪽이 아니라 **클라우드 내부 "운영자 웹 콘솔(프론트엔드) ↔ eSync Server 백엔드" 경계**. 백서 §19 **"Between Server and Application"**, 5·19쪽 다이어그램의 **User Interface ↔ eSync Server** 링크.

**근거 — 스펙 메타데이터 (🌐)**
- 제목: *"eSync Core **Operator UI** Web Service API"*
- 설명: *"providing **operator support** ... consumed by a **front-end** ... managing eSync update processes."*
- 서버 경로: `omaui/sapi/v1` (OTA 관리 UI / server API)

**리소스도 관리 평면 (🌐)** — 203경로 상위: campaigns, componentItems/cdb, packages, binary/operatorBinaries, installations, vdb/device, userManagement, workflow, telemetry — 전부 콘솔의 관리 작업(업로드·캠페인·사용자·설치현황). Q1 데이터 모델의 출처.

**구분 (📄문서)** — 이 API ≠ Server↔Client(차량, eSync Messaging/MQTT, 24쪽) ≠ Client↔Agent(차량내부 TLS 버스, 26·27쪽).

**정리 (💬추론)** — "OTA 관리 콘솔이 eSync Server를 조작하는 클라우드 내부 백오피스 API". 차량 전송 프로토콜 아님.
⚠️ 스펙에 인증 스킴 비어있음(`securitySchemes: []`)+CSRF만 존재 → 실제 접근 인증(SSO/OKTA/MFA, 22쪽)은 이 OpenAPI 밖 게이트웨이/세션 계층 처리로 추정.

## Q8. Client와 Agent가 모두 TGU에 있으면 인증서도 같은가?

**결론** — **아니오. 같은 하드웨어라도 인증서는 별개.** 둘은 하드웨어가 아니라 역할(신원)로 구분되고 서로를 인증하는 관계이므로 각자 자기 인증서 필요.

**근거 (📄문서)**
1. 상호 인증(26·27쪽): Client는 "Agent에 발급된 X.509", Agent는 "Client에 발급된 X.509"를 각각 검증 → 같은 인증서 공유 시 상호 인증 불성립. 27쪽 다이어그램도 Client/Broker/Agent 각각 별도 인증서 아이콘.
2. CN(주체)이 다름(10쪽): Client CN=시스템(TGU) 시리얼 / Agent CN=엔드포인트(ECU) 시리얼 → 주체·키 쌍 상이.
3. 검증 경계 상이: Client는 서버와 인증(OCSP, 24쪽), Client↔Agent는 OCSP 없음(15쪽).

**공유 가능한 것 (💬추론)** — 발급 CA(신뢰 뿌리)는 공유 가능. 27쪽 차량 내 CA(in-vehicle CA)가 Client·Agent·Broker에 각각 발급 → **"뿌리(발급자) 공통, 잎(개별 인증서·CN·키) 별개"**.

| 구분 | Client 인증서 | Agent 인증서 |
|------|------|------|
| CN | 시스템(TGU) 시리얼 | 엔드포인트/ECU 시리얼 |
| 키 쌍 | 별개 | 별개 |
| 검증 경계 | 서버(OCSP) | Client(OCSP 없음) |
| 발급 CA | 공통 가능(차량 내 CA) | 공통 가능(차량 내 CA) |

⚠️ 실제로 Client+Agent를 한 프로세스로 합치면 로컬 IPC로 상호 TLS 간소화·생략 최적화 가능하나, 백서는 둘을 별개 인증 주체로 규정 → 표준 밖 구현 선택, 별도 검토 필요.

## Q9. Dashboard의 목적 — 서버 관리자용인가, OTA 서비스 비즈니스 프론트엔드인가?

**전제 (📄문서)** — 백서는 Dashboard를 4쪽 다이어그램 박스로만 표시, 목적·대상 미정의. 아래는 다이어그램 위치+제품정보 기반 해석.

**위치 (📄문서)** — Dashboard는 **eDiag Server + MongoDB와 한 묶음(오른쪽=데이터 수집·진단·리포팅)**, 왼쪽(업데이트 관리: Uploader·Packages·Campaigns)과 대비. eSync 정의(3쪽): "software updating **and data gathering**". → Dashboard = 수집 데이터·진단·현황 시각화 면.

**제품 확인 (🌐)** — Excelfore: eDiag(OTX/SOVD 원격진단), eDatX("real-time fleet monitoring, widget-based 시각화·분석"). → 운영자용 모니터링/분석 대시보드.

**직접 답 (💬추론)** — 이분법상 어느 쪽도 딱 아니고 **"OTA·진단 서비스 운영 프론트엔드"에 가까움**:

| 후보 | 판정 |
|------|------|
| Server 관리자용(저수준 설정) | ✕ 그건 Operator UI/API(Q7) 담당 |
| OTA/진단 운영 프론트엔드(모니터링·진단·현황) | ○ 부합 |
| 최종 소비자 상용앱 | ✕ 내부 운영자 도구, 백서 범위 밖 |

**정리** — 관리(설정·배포) 평면=Operator UI+`omaui/sapi`(Q7) / 관찰(모니터링·진단) 평면=Dashboard+eDiag+MongoDB. **Dashboard는 후자(운영자용 모니터링/분석 프론트엔드).**
⚠️ 백서에 명문 정의 없음 → Dashboard가 eDatX인지 Operator UI 리포팅탭인지는 벤더 확인 권장.

## Q10. CDN의 용도·약어, 그리고 AWS S3가 CDN 역할을 할 수 있나?

**약어 (🌐)** — **CDN = Content Delivery Network**(콘텐츠 전송 네트워크). 엣지 캐시로 대용량 파일을 근거리에서 빠르게 전송. ※ 백서는 CDN을 풀어쓰지 않고 사용(19~20쪽).

**eSync에서 용도 (📄문서)** — **대용량 업데이트 바이너리 다운로드를 서버에서 분리해 CDN에 맡김**.
- 16쪽: Client가 "Download Update Binary"를 CDN에서 pull. 서버는 버전/유무만 통지.
- 19~20쪽 Secure CDN Workflow: ①메타데이터는 TLS로 전달(CDN 비보안 시 Payload Encryption Key 동봉) ②페이로드는 CDN이 HTTPS면 그대로, 아니면 그 키로 암호화 후 전송.

**보안 설계 (📄문서)** — 전달물은 이미 **서명+암호화된 페이로드**. CDN 비보안이어도 내용 노출·위변조 안 됨 → **CDN은 "바이트만 나르는" 신뢰 불필요 요소**.

**AWS S3가 CDN 역할? (🌐)** — 엄밀히 아니나 "다운로드 호스트" 요건은 S3 단독도 충족 가능. 진짜 CDN은 CloudFront.

| | S3 | CloudFront |
|---|---|---|
| 정체 | 오브젝트 스토리지(오리진) | CDN(엣지 분산) |
| HTTPS 서빙 | 가능 | 가능 |
| 엣지캐싱·글로벌저지연·DDoS완화 | ✕/제한적 | ○ |

- S3는 다운로드 오리진 역할 단독 수행 가능(+페이로드 암호화라 신뢰 불필요).
- 대규모 차량 대상 엣지·확장 원하면 **CloudFront(=CDN)+S3(=오리진)** 정석. 다이어그램 "CDN"↔CloudFront, 저장소↔S3.

**정리 (💬추론)** — CDN=대용량 바이너리 전송 분리 계층, 서명+암호화로 신뢰 불필요. AWS는 S3 단독도 호스트 가능하나 CDN 이점 원하면 CloudFront+S3.
