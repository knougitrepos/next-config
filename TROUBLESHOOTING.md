# ArgoCD 배포 문제 해결 가이드

이 문서는 이 GitOps 레포지토리를 사용하여 ArgoCD로 배포할 때 발생할 수 있는 일반적인 문제와 해결 방법을 설명합니다.

---

## 🔴 Critical: Kustomize Images 설정 충돌

### 문제
`base/kustomization.yaml`과 `overlays/{env}/kustomization.yaml` **모두에 `images` 섹션이 있으면** ArgoCD가 overlay의 이미지 태그를 적용하지 못합니다.

### 증상
- ✅ 로컬: `kubectl kustomize overlays/prod` → 올바른 이미지
- ❌ ArgoCD 적용: Deployment가 잘못된 이미지 사용 (latest 또는 base의 태그)

### 해결
**base에서 images 섹션 완전히 제거하고, overlay에서만 관리:**

```yaml
# ❌ base/kustomization.yaml - 이렇게 하지 마세요!
images:
  - name: next-app
    newName: ghcr.io/owner/app
    newTag: v1.0.0

# ✅ base/kustomization.yaml - 올바른 설정
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
# images 섹션 없음!

# ✅ overlays/prod/kustomization.yaml - 여기서만 관리
images:
  - name: next-app
    newName: ghcr.io/owner/app
    newTag: prod-v1.0.0
```

---

## 일반적인 문제들

### 1. 레포지토리 접근 권한

**문제**: ArgoCD가 `authentication error` 발생

**원인**: Private 레포지토리는 ArgoCD가 접근할 수 없음

**해결**:
- Option 1: 레포지토리를 Public으로 변경 (권장)
- Option 2: ArgoCD에 Repository Credentials 추가

### 2. 브랜치 이름 불일치

**문제**: `Unable to resolve 'main' to a commit SHA`

**원인**: ArgoCD Application의 `targetRevision`과 실제 브랜치 이름이 다름

**해결**:
```yaml
# ArgoCD Application manifest
spec:
  source:
    targetRevision: master  # 실제 브랜치 이름과 일치해야 함
```

### 3. 이미지 Pull 실패 (ImagePullBackOff)

**문제**: Pod가 `ImagePullBackOff` 또는 `ErrImagePull` 상태

**가능한 원인**:
1. **이미지가 존재하지 않음**: GitHub Actions가 해당 태그로 이미지를 push하지 않음
2. **이미지가 Private**: GHCR 패키지가 Private으로 설정됨
3. **태그 불일치**: overlay의 `newTag`와 실제 push된 태그가 다름

**해결**:
1. GitHub Packages에서 이미지 태그 확인
2. GHCR 패키지를 Public으로 변경
3. overlay의 `newTag`를 실제 태그로 업데이트

### 4. ArgoCD 동기화 안 됨

**문제**: GitOps 저장소를 업데이트했지만 ArgoCD가 반영하지 않음

**해결**:
```bash
# Hard refresh 시도
kubectl patch application {app-name} -n argocd \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}' \
  --type=merge

# 또는 ArgoCD UI에서 "Refresh" 클릭
```

---

## 디버깅 방법

### 1. Kustomize 빌드 테스트
```bash
# 로컬에서 overlay 빌드 결과 확인
kubectl kustomize overlays/prod | grep "image:"
```

### 2. ArgoCD가 적용한 실제 manifest 확인
```bash
# ArgoCD가 배포한 Deployment 확인
kubectl get deployment {deployment-name} -n {namespace} -o yaml | grep "image:"
```

### 3. ArgoCD Application 상태 확인
```bash
# Application 상태 및 에러 확인
kubectl describe application {app-name} -n argocd

# 상세 로그
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=100
```

---

## 이미지 태그 관리 규칙

### 태그 형식
GitHub Actions에서 생성하는 이미지 태그:
```
{branch}-{commit_sha}-{timestamp}
```
예: `master-328ac49-20251224001714`

### 업데이트 절차
1. GitHub Actions가 새 이미지 빌드 및 GHCR에 push
2. `overlays/{env}/kustomization.yaml`의 `newTag` 수동 업데이트
3. Git commit & push
4. ArgoCD 자동 동기화 (또는 수동 refresh)

---

## 참고 링크

- [Kustomize 공식 문서](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [ArgoCD 공식 문서](https://argo-cd.readthedocs.io/)
- Application 레포지토리: https://github.com/knougitrepos/next
