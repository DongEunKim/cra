# DOC-20 · 업데이트 사양 (A2)

**본문:** [업데이트-보안명세서.md](업데이트-보안명세서.md) (v1.4)
**원본:** `legacy/fota_v5.1.html` + `legacy/fota_v5.2.html` (구 DOC-10)
**상태:** 🟢 정제본 (스파인 A2 · 문서 ID 10→20 재배치)

FW 업데이트의 전자서명 PKI(이중서명·신뢰연쇄) · 패키지 트리(평문/Secure Flash) · 검증 절차 ·
대칭키(SW-Group-Key) 암호화 · Secure Boot·Factory Flashing.

## 본문 목차

| 절 | 제목 |
|----|------|
| 1 | FW 업데이트 패키지 전자서명 PKI (계층·이중서명·신뢰연쇄) |
| 2 | FW 업데이트 패키지 구성트리 (Type A 평문 / Type B Secure Flash) |
| 3 | 패키지 검증 주체 및 절차 (배포/설치 2단계 + 통합 무결성) |
| 4 | 대칭키(SW-Group-Key) 기반 FW 암호화 |
| 5 | Secure Boot 및 공정 초기 주입(Factory Flashing)·Anti-Rollback |

## 다이어그램
- `assets/diagrams/fota-그림1-검증시퀀스.svg`
- `assets/diagrams/fota-그림2-대칭키주입시퀀스.svg`

## 후속 TODO
- [ ] 절차 사양 표준 템플릿 적용(생성·서명·배포 / 설치·검증)
- [ ] **유선/공정 업데이트(Factory Flashing) 절차를 독립 절로 확장** — 현재 §5.1 요약 수준
- [ ] eSync 배포 캠페인 모델과의 정합 점검(호환 지향)
- [ ] 참조표준(ISO 21434, UNECE R156 등) 정확 인용
