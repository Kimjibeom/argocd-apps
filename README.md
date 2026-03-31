# ArgoCD App of Apps Repository

이 저장소는 쿠버네티스(Kubernetes) 클러스터에서 동작 중인 주요 서비스들을 **ArgoCD App of Apps** 패턴을 통해 GitOps 방식으로 통합 관리하기 위해 구성되었습니다.

## 📂 폴더 구조

```text
.
├── apps/               # 각 애플리케이션들을 정의하는 ArgoCD Application 매니페스트 폴더
├── values/             # 각 애플리케이션(Helm Release)의 설정이 담긴 values.yaml 파일 폴더
└── root-app.yaml       # 위 apps/ 내의 모든 애플리케이션을 배포 및 관리하는 최상위 Root App 매니페스트
```

## 🛠️ 관리 중인 서비스 목록 (Apps)

본 저장소는 아래의 주요 서비스 스택들을 포함하여 관리합니다:
- **Monitoring & Logging**: `prometheus-stack`, `loki`, `alloy`
- **CI/CD & Source Control**: `gitlab`, `jenkins`, `argo-workflows`, `argocd`
- **Registry & Storage**: `harbor`, `rook-ceph`
- **Database & Queue**: `mariadb`, `kafka`, `mlflow-db` (PostgreSQL)
- **AI/ML & Vector DB**: `mlflow`, `qdrant`, `kuberay-operator`
- **Security & Networking**: `vault`, `cert-manager`, `keycloak`, `my-traefik`
- **Automation**: `n8n`

---

## 🚀 시작하기 (배포 방법)

이 저장소를 기반으로 클러스터 전체 서비스를 ArgoCD에 한 번에 연동하려면 최초 1회 아래 명령어를 서버 터미널에서 실행하세요.

```bash
# ArgoCD가 설치된 클러스터에서 Root App 매니페스트 배포
kubectl apply -f root-app.yaml
```

명령어 실행 후 ArgoCD UI에 접속하면 `root-app`이 생성되고, 이 앱이 하위에 있는 모든 서비스 앱들을 순차적으로 배포 및 동기화합니다.

---

## 💡 주요 구성 및 문제 해결 가이드

1. **Multi-Source Application 활용**
   ArgoCD의 멀티 소스 기능(v2.6+)을 사용하여 외부의 공식 Helm 차트 주소(`https://...`)에, 본 Git 저장소의 커스텀 `values.yaml` 파일을 참조하여 설정값을 덮어쓰도록 구성되었습니다. 

2. **용량이 큰 CRD 처리를 위한 Server-Side Apply**
   `prometheus-stack` 등 일부 차트는 CustomResourceDefinition(CRD) 파일 용량이 매우 커서 쿠버네티스의 `kubectl apply` 애너테이션 길이 제한(256KB)을 초과할 수 있습니다. 이를 해결하기 위해 ArgoCD 앱 설정에 `ServerSideApply=true` 동기화 옵션이 적용되어 있습니다.

3. **랜덤 패스워드 방지를 위한 시크릿 고정 (Harbor 등)**
   `harbor`와 같은 일부 서비스는 `values.yaml`에 비밀번호를 넣지 않을 경우 렌더링 시마다 새로운 난수 비밀번호를 생성합니다. 이는 ArgoCD 자동 동기화 시 매번 Config 불일치(OutOfSync) 및 파드 재시작을 유발합니다. 이를 방지하기 위해 실제 생성됐던 비밀번호들을 추출해 `values/harbor.yaml`에 명시적으로 하드코딩하여 안전성을 높였습니다.

---

## ➕ 새로운 앱 추가하기

새로운 서비스를 본 저장소 구조에 연동하려면 다음 절차를 따릅니다.
1. `values/<새_서비스명>.yaml` 파일에 배포할 Helm 차트 설정값을 작성합니다.
2. `apps/<새_서비스명>.yaml` 에 ArgoCD Application 매니페스트를 작성합니다. (기존 앱들의 매니페스트를 참고하여 `targetRevision`이나 `repoURL` 등을 알맞게 수정합니다.)
3. 작업 내용을 본 저장소에 **Commit & Push** 합니다.
4. ArgoCD가 변경사항을 자동으로 감지(`Sync`)하여 클러스터에 새로운 앱을 띄우게 됩니다.