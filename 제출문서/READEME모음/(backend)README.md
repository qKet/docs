# backend

Qket(공연 예매 시스템)의 백엔드. Spring Boot 3.5 / Java 21, MyBatis로 MySQL에 접근하고 Redis를 세션·대기열·분산 락에 함께 쓴다.

## 기술 스택

| 영역 | 사용 기술 |
|---|---|
| 언어/프레임워크 | Java 21, Spring Boot 3.5.6 |
| DB 접근 | MyBatis, MySQL 8 |
| 세션/대기열/락 | Redis(Spring Session), Redisson(분산 락) |
| 인증 | 세션 기반 자체 로그인 + Google/Kakao/Naver OAuth2 |
| 결제 | Toss Payments |
| 외부 연동 | AWS S3(포스터), AWS SQS(이메일 인증·예매 알림 발행), OpenAI API(리뷰 스포일러 자동판별) |
| 관측성 | Spring Actuator(`/actuator/health`, 8081 포트) + Micrometer(Prometheus) |

## 패키지 구조

```
com.exam
├── admin          관리자 기능(사용자/카테고리/프로그램/메뉴/공연 관리)
├── auth            회원가입/로그인/OAuth2/세션
├── common          공통 설정, 인터셉터, 예외 처리, 유틸
├── notification   SQS 발행(이메일 인증, 예매확정/취소, 오픈 알림)
├── payment         Toss Payments 연동
├── queue           대기열(Redis 기반 순번 관리)
├── reservation     공연/회차/좌석/예매
└── review          리뷰, 스포일러 자동판별(OpenAI)
```

Controller → Service → Mapper(MyBatis XML) 4단 구조를 따른다. 자세한 레이어 규칙은 `CLAUDE_LLM_WIKI`의 `wiki/architecture/backend-layer-and-package-structure.md` 참고.

## 로컬 실행

```bash
# 1. MySQL/Redis 컨테이너 기동 (schema.sql/data.sql 자동 초기화됨)
docker-compose up -d

# 2. 애플리케이션 실행
./gradlew bootRun
```

- API는 `http://localhost:8080/api`로 뜬다 (`server.servlet.context-path: /api`).
- 헬스체크는 별도 포트(`http://localhost:8081/actuator/health`) — 비즈니스 트래픽이 몰려도 헬스체크가 영향받지 않도록 분리돼 있음.
- `application.yml`의 기본값(DB/Redis `localhost`, DB 비밀번호 `1234`)이 `docker-compose.yml`과 맞춰져 있어서 별도 설정 없이 바로 붙는다. **이 기본값은 로컬 전용이며 배포 환경(release/prod)에서는 절대 쓰이지 않는다** — 실배포는 K8s Secret(ESO가 Secrets Manager에서 채움)이 항상 덮어쓴다.

## 환경변수

로컬 실행에는 필요 없고(기본값으로 동작), 배포 환경에서 K8s Secret/ConfigMap으로 주입되는 값들이다.

| 변수 | 용도 |
|---|---|
| `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD`, `DB_POOL_SIZE` | MySQL 접속 |
| `REDIS_HOST`, `REDIS_PORT` | Redis 접속(세션/대기열/락 공유) |
| `S3_BUCKET`, `AWS_REGION`, `CLOUDFRONT_DOMAIN` | 포스터 이미지 저장/제공 |
| `NOTIFICATION_QUEUE_URL`, `OPEN_ALERT_QUEUE_URL` | SQS 발행 대상(이메일 인증·예매 알림 / 오픈 알림, 각각 별도 큐) |
| `OPENAI_API_KEY` | 리뷰 스포일러 자동판별 |
| `APP_BASE_URL` | 소셜 로그인 완료 후 리다이렉트할 프론트 origin |
| `CORS_ALLOWED_ORIGINS` | CORS 허용 도메인(콤마 구분) — 배포 환경엔 필수, 안 넣으면 배포된 프론트가 CORS로 막힘 |
| `GOOGLE_CLIENT_ID/SECRET`, `KAKAO_CLIENT_ID/SECRET`, `NAVER_CLIENT_ID/SECRET` | 소셜 로그인 |
| `TOSS_SECRET_KEY` | Toss Payments 시크릿 키 |

실제 값은 AWS Secrets Manager + External Secrets Operator(ESO)로 자동 동기화된다 — GitHub Secrets나 코드에 직접 두지 않는다.

## 배포

`release`/`main` 브랜치에 push하면 GitHub Actions가 테스트 → 이미지 빌드 → ECR push → `qKet/CD` 레포 write-back까지 수행하고, ArgoCD가 이를 감지해 EKS에 배포한다. 파이프라인 전체 구조는 `docs` 레포의 `CICD_파이프라인.md` 참고.

## 참고

이 서비스의 설계 배경(대기열 구조, 인가 체계, 컨벤션 등)은 `CLAUDE_LLM_WIKI` 레포에 정리돼 있다:

- `wiki/architecture/backend-layer-and-package-structure.md`
- `wiki/architecture/auth-and-authorization.md`
- `wiki/conventions/comment-rules.md`, `wiki/conventions/mybatis-conventions.md`, `wiki/conventions/api-response-format.md`
- `wiki/troubleshooting/` — 실제 겪은 버그와 해결 과정
