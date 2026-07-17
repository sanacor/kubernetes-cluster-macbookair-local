# kubernetes-cluster-local

로컬 kind 클러스터 위에서 ArgoCD를 학습하기 위한 환경.

## 구성

- **kind** 클러스터 `argocd-lab` (단일 노드, 호스트 80/443 포트 매핑)
- **ingress-nginx** — helmfile로 관리
- **ArgoCD** — helmfile로 관리 (argo-helm 차트)

```
.
├── kind-config.yaml        # kind 클러스터 정의
├── helmfile.yaml           # ingress-nginx + argocd 릴리스 정의
├── values/
│   ├── ingress-nginx.yaml  # kind용 hostPort 설정
│   └── argocd.yaml         # 로컬 학습용 ArgoCD 설정
├── apps/
│   └── demo/               # GitOps로 배포되는 데모 앱 (nginx)
└── argocd/
    └── demo-app.yaml       # ArgoCD Application (이 저장소의 apps/demo를 감시)
```

## 처음부터 다시 만들기

```bash
kind create cluster --config kind-config.yaml
helmfile apply
```

## 접속

- UI: http://argocd.localhost (브라우저에서 바로 열림)
- 계정: `admin` / 초기 비밀번호는 아래 명령으로 확인

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

> 비밀번호를 바꾼 뒤에는 초기 시크릿을 삭제하는 것이 권장됨:
> `kubectl -n argocd delete secret argocd-initial-admin-secret`

## GitOps 흐름

이 저장소 자체가 ArgoCD의 소스 저장소다. `apps/demo/`의 매니페스트를 수정하고
main에 push하면 ArgoCD가 변경을 감지해 OutOfSync로 표시한다 (기본 폴링 주기 3분,
UI에서 REFRESH를 누르면 즉시).

```bash
kubectl apply -f argocd/demo-app.yaml   # Application 등록 (최초 1회)
```

- 데모 앱: http://demo.localhost
- sync 정책: **수동** — git 변경 후 UI의 SYNC 버튼(또는 `argocd app sync demo`)으로 반영

## 자주 쓰는 명령

```bash
helmfile diff                # 적용 전 변경사항 미리보기
helmfile apply               # 차트 설치/업그레이드
kubectl get pods -n argocd   # ArgoCD 파드 상태
kind delete cluster --name argocd-lab   # 클러스터 삭제
```

## 메모

- ArgoCD는 `server.insecure: true`로 HTTP 서빙 (로컬 학습용)
- dex(SSO), notifications는 메모리 절약을 위해 비활성화
- Docker Desktop 메모리 할당: 4GB
