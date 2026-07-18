# 오늘은 내가 총무 (오.내.총)

모임에서 결제를 도맡은 총무가 결제 항목만 입력하면 1인당 정산 금액을 자동 계산하고, 감열지 영수증 스타일의 "감성 영수증" 이미지로 만들어 카카오톡 등으로 바로 공유할 수 있는 웹앱입니다.

**🔗 바로 사용하기: [jstd-jeostdu.github.io/chongmu](https://jstd-jeostdu.github.io/chongmu/)**
**📖 이용 안내서: [jstd-jeostdu.github.io/chongmu/guide.html](https://jstd-jeostdu.github.io/chongmu/guide.html)**

<img src="icons/icon.svg" alt="오.내.총 아이콘" width="96" />

## 주요 기능

- **자동 정산 계산** — 항목별 참여자와 고정 금액을 반영해 1인당 금액을 원 단위까지 자동 계산하고, 합계가 항상 일치하는지 검증
- **감성 영수증** — 도트 폰트 + 하이라이트 사진 + 랜덤 멘트로 완성되는 실제 영수증 스타일 이미지, `html2canvas`로 PNG 저장/공유
- **비로그인 공유 링크** — 참여자는 로그인 없이 링크만으로 정산 내역과 계좌 정보 확인 가능
- **클라우드 동기화 (선택)** — Google 로그인 시 Firebase Firestore로 기기 간 데이터 동기화, 로그인하지 않아도 로컬에서 전체 기능 사용 가능
- **PWA 지원** — 홈 화면에 추가해 앱처럼 실행 가능

## 사용 방법

간단한 사용법은 [이용 안내서](https://jstd-jeostdu.github.io/chongmu/guide.html)를 참고하세요. 요약하면:

1. 모임 만들기 (모임명 · 날짜 · 계좌 정보)
2. 멤버 추가
3. 결제 항목 입력 (항목명 · 금액 · 참여자)
4. 정산 결과 확인
5. 멤버별 감성 영수증 생성 후 공유

## 기술 스택

- 단일 `index.html` — 빌드 도구/프레임워크 없는 순수 HTML/CSS/JS
- Firebase Auth(Google 로그인) + Firestore (CDN, compat 모드)
- `html2canvas` — 영수증 이미지 캡처
- Galmuri 웹폰트 — 영수증 도트 폰트
- GitHub Pages로 배포

## 로컬에서 열어보기

빌드 과정이 없으므로 `index.html`을 브라우저로 바로 열면 됩니다. 단, Firebase 관련 기능(로그인·동기화·공유 링크)은 `CONFIG.FIREBASE` 값이 설정되어 있어야 동작하며, 그 외 정산·영수증 기능은 설정 없이도 로컬 전용으로 바로 사용할 수 있습니다.

## 프로젝트 문서

- `오내총_PRD_v1.md` — 제품 요구사항 정의서
- `CLAUDE.md` — 코드베이스 구조와 개발 가이드
- `docs/superpowers/` — 단계별 설계 문서(specs)와 구현 계획(plans)

---

made by JSTD
