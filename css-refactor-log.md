# CSS 표준화 / 컴포넌트화 작업 로그

작
## 1단계 — CSS 값 표준화 (완료)

하드코딩된 px 값들을 `:root`에 정의한 CSS 변수로 교체.
폰트 크기, 여백/간격, 색상변수(기존) 추가해서 하드코딩된 값들 찾아서 변수로 바꿔놨어요
지금 styles/base.css 파일에 설명 되어있을거에요.

적용 파일: `globals.css`, `BookButton.tsx`, `app/mypage/page.tsx`, `app/admin/performances/page.tsx` 등 인라인 스타일 쓰던 컴포넌트 전체.

---

## 2단계 — CSS 파일 분리 + 주석 (완료)

### 구조 변경
`app/globals.css` 1346줄 → 역할별로 13개 파일로 분리, `@import`로 재조립.
13개로 이미 코드에 분류가 되어있길래 분류해서 파일들 만들어놨고 기존에 있던 globals.css는 조립창구 역할로 쓰면될거같아요.
.tsx 파일을 건드려서 구조 바꿀까 했는데 그냥 import로만 해도 충분할거같아서 기존에있던 .tsx파일들은 안 건드렸어요
각 파일 상단에 "이 파일 역할 / 담당 화면" 주석 + 클래스별 설명 주석 추가했습니다.
```
app/globals.css 
styles/
 ├─ base.css       (reset, :root 변수)
 ├─ nav.css        (상단 네비게이션)
 ├─ layout.css     (페이지 공통 레이아웃 틀)
 ├─ auth.css       (로그인/회원가입 + 폼)
 ├─ button.css     (버튼 4종)
 ├─ card.css       (카드 / 공연 목록 그리드)
 ├─ badge.css      (상태 뱃지)
 ├─ queue.css      (대기열 화면)
 ├─ seat.css       (좌석 선택 화면)
 ├─ mypage.css     (마이페이지)
 ├─ message.css    (공통 에러/성공/로딩 메시지)
 ├─ admin.css      (관리자 전체 화면)
 └─ responsive.css (반응형 미디어쿼리, 항상 마지막 import)
```

### 발견 및 수정한 버그 
버그 2가지 있었는데 
1. 기존파일에 버튼에 대한 css가 중복 정의 되어있어서 나중 정의로 되어있는걸로 놔두고 나머지하나는 삭제했습니다. '.btnSecondary' / '.btnDanger' 이거 두개 였습니다. -> 수정완료!
2. 관리자페이지 드롭다운이 원래 선택되면 보라색으로 바뀌고 수정된 행은 색이 바뀌는걸로 되어있었는데 코드에 정의해놓지 않은 변수로 되어있어서 적용이 안되고있더라고요. `.adminSelect` / `.adminInput` 두 개 교체해서 색 변경 되도록 해놨습니다 -> 수정완료!

---

## 3단계 — JSX 컴포넌트화 (Button, Badge 완료)

반복되는 UI 패턴을 재사용 컴포넌트로 추출. 값/디자인은 그대로, className만 컴포넌트 호출로 교체.

### Button — 완료
`<button className="btnPrimary/btnSecondary/btnDanger">` → `<Button variant="...">`

- `frontend/components/ui/Button.tsx` 생성. `variant`(primary/secondary/ghost/danger) + `fullWidth` prop으로 button.css의 4종 스타일 처리.
- 아래 8개 파일 전부 교체 완료:

- [x] `components/BookButton.tsx`
- [x] `app/(auth)/signup/page.tsx`
- [x] `app/(auth)/login/page.tsx`
- [x] `app/mypage/page.tsx` (2곳: "공연 보러 가기", "취소")
- [x] `app/admin/performances/page.tsx` (7곳: 추가/수정/삭제/회차삭제/회차추가/닫기/저장)
- [x] `app/admin/performances/new/page.tsx` (4곳: 목록으로/회차추가/회차삭제/공연등록)
- [x] `app/admin/users/page.tsx` (저장)
- [x] `app/seats/[scheduleId]/page.tsx` (예매하기)

건드리지 않은 버튼: `admin/performances/page.tsx`의 모달 닫기(X) 버튼(`adminModalClose`)과 `seats` 페이지의 개별 좌석 버튼(`seat` 클래스) — 이 둘은 Button의 4가지 variant에 해당하지 않는 별도 스타일이라 원래 코드 그대로 둠.

### Badge — 완료
`<span className="badge badge종류">` → `<Badge variant="...">`

- `frontend/components/ui/Badge.tsx` 생성. `variant`(open/closed/soldout/vip/r/s)로 badge.css의 6종 색상 처리.
- 아래 4개 파일, 총 6곳 교체 완료:

- [x] `components/BookButton.tsx` (2곳: "예매 전", "예매 마감" — 둘 다 `closed`)
- [x] `app/page.tsx` (1곳: "매진" — `soldout`)
- [x] `app/seats/[scheduleId]/page.tsx` (2곳: 좌석 구역 등급 표시, 선택한 좌석 등급 표시 — 좌석 grade 값(VIP/R/S)을 소문자로 변환해서 variant로 사용)
- [x] `app/mypage/page.tsx` (1곳: 예매 상태 뱃지 — 기존 `STATUS_CLASS` 문자열 매핑을 `STATUS_VARIANT`로 교체)

### 컴포넌트화 로드맵 (실제 코드 반복 빈도 기준)

| 순서 | 컴포넌트 | 대상 className | 반복 횟수 | 비고 |
|---|---|---|---|---|
| 1 | Button | `btnPrimary`/`btnSecondary`/`btnGhost`/`btnDanger` | 20+ | 완료 |
| 2 | Badge | `badge` + `badgeOpen`/`badgeClosed`/`badgeSoldout`/`badgeVip`/`badgeR`/`badgeS` | 6곳 | 완료 |
| 3 | FormField / Input | `field`+`fieldLabel`+`fieldInput`(로그인/회원가입), `adminFormRow`+`adminLabel`+`adminInput`(관리자 폼) | 7~13회 | 완료 |
| 4 | PageHeader | `pageWrap`+`pageTitle`+`pageSubtitle`+`pageHeader` | 6~8회 | 완료 |
| 5 | StatusMessage | `loadingMsg`, `errorMsg` | 3~7회 | 완료 |
| 낮음 | 기타 | `seatLegendItem`/`seatLegendDot`(좌석 범례), `adminPosterWrap` 등 포스터 업로드 영역 | 2~4회 | 반복 적어서 급하지 않음, 여유 있을 때 |

### FormField / Input — 완료

`<div className="field/adminFormRow"><label className="fieldLabel/adminLabel">...</label><input className="fieldInput/adminInput" .../></div>` → `<FormField variant="auth|admin" label="..." required?><Input variant="auth|admin" .../></FormField>`

- `frontend/components/ui/FormField.tsx` 생성. `variant`(auth/admin) + `label` + `required` prop으로 라벨+래퍼 div 처리. children으로 실제 입력 요소(Input, select 등)를 그대로 받음.
- `frontend/components/ui/Input.tsx` 생성. `variant`(auth/admin)로 fieldInput/adminInput 처리. `forwardRef` 지원(관리자 폼 유효성 검사 실패 시 focus 이동에 필요).
- select 태그(공연장 선택 등)는 Input 대상이 아니라서 className="adminInput" 그대로 유지, FormField로 라벨/래퍼만 감쌈.
- 아래 4개 파일 전부 교체 완료:

- [x] `app/(auth)/login/page.tsx` (2곳: 아이디, 비밀번호)
- [x] `app/(auth)/signup/page.tsx` (5곳: 아이디/이름/이메일/비밀번호/비밀번호 확인)
- [x] `app/admin/performances/page.tsx` (수정 모달: 제목/포스터/회차별 시간 2개×N/회차 추가 시간 2개)
- [x] `app/admin/performances/new/page.tsx` (제목/공연장/포스터/회차별 시간 2개×N)

건드리지 않은 부분: `adminFormSectionHeader` 안의 "회차 목록 *" 라벨(래퍼 div 없이 flex 헤더의 일부라 FormField 구조와 안 맞음), 회차 목록 섹션 제목 라벨(`adminLabel` 단독 사용) — 둘 다 원래 코드 그대로 둠.

### PageHeader — 완료

`<div className="pageWrap(Wide)"><div className="pageHeader/adminPageHeader">...</div>...나머지 페이지...</div>` → `<PageHeader title="..." subtitle="..." variant="default|admin" wide? actions?>...나머지 페이지...</PageHeader>`

- `frontend/components/ui/PageHeader.tsx` 생성. 페이지 최상위 wrapper(`pageWrap`/`pageWrapWide`)까지 함께 감싸서 `children`으로 나머지 내용을 받음. `variant="admin"`이면 `adminPageHeader` 레이아웃 + `actions` slot(우측 버튼/영역, 래핑 여부는 호출부 재량) 사용.
- 아래 6개 파일 전부 교체 완료:

- [x] `app/page.tsx` (공연 목록, default)
- [x] `app/mypage/page.tsx` (default)
- [x] `app/seats/[scheduleId]/page.tsx` (default, `wide`)
- [x] `app/admin/users/page.tsx` (admin, actions에 상태 메시지+저장 버튼)
- [x] `app/admin/performances/page.tsx` (admin, actions에 "+ 공연 추가" 버튼)
- [x] `app/admin/performances/new/page.tsx` (admin, actions에 "← 목록으로" 버튼)

건드리지 않은 부분: 각 파일의 데이터 로딩 중 조기 return(`{isLoading && return <div className="pageWrap">...}`) — 제목 없이 로딩 문구만 있는 상태라 PageHeader 구조와 안 맞아서 `pageWrap` div는 그대로 두고 안의 문구만 StatusMessage로 교체.

### StatusMessage — 완료

`<p className="loadingMsg/errorMsg/successMsg">...</p>` → `<StatusMessage variant="loading|error|success">...</StatusMessage>`

- `frontend/components/ui/StatusMessage.tsx` 생성. `variant`(loading/error/success)로 loadingMsg/errorMsg/successMsg 3종 처리. 기본 태그는 `<p>`, 인라인 배치가 필요한 자리(버튼 옆 등)는 `as="span"`으로 교체 가능.
- 로드맵에는 loadingMsg/errorMsg만 적혀있었지만, 기존 코드에 `msg.ok ? "successMsg" : "errorMsg"` 식 삼항연산자로 두 클래스가 항상 짝지어 쓰이고 있어서 successMsg도 같은 컴포넌트의 variant로 포함— CSS 값 변경 없이 동일 패턴 재사용.
- 아래 8개 파일, 총 12곳 교체 완료:

- [x] `app/(auth)/login/page.tsx` (에러 1곳)
- [x] `app/(auth)/signup/page.tsx` (에러 1곳)
- [x] `app/queue/page.tsx` (에러 1곳)
- [x] `app/page.tsx` (로딩/빈 목록 1곳)
- [x] `app/mypage/page.tsx` (로딩 1곳)
- [x] `app/seats/[scheduleId]/page.tsx` (로딩 1곳, 성공 1곳)
- [x] `app/admin/users/page.tsx` (로딩 1곳, 저장 성공/실패 1곳 — `as="span"`)
- [x] `app/admin/performances/page.tsx` (로딩 2곳, 저장 성공/실패 1곳)
- [x] `app/admin/performances/new/page.tsx` (로딩 1곳, 등록 성공/실패 1곳)

### 검증

- `npx tsc --noEmit` 통과 (타입 에러 0건)
- 실행 중인 dev 서버(localhost:3000)에서 로그인/회원가입/공연 목록 페이지 실제 렌더링 확인 — `document.querySelectorAll` 로 렌더된 DOM의 className이 기존 값(`fieldInput`, `field`, `pageHeader`/`pageTitle`/`pageSubtitle` 등)과 완전히 동일함을 확인. 콘솔 에러 없음.
- 관리자 페이지는 로그인 세션이 없어 실제 화면 클릭 테스트는 못 했고, 타입체크 + 코드 리뷰로만 검증함. 실제 로그인 후 한 번 확인 권장.

---

*최종 업데이트: 2026-07-30*
