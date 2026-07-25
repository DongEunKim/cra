# OTA 시스템 사양서 (EPOS-10i)

| 항목 | 내용 |
|------|------|
| **문서 성격** | eSync 기반 OTA 워크플로 사양 |
| **상태** | Draft |
| **적용 대상** | eSync Server · TGU(eSync Client·Agent) · EPOS-10i |
| **관계** | [10 PKI 사양서](10-PKI-사양서.md) · [20 프로비저닝 사양서](20-프로비저닝-사양서.md)를 전제로 한다. |
| **용어 기준** | [00 용어집](00-용어집.md) |

건설기계 제어기의 무선 소프트웨어 업데이트(OTA)를 Excelfore eSync 위에 정의한다. OTA 워크플로는 하나이며, 대상 제어기의 능력에 따라 일부 단계가 달라진다. 본 문서는 **EPOS-10i**(HSM 없음·서명검증/암복호화 불가·싱글뱅크·OSEK)를 기준 사례로 종단간(End-to-End) 상술한다.

---

## 1. 개요·아키텍처

시스템을 **클라우드 · 통신 · 장비** 세 계층으로 나눈다.

![그림 1 · 전체 구조](assets/조감도-전체구조.svg)

*[그림 1] 전체 구조*

### 1.1 세 계층 개요

- **클라우드** — eSync Server(백엔드), 운영 프론트엔드, 소스·신원 서비스(GPDM·VLM·KMS).
- **통신(Air Interface)** — AWS IoT Core를 통한 eSync Server ↔ TGU 채널(§5).
- **장비** — TGU(eSync Client·Agent)와 대상 제어기(EPOS-10i).

### 1.2 계층별 구성

- **eSync Server** — 캠페인·타겟팅, Component·Package DB, 서명, 감사.
- **GPDM** — Component와 SW BOM의 출처 원장.
- **VLM** — 시리얼↔VIN↔장비 옵션 매핑, SBOM 이력.
- **KMS** — 서명 개인키 보관([10 §4](10-PKI-사양서.md)).
- **TGU** — eSync Client(오케스트레이션)와 eSync Agent(검증·플래시)를 한 노드에 둔다.
- **EPOS-10i** — eSync Agent가 UDS on CAN으로 플래시하는 대상.

### 1.3 신뢰 경계

- 서명은 클라우드에서 이뤄지고 **eSync Agent가 검증**한다.
- 검증 신뢰앵커(Root CA 인증서)는 각 노드가 로컬 보유한다([10 §3.1](10-PKI-사양서.md)).
- EPOS-10i는 검증 능력이 없어, 무결성 검증이 **TGU까지** 보장된다(§4.2). TGU→EPOS 구간은 평문이며 UDS 세션으로 보호한다(§2.2, §10).

### 1.4 제어기 특성별 변주

| 제어기 능력 | 무결성 검증 위치 | 페이로드 | 플래시 |
|------------|------------------|----------|--------|
| **EPOS-10i** (HSM 없음·검증/복호화 불가·싱글뱅크) | TGU(eSync Agent) | 평문 | UDS on CAN |
| HSM 보유 제어기 | 대상 제어기 자체 | Secure Flash(암호화) | (별도) |
| 이더넷 Linux/Android | 대상 자체(Verified Boot 등) | (별도) | (별도) |

**설치 경로**는 두 가지다. 같은 서명 Component/Package를 재사용한다.

- **무선(OTA):** eSync Server → 통신 계층 → TGU. 본 문서가 다룬다.
- **유선(DMS):** eSync Server/Dashboard → DMS(진단툴) → OBD-II → SGW/TGU 게이트웨이 → 대상. → [40 유선 업데이트 사양서](40-유선업데이트-사양서.md).

---

## 2. 종단간 업데이트 워크플로 (EPOS-10i)

### 2.1 개요

소스(GPDM)에서 시작해 eSync Server에서 서명·패키징하고, 캠페인으로 대상 장비를 선정한 뒤, AWS IoT Core를 통해 TGU에 전달한다. TGU가 검증·호환성 확인·설치 정책을 거쳐 EPOS-10i를 플래시하고 상태를 보고한다.

![그림 2 · EPOS-10i 업데이트 워크플로](assets/epos-10i-워크플로.svg)

*[그림 2] EPOS-10i 업데이트 워크플로*

발행 단계(빌드·서명·패키징·캠페인)는 §3·§4·§6에서, 전송은 §5에서, 실행 단계는 아래 §2.2에서 상술한다.

### 2.2 런타임 실행 단계

아래 번호는 [그림 3]의 시퀀스 번호와 일치한다.

![그림 3 · EPOS-10i 런타임 시퀀스](assets/epos-10i-런타임-시퀀스.svg)

*[그림 3] EPOS-10i 런타임 시퀀스*

1. **업데이트 확인** — eSync Client가 현재 버전을 보고하고 업데이트를 확인한다.
2. **캠페인 매칭** — 캠페인이 있으면 eSync Server가 다운로드 메타데이터를 내려준다(대상 선정 §6).
3. **다운로드 요청** — eSync Client가 펌웨어 파일 서버에 컴포넌트를 요청한다(§5.3).
4. **컴포넌트 수신** — 컴포넌트(ZIP)를 청크 해시로 무결성 확인하며 받아 보호 저장한다(§5.3).
5. **Agent 전달** — eSync Client가 컴포넌트를 eSync Agent로 전달한다(IPC).
6. **서명 검증** — eSync Agent가 체인·서명·재해시를 검증한다(§4.2).
7. **호환성 조회** — eSync Agent가 UDS `0x22`로 EPOS-10i의 HW ID를 읽는다(§6.2).
8. **HW ID 응답** — EPOS-10i가 HW ID를 반환한다.
9. **호환성 판정** — manifest의 호환성 정보와 HW ID를 대조한다(§6.2).
10. **세션 게이트** — eSync Agent가 UDS `0x27`로 프로그래밍 세션을 연다(§10).
11. **플래시** — UDS `0x34/0x36/0x37`로 평문 바이너리를 전송한다.
12. **결과 수신** — EPOS-10i가 플래시 결과를 반환한다.
13. **원자적 커밋** — eSync Agent가 새 이미지를 활성화한다(§8.2).
14. **상태 보고** — `INSTALL_COMPLETE`를 eSync Client 경유로 eSync Server에 보고한다(§9).

각 단계의 실행 시점은 설치 정책(§7)이 통제한다.

---

## 3. 컴포넌트·패키지·매니페스트

### 3.1 데이터 모델

- **Component** — 대상 엔드포인트용 소프트웨어 단위. `name`, `nodeName`(업데이트 엔드포인트 ID), `type`(담당 eSync Agent), `versions`.
- **ComponentVersion** — `version`, `deltaSha`(바이너리 SHA-256), `dependencies`.
- **Package** — Component 묶음. `components`, `componentRules`(§6.3), `atomicRollback`.

### 3.2 매니페스트 (manifest.xml)

#### 3.2.1 패키징

컴포넌트는 다음을 담은 ZIP이다. manifest.xml은 컴포넌트 안에 포함되어 서명 대상에 함께 들어간다(§4.1).

```
📦 eSync Component (ZIP)
├── 📄 manifest.xml
├── 📦 binary
└── 📄 manifest_diff.xml   (델타 시)
```

#### 3.2.2 필드

| 필드 | 용도 |
|------|------|
| 대상 노드 · type | 업데이트 엔드포인트와 담당 eSync Agent |
| version | 소프트웨어 버전 |
| SW 품번 | 장비 옵션별 SW 변종의 **구분자**(호환성 정보 아님, §6.1) |
| 호환성(HW) | 적용 가능한 HW 버전·모델(호환성 판정용, §6.2) |
| rollback | 다운그레이드 방지 기준 버전(§8.3) |
| delta | 델타 기준 버전(§3.3) |
| binary sha256 | 바이너리 무결성 |

#### 3.2.3 예시

```xml
<manifest>
  <package name="EPOS-10i-APP" version="2.4.0"/>
  <component node="epos-10i-app" type="uds-can-agent"/>

  <!-- 옵션별 SW 변종 구분자 (호환성 정보 아님; 타겟팅은 VLM+정책 §6.1) -->
  <software partNumber="EPOS10I-SW-A21"/>

  <!-- 호환성 판정 (디바이스측 대조, §6.2) -->
  <compatibility>
    <hardware model="EPOS-10i" hwVersion=">=1.2"/>
  </compatibility>

  <rollback allowedFrom="2.3.0"/>
  <binary sha256="a1b2c3…" size="1048576"/>
  <!-- 델타 시: <delta from="2.3.0" reference="…"/> -->
</manifest>
```

> 태그 구성은 eSync manifest.xml 규약을 따르며 업로드 시 검증된다. 위는 대표 예시다.

#### 3.2.4 작성 주체·툴

- manifest.xml은 사람이 손으로 쓰지 않는다. **릴리스 패키징 툴**이 GPDM 릴리스 메타데이터(버전·품번·호환성·롤백)에서 **생성**한다.
- 컴포넌트 ZIP은 eSync Server 업로드 시 사전검증(preValidate)되어 필수 태그·형식이 강제된다.

### 3.3 델타 업데이트

- 델타 컴포넌트는 `manifest_diff.xml`과 기준 버전 참조를 포함한다.
- eSync Agent가 기준 버전을 확인하고 이미지를 재구성한 뒤 검증·플래시한다.

---

## 4. 서명과 검증

### 4.1 서명

- **알고리즘:** ECDSA `P-256` / `SHA-256`([10 §1.3](10-PKI-사양서.md)).
- **주체·키:** eSync Server가 KMS에 서명을 요청하고, KMS가 CS Cert(코드서명 인증서) 개인키로 서명한다. 개발자·벤더는 서명 키를 갖지 않는다.
- **대상:** 컴포넌트(ZIP) 전체. manifest.xml이 안에 있으므로 하나의 서명이 바이너리와 메타데이터를 함께 보증한다.

![그림 4 · 서명 발행 시퀀스](assets/서명-발행-시퀀스.svg)

*[그림 4] 서명 발행 시퀀스*

1. GPDM이 릴리스된 컴포넌트(ZIP)를 eSync Server에 업로드한다.
2. eSync Server가 컴포넌트를 조립하고 SHA-256으로 해시한다.
3. eSync Server가 KMS에 서명을 요청한다.
4. KMS가 CS Cert 개인키(P-256)로 ECDSA 서명한다.
5. 서명값을 반환한다.
6. eSync Server가 서명값과 CS Cert(체인 포함)를 컴포넌트에 동봉한다.
7. 서명된 컴포넌트를 저장한다.

### 4.2 검증

EPOS-10i는 서명을 검증하지 못하므로 **TGU의 eSync Agent가 대행 검증**한다. TGU는 Root CA 인증서를 신뢰앵커로 보유한다([20 §1.3](20-프로비저닝-사양서.md)).

![그림 5 · 서명 검증 상세](assets/서명검증-상세.svg)

*[그림 5] 서명 검증 상세*

1. **체인 구성** — 동봉된 CS Cert·Code Sign CA 인증서로 `Root → Code Sign CA → CS Cert` 체인을 만든다.
2. **서명 검증** — CS Cert 공개키로 서명을 검증한다(ECDSA P-256).
3. **재해시** — 수신 컴포넌트를 SHA-256으로 다시 해시해 서명값과 대조한다.
4. **판정** — 모두 통과해야 진행한다. 하나라도 실패하면 폐기한다.

체인 검증 규칙과 신뢰앵커는 [10 §3](10-PKI-사양서.md)을 따른다.

---

## 5. 전송·연결 (AWS IoT Core)

### 5.1 mTLS 연결

- TGU는 자기 디바이스 인증서로 AWS IoT Core에 상호 TLS(mTLS)로 접속한다(TLS 1.2/1.3, MQTT/TLS 8883 또는 443+ALPN, SNI 전송).
- 하나의 TGU 인증서/키가 전송 인증과 애플리케이션 신원을 겸한다([20 §4.2](20-프로비저닝-사양서.md)).
- **TGU의 서버 검증 앵커(인증서)는 대상에 따라 다르다:** AWS IoT Core 엔드포인트는 **Amazon Root CA**로, eSync Server 애플리케이션 신원은 **OEM Root CA**로 검증한다([20 §2.3](20-프로비저닝-사양서.md)). 반대 방향(서버가 TGU 검증)의 앵커는 등록·공유된 Device CA 인증서다.

![그림 6 · mTLS 핸드셰이크 시퀀스](assets/mtls-핸드셰이크-시퀀스.svg)

*[그림 6] mTLS 핸드셰이크 시퀀스 (eSync Server ↔ eSync Client)*

1. ClientHello.
2. 서버 인증서 제시.
3. 클라이언트가 서버 인증서 검증(해당 앵커 인증서로 체인·유효기간·CN·OCSP).
4. 클라이언트 인증서 제시(CN=Serial).
5. 서버가 클라이언트 인증서 검증(Device CA 앵커로 체인·CN·OCSP).
6. 검증 성공 시 mTLS 수립.

### 5.2 시그널링

- eSync Server ↔ TGU 신호는 AWS IoT Core를 통한다. 업데이트 잡(job)과 상태를 MQTT 토픽으로 주고받는다.
- eSync Messaging Protocol이 제어·메타데이터를 나르고, 전송은 이 채널이 담당한다.

### 5.3 다운로드·보안 저장

- 펌웨어 bulk는 시그널링 채널로 보내지 않는다. 메타데이터로 다운로드 위치만 전달하고, TGU가 펌웨어 파일 서버에서 HTTPS로 직접 내려받는다.
- 다운로드는 청크 해시로 무결성을 확인하며, 완료분은 **보호 저장소**에 둔다. 중단 시 누락분만 재요청한다.

---

## 6. 대상 선정·호환성

### 6.1 캠페인 타겟팅

- 캠페인은 **차량(장비) 단위**로 대상을 선정한다(VIN·그룹). 개별 EPOS를 등록하지 않는다.
- **SW 품번(옵션 변종) 선택은 VLM·정책과 연동**한다. VLM이 장비의 옵션→적용 품번을 알고, 캠페인이 그에 맞는 변종을 대상 장비군에 배포한다. 품번만으로 디바이스에서 자동 설치가 일어나지 않는다.
- 단계적 롤아웃(일부→확대)과 중단(halt)을 지원한다.

### 6.2 호환성 판정

정품이지만 비호환인 펌웨어의 오설치를 막기 위해, 서명이 아니라 메타데이터로 호환성을 판정한다.

- **HW 호환성:** eSync Agent가 UDS `0x22`로 EPOS-10i의 HW 버전·모델을 읽어 manifest의 호환성 정보와 대조한다.
- **SW 버전·의존성:** rollback 기준(§8.3)과 `dependencies`/`componentRules`(§6.3)로 버전 적합성을 판정한다.
- SW 품번은 판정 기준이 아니라 구분자다(§6.1).

### 6.3 componentRules (설치 규칙)

Package는 컴포넌트별 설치 규칙과 원자적 롤백을 정의한다.

```json
{
  "componentRules": [
    { "id": "<componentVersionId>", "atomicity": "PersistentAtomic" }
  ],
  "atomicRollback": {
    "layers": [
      { "items": [ { "atomic": true, "componentVersion": "<componentVersionId>" } ] }
    ]
  }
}
```

- `atomicity` — `NonAtomic` / `PersistentAtomic` / `NonPersistentAtomic` / `ServerAtomic`. 묶인 컴포넌트는 모두 성공하거나 모두 롤백된다.
- `atomicRollback.layers` — 이전 계층 설치 실패 시 설치할 대체(롤백) 계층.

---

## 7. 설치 정책·트리거

### 7.1 설치 정책 (InstallationPolicy)

설치 시점·조건을 정책으로 통제한다.

```json
{
  "policies": [
    { "type": "IPDrivingMode",       "mode": "ignition_off" },
    { "type": "IPNetworkConnection", "connection": "any" },
    { "type": "IPTimeWindow",        "start_time": "01:00", "end_time": "05:00",
                                     "timezone": "Asia/Seoul", "source": "fixed" }
  ]
}
```

- **IPDrivingMode** — `any` / `ignition_off`. 건설기계는 안전을 위해 `ignition_off`(시동/작동 정지) 조건을 권장한다.
- **IPNetworkConnection** — `any` / `broadband`. 다운로드 네트워크 조건.
- **IPTimeWindow** — 설치 허용 시간창.

### 7.2 업데이트 확인·설치 시점

- eSync Client는 설정된 주기/이벤트(예: 시동 시)에 업데이트를 확인한다(§2.2-1).
- 다운로드·설치는 설치 정책(§7.1)을 만족할 때만 진행한다.

---

## 8. 실패·재시도·복구·롤백

### 8.1 폴트 톨러런스

- 캠페인 중 실패는 일시적으로 보고 정해진 횟수만큼 **재시도**한다. `Fail`은 캠페인 종료 시에만 확정된다.
- 연결 실패 시 재시도를 예약하고, 전송 중단 시 누락분만 재전송한다(§5.3).

### 8.2 원자적 커밋·복구

- 플래시는 **원자적 커밋**(전송·검증 완료 후 활성화)으로 처리한다.
- EPOS-10i는 싱글뱅크라 A/B 대체 뱅크가 없다. 보호 영역의 **복구 부트로더(Insurance Kernel)**가 플래시 실패 시에도 재플래시를 가능하게 한다([20 §1.4](20-프로비저닝-사양서.md)).

### 8.3 롤백 방지·정상 복귀

- **롤백 방지:** manifest의 rollback 기준·버전으로 TGU가 다운그레이드 배포를 차단한다.
- **정상 복귀:** 설치 실패로 판정되면 이전 정상 버전으로 되돌린다(§6.3 `atomicRollback`).

---

## 9. 상태·감사

### 9.1 설치 상태 보고

- eSync Agent가 단계별 상태(진행·성공·실패)를 eSync Client 경유로 eSync Server에 보고한다(§2.2-14).
- 최종 상태는 `INSTALL_COMPLETE` / `INSTALL_FAILED`로 확정한다.

### 9.2 감사·추적

- 설치 이력은 이벤트 순서대로 추적(trail)으로 남긴다.
- 인증·서명 검증·설치 결과·사용자/역할 변경을 감사 로그로 기록한다.

---

## 10. EPOS-10i 특성 종합

| 특성 | 대응 |
|------|------|
| HSM 없음·서명검증 불가 | TGU(eSync Agent)가 대행 검증(§4.2) |
| 암복호화 불가 | 평문 페이로드 수용, 전송 구간은 상위에서 보호 |
| UDS `0x29` 미지원 | 인증서 기반 인증 미사용 |
| 플래시 세션 보호 | UDS `0x27` SecurityAccess(암호 하드웨어 불요, §2.2-10) |
| 싱글뱅크 | 원자적 커밋 + Insurance Kernel 복구(§8.2) |
| 롤백 카운터 없음 | TGU측 버전 게이트로 다운그레이드 차단(§8.3) |

---

## 11. 미정 (TBD)

- eSync Server ↔ TGU 시그널링 상세(MQTT 토픽·잡 구조) — 전송 담당 팀과 협의.
- 업데이트 확인 주기·이벤트 정의.
- UDS `0x27` 시드-키 비밀의 관리 정책.
- SW 벤더 납품 경로(참고).
