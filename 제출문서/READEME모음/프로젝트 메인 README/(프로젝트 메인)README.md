# Q-Ket (Qket)

공연 예매 시 발생하는 동시접속 트래픽 폭주를, 대기열 시스템과 프로덕션 수준의 인프라로 안정적으로 처리하는 공연 예매 플랫폼입니다.

인기 콘서트·뮤지컬 예매가 열리는 순간 수천~수만 명이 동시에 접속하는 "티켓 오픈런"을, 서버를 무작정 키우는 대신 **대기열을 통한 트래픽 관리**로 풀어내는 걸 핵심 과제로 삼았습니다. 화면과 API가 동작하는 것에서 그치지 않고, 실제로 부하가 몰렸을 때 시스템이 어디서 무너지고 어떻게 버티는지까지 직접 설계하고 실측했습니다.

## 이렇게 구현했습니다

<table>
<tr>
<td width="50%">

**① 공연 목록**
카테고리·검색으로 원하는 공연을 찾습니다.
<img src="../photo/01-list.png" width="100%">

</td>
<td width="50%">

**② 공연 상세 · 회차 선택**
캘린더에서 회차를 고르고 출연진을 확인합니다.
<img src="../photo/02-detail.png" width="100%">

</td>
</tr>
<tr>
<td width="50%">

**③ 대기열 진입**
예매 오픈 순간 몰린 사용자를 대기열에 순서대로 세웁니다.
<img src="../photo/03-queue.png" width="100%">

</td>
<td width="50%">

**④ 순번 도달 → 자동 입장**
처리 가능한 만큼만 실시간으로 좌석 선택 페이지로 들여보냅니다.
<img src="../photo/04-queue-success.png" width="100%">

</td>
</tr>
<tr>
<td width="50%">

**⑤ 좌석 선택**
등급별 좌석을 실시간 상태로 보여주고, 동시 클릭도 분산락으로 막습니다.
<img src="../photo/05-seats.png" width="100%">

</td>
<td width="50%">

**⑥ 결제 승인**
Toss Payments 연동 결제 — 승인과 동시에 예매가 확정됩니다.
<img src="../photo/06-payment.png" width="100%">

</td>
</tr>
<tr>
<td width="50%">

**⑦ 마이페이지**
예매/결제 내역을 확인하고 환불을 요청할 수 있습니다.
<img src="../photo/07-mypage.png" width="100%">

</td>
<td width="50%"></td>
</tr>
</table>

## 아키텍처

**AWS 인프라** — VPC/서브넷 이중화, ALB(공개용/관리자용 분리), EKS 워커노드 오토스케일링, RDS/ElastiCache Multi-AZ, ECR·Secrets Manager·SQS-Lambda-SES까지 전부 Terraform으로 관리합니다.
<img src="../photo/08-arch-aws.png" width="100%">

**EKS 클러스터 내부** — KEDA(파드)·Karpenter(노드) 오토스케일링, External Secrets Operator로 시크릿 자동 동기화, ArgoCD·Prometheus/Grafana/Alertmanager로 배포와 관측을 자동화했습니다.
<img src="../photo/09-arch-eks.png" width="100%">

**GitOps CI/CD 흐름** — 앱 배포는 GitHub Actions → CD 레포 → ArgoCD Sync로 자동화하고, 인프라 배포는 의도적으로 수동(Terraform apply)으로 분리해 실수로 인한 인프라 변경을 막습니다.
<img src="../photo/10-arch-cicd.png" width="100%">

## 기능

- 회원가입/로그인(자체 + Google·Kakao·Naver 소셜 로그인)
- 공연 목록/상세, 회차별 좌석 조회
- **대기열 진입 및 실시간 순번 확인** — 예매 오픈 시 자동 적용
- 좌석 선택·예매, Toss Payments 결제 연동
- 마이페이지(예매 내역 조회)
- 관리자 기능(공연/회차/좌석/사용자/메뉴 관리, 예매 활동 로그)

## 고도화 — 프로덕션 수준으로

2차 프로젝트에서 기본 예매 플로우와 대기열을 완성한 뒤, 최종 프로젝트에서는 기능 추가 대신 **인프라를 프로덕션 수준으로 끌어올리는 것**에 집중했습니다.

- **CI/CD 신뢰성** — "빌드 성공"과 "배포 후 실제 동작"을 구분하고 둘 다 보장
- **인프라의 코드화(IaC)** — 전 구성을 Terraform으로 관리, destroy 순서/시크릿 유실 같은 장애 상황도 복구 가능하게 설계
- **시크릿 관리 자동화** — AWS Secrets Manager + External Secrets Operator로 자동 동기화
- **관측성** — Prometheus/Grafana로 대기열·서비스 상태 실시간 모니터링
- **오토스케일링 검증** — KEDA(파드)·Karpenter(노드)·오버프로비저닝을 조합해 실제 수천 명 규모 부하테스트로 반복 검증

## 기술 스택

**Backend** Spring Boot · MyBatis · Redis(대기열/세션) · MySQL
**Frontend** Next.js
**Infra** AWS EKS · Terraform · Karpenter · KEDA · ArgoCD(GitOps) · External Secrets Operator
**모니터링** Prometheus · Grafana · Amazon Managed Prometheus

## 저장소

| 저장소 | 내용 |
|---|---|
| [`backend`](https://github.com/qKet/backend) | Spring Boot API 서버 |
| [`frontend`](https://github.com/qKet/frontend) | Next.js 프론트엔드 |
| [`Infra`](https://github.com/qKet/Infra) | Terraform 인프라 코드 |
| [`CD`](https://github.com/qKet/CD) | ArgoCD GitOps 매니페스트(Helm) |
| [`docs`](https://github.com/qKet/docs) | 제출용 문서, 화면/아키텍처 자료 |
| [`CLAUDE_LLM_WIKI`](https://github.com/qKet/CLAUDE_LLM_WIKI) | 컨벤션·아키텍처·설계 결정(ADR)·트러블슈팅 위키 |

## 팀

이채영 · 강진호 · 나윤준 · 문준혁 · 정우진
