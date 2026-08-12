# Qket 인프라 아키텍처 (2026-08-12 기준)

이 문서는 현재 실제로 떠있는(라이브) 인프라 구조를 정리한 것이다. Terraform root 구조(`01_infrastructure`/`02_k8s-addon`/`03_registry`/`04_data`)와 실제 AWS/K8s 배치가 어떻게 대응되는지 한눈에 보기 위한 용도 — 코드 자체의 이유/트레이드오프는 `CLAUDE_LLM_WIKI`(architecture/decisions/troubleshooting)에 더 자세히 있음.

> ⚠️ `04_data`는 지금 `release` workspace만 실제로 적용돼 있고, `prod`는 아직 apply 전이다(코드는 준비됨). 아래 다이어그램은 release 기준.
> ⚠️ `01_infrastructure`/`02_k8s-addon`은 비용 절감을 위해 매일 밤 destroy → 아침 재생성한다(`README.md`의 "매일 아침/저녁" 참고) — 그래서 이 다이어그램은 "낮 시간대 기준"이다.

## 전체 구조

```mermaid
flowchart TB
    subgraph internet["인터넷"]
        users["일반 사용자"]
        team["팀원(허용 IP만)"]
        gh["GitHub Actions<br/>(backend/frontend CI)"]
    end

    subgraph r53["Route53 — jun979.click"]
        dns_dev["dev.jun979.click"]
        dns_app["app.jun979.click"]
        dns_grafana["grafana.jun979.click"]
        dns_cd["cd.jun979.click"]
    end

    subgraph vpc["VPC 10.70.0.0/16 (ap-northeast-2)"]
        subgraph pub["Public Subnet x2"]
            alb_app["ALB: team5-qket-alb<br/>(공개, IP 제한 없음)"]
            alb_admin["ALB: team5-qket-admin-alb<br/>(비공개, IP 허용목록)"]
            nat["NAT Gateway x2"]
        end

        subgraph priv["Private Subnet x2"]
            bastion["Bastion EC2<br/>(SSM 전용, 공인 IP 없음)"]

            subgraph eks["EKS: team5-qket-cluster"]
                subgraph ns_addon["kube-system / argocd / monitoring / external-secrets"]
                    albc["ALB Controller"]
                    extdns["ExternalDNS"]
                    argocd["ArgoCD"]
                    eso["ESO"]
                    mon["Prometheus/Grafana/Alertmanager"]
                end
                subgraph ns_release["qket-release"]
                    fe_r["frontend"]
                    be_r["backend"]
                end
                subgraph ns_prod["qket-prod (미배포)"]
                    fe_p["frontend"]
                    be_p["backend"]
                end
            end

            rds[("RDS MySQL<br/>team5-qket-mysql-release")]
            redis[("ElastiCache Redis<br/>team5-qket-redis-release")]
        end
    end

    subgraph aws_managed["VPC 밖 AWS 관리형 서비스"]
        s3[("S3: team5-qket-posters-release")]
        cf["CloudFront"]
        ecr["ECR: team5/ecr/qket"]
        sm["Secrets Manager<br/>(connection / external-api / RDS 자동생성)"]
    end

    subgraph cd_repo["qKet/CD 레포"]
        cdvalues["helm/values.yaml<br/>(이미지 태그)"]
    end

    users -->|"HTTPS"| dns_app
    users -->|"HTTPS"| dns_dev
    team -->|"HTTPS + IP 검증"| dns_grafana
    team -->|"HTTPS + IP 검증"| dns_cd

    dns_dev --> alb_app
    dns_app --> alb_app
    dns_grafana --> alb_admin
    dns_cd --> alb_admin

    alb_app -->|"/api"| be_r
    alb_app -->|"/"| fe_r
    alb_admin --> mon
    alb_admin --> argocd

    albc -.->|"Ingress 감지→ALB 생성"| alb_app
    albc -.-> alb_admin
    extdns -.->|"Route53 레코드 자동 갱신"| r53

    be_r --> rds
    be_r --> redis
    be_r -->|"IRSA"| s3
    s3 --> cf
    cf --> fe_r

    eso -->|"동기화"| sm
    sm -.->|"DB_HOST/PW, 토스/OAuth 키"| be_r

    bastion -.->|"SSM 포트포워딩"| rds
    bastion -.->|"SSM 포트포워딩"| redis

    gh -->|"OIDC(장기키 없음)"| ecr
    gh -->|"이미지 push"| ecr
    gh -->|"GitHub App으로 write-back"| cdvalues
    argocd -->|"sync"| cdvalues
    argocd -.->|"배포"| ns_release
```

## CI/CD 플로우

```mermaid
sequenceDiagram
    participant Dev as 팀원
    participant GH as GitHub Actions<br/>(backend/frontend)
    participant ECR as ECR
    participant CD as qKet/CD 레포
    participant Argo as ArgoCD
    participant K8s as EKS(qket-release)

    Dev->>GH: release 브랜치 push
    GH->>GH: test(mysql/redis 서비스 컨테이너) + build
    GH->>ECR: OIDC 인증 → 이미지 push
    GH->>CD: GitHub App 토큰으로 values.yaml 이미지 태그 write-back
    Argo->>CD: 주기적으로 git 변경 감지
    Note over Argo: syncPolicy 수동 — 사람이 SYNC 버튼 눌러야 함
    Argo->>K8s: Deployment/Service 반영
```

## 레이어별 Terraform root 대응

| Root | 실제로 관리하는 것 | 매일 밤 destroy? |
|---|---|---|
| `01_infrastructure` | VPC, 서브넷, IGW, NAT, EKS(클러스터+노드그룹), bastion EC2 | ✅ (EKS/bastion/NAT만, VPC/서브넷은 유지) |
| `02_k8s-addon` | namespace, ArgoCD, ALB Controller, ExternalDNS, ESO 컨트롤러, 모니터링 스택, Ingress(app/admin) | ✅ |
| `03_registry` | ECR, GitHub Actions OIDC role | ❌ (절대 안 지움) |
| `04_data` | RDS, Redis, S3+CloudFront, Secrets Manager(connection/external-api), IRSA | ❌ (AWS 리소스는 유지, K8s 오브젝트만 매일 아침 재생성) |

## 관련 문서
- `CLAUDE_LLM_WIKI/wiki/troubleshooting/eks-destroy-layer-separation.md` — 이렇게 나눈 이유
- `CLAUDE_LLM_WIKI/wiki/decisions/2026-08-11-monitoring-stack-design.md` — 모니터링 설계
- `Infra/README.md` — 매일 아침/저녁 명령어
