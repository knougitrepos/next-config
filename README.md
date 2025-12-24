# next-config

이 레포지토리는 [next](https://github.com/knougitrepos/next) 애플리케이션의 GitOps 배포 매니페스트를 관리합니다.

## 구조

```
next-config/
├── base/                    # 공통 기본 설정
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
└── overlays/                # 환경별 오버레이
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── namespace.yaml
    │   └── patch-deployment.yaml
    ├── stage/
    └── prod/
```

## 브랜치 전략

- **master**: 모든 환경의 manifest 저장
- ArgoCD Application별로 다른 overlay 경로 사용:
  - `overlays/dev` → dev 환경
  - `overlays/stage` → stage 환경
  - `overlays/prod` → prod 환경

## ArgoCD 배포

### Application 설정 예시

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: next-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/knougitrepos/next-config.git
    targetRevision: master
    path: overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: next-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 이미지 태그 업데이트

GitHub Actions가 새 이미지를 빌드하면 수동으로 각 overlay의 `kustomization.yaml`을 업데이트:

```yaml
# overlays/prod/kustomization.yaml
images:
  - name: next-app
    newName: ghcr.io/knougitrepos/next
    newTag: master-{commit_sha}-{timestamp}  # 여기를 업데이트
```

## 문제 해결

ArgoCD 배포 시 문제가 발생하면 [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)를 참고하세요.

**⚠️ Critical**: base와 overlay 모두에 `images` 섹션을 두지 마세요! 자세한 내용은 TROUBLESHOOTING.md 참조.

## 관련 링크

- Application 코드: https://github.com/knougitrepos/next
- CI/CD 워크플로우: https://github.com/knougitrepos/next/blob/master/.github/workflows/ci.yml
