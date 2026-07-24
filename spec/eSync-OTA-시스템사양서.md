# eSync 기반 OTA 시스템 사양서

| 항목 | 내용 |
|------|------|
| **문서 성격** | eSync 기반 OTA 시스템 사양 |
| **상태** | Draft |
| **적용 대상** | eSync Server · TGU(eSync Client·Agent) · 대상 제어기 |
| **용어 기준** | 본문은 eSync 용어를 사용한다. 우리 시스템 용어 대응은 [용어 대응표](용어-대응표.md). |

건설기계 제어기의 무선 소프트웨어 업데이트(OTA)를 Excelfore **eSync** 위에 정의한다.
OTA 워크플로는 하나이며, 대상 제어기의 능력에 따라 일부 단계가 달라진다. 본 문서는 **EPOS-10i**를 기준 사례로 상술한다.
읽는 순서는 **아키텍처 → 워크플로 → 세부 설계**다.

---

## 1. 아키텍처

시스템을 **클라우드 · 통신 · 장비** 세 계층으로 나눈다.

![그림 1 · 전체 구조](assets/조감도-전체구조.svg)

*[그림 1] 전체 구조*

### 1.1 세 계층 개요

- **클라우드** — eSync Server(백엔드)와, 이를 쓰는 운영·비즈니스 프론트엔드, 소스·신원·서명 서비스.
- **통신(Air Interface)** — eSync Server와 eSync Client 사이의 채널.
- **장비** — eSync Client·eSync Agent와 플래시 대상 제어기.

### 1.2 계층별 구성

#### 1.2.1 클라우드

- **eSync Server** — 캠페인·타겟팅, Component·Package 데이터베이스, 서명, 감사를 담당한다.
- **OTA 운영 플랫폼(Operator UI)** — 캠페인·패키지를 운영하는 단일 콘솔. RBAC로 딜러·AS에 부분 개방한다.
- **GPDM** — Component와 SW BOM의 출처 원장(§3.1.2).
- **KMS** — 서명 개인키를 보관한다. 개인키는 KMS 밖으로 나오지 않는다.
- **펌웨어 파일 서버** — 펌웨어 bulk를 HTTPS로 배포한다.

#### 1.2.2 통신 (Air Interface)

- **eSync Messaging Protocol** — 제어·메타데이터 채널.
- **전송** — 이 메시지를 실어 나르는 전송(예: AWS IoT Core, MQTT)은 교체 가능하다.
- **펌웨어 파일 서버(HTTPS)** — 통신 채널은 다운로드 위치만 전달하고, 기기가 펌웨어 bulk를 직접 내려받는다.

#### 1.2.3 장비

- **eSync Client**과 **eSync Agent**는 TGU 한 노드에 함께 있다.
- **대상 제어기(EPOS-10i)** — eSync Agent가 UDS on CAN으로 플래시한다.

### 1.3 신뢰 경계

- 서명은 클라우드(eSync Server)에서 이뤄지고, **eSync Agent가 이를 검증**한다.
- 검증에 필요한 신뢰앵커는 각 노드가 로컬 보유하여, 인터넷 없이도 서명을 검증한다.

### 1.4 제어기 특성별 변주

OTA 워크플로는 하나지만, 대상 제어기의 능력에 따라 검증·암호화·플래시 단계가 달라진다.

| 제어기 능력 | 무결성 검증 위치 | 페이로드 | 플래시 |
|------------|------------------|----------|--------|
| **EPOS-10i** (HSM 없음·서명검증/암복호화 불가·싱글뱅크·OSEK) | **TGU(eSync Agent)** | 평문 | UDS on CAN |
| HSM 보유 제어기 | 대상 제어기 자체 | Secure Flash(암호화) | (해당 사양 별도) |
| 이더넷 Linux/Android | 대상 자체(Verified Boot 등) | (해당 사양 별도) | (해당 사양 별도) |

EPOS-10i는 서명 검증·복호화를 스스로 하지 못한다. 이런 제어기는 페이로드가 **가장 가까운 능력 노드(TGU)까지만** 보호되며, TGU가 검증을 마친 뒤 평문으로 플래시한다.

---

## 2. 업데이트 워크플로 (EPOS-10i)

### 2.1 개요

소스(GPDM)에서 시작해 eSync Server에서 서명·패키징하고, mTLS로 TGU에 전달한 뒤, TGU가 검증·호환성 확인을 거쳐 EPOS-10i를 플래시한다.

![그림 2 · EPOS-10i 업데이트 워크플로](assets/epos-10i-워크플로.svg)

*[그림 2] EPOS-10i 업데이트 워크플로*

### 2.2 입력·빌드·소스

- 개발자는 소스를 **SVN(ALM)**에 커밋하고, **Jenkins(Build)**가 빌드해 바이너리를 생성한다.
- 빌드 산출물과 SW BOM은 **GPDM(PLM)**에 등록된다. SW 벤더 납품분도 GPDM으로 들어온다.
- **GPDM에서 릴리스된 산출물만** eSync Server로 업로드된다. 업로드 단위는 **컴포넌트**(§2.3)다.
- 업로드에 필요한 입력:

| 입력 | 내용 |
|------|------|
| 바이너리 | 대상 제어기용 펌웨어 이미지 |
| manifest.xml | 대상 노드·버전·롤백·호환성 메타데이터 (§2.3.2) |
| Component 메타 | `name`, `nodeName`(대상 엔드포인트), `type`(담당 Agent) |
| 버전 메타 | `version`, 이전 버전 대비 델타 여부 |

### 2.3 Component 구성

#### 2.3.1 컴포넌트 패키징

컴포넌트는 다음을 담은 하나의 패키지(ZIP)다.

```
📦 eSync Component (ZIP)
├── 📄 manifest.xml       * 대상·버전·호환성 메타데이터
├── 📦 binary             * 펌웨어 이미지 (평문)
└── 📄 manifest_diff.xml  * (델타 업데이트 시에만)
```

서명과 CS Cert는 서버 서명 단계(§2.4)에서 컴포넌트에 결합된다.

#### 2.3.2 manifest.xml 필드

| 필드 | 내용 |
|------|------|
| 대상 노드(OMA node) | 업데이트 대상 엔드포인트 식별자 |
| type | 담당 eSync Agent(핸들러) |
| version | 소프트웨어 버전 |
| rollback 참조 | 다운그레이드 방지를 위한 이전 버전 참조 |
| delta 참조 | 델타 업데이트의 기준 버전 |
| Target HW ID | 적용 가능한 품번·HW 버전 (호환성 판정용, §2.8) |

#### 2.3.3 데이터 모델

- **Component** — `name`, `nodeName`(전개 범위에서 유일한 엔드포인트 ID), `type`(담당 Agent), `versions`.
- **ComponentVersion** — `version`, `deltaSha`(바이너리의 SHA-256), `dependencies`.

### 2.4 서명

#### 2.4.1 알고리즘과 키

- **서명 알고리즘:** ECDSA, 곡선 `secp256r1(P-256)`, 해시 `SHA-256`.
- **서명 주체:** eSync Server. 서명에 쓰는 개인키는 **CS Cert(코드서명 리프 인증서)의 개인키**이며 **KMS 안에만** 있다. eSync Server는 KMS에 서명을 요청한다.
- **서명 대상:** 컴포넌트(ZIP) 전체. manifest.xml이 컴포넌트 안에 있으므로 **하나의 서명이 바이너리와 메타데이터를 함께 보증**한다.

> **근거(설계):** 개발자·벤더에게 서명 키를 배포하지 않고 서명을 eSync Server로 일원화한다. 분산된 서명 키의 유출로 가짜 펌웨어가 서명되는 위협을 없앤다. "누가 만든 펌웨어인가"는 서명 요청·승인 기록(GPDM 릴리스·감사 로그)으로 담보한다.

#### 2.4.2 서명 절차

![그림 3 · 서명 발행 시퀀스](assets/서명-발행-시퀀스.svg)

*[그림 3] 서명 발행 시퀀스*

1. GPDM이 릴리스된 컴포넌트(ZIP: manifest.xml + binary)를 eSync Server에 업로드한다.
2. eSync Server가 컴포넌트를 조립하고 `SHA-256`으로 해시한다.
3. eSync Server가 KMS에 서명을 요청한다(digest 전달).
4. KMS가 CS Cert 리프 개인키(P-256)로 ECDSA 서명한다.
5. 서명값을 eSync Server로 반환한다.
6. eSync Server가 서명값과 **CS Cert(체인 포함)를 컴포넌트에 동봉**한다.
7. 서명된 컴포넌트를 저장한다.

### 2.5 Package 구성

- **Package**는 하나 이상의 Component를 묶는다.
- `componentRules`로 설치 순서·원자성을 지정하고, `atomicRollback`으로 실패 시 원자적 롤백을 정의한다.
- Campaign이 Package를 대상 장비군에 배포한다.

### 2.6 전달 (mTLS)

eSync Server와 eSync Client는 **상호 TLS(mTLS)**로 연결한다.

![그림 4 · mTLS 핸드셰이크 시퀀스](assets/mtls-핸드셰이크-시퀀스.svg)

*[그림 4] mTLS 핸드셰이크 시퀀스*

- 양측은 각자 X.509 인증서와 개인키를 보유한다. eSync Client 인증서의 `CN`은 시스템 시리얼 번호다.
- 상대 인증서를 **신뢰 체인(Root→Intermediate)·유효기간·CN·OCSP**로 검증한 뒤 TLS를 수립한다.
- 펌웨어 bulk는 이 채널로 보내지 않는다. 통신 채널은 다운로드 위치만 전달하고, 기기가 펌웨어 파일 서버에서 HTTPS로 직접 내려받는다.

### 2.7 검증 (eSync Agent)

EPOS-10i는 서명을 스스로 검증하지 못하므로, **TGU의 eSync Agent가 대신 검증**한다.

1. Root CA 공개키(신뢰앵커)로 동봉된 **CS Cert 체인**(Root→Code Sign CA→CS Cert)을 검증한다.
2. CS Cert의 공개키로 컴포넌트 **서명을 검증**한다(ECDSA P-256).
3. 수신 컴포넌트를 다시 해시해 서명된 값과 **일치하는지 대조**한다.
4. 하나라도 실패하면 폐기하고 플래시하지 않는다.

### 2.8 호환성 확인

정품이지만 비호환인 펌웨어의 오설치를 막기 위해, 서명이 아니라 **메타데이터로 호환성을 판정**한다.

- eSync Agent가 **UDS 0x22(ReadDataByIdentifier)**로 EPOS-10i의 실측 HW ID(품번·HW 버전)를 읽는다.
- manifest.xml의 `Target HW ID`와 대조한다. 불일치면 플래시하지 않는다.

### 2.9 플래시 (EPOS-10i)

- **세션 게이트:** eSync Agent가 **UDS 0x27(SecurityAccess)**로 프로그래밍 세션을 연다.
- **전송:** UDS `0x34(RequestDownload)` / `0x36(TransferData)` / `0x37(RequestTransferExit)`로 **평문** 바이너리를 전송한다.
- **활성화:** 플래시 성공 후 **원자적 커밋**으로 새 이미지를 활성화하고, 상태를 보고한다.
- EPOS-10i는 인증서 기반 인증(UDS 0x29)을 지원하지 않으므로 사용하지 않는다.

### 2.10 런타임 시퀀스

![그림 5 · EPOS-10i 런타임 시퀀스](assets/epos-10i-런타임-시퀀스.svg)

*[그림 5] EPOS-10i 런타임 시퀀스*

업데이트 확인 → 다운로드 → 검증 → 호환성 확인 → 세션 게이트 → 플래시 → 상태 보고의 전 과정을 나타낸다.

---

## 3. 세부 설계

### 3.1 서명 주체와 출처

#### 3.1.1 중앙 서명

- 디바이스(eSync Agent)가 검증하는 서명은 **eSync Server 서명 하나**다.
- 서명 개인키는 KMS 안 CS Cert 리프 개인키이며 반출되지 않는다.

#### 3.1.2 출처 관리 (GPDM)

- **GPDM**(PLM)이 SW와 SW BOM의 출처 원장이다. 작성자·승인·버전 관리를 GPDM이 담당한다.
- eSync Server는 **GPDM에서 릴리스된 산출물만** 수용한다. GPDM은 Component·SW BOM의 유일 소스로 정책화한다.
- GPDM의 SW BOM이 그대로 업데이트의 SBOM이 된다.

### 3.2 PKI 구성

![그림 6 · PKI 계층](assets/pki-계층.svg)

*[그림 6] PKI 계층*

#### 3.2.1 인증기관 계층

- **OEM Root CA** — 최상위 인증기관. 개인키(서명키)는 오프라인 보관한다.
- **Device CA** — 디바이스 인증서 발급기관. 개인키는 KMS에 보관한다.
- **Code Sign CA** — 코드서명 인증서 발급기관. 개인키는 KMS에 보관한다.

eSync Server는 인증기관이 아니다. **CS Cert(코드서명 리프)의 개인키로 서명하는 서명자**이며, 디바이스 인증서에 대해서는 **검증자**다.

#### 3.2.2 인증서 프로파일

| 인증서 | 발급기관 | 주요 필드 |
|--------|----------|-----------|
| **CS Cert** (코드서명 리프) | Code Sign CA | `KeyUsage=digitalSignature`, `EKU=codeSigning`, 단수명(회전) |
| **디바이스 인증서** (eSync Client) | Device CA | `CN=Serial`, `KeyUsage=digitalSignature`, mTLS 클라이언트 인증 |

#### 3.2.3 신뢰앵커와 체인 검증

- 각 검증 노드는 **Root CA 공개키만 신뢰앵커로** 공장 주입받는다.
- 서명 검증에 필요한 **CS Cert와 중간 CA 인증서는 컴포넌트에 동봉**된다. 그래서 인터넷 없이도 `Root → Code Sign CA → CS Cert` 체인을 구성해 검증할 수 있다.
- CS Cert(리프)는 단수명으로 회전하되, 신뢰앵커(Root)는 고정되므로 필드 재프로비저닝이 필요 없다.

### 3.3 mTLS 구성

- **eSync Client 보유:** 자기 디바이스 인증서(`CN=Serial`)·개인키, Root CA 신뢰앵커.
- **eSync Server 보유:** 자기 인증서·개인키, Device CA 신뢰앵커(클라이언트 인증서 검증용).
- 전송을 담당하는 연결점(예: AWS IoT Core)은 디바이스 인증서 체인을 검증할 수 있도록 **Device CA 인증서를 등록**해 둔다. 최초 접속 시 디바이스가 자동 등록(JITP)된다.

### 3.4 프로비저닝

- **디바이스 인증서:** 제어기 공정(Tier-1)에서 발급한다. 제어기가 키쌍을 자체 생성하고 CSR을 만들어 Device CA에 제출하면, `CN=Serial`로 인증서가 발급된다. 개인키는 제어기 밖으로 나오지 않는다.
- **신뢰앵커:** 공장에서 Root CA 공개키를 제어기에 주입한다.

### 3.5 EPOS-10i 특성 대응

#### 3.5.1 무결성 검증 대행

EPOS-10i는 서명 검증·복호화 능력이 없다. 무결성 검증은 **TGU의 eSync Agent가 대행**하며(§2.7), TGU까지 검증이 완료된 뒤 평문으로 플래시한다.

#### 3.5.2 플래시 세션 보호

TGU와 EPOS-10i 사이 구간은 UDS 평문 전송이다. 임의 노드의 플래시를 막기 위해 **UDS 0x27(SecurityAccess)**로 프로그래밍 세션을 보호한다. 시드-키 방식은 암호 하드웨어를 요구하지 않아 EPOS-10i에서 구현 가능하다.

#### 3.5.3 복구와 롤백 방지

- **복구:** EPOS-10i는 싱글뱅크라 A/B 대체 뱅크가 없다. 보호 영역의 **복구 부트로더(Insurance Kernel)**를 두어, 플래시 실패 시에도 재플래시할 수 있게 한다. 플래시는 **원자적 커밋**(전송·검증 완료 후 활성화)으로 처리한다.
- **롤백 방지:** manifest.xml의 rollback 참조와 버전을 기준으로, **TGU가 강등(다운그레이드) 배포를 차단**한다.

---

## 4. 미정 항목 (TBD)

- eSync Client 인증서의 유효기간.
- GPDM ↔ eSync Server 연동 방식 — 설계플랫폼팀과 협의.
- UDS 0x27 시드-키 비밀의 관리·배포 정책.
- SW 벤더 납품 경로(SRM)의 상세.
- 장수명 대응을 위한 암호 알고리즘 식별자(크립토 애자일리티).
