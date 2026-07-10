---
name: security-diagram
description: >
  CRA 보안 기술문서용 SVG 다이어그램(시퀀스·아키텍처)을 하우스 스타일로 만들고 유지하는 스킬.
  assets/diagrams/에 새 다이어그램을 만들거나 기존 것을 고칠 때, 검증 시퀀스·인증서 발급 흐름·PKI 계층·
  UDS 0x29 흐름·키 주입 시퀀스 같은 그림을 그릴 때 반드시 사용한다. "다이어그램/그림/시퀀스/SVG 그려줘·고쳐줘",
  또는 명세서 절에 도식을 넣어야 하는 작업이면 발동한다. 자기완결(외부 의존 0)·텍스트 잘림 금지·일관된
  색/레인 규격이 목적이다.
---

# security-diagram — SVG 다이어그램 하우스 스타일

이 프로젝트의 다이어그램은 사내망·오프라인·PDF로 배포될 수 있으므로 **자기완결**이어야 하고,
4개의 기존 그림과 **같은 시각 언어**를 써야 한다. 기존 예시: `assets/diagrams/fota-그림1·2.svg`,
`assets/diagrams/pki-그림1·2.svg` — 새 그림 전에 하나 열어 톤을 맞춘다.

## 절대 규칙 (자기완결)

CSP·오프라인·PDF에서 깨지지 않게 한다.

- **외부 의존 0:** 원격 폰트/CSS/이미지/스크립트 금지. `<script>` 금지. 폰트는 `font-family="Inter, sans-serif"`
  로 시스템 폴백만 지정(임베드하지 않음).
- **standalone SVG:** 루트에 `xmlns="http://www.w3.org/2000/svg"`, `viewBox`, 명시적 `width`/`height`.
- **텍스트 잘림 금지:** 박스 폭 안에 문구가 들어가는지 확인한다(원본 HTML에서 "…패키 전송"처럼 잘린 사례가 있었다).
  길면 줄을 나누거나 폰트를 줄인다. LaTeX 표기(`$\rightarrow$`) 금지 — 유니코드 `→`를 직접 쓴다.
- **한글 안전:** 한글은 `text-anchor="middle"` 기준으로 폭을 넉넉히. 혼합(한/영) 라벨은 특히 여유를.

## 파일 규약

- 경로: `assets/diagrams/<문서>-그림N-<슬러그>.svg` (예: `fota-그림1-검증시퀀스.svg`).
- 명세서에서 삽입: `![그림 N · 제목](../../assets/diagrams/<파일>.svg)` 다음 줄에 `*[그림 N] 제목*` 캡션.

## 색 팔레트 (역할 고정)

의미에 색을 고정해 문서 전체에서 같은 주체가 같은 색으로 보이게 한다.

| 역할 | 색 |
|------|-----|
| Root/최상위·강조 헤더 | `#1e3a8a` (남색) |
| 인스톨러/네트워크(TGU·파랑 계열) | `#0ea5e9` / `#3b82f6` |
| 타겟/HSM·주의(주황) | `#f59e0b` / `#fef3c7` 박스 |
| 게이트웨이/오프라인 앵커(SGW·teal) | `#0d9488` / `#ccfbf1` 박스 |
| 진단/DMS(보라) | `#9333ea` / `#f3e8ff` 박스 |
| 성공/완료(초록) | `#22c55e` / `#059669` |
| 레인·점선·테두리(중립) | `#cbd5e1` / `#94a3b8` / `#e2e8f0` |
| 본문 텍스트 | `#334155` / 강조 `#0f172a` |

## 시퀀스 다이어그램 패턴

- 상단에 액터 박스(`rect rx=6` + 흰 볼드 텍스트), 아래로 점선 레인(`stroke-dasharray="5,5"`).
- 메시지는 `path` + `marker-end` 화살표. 번호를 붙인다(`1. …`, `2. …`).
- 내부 연산/검증은 연한 배경 박스(위 팔레트의 계열색 `-50/-100`).
- 화살표 마커는 `<defs>`에 색상별 `<marker>` 정의(`refX≈7, markerWidth/Height=8`).

## 아키텍처 다이어그램 패턴

- 계층/영역은 라운드 사각형 존(`rx=8`, 연한 fill + 얇은 테두리)으로 묶고 좌상단에 라벨.
- CA 계층은 위→아래 발급 화살표(점선). 노드 박스는 제목/부제/용도 3줄.

## 최소 골격 (복붙 시작점)

```svg
<svg viewBox="0 0 800 320" width="800" height="320" xmlns="http://www.w3.org/2000/svg" font-family="Inter, sans-serif">
  <rect x="100" y="20" width="120" height="40" rx="4" fill="#9333ea"/>
  <text x="160" y="45" fill="white" font-weight="bold" text-anchor="middle">액터 A</text>
  <line x1="160" y1="60" x2="160" y2="300" stroke="#94a3b8" stroke-dasharray="5,5" stroke-width="2"/>
  <!-- 메시지 -->
  <text x="400" y="90" fill="#334155" font-size="12" font-weight="bold" text-anchor="middle">1. 요청</text>
  <path d="M 170 95 L 630 95" stroke="#334155" stroke-width="2" marker-end="url(#arrow)"/>
  <defs>
    <marker id="arrow" markerWidth="8" markerHeight="8" refX="7" refY="3" orient="auto" markerUnits="strokeWidth">
      <path d="M0,0 L0,6 L8,3 z" fill="#334155"/>
    </marker>
  </defs>
</svg>
```

## 다이어그램 내 용어

라벨도 문서다. [term-guard] 표준형을 따른다(예: "통합 디바이스 CA"가 아니라 "디바이스 CA (Dev-IC)").
그림을 고치면 대응 명세서 캡션·본문과 어긋나지 않는지 확인한다.
