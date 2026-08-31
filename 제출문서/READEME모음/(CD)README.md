# CD

Qket의 GitOps 매니페스트 저장소. ArgoCD가 이 레포의 `helm/` 경로를 보고 `qket-release`/`qket-prod` 네임스페이스에 배포합니다.

```
helm/
├── Chart.yaml
├── values-release.yaml   # release 기준값 (base)
├── values-prod.yaml      # prod가 values-release.yaml 위에 덮어쓰는 값 (레이어링)
└── templates/
    ├── backend-deployment.yaml
    ├── backend-scaledobject.yaml    # KEDA — backend replica 오토스케일링
    ├── frontend-deployment.yaml
    ├── frontend-scaledobject.yaml   # KEDA — frontend replica 오토스케일링
    ├── namespace.yaml
    └── networkpolicy.yaml
```

## 배포 흐름

1. `backend`/`frontend` 레포에서 CI가 이미지를 빌드해 ECR에 push하고, 이 레포의 `values-*.yaml`에 이미지 태그를 write-back
2. ArgoCD Application(`qket-cd-release`/`qket-cd`, `Infra/modules/addons/argocd`가 등록)이 이 레포의 변경을 감지
3. 배포는 **수동 Sync** — automated(prune/selfHeal)는 켜져 있지 않음. ArgoCD UI에서 직접 Sync를 눌러야 실제로 반영됨

## 값을 고칠 때

- release/prod 공통으로 바꿀 값은 `values-release.yaml`에 — prod가 명시적으로 덮어쓰지 않는 키는 release 값을 그대로 물려받습니다
- prod만 다르게 가져가야 하는 값(replica 수, DB 커넥션 풀 사이즈 등)은 `values-prod.yaml`에 명시
- `backend.dbPoolSize × backend.autoscaling.maxReplicas`가 RDS `max_connections`를 넘지 않는지 항상 확인할 것

## 더 자세한 내용

설계 배경, 트러블슈팅, ADR은 `CLAUDE_LLM_WIKI` 레포를 참고하세요.
