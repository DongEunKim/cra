# PKI 사양서

| 항목 | 내용 |
|------|------|
| **문서 성격** | eSync 기반 OTA의 PKI(공개키 기반구조) 사양 |
| **상태** | Draft |
| **관계** | [20 프로비저닝 사양서](20-프로비저닝-사양서.md) · [30 eSync OTA 공통 아키텍처](30-eSync-OTA-공통아키텍처.md)와 함께 읽는다. 제어기별 사양은 [31 EPOS-10i](31-EPOS-10i-OTA사양서.md) · [32 EPOS-30i](32-EPOS-30i-업데이트-사양서.md). |
| **용어 기준** | [00 용어집](00-용어집.md) |

OTA에 쓰이는 인증기관 계층, 인증서 프로파일, 신뢰 체인, 키 보관, 인증서 생애주기를 정의한다.

---

## 1. PKI 계층

![그림 · PKI 계층](assets/pki-계층.svg)

*[그림] PKI 계층*

### 1.1 인증기관 계층

- **OEM Root CA** — 회사 보안팀이 관리하는 최상위(발급) 인증기관. 개인키는 오프라인 보관한다. 사내 시스템·서버 인증서를 발급하는 신뢰의 뿌리이며, 각 제어기가 보유하는 신뢰앵커(Root CA 인증서)의 출처다. (자기서명 여부·하위 발급 CA 위임 정책은 보안팀 확인 대상 → §7.)
- **코드서명(FW 서명)** — 코드서명 인증서(CS Cert)는 **별도 중간 CA 없이 OEM Root CA가 직접 발급**한다(중간 CA는 필요 시 이후 도입). 상세는 §2.2·§4.
- **eSync Server 인증서** — eSync Server가 장비에 자신의 신원을 증명하는 애플리케이션 신원 인증서로, 용도는 `serverAuth`다. 코드서명 인증서와는 키·용도가 다른 **별개 인증서**이며, 한 키가 두 역할을 겸하지 않는다. 발급 경로(중간 CA 경유 여부)는 §7. 상세는 §2.6.
- **Device CA / Diagnostic License CA** — 디바이스 인증서·DMS License를 발급하는 인증기관(개인키 KMS). 이 두 갈래의 CA 구조는 **이후 재검토**한다.

### 1.2 신뢰 구조

- 자체 PKI 검증의 신뢰앵커는 **OEM Root CA 인증서 하나**다(전송용 Amazon Root CA는 별개, §3.1).
- 디바이스는 **"Root → (필요 시 중간 CA) → 최종 인증서"** 규칙으로 체인을 구성해 검증한다.
- **코드서명**은 중간 CA 없이 `OEM Root CA → CS Cert`로 단순화한다(§2.2). 디바이스 신원·진단 라이선스 갈래는 각각 Device CA·Diagnostic License CA를 두며, 이 구조는 이후 재검토한다(§1.1).

### 1.3 암호 스위트

- 서명: **ECDSA**, 곡선 `secp256r1(P-256)`, 해시 `SHA-256`.
- 전 계층(Root CA·중간 CA·최종 인증서)에 동일 스위트를 적용한다.

---

## 2. 인증서 프로파일

- **최종 인증서** — 디바이스 인증서(§2.1) · CS Cert(§2.2) · DMS License(§2.3) · eSync Server 인증서(§2.6). 증명 대상과 EKU가 각각 다르다.
- **인증기관 인증서** — Root CA와 하위 CA(§2.4).
- **비인증서 앵커** — EPOS 이미지 서명 검증용 원시 공개키(§2.5).

### 2.1 디바이스 인증서

| 항목 | 값 |
|------|-----|
| 발급기관 | Device CA |
| Subject CN | 시스템 시리얼 번호 |
| Key Usage | `digitalSignature` |
| Extended Key Usage | `clientAuth (1.3.6.1.5.5.7.3.2)` |
| 용도 | mTLS 클라이언트 인증(eSync Client 신원) |

### 2.2 코드서명 인증서 (CS Cert)

코드서명 인증서(CS Cert, Code Signing Certificate)는 "이 공개키(cs-pub)로 검증되는 서명은 정품 FW다"를 보증하는 최종 인증서다. 짝이 되는 개인키(cs-pri, KMS 보관)로 eSync Server가 Component에 서명하고, 이 인증서를 Component에 동봉해 기기가 검증하게 한다.

| 항목 | 값 |
|------|-----|
| 발급기관 | OEM Root CA (보안팀 CA) — 별도 중간 CA 없음 |
| Subject CN | eSync Server 서명 신원 — **고정값**이며 검증자가 대조한다(§3.3-5, 확정값은 §7) |
| 보증 대상 | 코드서명 공개키(cs-pub) |
| Key Usage | `digitalSignature` (keyCertSign 없음) |
| Extended Key Usage | `codeSigning (1.3.6.1.5.5.7.3.3)` |
| 용도 | Component 서명. 개인키(cs-pri)는 온프렘 KMS 보관 |
| 배포 | 검증을 위해 Component에 동봉 |

`codeSigning` EKU는 이 인증서의 용도를 **코드서명으로 한정한다**. 같은 OEM Root CA가 발급하더라도 eSync Server 인증서(§2.6)와는 용도·키·생애주기가 모두 다르다.

| 축 | CS Cert (코드서명) | eSync Server 인증서 (§2.6) |
|----|--------------------|----------------------------|
| 증명 대상 | 소프트웨어가 정품·무변조임 | 접속 상대가 정품 eSync Server임 |
| EKU | `codeSigning` | `serverAuth` |
| 개인키 위치 | 온프렘 KMS (서버 밖, §4) | eSync Server 로컬 |
| 사용 시점 | 릴리스마다 1회 | 접속마다 |
| 검증 기준시점 | **서명 시점** (§3.3-3) | **접속 현재** (§3.3-3) |
| 검증자에게 전달 | Component에 동봉 (§3.2) | TLS 핸드셰이크에서 제시 |
| 폐기 시 영향 | 기존 Component **재서명 필요** (§5.3) | 새 인증서로 교체 |

> **eSync도 둘을 따로 둔다.** *Client Authentication Integration*에서 바이너리 검증 체인(Binary Verification CA)과 서버 인증 체인(eSync Server CA)은 독립된 체인이며(18쪽), eSync Server 안에도 바이너리 서명용 인증서와 서버 인증용 인증서가 별개로 놓인다(19쪽). 클라이언트도 두 앵커를 각각 보유한다(20쪽). eSync 백서도 인증서 서명 키·컴포넌트 서명 키·클라이언트 통신 키를 각각 분리하라고 적는다(12쪽).

### 2.3 DMS License 인증서

| 항목 | 값 |
|------|-----|
| 발급기관 | Diagnostic License CA |
| Subject | `CN=<PC 고유 ID>, O=<OEM>, OU=Diagnostics` |
| 공개키·서명 | ECDSA P-256 · `ecdsa-with-SHA256` |
| Key Usage | `digitalSignature` |
| Extended Key Usage | `clientAuth (1.3.6.1.5.5.7.3.2)` |
| Role 확장(사용자정의) | OID `1.3.6.1.4.1.<PEN>.4.1` — 진단 Role 값 |
| 유효기간 | 단수명. 단, 만료 강제는 절대시각이 아닌 방식(→ [40 §9](40-유선업데이트-사양서.md)) |
| 용도 | 이중 — SGW의 UDS `0x29` APCE 검증 + TGU의 mTLS 클라이언트 인증 |

**Role 값**

| 값 | Role | 개요 |
|----|------|------|
| `0x01` | Engineering | 전체 서비스·대상 |
| `0x02` | Dealer-Service | 서비스 선로·승인 대상, 플래시 |
| `0x03` | Read-Only | 진단 조회만 |

Role → 허용 CAN 선로/대상/서비스의 구체 매핑은 [40 §8](40-유선업데이트-사양서.md)에 둔다.

**예시** (openssl `x509 -text` 발췌)

```
Certificate:
  Data:
    Signature Algorithm: ecdsa-with-SHA256
    Issuer: CN=Diagnostic License CA, O=<OEM>
    Subject: CN=DMS-PC-3F2A91, O=<OEM>, OU=Diagnostics
    Subject Public Key Info:
      Public Key Algorithm: id-ecPublicKey (prime256v1)
    X509v3 extensions:
      X509v3 Key Usage: critical
        Digital Signature
      X509v3 Extended Key Usage:
        TLS Web Client Authentication
      1.3.6.1.4.1.<PEN>.4.1: 0x02   # Role = Dealer-Service
  Signature Algorithm: ecdsa-with-SHA256
```

DMS License는 하나의 인증서를 두 게이트웨이가 각자의 방식으로 검증한다([40 §5](40-유선업데이트-사양서.md)). `<PEN>`(Private Enterprise Number)은 TODO(§7).

### 2.4 인증기관 인증서

Root CA·Device CA·Diagnostic License CA 인증서는 `basicConstraints: cA=TRUE`, `Key Usage: keyCertSign`을 가진다. Root는 자기서명(self-signed)이다(자기서명 여부는 보안팀 확인 대상, §7). 코드서명은 중간 CA를 두지 않으므로 이 목록에 없다.

### 2.5 EPOS 이미지 서명 검증 공개키 (HSM 제어기)

HSM 보유 제어기(EPOS-30i)는 코드서명이 **두 계층**으로 나뉜다. 검증 주체와 검증 자산이 다르다.

| 계층 | 서명 주체 | 검증 주체 | 검증 자산 |
|------|----------|----------|-----------|
| eSync CS Cert (외부) | KMS(cs-pri) | TGU eSync Agent | **인증서 체인**(Root → CS Cert) |
| EPOS 이미지 서명 (내부) | KMS(이미지 서명 개인키) | EPOS-30i HSM | **raw 공개키 앵커**(HSM 슬롯) |

- 외부 계층은 §2.2·§3의 인증서 체인 검증을 따른다.
- 내부 계층은 EPOS-30i HSM 슬롯에 주입한 **원시 P-256 공개키**로 이미지 서명(ECDSA)을 직접 검증한다. 인증서 체인이 아니라 앵커 공개키 하나로 검증하며(경량), 공개키는 유출돼도 위조가 불가능하므로 플릿 공통으로 둔다. 슬롯 배치·주입은 [32 §7](32-EPOS-30i-업데이트-사양서.md)·[20](20-프로비저닝-사양서.md).
- 두 계층 모두 알고리즘은 ECDSA P-256이다(§1.3). 개발자 서명은 두 계층 어디에도 없다(중앙 KMS 서명, §4.2).

### 2.6 eSync Server 인증서

eSync Server가 eSync Client(TGU)에게 "이 서버가 정품 eSync Server다"를 증명하는 최종 인증서다. 코드서명 인증서(§2.2)와는 키·용도가 다른 **별개 인증서**이며, 한 키가 두 역할을 겸하지 않는다(§4.2).

| 항목 | 값 |
|------|-----|
| 발급기관 | OEM Root CA (보안팀 CA) — 발급 경로는 §7 |
| **subjectAltName** | **`dNSName = eSync Server FQDN` (필수)** — 신원 매칭의 기준 |
| Subject CN | eSync Server FQDN (표시용. 신원 매칭에 쓰지 않는다) |
| Key Usage | `digitalSignature` |
| Extended Key Usage | `serverAuth (1.3.6.1.5.5.7.3.1)` (→ §7) |
| 용도 | eSync Client(TGU)에 대한 서버 신원 증명 |
| 개인키 보관 | eSync Server 측 — 보관 수단은 §7 |
| 배포 | 접속마다 서버가 핸드셰이크에서 제시 (→ §7) |

`Key Usage`는 `digitalSignature` 단독이다. `keyEncipherment`는 RSA 키 전송용, `keyAgreement`는 정적 ECDH용이라 ECDSA + ECDHE 조합(§1.3)에는 어느 쪽도 넣지 않는다. 위 표는 서버가 TLS 핸드셰이크에서 직접 인증서를 제시하는 배치를 전제한다. 앱 신원을 어느 계층에서 제시하는지에 따라 EKU 값이 달라질 수 있어 §7에서 함께 확정한다.

**FQDN은 `subjectAltName`에 담는다.** 검증자는 CN이 아니라 SAN `dNSName`으로 서버 신원을 매칭하며, CN 대체 매칭은 허용하지 않는다(RFC 9525). CN에만 FQDN을 두면 최신 TLS 구현이 신원을 확인하지 못한다. eSync도 서버 인가 요건으로 "지정된 CA의 서명"과 "소지자를 eSync Server로 확정하는 명시적 속성" 둘을 요구한다(*Client Authentication Integration* 4쪽) — `serverAuth` EKU만으로는 후자가 채워지지 않는다.

> **전송 엔드포인트 인증서와는 다른 인증서다.** TGU가 검증하는 서버측 인증서는 둘이다 — AWS IoT Core 엔드포인트 인증서(Amazon Root CA 앵커, 전송 계층)와 본 절의 eSync Server 인증서(OEM Root CA 앵커, 앱 신원)다([20 §2.3](20-프로비저닝-사양서.md), [30 §5.1](30-eSync-OTA-공통아키텍처.md)). eSync 백서 7쪽은 이 인증서를 외부 PKI 제공자가 FQDN 검증(메일·CNAME·루트 파일) 후 발급하는 것으로 기술한다. 본 사양은 이를 OEM Root CA 발급으로 바꾸므로, 장비 앵커가 OEM Root 하나로 성립하는 대신 공인 PKI의 FQDN 검증 절차를 사내 발급 절차로 대체해야 한다(§7).

---

## 3. 신뢰앵커와 체인 검증

### 3.1 신뢰앵커

- **신뢰앵커는 공개키가 아니라 인증서다(자기서명 CA 인증서).** 경로 검증에는 앵커의 이름(DN)·공개키·제약(basicConstraints·keyUsage)이 필요하다(RFC 5280 §6.1.1). 공개키만으로는 서명 확인은 되어도 이름 체이닝·제약 검사를 할 수 없다.
- 디바이스는 **목적별로 복수의 신뢰앵커를 인증서로** 보유한다.
  - **OEM Root CA 인증서** — 자체 PKI 검증. 코드서명(CS Cert 체인, §2.2)과 eSync Server 앱 신원(§2.6)을 검증한다. **검증 대상이 둘이지만 앵커는 하나다** — 두 인증서는 EKU와 신원으로 구분한다(§3.3-4·5).
  - **Amazon Root CA 인증서** — AWS IoT Core 엔드포인트(전송 mTLS) 검증([20 §2.3](20-프로비저닝-사양서.md)).
- 앵커는 공장에서 주입한다([20 §1](20-프로비저닝-사양서.md)). 중간 CA·리프 인증서는 앵커가 아니라 검증 대상이며, 검증 시점에 동봉/제시된다.

> eSync는 바이너리 검증과 서버 인증에 **각각 다른 앵커**를 두라고 한다(*Client Authentication Integration* 20쪽 — 클라이언트가 `Binary Verification CA`와 `eSync Server CA`를 따로 보유). 본 사양은 두 갈래를 **하나의 OEM Root CA 앵커 아래 두되 EKU와 신원으로 분리**한다(§3.3-4·5). eSync는 중간 인증서를 쓰는 경우에 한해 CA 통합을 허용하므로(같은 문서 18쪽), 중간 CA 없이 통합한 현 구조의 타당성은 코드서명 중간 CA 도입 여부와 함께 §7에서 다룬다.
>
> **앵커 통합의 대가는 둘이다.** 첫째, OEM Root CA는 사내 시스템·서버 인증서도 발급하는 전사 루트이므로(§1.1), 같은 루트에서 나온 제3의 인증서가 검증을 통과할 수 있다 — EKU는 용도만 한정할 뿐 발급 자체를 구획하지 못하므로, 신원 확인(§3.3-5)이 실질적인 방어선이다. 둘째, 중간 CA가 없어 **가지 단위 폐기가 불가능하다** — Root 키가 침해되면 코드서명·서버 인증이 동시에 무너지고, 복구하려면 전 장비의 앵커를 다시 주입하고 배포된 Component를 전량 재서명해야 한다. 현재 앵커 주입 경로는 공장뿐이다([20 §1](20-프로비저닝-사양서.md)).

### 3.2 체인 구성

- 코드 서명 검증: Component에 **CS Cert를 동봉**한다. 디바이스는 `Root → CS Cert` 체인을 구성해 검증한다(중간 CA를 이후 도입하면 그 인증서도 함께 동봉).
- 디바이스 신원(mTLS) 검증: 상대가 제시한 리프와 중간 CA로 `Root → Device CA → 디바이스 인증서` 체인을 구성해 검증한다.
- eSync Server 신원 검증: 서버가 핸드셰이크에서 제시한 리프로 `Root → eSync Server 인증서` 체인을 구성해 검증한다(중간 CA 없음, §2.6). 코드서명과 같은 앵커를 쓰므로 EKU·SAN 확인이 반드시 따라붙는다(§3.3-4·5).
- 인터넷 없이도 신뢰앵커(Root)만으로 체인을 수학적으로 검증할 수 있다. (폐기 확인은 §5.2.)

### 3.3 통일 검증 규칙

두 갈래 모두 다음 순서로 검증한다.

1. 리프에서 Root까지 인증서 체인을 구성한다.
2. 각 상위 인증서의 공개키로 하위 서명을 검증해 Root 앵커까지 잇는다.
3. 유효기간을 확인한다. **기준 시점이 용도별로 다르다:**
   - **코드서명(CS Cert):** **서명 시점**의 유효성을 확인한다(신뢰 타임스탬프). 이후 CS Cert가 만료·회전돼도 과거에 서명된 Component는 계속 검증된다.
   - **전송(mTLS 인증서):** **접속 현재 시점**의 유효기간을 확인한다(§6 신뢰 시각 필요).
4. 용도(EKU)를 확인한다. 코드서명 검증은 `codeSigning`(§2.2)을, 상대 서버 검증은 `serverAuth`(§2.6)를, 상대 클라이언트 검증은 `clientAuth`(§2.1·§2.3)를 요구한다. **같은 OEM Root CA에서 나온 인증서라도 EKU가 맞지 않으면 거부한다.** 리프에 EKU가 없거나 `anyExtendedKeyUsage`(`2.5.29.37.0`)를 포함해도 거부한다. 하나의 리프에 두 용도를 함께 담지 않는다.
5. **신원을 확인한다.** EKU는 용도만 한정할 뿐 발급 주체를 구획하지 않으므로(RFC 5280 §4.2.1.12), 리프가 누구의 것인지를 따로 확인해야 한다. **이 단계가 없으면 같은 Root에서 나온 다른 인증서가 그대로 수용된다.**
   - **코드서명:** 리프의 Subject가 §2.2가 고정한 서명 신원과 일치해야 한다.
   - **서버:** 접속한 FQDN이 리프의 `subjectAltName` `dNSName`과 일치해야 한다(§2.6). **CN 값으로 대체 매칭하지 않는다**(RFC 9525).
   - **클라이언트:** 리프의 `CN`이 등록된 시리얼과 일치해야 한다(§2.1·§2.3).
6. 폐기 상태를 확인한다(§5.2).

> **Component/Package는 독립적 만료를 갖지 않는다.** 코드서명을 서명 시점 기준으로 검증하므로(3항), 짧은 수명의 CS Cert가 회전·만료돼도 이미 서명된 펌웨어는 유효하다. 펌웨어 수명(장비 생애)과 CS Cert 수명이 분리된다. 신뢰 타임스탬프 소스는 §7. (EPOS/유선 경로는 신뢰 시계가 없어 시각을 게이트로 쓰지 않는다 → [40 §9](40-유선업데이트-사양서.md).)

---

## 4. 키 보관·서명 서비스

### 4.1 키 보관 (KMS)

- 코드서명 개인키(cs-pri)는 **온프렘 KMS/HSM 안에만** 존재하며 밖으로 나오지 않는다.
- OEM Root CA 개인키는 보안팀이 오프라인(에어갭)에 보관한다.
- (Device CA·Diagnostic License CA 개인키 보관은 이후 재검토 — §1.1.)

### 4.2 서명 서비스(Code Sign)와 서명 게이트웨이

- **Code Sign**은 FW에 서명해 Component를 만드는 eSync Server의 *기능*이며, 실제 서명은 KMS에 위탁한다. (인증서를 발급하는 CA가 아니다.)
- 개발자·벤더는 서명 키를 갖지 않는다(중앙 KMS 서명).
- 클라우드의 OTA Platform(eSync Server)과 온프렘 KMS는 **서명 게이트웨이**로 잇는다. eSync Server가 Component의 **해시**를 게이트웨이로 보내면 KMS가 cs-pri로 서명해 **서명값**만 돌려준다 — 개인키(cs-pri)는 KMS 밖으로 나오지 않는다. 공개키(cs-pub) 추출도 이 경로를 쓴다.
- **코드서명 키 ≠ 서버 신원 키:** 코드서명 키(cs-pri, KMS·릴리스마다 드물게 사용)와 eSync Server 인증서의 개인키(서버 로컬·접속마다 사용, §2.6)는 **별개**다. 한 키가 두 역할을 겸하지 않는다. 보관 위치가 다르다 — 코드서명 개인키는 eSync Server가 접근할 수 없는 온프렘 KMS 안에만 있고, 서버 신원 키는 접속을 처리해야 하므로 서버 로컬에 있다. eSync 백서도 컴포넌트 서명을 "보안상 eSync Server 밖에 둔다"고 명시한다(14쪽). 두 인증서의 대비는 §2.2.

발급→서명→검증의 전체 흐름은 [그림 · 코드서명 발급·서명·검증](assets/코드서명-발행검증.svg)과 같다.

![그림 · 코드서명 발급·서명·검증](assets/코드서명-발행검증.svg)

*[그림] 코드서명 발급·서명·검증*

---

## 5. 인증서 수명·폐기·회전

### 5.1 수명

| 인증서 | 수명 정책 |
|--------|-----------|
| 디바이스 인증서 | TBD (§7) |
| CS Cert (코드서명 리프) | 단수명 · 주기 회전 |
| DMS License | 단수명. 만료 강제는 절대시각이 아닌 방식(→ [40 §9](40-유선업데이트-사양서.md)) |
| 중간 CA | 장수명 |
| Root CA | 장수명 |

### 5.2 폐기 (Revocation)

- **온라인 경계(TGU ↔ 클라우드):** OCSP로 폐기를 확인한다. 확인 강도는 설정값으로 정한다.

| 설정값 | 동작 |
|--------|------|
| `DISABLED` | 폐기 확인 안 함 |
| `NONE` | 확인하되 OCSP 실패는 수용 |
| `REQUIRED` | OCSP 속성이 없으면 인증 실패 |
| `ENFORCED` | 응답이 명시적 `GOOD`이 아니면 실패 |

- OCSP Stapling / Must-Staple을 사용한다.
- **오프라인 경계(TGU → EPOS-10i, CAN):** EPOS-10i는 폐기를 확인할 수 없다. 이 구간의 무결성은 TGU가 대행 검증하며([31 §4](31-EPOS-10i-OTA사양서.md)), CS Cert는 단수명 회전으로 폐기 노출을 줄인다.

### 5.3 갱신·교체·회전

- **디바이스 인증서:** 만료 전 갱신한다. 폐기 시 새 인증서를 발급해 Campaign으로 내려보내고, 검증 성공 후 교체한다(실패 시 이전 인증서로 복귀).
- **CS Cert:** OEM Root CA가 새 CS Cert를 발급해 회전한다. 신뢰앵커(Root)는 고정되므로 디바이스 재프로비저닝이 필요 없다.
  - **통상 회전·만료**는 **이미 서명된 Component의 유효성에 영향을 주지 않는다**(서명 시점 검증, §3.3).
  - **폐기는 다르다.** CS Cert가 폐기되거나 상위 CA를 교체하면 그 키로 만든 서명은 더 이상 신뢰할 수 없으므로, **기존 Component를 새 키·인증서로 재서명해야 한다**(*Client Authentication Integration* 21쪽). 재서명은 Component 버전 단위로 수행하며, eSync는 이를 위한 단건·대량 재서명 API를 제공한다(OpenAPI 6.8.1 `/componentItems/resign` · `/componentItems/bulkResign`).
  - 재서명은 **정품 배포를 되살릴 뿐, 침해된 키로 만든 서명의 수용을 막지는 못한다.** 장비는 서명 시점 기준으로 판정하므로(§3.3-3), 폐기 사실이 장비에 닿지 않으면 공격자가 유효기간 안쪽 시점으로 만든 서명을 그대로 받아들인다. 폐기를 장비까지 전달하는 수단은 온라인 경계에서는 OCSP이고(§5.2), 오프라인 경계는 §7에서 정한다.
- **eSync Server 인증서(§2.6):** 통상 교체는 서버에 새 인증서를 배치하는 것으로 끝나며, 배포된 Component에는 영향이 없다. 폐기 시에도 **재서명이 필요 없다** — 이 인증서는 소프트웨어에 서명하지 않기 때문이다. 그래서 CS Cert와 생애주기가 독립적이다.

---

## 6. 신뢰 시각 (Trusted Time)

인증서 유효기간(§3.3-3)과 OCSP 검증(§5.2)은 신뢰할 수 있는 현재 시각을 전제한다.

- **온라인 경계(TGU ↔ 클라우드):** TGU는 신뢰 시각 소스를 보유해, 유효기간·OCSP를 검증한다. 소스·동기화 방식은 TBD(§7).
- **유선 경계(SGW·DMS PC·EPOS):** 시각 동기화가 불가능하고(EPOS는 RTC 없음) 조작될 수 있다. 이 경계에서는 **절대시각을 보안 게이트로 쓰지 않는다.** 유효성은 세션 nonce·폐기·단조 방식으로 판단한다(→ [40 §9](40-유선업데이트-사양서.md)).

---

## 7. 미정 (TBD)

- OEM Root CA 자기서명 여부·하위 발급 CA 위임·코드서명 EKU 발급 정책(보안팀 확인).
- 코드서명 중간 CA 도입 여부(현재 미도입 — OEM Root CA 직접 발급). eSync는 중간 인증서를 쓰는 경우에만 CA 통합을 허용한다(*Client Authentication Integration* 18쪽). 코드서명 인증서와 eSync Server 인증서가 같은 Root를 공유하는 현 구조(§3.1)와 함께 판단한다.
- eSync Server 인증서(§2.6)의 발급 경로 — 중간 CA를 경유할지, 그리고 eSync 백서 7쪽이 전제하는 외부 PKI 제공자 발급을 본 사양의 OEM Root CA 앵커 검증([20 §2.3](20-프로비저닝-사양서.md))과 어떻게 맞출지.
- **서명자 신원 고정값** — 코드서명 리프의 Subject(§2.2)와 eSync Server 리프의 SAN `dNSName`(§2.6)에 넣을 확정값. 검증자가 이 값으로 대조하므로(§3.3-5) 전사 Root에서 나온 다른 인증서를 걸러내는 실질 기준이 된다. 정책 OID(`certificatePolicies`)를 함께 쓸지도 보안팀과 정한다.
- eSync Server 인증서 개인키 보관 수단(§2.6) — 서버 파일시스템 대 HSM 간접 호출. eSync는 후자를 제시한다(*Client Authentication Integration* 19쪽). 접속마다 쓰는 온라인 상주 키다.
- OCSP 발급 경로 — 세 인증서 프로파일(§2.1·§2.2·§2.6)에 `authorityInfoAccess(OCSP)`를 넣을지, 그리고 Root가 오프라인(§4.1)이므로 응답을 누가 서명할지(Root가 발급한 위임 OCSP 서명 인증서, RFC 6960 §4.2.2.2). 현재 §5.2의 `REQUIRED` 설정값은 이 항목이 정해져야 성립한다.
- eSync Server 앱 신원의 제시 계층 — AWS IoT Core가 TLS를 종단하면 eSync Server가 자기 인증서를 내미는 두 번째 TLS 핸드셰이크가 없다. 이 경우 앱 신원 검증은 `serverAuth`가 아니라 애플리케이션 계층 구성물이 되므로, §2.6의 EKU 값도 함께 확정한다([30 §5.1](30-eSync-OTA-공통아키텍처.md)).
- eSync 기본 배포가 요구하는 디바이스 인증서 제약을 채택할지 여부 — `OU=x14_device`, OID `1.3.6.1.4.1.45473.1.1`(`sota` 표식)·`.1.10`(디바이스 식별자)·`.1.11`(테넌시). 구현 측이 프로파일을 정의하면 그것을 따르므로(*Client Authentication Integration* 23쪽), §2.1을 유지할지 eSync 기본값에 맞출지 Excelfore와 협의한다.
- 디바이스 인증서 유효기간 값.
- CS Cert 회전 주기.
- OCSP 설정값(DISABLED/NONE/REQUIRED/ENFORCED) 확정.
- 신뢰 시각 소스·동기화 방식.
- 코드서명 **서명 시점 검증용 신뢰 타임스탬프** 소스·형식(§3.3).
- 장수명 대응 암호 알고리즘 식별자(크립토 애자일리티).
