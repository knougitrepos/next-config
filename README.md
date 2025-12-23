# next-config

GitOps 매니페스트 저장소 구조입니다. 현재 이 폴더 자체가 `next-config` 저장소로 사용될 수 있게 구성되어 있습니다.

## 분리 방법 (권장)

```powershell
# 1. 별도 위치로 복사
mkdir c:\git\next-config
cp -r c:\git\next\next-config\* c:\git\next-config\

# 2. git 초기화 및 push
cd c:\git\next-config
git init
git add .
git commit -m "Initial GitOps setup"
git remote add origin https://github.com/{your-username}/next-config.git
git push -u origin main
```

## 폴더 구조

```
base/                    # 공통 K8s 리소스
├── kustomization.yaml
├── deployment.yaml
├── service.yaml
└── configmap.yaml
overlays/
├── dev/                 # 개발 환경
├── stage/               # 스테이징 환경
└── prod/                # 프로덕션 환경
```
