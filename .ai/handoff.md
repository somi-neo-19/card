# 세션 인계 문서

마지막 갱신: 2026-09-06 (세션 https://claude.ai/code/session_016m2vCMKmafEsxUHakZWaLK)

## 프로젝트 개요

- 신용카드 실적 허들 관리용 대시보드. `index.html` 단일 파일 + `firestore.rules`(Firestore 보안 규칙) + `FIREBASE_SETUP.md`(비개발자용 설정 가이드)로 구성.
- **원래는 Claude 아티팩트 전용(`window.claude.use('db')`)이었으나, 이번 세션에서 Firebase(Firestore)로 완전히 이전**해서 GitHub Pages로 독립 배포했다. 기존 Claude 아티팩트 링크는 이 작업과 무관하게 별도로 계속 살아있다(같은 코드베이스에서 갈라져 나간 것, 서로 데이터 공유 안 함).
- **실제 배포 주소**: <https://somi-neo-19.github.io/card/> — 사용자가 실사용 중이며 정상 동작 확인됨(커스텀 카드 "토스하나카드" 등 입력 후 기기 간 동기화 확인 완료).
- **Firebase 프로젝트**: `card-9c857` (Firestore + Anonymous Auth 사용, Spark 무료 플랜).

## ⚠️ 중요: git 저장소와 실제 배포본의 관계

이 세션에서는 **사용자가 GitHub 웹 UI로 파일을 직접 업로드**하는 방식으로 작업해왔다(Claude가 git commit/push를 하지 않음 — 사용자가 "Github에 올리는 건 내가 할게"라고 명시적으로 요청함). 그 결과:

- 로컬 git의 `main` 브랜치 HEAD(`1918660`)는 **정기결제 이중계산 수정까지만** 반영돼 있고, 그 이후의 Firebase 이전 작업 전체가 **커밋되지 않은 워킹 디렉터리 변경사항**으로만 존재한다(`git status`에 `index.html` modified, `FIREBASE_SETUP.md`/`firestore.rules` untracked로 나타남).
- 반면 **실제 GitHub Pages에 배포된 코드는 이 워킹 디렉터리 상태와 거의 동일하거나 그 이전 버전**이다 — 매 기능 추가마다 `index.html`을 사용자에게 파일로 전달했고, 사용자가 그걸 GitHub에 업로드해왔기 때문. 다만 **가장 최근에 보낸 파일(요약 대시보드 추가본)을 사용자가 아직 업로드했는지는 이 세션에서 확인되지 않았다.**
- 즉 3곳의 상태가 서로 다를 수 있다: ① 로컬 git 커밋 이력(오래됨), ② 로컬 워킹 디렉터리의 `index.html`(최신, 아래 작업 전부 반영됨), ③ 실제 GitHub Pages 배포본(사용자가 마지막으로 업로드한 시점 기준, 로컬 워킹 디렉터리와 같거나 한두 단계 뒤처져 있을 수 있음).
- **다음 담당자가 git commit/push를 하려면 먼저 사용자에게 확인**: 지금까지 파일 단위로 전달한 방식을 유지할지, 아니면 이 시점부터는 Claude가 직접 git으로 관리(commit + push)할지 물어볼 것. 자동으로 push하지 말 것(이 세션 내내 명시적 요청 없이는 안 했음).

## 이번 세션에서 한 일 (시간순)

1. **씨드 데이터 "변형" 의혹 조사** → 근거 없음으로 결론, 원본 유지.
2. **진행 바 계산 버그 수정** (`barSegmentsHtml`의 변동 구간 폭 계산 연산자 우선순위 실수) → PR #1로 머지 완료(`main`, 커밋 `2e29ce9`).
3. **정기결제 "이미 지난 결제일" 처리 로직**: 처음엔 지난 결제를 실적에 "더하는" 방향으로 잘못 구현했다가, 사용자 피드백("현재 실적은 카드앱에서 본 값이라 이미 정기결제가 반영돼 있다")을 받고 **결제일이 지난 항목은 예정 금액 계산에서 완전히 제외**하는 방향으로 수정(`isPastDue()` + `computeStats()`). 커밋 `1918660`(git에 반영됨, main).
4. **Firebase(Firestore) 이전**: `window.claude.use('db')` API가 Firestore v8 compat SDK와 거의 동일한 문법(`collection()/doc()/onSnapshot()`)이라 비즈니스 로직은 그대로 두고 초기화 부분만 교체. 초기화 흐름: `firebaseConfig`(파일 상단에 실제 값 채워짐, 비밀값 아님) → `firebase.initializeApp()` → `firebase.auth().signInAnonymously()`로 조용히 로그인 → `onAuthStateChanged`에서 `months`/`cards` 컬렉션 구독 시작. 보안 규칙(`firestore.rules`)은 "로그인한 사람만 읽기/쓰기 가능"(익명 로그인도 포함) 수준.
5. **뷰포트 메타 태그 누락 버그 발견 및 수정**: 원래 `<meta name="viewport">`가 아예 없어서(Claude 아티팩트 안에서 열 때는 플랫폼이 자동으로 넣어줘서 몰랐음), GitHub Pages 단독 배포 시 아이폰에서 전체가 축소되어 보이고 카드 2개가 한 줄에 끼어 보이는 문제 발생 → `<!doctype html><html><head>` 정식 구조로 감싸고 뷰포트 메타 추가. 640px 이하에서 카드 1열 강제 + 전체 폰트 확대하는 미디어 쿼리 추가.
6. **포인트 결제 안내문구 CSS 버그 수정**: `class="prow points-note"`가 `.prow`의 5열 그리드에 갇혀 세로로 글자가 깨지던 문제(최초 업로드본부터 있던 버그) → `points-note` 단독 클래스로 분리.
7. **정기결제 입력란 모바일 2줄 재배치**: 640px 이하에서 `grid-template-areas`로 "날짜+항목명+삭제" / "종류+금액" 2줄 배치, 금액 칸 폭 확보.
8. **금액 세자리 콤마 자동 포맷**: 허들/실적/정기결제 금액/추천 금액 입력란을 `type="text" inputmode="numeric"` + `money-input` 클래스로 바꾸고, 입력 중 실시간 콤마 삽입(`onMoneyInput`) + 저장 시 콤마 제거(`parseMoney`) 로직 추가.
9. **당겨서 새로고침(pull-to-refresh)**: 화면 최상단에서 아래로 당기면 캐시를 무시하는 URL(`?_r=타임스탬프`)로 강제 이동해서 항상 최신 배포본을 받아오도록 구현. 터치 이벤트 기반, 라이브러리 없음.
10. **카드 배치 순서 변경 기능**: 각 카드 헤더에 ▲▼ 버튼 추가. 카드 문서에 `order`(숫자) 필드를 추가하고, `activeCards()`가 이 값으로 정렬하도록 변경. 버튼 클릭 시 활성 카드 전체의 `order`를 현재 화면 순서 기준으로 재기록. `order` 없는 기존 카드는 자동으로 맨 뒤 취급(Infinity).
11. **카드별 실적 현황 요약 대시보드 추가**: "지금 결제하려는 금액은?" 섹션 바로 아래, 카드명 + 기존과 동일한 진행률 막대 + 상태 텍스트만 모은 목록. 부족 금액이 큰 카드가 위로 오도록 정렬(`short` > `likely` > `achieved`, 동일 상태 내에서는 `gapMin` 내림차순).

## 데이터 모델 (Firestore)

- `cards/{cardId}`: `{name, archived, createdAt, order?}` — `order`는 10단계에서 추가된 필드, 없으면 정렬 시 맨 뒤로 취급.
- `months/{monthId}`: `{label, createdAt}` (monthId 형식 `YYYY-MM`)
- `months/{monthId}/records/{cardId}`: `{hurdleAmount, currentPerformance, recurringPayments:[{day,label,amountType,amount|min/max,note?}]}`

## 알려진 제한 / 다음 담당자가 참고할 점

- **보안 수준**: 익명 로그인 + "로그인만 하면 전체 읽기/쓰기 가능" 규칙. 은행 앱 수준 아님(사용자에게 이미 설명·동의됨). 카드번호·비밀번호 등 진짜 민감정보는 저장하지 않는 개인용 앱이라는 전제.
- **정기결제 "지난 결제일 제외" 로직의 근본 한계**: 사용자가 카드앱을 확인한 시점과 실제 카드사 정산 반영 시점이 정확히 "오늘"과 안 맞을 수 있음 — 수동 입력 구조 자체의 한계로 이번 세션에서 만들거나 악화시킨 문제는 아님.
- **아이스하키 프로젝트 Firebase 이전**: 사용자가 장기적으로 원한다고 언급했으나, 이번 세션에서는 card 프로젝트가 안정화되는 것까지만 다루고 실제 이전 작업은 아직 시작 안 함. 요청 시 `FIREBASE_SETUP.md`의 1~3단계(프로젝트 생성/Firestore 켜기/규칙 설정) 절차를 그대로 재사용 가능.
- **가장 최근 기능(카드 순서 변경, 요약 대시보드)이 사용자의 실기기에서 아직 검증되지 않음** — 다음 세션/담당자는 사용자에게 결과를 먼저 물어볼 것.
- 이 저장소에는 자동화 테스트나 CI가 없음 — 변경 검증은 매번 Node로 함수를 추출해 실행하는 수동 방식(이번 세션에서도 계산 로직·포맷 로직마다 이 방식으로 검증함).

## 참고 링크

- GitHub 저장소: `somi-neo-19/card` (main 브랜치)
- GitHub Pages(실제 서비스): <https://somi-neo-19.github.io/card/>
- Firebase 콘솔 프로젝트: `card-9c857`
- 이번 세션: https://claude.ai/code/session_016m2vCMKmafEsxUHakZWaLK
