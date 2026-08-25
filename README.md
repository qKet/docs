# Q-Ket 문서

공연 예매 시 발생하는 동시접속 트래픽 폭주를, 대기열 시스템과 프로덕션 수준의 인프라로 안정적으로 처리하는 공연 예매 플랫폼 — 제출/발표용 문서를 모아두는 저장소입니다.

**팀**: 강진호, 나윤준, 문준혁, 이채영, 정우진

## 폴더 구조

| 폴더 | 내용 |
|---|---|
| [`아키텍처/`](./아키텍처) | 시스템 구성도, AWS 인프라, EKS 클러스터 내부 동작, GitOps CI/CD 흐름, ERD, 사용자·서비스 흐름도, 화면 정의서 |
| [`화면/`](./화면) | 실제 화면 캡처 — 로그인/회원가입, 공연 목록·상세·좌석선택, 대기열, 결제, 마이페이지, 관리자(공연/카테고리/메뉴/사용자/프로그램 관리, 예매 활동 로그) |
| [`문서/`](./문서) | 제출용 요구사항정의서·API명세서 (엑셀) |
| [`cluadeDocs/`](./cluadeDocs) | 팀 자체 기술 문서 — 기획서, 인프라 아키텍처, CI/CD 파이프라인, ALB/HTTPS 트러블슈팅 등 |
| `API명세서.png`, `API명세서2.png`, `요구사항정의서.png` | 루트에 남아있는 스냅샷 — 최신 버전은 `문서/`(엑셀)와 `아키텍처/`(화면 정의서) 쪽을 우선 참고 |

## 관련 저장소

- [`qKet/backend`](https://github.com/qKet/backend) — Spring Boot API 서버
- [`qKet/frontend`](https://github.com/qKet/frontend) — Next.js 프론트엔드
- [`qKet/Infra`](https://github.com/qKet/Infra) — Terraform 인프라(AWS/EKS)
- [`qKet/CD`](https://github.com/qKet/CD) — ArgoCD GitOps 매니페스트(Helm)
- [`qKet/CLAUDE_LLM_WIKI`](https://github.com/qKet/CLAUDE_LLM_WIKI) — 컨벤션·아키텍처·설계 결정(ADR)·트러블슈팅 위키

## 기술 스택

Spring Boot · Next.js · MySQL · Redis · AWS EKS · Terraform · ArgoCD · KEDA · Karpenter
