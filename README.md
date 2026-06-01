# team1-config

> Team1 수강신청 시스템의 Kubernetes 매니페스트 및 GitOps 배포 구성 저장소

## 📌 프로젝트 소개

본 저장소는 Amazon EKS 환경에서 실행되는 수강신청 시스템의 Kubernetes 매니페스트와 GitOps 배포 구성을 관리합니다.

ArgoCD는 본 저장소를 **Single Source of Truth**로 사용하여 EKS 클러스터와 지속적으로 동기화합니다.
애플리케이션 배포에 필요한 Deployment, Service, HPA, Secret, ArgoCD Application 설정을 모두 이곳에서 관리합니다.

| 구분 | 저장소 | 역할 |
| --- | --- | --- |
| App | [team1-app](https://github.com/CLD-05/team1-app) | 애플리케이션 소스코드, CI 파이프라인 |
| Config | [team1-config](https://github.com/CLD-05/team1-config) | Kubernetes 매니페스트, GitOps 배포 구성 |
| Infra | [team1-infra](https://github.com/CLD-05/team1-infra) | Terraform IaC, AWS 인프라 |

<br>

## 🔄 GitOps 구조

본 프로젝트는 소스코드 저장소와 배포 설정 저장소를 분리한 GitOps 구조를 채택합니다.

```text
App Repository (team1-app)
        │
        │  develop 브랜치 머지 시
        │  GitHub Actions 자동 실행
        ▼
GitHub Actions
        │
        │  Docker 빌드 + ECR Push
        │  kustomization.yaml 이미지 태그 자동 갱신
        ▼
Amazon ECR
        │
        │  course-service:{github.sha}
        ▼
Config Repository (team1-config) ◀── Single Source of Truth
        │
        │  변경 감지 (polling / webhook)
        ▼
ArgoCD
        │
        │  manual sync
        │  롤링 업데이트
        ▼
Amazon EKS (team1-dev 네임스페이스)
```

> 소스코드 변경과 배포 설정 변경을 분리함으로써 배포 이력 추적과 롤백이 용이합니다.

<br>

## 📁 디렉토리 구조

```text
team1-config
│
├── apps
│   └── team1-app
│       ├── base                        # 공통 매니페스트 (환경 무관)
│       │   ├── deployment.yaml         # Pod 생성 및 롤링 업데이트
│       │   ├── service.yaml            # 내부 네트워크 접근점
│       │   ├── httproute.yaml          # Gateway API 라우팅 규칙
│       │   ├── keda-scaled.yaml        # KEDA 기반 오토스케일링
│       │   └── kustomization.yaml      # base 리소스 묶음
│       │
│       └── overlays                    # 환경별 설정 오버레이
│           ├── dev                     # 개발 환경
│           │   ├── external-secret.yaml    # AWS Secrets Manager 연동
│           │   └── kustomization.yaml      # 이미지 태그 갱신 대상
│           └── prod                    # 운영 환경
│               ├── external-secret.yaml
│               └── kustomization.yaml
│
├── argocd                              # ArgoCD 구성
│   ├── project.yaml                    # 배포 권한 범위 정의
│   ├── application-dev.yaml            # dev 환경 배포 설정
│   ├── application-prod.yaml           # prod 환경 배포 설정
│   └── application-monitoring.yaml     # 모니터링 스택 배포 설정
│
└── infra                               # 클러스터 수준 인프라 리소스
    ├── namespace.yaml                  # 네임스페이스 정의
    ├── gateway-dev.yaml                # dev Gateway 설정
    ├── gateway-prod.yaml               # prod Gateway 설정
    └── prometheus                      # Prometheus 스택
        ├── namespace.yaml
        ├── kustomization.yaml
        └── servicemonitor.yaml
```

<br>

## ⚙️ Kubernetes 매니페스트 구성

### Deployment (`base/deployment.yaml`)
애플리케이션 Pod를 생성하고 관리하는 리소스입니다.

| 항목 | 값 |
| --- | --- |
| 기본 Replica | 2 |
| 컨테이너 포트 | 8080 (HTTP), 8081 (Actuator) |
| CPU Request/Limit | 250m / 500m |
| Memory Request/Limit | 512Mi / 1Gi |
| Health Check | `/actuator/health` (port 8081) |
| 프로파일 | `SPRING_PROFILES_ACTIVE=dev` |

DB 접속 정보 및 JWT Secret은 `team1-secret` (ExternalSecret)에서 주입합니다.

---

### Service (`base/service.yaml`)
Pod 집합에 대한 안정적인 네트워크 엔드포인트를 제공하는 리소스입니다.
Gateway API(HTTPRoute)가 Service를 대상으로 트래픽을 전달합니다.

| 항목 | 값 |
| --- | --- |
| 타입 | ClusterIP (클러스터 내부 통신) |
| 포트 | 80 → 8080 |

---

### HTTPRoute (`base/httproute.yaml`)
Gateway API를 통해 외부 요청을 애플리케이션 Service로 전달합니다.
기존 Ingress보다 확장성과 역할 분리가 뛰어난 Kubernetes 차세대 네트워킹 표준을 사용합니다.

| 항목 | 값 |
| --- | --- |
| Gateway | team1-gateway |
| 경로 | `/` (PathPrefix) |
| 대상 Service | team1-app:80 |

---

### KEDA ScaledObject (`base/keda-scaled.yaml`)
CPU 사용률 기반 오토스케일링과 시간 기반 Scheduled Scaling을 함께 적용하여 수강신청 시간대의 급격한 트래픽 증가에 대응합니다.

| 항목 | 값 |
| --- | --- |
| 최소 Pod | 2 |
| 최대 Pod | 30 |
| CPU 임계값 | 50% 초과 시 Scale-Out |
| CoolDown | 300초 |

수강신청 오픈 시간대에 맞춰 사전 Scale-Out 스케줄이 설정되어 있습니다.

| 시간대 | 스케줄 | 목표 Pod |
| --- | --- | --- |
| 오전 수강신청 | 09:30 ~ 10:40 (평일) | 15 |
| 오후 1차 | 13:30 ~ 14:40 (평일) | 15 |
| 오후 2차 | 15:30 ~ 16:40 (평일) | 15 |

---

### Kustomize 구성

**base** — 공통 리소스 묶음
```yaml
resources:
  - deployment.yaml
  - service.yaml
  - keda-scaled.yaml
```

**overlays/dev** — dev 환경 오버레이
- 네임스페이스: `team1-dev`
- 이미지 태그: GitHub Actions가 `github.sha`로 자동 갱신
- ExternalSecret으로 AWS SSM Parameter Store에서 시크릿 주입

---

### ExternalSecret (`overlays/dev/external-secret.yaml`)
AWS SSM Parameter Store에서 민감 정보를 자동으로 가져와 Kubernetes Secret으로 생성합니다.

| Secret Key | SSM 경로 |
| --- | --- |
| db-host | `/team1/eks-dev/db-host` |
| db-username | `/team1/eks-dev/db-username` |
| db-password | `/team1/eks-dev/db-password` |
| db-name | `/team1/eks-dev/db-name` |
| jwt-secret | `/team1/eks-dev/jwt-secret` |

<br>

## 🔧 ArgoCD 구성

### Project (`argocd/project.yaml`)
ArgoCD에서 배포 권한 범위를 정의하는 리소스입니다.

| 항목 | 값 |
| --- | --- |
| 프로젝트명 | team1-project |
| 소스 저장소 | https://github.com/CLD-05/team1-config.git |
| 배포 가능 네임스페이스 | team1-dev, team1-prod, monitoring |
| 클러스터 리소스 | Namespace 생성 허용 |

---

### Application (`argocd/application-dev.yaml`)
ArgoCD가 실제 배포를 수행하기 위한 설정 파일입니다.

| 항목 | 값 |
| --- | --- |
| 애플리케이션명 | team1-app-dev |
| 소스 경로 | `apps/team1-app/overlays/dev` |
| 대상 브랜치 | main |
| 배포 네임스페이스 | team1-dev |
| Sync 방식 | automated (prune + selfHeal) |
| 네임스페이스 자동 생성 | 활성화 |

> `prune: true` — Git에서 삭제된 리소스는 클러스터에서도 자동 삭제
> `selfHeal: true` — 클러스터 상태가 Git과 달라지면 자동으로 원복

<br>

## 🚀 배포 흐름

### 전체 흐름

```text
App Repository (team1-app)
        │
        │  develop 브랜치 머지
        ▼
GitHub Actions
        │
        ├── OIDC 인증 (AWS IAM)
        ├── Docker 멀티스테이지 빌드
        ├── ECR Push (course-service:{github.sha})
        └── kustomization.yaml 이미지 태그 자동 갱신
        │
        ▼
Config Repository (team1-config) ← 현재 저장소
        │
        │  변경 감지
        ▼
ArgoCD (automated sync)
        │
        ├── prune: Git 삭제 리소스 자동 제거
        └── selfHeal: 클러스터 상태 자동 원복
        │
        ▼
Amazon EKS (team1-dev 네임스페이스)
        │
        ├── 롤링 업데이트 (무중단)
        ├── Readiness Probe 통과 후 트래픽 수신
        └── 신규 Pod 정상화 → 구 Pod 종료
```

### 롤백 방법

장애 발생 시 이전 버전으로 즉각 롤백할 수 있습니다.

**방법 1 — Config 레포에서 태그 변경**
```yaml
# overlays/dev/kustomization.yaml
images:
  - name: 495599735720.dkr.ecr.ap-northeast-2.amazonaws.com/team1-app
    newTag: <이전 정상 커밋 SHA>  # ← 이전 sha로 변경 후 push
```

**방법 2 — ArgoCD 대시보드에서 이전 버전으로 Sync**
ArgoCD UI → team1-app-dev → History → 이전 버전 선택 → Rollback

<br>

## 🌿 환경 분리 전략

본 프로젝트는 Kustomize overlay 방식으로 dev/prod 환경을 분리합니다.

### 구조

```text
apps/team1-app/
├── base/                   # 공통 설정 (환경 무관)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── httproute.yaml
│   └── keda-scaled.yaml
│
└── overlays/
    ├── dev/                # 개발 환경
    │   ├── kustomization.yaml   (team1-dev 네임스페이스, dev 이미지 태그)
    │   └── external-secret.yaml (dev SSM 시크릿 경로)
    └── prod/               # 운영 환경
        ├── kustomization.yaml   (team1-prod 네임스페이스, prod 이미지 태그)
        └── external-secret.yaml (prod SSM 시크릿 경로)
```

### 환경별 차이점

| 항목 | dev | prod |
| --- | --- | --- |
| 네임스페이스 | team1-dev | team1-prod |
| 이미지 태그 | github.sha (자동 갱신) | github.sha (수동 갱신) |
| SSM 경로 | `/team1/eks-dev/` | `/team1/eks-prod/` |
| ArgoCD Application | application-dev.yaml | application-prod.yaml |
| Gateway | gateway-dev.yaml | gateway-prod.yaml |

### base vs overlay 역할

| 구분 | 역할 |
| --- | --- |
| base | 환경 무관 공통 리소스 정의 |
| overlay | 환경별 네임스페이스, 이미지 태그, 시크릿 경로 오버라이드 |

> base를 직접 수정하지 않고 overlay에서만 환경별 차이를 관리하여
> 코드 중복 없이 다중 환경을 운영합니다.

<br>

## 📋 팀 운영 규칙

### Config 레포 수정 원칙

| 상황 | 방법 |
| --- | --- |
| 이미지 태그 갱신 | GitHub Actions 자동 처리 (직접 수정 금지) |
| 매니페스트 변경 | PR → 팀원 1인 이상 approve → merge |
| 긴급 롤백 | ArgoCD 대시보드 또는 kustomization.yaml 태그 직접 수정 |

---

### 브랜치 전략

```text
main  ← 배포 기준 브랜치 (ArgoCD가 추적)
 └── feature/이름/기능명
```

- 모든 변경은 `feature/*` 브랜치에서 작업 후 `main`으로 PR
- approve 없이 main 직접 push 금지
- GitHub Actions의 자동 태그 갱신은 main에 직접 push (예외)

---

### 주의사항

> ⚠️ `overlays/dev/kustomization.yaml`의 이미지 태그는
> GitHub Actions가 자동으로 갱신합니다. 직접 수정하면
> 다음 배포 시 덮어씌워집니다.

> ⚠️ ArgoCD `selfHeal: true` 설정으로 인해 클러스터에서
> 직접 수정한 리소스는 자동으로 Git 상태로 원복됩니다.
> 영구 변경은 반드시 Config 레포를 통해 진행하세요.

---
