# frontend

Qket(공연 예매 시스템)의 프론트엔드. Next.js 14(App Router) + TypeScript.

## 기술 스택

| 영역 | 사용 기술 |
|---|---|
| 프레임워크 | Next.js 14.2 (App Router, `output: "standalone"`) |
| 언어 | TypeScript, React 18 |
| 결제 | `@tosspayments/tosspayments-sdk` |
| 관측성 | `@grafana/faro-web-sdk` (프론트 RUM — 에러/성능 지표 수집) |

## 디렉토리 구조

```
app/
├── (auth)/          로그인, 회원가입, 비밀번호 찾기
├── admin/           관리자 페이지
├── mypage/          예매 내역 조회
├── payments/        결제(체크아웃)
├── performances/    공연 목록/상세/등록
├── seats/           좌석 선택
├── events/          공연 오픈 이벤트 관련
└── healthz/         헬스체크 엔드포인트
components/          공용 UI 컴포넌트(components/ui/ 재사용 컴포넌트 목록은 wiki 참고)
context/             AuthContext — 로그인 세션 상태
lib/api/             백엔드 API 호출 함수(fetch 래퍼) — 엔드포인트 도메인별로 분리
middleware.ts        경로 기반 서버 사이드 인가 가드
```

## 로컬 실행

```bash
npm install
npm run dev
```

- `http://localhost:3000`에서 뜬다.
- 로컬(`next dev`)에서는 `/api/*` 요청을 `next.config.mjs`의 `rewrites()`가 백엔드(`CLUSTER_IP` 환경변수, 기본값 `http://localhost:8080`)로 프록시해준다 — 배포 빌드(`next build`, standalone)에는 이 rewrites가 아예 안 들어가고, 실배포에서는 ALB가 path 라우팅(`/api` → backend, 나머지 → frontend)으로 대신 처리한다. 이 차이 때문에 로컬 개발용 프록시와 배포용 라우팅 로직이 코드상 분리돼 있다.
- backend가 먼저(`docker-compose up -d` + `./gradlew bootRun`) 떠있어야 로그인 등 API 호출이 정상 동작한다.

## 환경변수

| 변수 | 용도 |
|---|---|
| `NEXT_PUBLIC_TOSS_CLIENT_KEY` | Toss Payments 위젯 클라이언트 키 — CI 빌드 시점에 Secrets Manager에서 OIDC로 직접 조회해 주입(GitHub Secrets 미사용) |
| `NEXT_PUBLIC_FARO_COLLECTOR_URL` | Grafana Faro(RUM) 수집 엔드포인트 |
| `NEXT_PUBLIC_APP_ENV` | 현재 환경 구분(release/prod) — Faro 등에서 환경 태깅용 |
| `CLUSTER_IP` | 로컬 `next dev`의 `/api/*` 프록시 대상. K8s Deployment의 env가 실제 배포값으로 덮어씀 |

## 인가 가드

`middleware.ts`가 `/admin/:path*`, `/performances/:path*`만 서버 사이드에서 role 체크한다 — 그 외 경로(`/mypage`, `/seats/[scheduleId]`, `/payments/checkout` 등)는 현재 클라이언트 사이드 가드가 없다. 화면 노출(메뉴)과 실제 API 접근 권한은 의도적으로 분리돼 있고, 보안 경계는 백엔드 API 레벨(`AdminAccessInterceptor` 등)이 최종 책임진다 — 프론트 가드는 UX 목적이 크다는 점을 염두에 둘 것.

## 배포

`release`/`main` 브랜치에 push하면 GitHub Actions가 `npm ci && npm run build` → 이미지 빌드 → ECR push → `qKet/CD` 레포 write-back까지 수행하고, ArgoCD가 이를 감지해 EKS에 배포한다. 파이프라인 전체 구조는 `docs` 레포의 `CICD_파이프라인.md` 참고.

## 참고

- `CLAUDE_LLM_WIKI`의 `wiki/architecture/frontend-folder-structure.md`
- `wiki/conventions/api-client-usage.md` — `apiFetch` 사용 규칙
- `wiki/conventions/reusable-ui-components.md` — `components/ui/` 목록
- `wiki/decisions/2026-08-11-frontend-api-routing-alb-not-rewrites.md` — `/api` 라우팅을 rewrites에서 ALB로 옮긴 이유
