# 세션 인계 문서

마지막 갱신: 2026-09-06 (세션 https://claude.ai/code/session_01TVu6DWyRiUNCXBaGsTcjwd)

## 프로젝트 개요

- 신용카드 혜택 실적 허들 관리용 대시보드. `index.html` 단일 파일(HTML+CSS+JS 인라인) + `firestore.rules`(Firestore 보안 규칙) + `FIREBASE_SETUP.md`(비개발자용 Firebase 설정 가이드)로 구성.
- Firebase(Firestore) 기반으로 GitHub Pages에 독립 배포되어 있다. **실제 서비스 주소**: <https://somi-neo-19.github.io/card/> — 사용자가 실사용 중.
- **Firebase 프로젝트**: `card-9c857` (Firestore + Anonymous Auth, Spark 무료 플랜). `index.html` 상단의 `firebaseConfig`에 실제 값이 채워져 있음(비밀값 아님, 공개 API 키).
- 원래는 Claude 아티팩트 전용(`window.claude.use('db')`)이었던 것을 과거 세션에서 Firebase로 이전했다. 그 시절 Claude 아티팩트 링크가 남아있다면 이 저장소와는 완전히 별개(데이터 공유 안 됨).

## 현재 워크플로우 (git ↔ 배포본 관계 — 과거와 달라졌음)

**과거(이전 세션들)에는 사용자가 GitHub 웹 UI로 파일을 직접 업로드**하는 방식이었고, 그래서 로컬 git 커밋 이력과 실제 배포본이 서로 어긋나 있던 시기가 있었다. **이번 세션부터는 다음 흐름으로 정착됨**:

1. Claude가 세션 브랜치(`claude/continue-session-kcfmni`)에서 직접 코드를 수정하고 git commit + push.
2. Claude가 `main`으로의 Pull Request를 생성.
3. **사용자가 GitHub 웹에서 직접 "Merge pull request" 버튼을 눌러 병합** (사용자가 명시적으로 "내가 직접 머지할게"라고 요청 — 병합은 Claude가 하지 않음).
4. 병합되는 순간 GitHub Pages가 자동 재배포(보통 30초~2분).

즉 **git의 `main` 브랜치와 실제 배포본은 항상 일치한다** (더 이상 어긋날 걱정 없음). 다음 세션을 시작할 때는 로컬 브랜치를 `origin/main`으로 fast-forward 동기화하는 것부터 시작할 것 — 지금까지는 항상 divergence 없이 단순 fast-forward였다.

세션 시작 시 표준 절차: `git fetch origin main` → 로컬 세션 브랜치가 뒤처져 있으면 `git merge --ff-only origin/main` 후 push. PR 생성 전에는 항상 사용자에게 머지 방식(Claude가 직접 머지 vs PR만 만들고 사용자가 머지)을 확인할 것 — 최근엔 항상 사용자가 직접 머지를 선택함.

## 화면 구성 (현재 UI, 이번 세션에서 크게 간소화됨)

- 상단: 월 탭(`monthbar`) + "새 달" 추가 폼.
- **"지금 결제하려는 금액은?" 추천 섹션과 "카드별 실적 현황" 요약 대시보드는 이번 세션에 완전히 삭제됨** (관련 CSS/JS 포함, 코드 흔적 없음). 알고리즘 추천이 실질적 도움이 안 된다는 사용자 피드백에 따른 것 — 되살릴 필요 없음, 의도된 삭제.
- 카드 그리드: 카드마다 이름 / ▲▼ 순서 변경 / 보관 버튼 / 달성 상태 pill / 허들·실적 입력란(콤마 자동 포맷) / 진행률 막대(`.bar`, 카드 자체에 남아있음 — 실적 확인은 여기서) / **정기결제 목록은 기본 접힘 상태**, "자세히보기 ▾" 클릭 시 펼쳐져서 목록 확인·추가·삭제 가능(다시 누르면 "접기 ▲"). 펼침 상태는 새로고침하면 초기화됨(의도된 동작, 저장 안 함).
- 하단: 보관된 카드 목록(펼치기 토글), 당겨서 새로고침(pull-to-refresh, 캐시 무시하고 최신 배포본 강제 로드).

## 데이터 모델 (Firestore)

- `cards/{cardId}`: `{name, archived, createdAt, order?}` — `order`(숫자)로 카드 그리드 정렬, 없으면 맨 뒤 취급.
- `months/{monthId}`: `{label, createdAt}` (monthId 형식 `YYYY-MM`)
- `months/{monthId}/records/{cardId}`: `{hurdleAmount, currentPerformance, recurringPayments:[{day,label,amountType:'fixed'|'range'|'points',amount|min/max,note?}]}`
- 정기결제 중 결제일(day)이 오늘(또는 조회 중인 달) 기준으로 이미 지난 항목은 "카드앱에 이미 반영된 값"으로 간주해 `computeStats()`의 예정 금액 계산에서 제외한다(`isPastDue()`). 이 로직 자체는 사용자 확인 완료된 의도된 설계 — 손대지 말 것.

## 이번 세션에서 한 일

1. **이전 세션 인계 파악 + git 동기화**: 세션이 옛 커밋 기준 브랜치로 시작돼서 이전 세션의 마지막 작업(Firebase 이전 전체)이 반영 안 된 상태였음 → `origin/main`으로 fast-forward 동기화.
2. **MyData API 연동 가능성 조사 (조사만, 코드 변경 없음)**: 카드사 실적을 API로 자동 수집할 수 있는지 문의받음 → 마이데이터 정식 API는 금융위 인가 사업자만 사용 가능해 개인 프로젝트로는 불가능하다고 결론. 대안(안드로이드 알림 자동화 등)도 사용자가 아이폰을 쓰고 있어 현실성이 낮다고 판단해 **수동 입력 유지로 결론**. 향후 세션에서 다시 꺼내지 않아도 됨(사용자가 이미 결정함).
3. **UI 간소화**: 위 "화면 구성" 항목 참고 — 추천 섹션·요약 대시보드 삭제, 정기결제 접기 UI 도입. PR #2로 병합 완료.
4. **모바일 실적 입력 버그 수정 (2단계)**: 사용자가 "모바일에서 현재 실적 입력이 반영 안 됨"을 제보.
   - **1차 원인 (PR #3로 수정, 병합 완료)**: 카드/달을 막 연 직후처럼 `months/{month}/records/{cardId}` 문서가 아직 생성되기 전에 값을 입력하면, 기존 `update()` 호출이 "문서 없음"으로 실패하고 그 실패 처리(`ensureRecord`)가 문서 전체를 허들 0/실적 0/정기결제 빈 배열로 되돌려버리는 경쟁 상태였음. `updateRecordField`/`writePayments`를 `update()` 대신 `set(..., {merge:true})`로 바꾸고, `ensureRecord`를 Firestore 트랜잭션으로 바꿔서 사용자 값이 먼저 반영된 경우 기본값 초기화가 그 값을 덮어쓰지 않도록 수정. Firestore를 버전 기반 트랜잭션 재시도까지 구현한 스텁으로 재현해 수정 전/후 차이를 직접 확인함.
   - **2차 원인 (PR #4로 수정, 병합 완료)**: 1차 수정 배포 후에도 증상이 계속돼서 추가 조사 → 모바일(특히 iOS) 숫자 전용 키패드는 '완료' 버튼이 없는 경우가 많아, 사용자가 값을 입력한 뒤 다른 곳을 탭해 포커스를 옮기지 않으면 `change`(blur) 이벤트 자체가 안 나서 저장 로직이 호출조차 안 될 수 있음. 허들/실적/정기결제 금액 입력란에 **입력을 멈춘 지 800ms 뒤 자동 저장하는 디바운스 로직**(`scheduleAutoSave`)을 추가 — blur 발생 여부와 무관하게 값이 저장됨. 헤드리스 브라우저로 실제 키 입력을 재현해 블러 없이도 자동 저장되는 것을 확인. **사용자가 실기기에서 정상 동작 확인 완료** ("확인 완료").
   - 이 샌드박스 환경에는 Playwright의 WebKit(사파리 엔진)이 설치돼 있지 않아 Chromium으로만 검증했음 — 실제 사파리 고유 동작까지 100% 재현한 것은 아니라는 점 참고. 다만 사용자 실기기 확인까지 끝났으므로 이 건은 종결.

## 알려진 제한 / 다음 담당자가 참고할 점

- **디바운스 자동 저장의 트레이드오프**: 800ms 자동 저장이 완료되면 Firestore 구독이 카드 그리드 전체를 다시 그리는데, 사용자가 800ms 넘게 멈췄다가 타이핑을 이어가는 드문 타이밍이면 그 순간 입력 포커스가 끊길 수 있음(재입력 필요). 데이터 유실보다 훨씬 가벼운 문제라 판단해 그대로 둠 — 재발 보고 시 debounce 시간을 늘리거나(현재 800ms) 포커스 유지 로직을 고려할 것.
- **보안 수준**: 익명 로그인 + "로그인만 하면 전체 읽기/쓰기 가능" Firestore 규칙(`firestore.rules`). 은행 앱 수준 보안 아님 — 사용자에게 이미 설명·동의됨. 카드번호·비밀번호 등 진짜 민감정보는 저장하지 않는 개인용 앱이라는 전제.
- **아이스하키 프로젝트 Firebase 이전**: 사용자가 장기적으로 원한다고 언급했으나 아직 착수 안 함. 요청 시 `FIREBASE_SETUP.md`의 1~3단계(프로젝트 생성/Firestore 켜기/규칙 설정) 절차를 그대로 재사용 가능.
- **자동화 테스트/CI 없음**: 변경 검증은 매번 Node로 관련 함수를 추출해 실행하거나, Firestore를 in-memory 스텁으로 대체한 뒤 Playwright(Chromium)로 렌더링·상호작용을 직접 재현하는 수동 방식. 이 세션에서 확립한 패턴이니 다음 세션도 동일하게 재사용 가능(임시 스텁/테스트 파일은 저장소에 커밋하지 않고 `/tmp`에서 검증 후 정리함).
- **PR 워치 관련**: 이 세션에서 `subscribe_pr_activity`로 PR #2를 지켜보다가 사용자가 직접 머지해서 자동 구독 해제됨. 이후 PR #3, #4는 사용자가 "내가 직접 머지할게"라고 해서 애초에 구독하지 않음. 다음 세션도 PR 생성 후 "지켜볼지 직접 머지할지" 매번 물어볼 것 — 최근 경향은 사용자가 직접 머지.

## 참고 링크

- GitHub 저장소: `somi-neo-19/card` (main 브랜치, git과 배포본 항상 동기화됨)
- GitHub Pages(실제 서비스): <https://somi-neo-19.github.io/card/>
- Firebase 콘솔 프로젝트: `card-9c857`
- 이번 세션에서 머지된 PR: [#2](https://github.com/somi-neo-19/card/pull/2)(UI 간소화), [#3](https://github.com/somi-neo-19/card/pull/3)(Firestore 문서-없음 경쟁 상태 수정), [#4](https://github.com/somi-neo-19/card/pull/4)(모바일 blur 없는 자동 저장)
- 이번 세션: https://claude.ai/code/session_01TVu6DWyRiUNCXBaGsTcjwd
