# ALB + HTTPS 연결 트러블슈팅 정리 (2026-07-21)

## 배경

목표: `https://app.jun979.click`에서 프론트(Next.js)와 백엔드(Spring Boot)가 ALB Ingress를 통해 모두 정상 동작하도록 하는 것 (HTTPS 포함).

결과: 프론트/백엔드/ALB/HTTPS 모두 정상 동작 확인 완료. `/api/events` 호출 시 실제 DB 데이터 정상 반환.

---

## 수정한 파일 목록

| 파일 | 수정 내용 |
|---|---|
| `backend/src/main/resources/application.yml` | DB 접속 정보(`url`, `username`, `password`)를 환경변수 참조로 변경 (핵심 원인, 아래 문제 4 참고) |
| `frontend/lib/api/accounts.ts` | 은행 템플릿 잔재 코드 제거 (빌드 에러 원인) |
| `frontend/lib/api/transactions.ts` | 은행 템플릿 잔재 코드 제거 (빌드 에러 원인) |
| `k8s/deploy_https_AWS_frontend.yaml` | 이미지 태그를 `frontend-7.1`로 변경, `imagePullPolicy: Always`로 변경 |
| `k8s/deploy_https_AWS_backend.yaml` | 이미지 태그를 `backend-7.2`로 변경, `imagePullPolicy: Always`로 변경 |

---

## 문제 1: ALB에 프론트/백엔드가 안 붙음, HTTPS 안 됨

- **증상**: `aws-load-balancer-controller`가 클러스터에 없어서 Ingress에 ALB가 안 붙음.
- **원인**: 컨트롤러 미설치, IAM Role(IRSA) 연결 누락/오류, IMDS에서 VPC ID를 못 가져옴, 오래된 nginx Ingress 잔재 충돌.
- **해결**: `aws-load-balancer-controller` Helm 설치 (region/vpcId 명시), ServiceAccount에 정확한 IAM Role(`team5-alb`) 어노테이션 연결, 중복 Ingress 삭제, `ingress_https_qket.yaml` 재적용.
- **결과**: ALB 정상 프로비저닝, `https://app.jun979.click`으로 HTTPS 접근 가능.

## 문제 2: `/api/*` 요청이 백엔드에서 404

- **증상**: `/api/health` 호출 시 Spring Boot 자체 404 JSON 응답.
- **원인**: ALB Ingress는 nginx-ingress와 달리 경로를 자르지 않고 그대로 백엔드로 전달함. 즉 `/api/health` 요청이 백엔드에 `/api/health` 그대로 도착하는데, 백엔드에 `server.servlet.context-path` 설정이 없어서 `/health`로만 매핑돼 있었음.
- **해결**: `application.yml`에 `server.servlet.context-path: /api` 추가.

## 문제 3: Docker 이미지 태그 재사용으로 인한 캐시 문제 (여러 번 발생)

- **증상**: 이미지를 재빌드/재푸시해도 파드가 예전 동작을 그대로 보임 (예: context-path가 계속 `/`로 나옴, 또는 완전히 다른 애플리케이션 코드가 실행됨).
- **원인**: `imagePullPolicy: IfNotPresent` + 같은 태그를 반복 사용 → 특정 노드가 이미 그 태그를 로컬에 캐싱해두면, 실제 ECR 이미지가 바뀌어도 재확인 없이 캐싱된 걸 그대로 사용함.
- **결정적 사례**: 프론트(`qket-frontend`)와 백엔드(`account-backend`)가 **같은 ECR 리포지토리(`team5/ecr/qket`)를 숫자 태그(`1.0, 2.0...`)로 공유**하고 있었음. 프론트 빌드와 백엔드 빌드가 우연히 같은 태그 번호(`:7.0`)를 사용하면서 서로의 이미지를 덮어씀 → 프론트 파드가 백엔드(Spring Boot) 이미지를, 백엔드 파드가 (구버전) 잘못된 이미지를 실행하는 사고 발생.
- **해결**:
  1. 태그를 역할별로 구분: 프론트 `frontend-*`, 백엔드 `backend-*`.
  2. `imagePullPolicy: Always`로 변경 (태그 존재 여부가 아니라 항상 레지스트리에서 재확인 후 pull).
  3. 각 파드가 실제로 어떤 이미지를 실행 중인지는 `kubectl get pod ... -o jsonpath='{.status.containerStatuses[0].imageID}'`로 다이제스트(sha256) 비교해서 확정 진단.
- **팀 공유 필수 사항**: 프론트/백엔드가 ECR 리포지토리를 공유하는 구조 자체가 태그 충돌의 근본 원인입니다. 가능하면 리포지토리를 분리하거나, 최소한 태그 접두사(`frontend-`, `backend-`) 규칙을 팀 전체가 지켜야 합니다.

## 문제 4: 프론트 빌드 실패 (TypeScript 타입 에러)

- **증상**: `npm run build` 시 `AccountDTO`, `TransactionDTO` 타입을 찾을 수 없다는 에러로 빌드 실패.
- **원인**: `frontend/lib/api/accounts.ts`, `frontend/lib/api/transactions.ts`가 이 프로젝트 이전에 쓰던 은행(bank) 템플릿의 잔재 코드였음 (`package.json`의 name도 `bank-frontend`). 연결된 페이지(`app/accounts/page.tsx`, `app/transactions/page.tsx`)도 실제로는 `redirect("/")`만 하는 빈 페이지라 Qket 기능에서 전혀 쓰이지 않음.
- **해결**: 두 파일 내용을 빈 모듈(`export {}`)로 정리해서 빌드만 통과하도록 처리. 기능 영향 없음.

## 문제 5 (근본 원인): `/api/events` 500 에러 — DB 연결 실패

- **증상**: 프론트/ALB/context-path 다 정상인데 `/api/events` 호출 시 계속 500. 로그 상 `java.net.ConnectException: Connection refused`.
- **디버깅 과정**: 보안그룹, RDS 상태, Secret 값, DNS, 파드 IP 대역, 심지어 파드와 완전히 같은 네트워크 네임스페이스에서의 순수 TCP 연결(`nc`)까지 전부 정상으로 확인됨 — 즉 네트워크 문제가 전혀 아니었음.
- **진짜 원인**: `application.yml`의 `spring.datasource.url`이 **환경변수 치환 없이 `jdbc:mysql://localhost:3308/qket...`로 하드코딩**되어 있었음. `username`/`password`도 마찬가지로 `root`/`1234` 하드코딩. Redis 설정(`${REDIS_HOST:localhost}`)은 환경변수를 쓰고 있었는데 DB 설정만 빠져 있었던 것. K8s Secret으로 `DB_HOST` 등을 아무리 잘 주입해도 애플리케이션이 그 값을 참조하지 않고 컨테이너 내부의 `localhost:3308`(아무것도 없는 포트)로 접속을 시도했기 때문에 항상 즉시 `Connection refused`가 발생함.
- **해결**: `application.yml`을 아래와 같이 수정.
  ```yaml
  spring:
    datasource:
      url: jdbc:mysql://${DB_HOST:localhost}:${DB_PORT:3308}/${DB_NAME:qket}?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Seoul
      driver-class-name: com.mysql.cj.jdbc.Driver
      username: ${DB_USERNAME:root}
      password: ${DB_PASSWORD:1234}
  ```
  환경변수가 없을 때는 기존 로컬 개발 값으로 폴백하므로 로컬 개발 환경은 그대로 동작함.
- **결과**: 재배포 후 `/api/events`가 실제 DB 데이터(공연 목록) 정상 반환 확인.

---

## 최종 확인된 상태

- `https://app.jun979.click/` → 프론트 정상 (Next.js)
- `https://app.jun979.click/api/health` → 200 OK
- `https://app.jun979.click/api/events` → 실제 DB 데이터 정상 반환
- ALB, HTTPS(ACM 인증서), Ingress 라우팅 모두 정상

## 남은 할 일 / 팀 공유 사항

1. **ECR 태그 규칙 통일 필요**: 프론트/백엔드가 같은 리포지토리를 쓰는 한, 태그 접두사(`frontend-*` / `backend-*`) 규칙을 반드시 지켜야 함. 가능하면 리포지토리 자체를 분리하는 것을 검토.
2. **`imagePullPolicy: Always`로 통일**: 배포 파일 두 개 모두 이미 반영함. 앞으로 다른 배포 파일을 새로 만들 때도 동일하게 적용 권장.
3. 브라우저에서 로그인/예매 플로우 등 실제 UI 동작까지 최종 확인 필요 (API 레벨 확인만 완료된 상태).
4. `team5-alb` IAM Role의 trust policy에 `sub` 조건이 빠져 있음 (현재는 동작에 문제없지만 보안상 나중에 보완 권장).
