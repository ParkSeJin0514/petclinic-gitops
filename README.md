# 🚀 PetClinic GitOps Repository

ArgoCD가 관리하는 Kubernetes 배포 설정 저장소입니다.

---

## 📁 디렉토리 구조

```
petclinic-gitops/
├── kustomization.yaml              # 이미지 태그 관리 (Jenkins가 자동 업데이트)
├── README.md
└── manifests/
    ├── 00-namespace.yaml           # petclinic 네임스페이스
    ├── 01-config-server.yaml       # Spring Cloud Config Server
    ├── 02-discovery-server.yaml    # Eureka Discovery Server
    ├── 03-customers-service.yaml   # 고객 서비스
    ├── 04-visits-service.yaml      # 방문 서비스
    ├── 05-vets-service.yaml        # 수의사 서비스
    ├── 06-api-gateway.yaml         # API Gateway
    ├── 07-admin-server.yaml        # Spring Boot Admin
    ├── 08-cluster-secret-store.yaml # External Secrets - AWS 연동
    ├── 09-external-secret.yaml     # External Secrets - DB Secret 동기화
    ├── 10-ingress.yaml             # ALB Ingress
    ├── 11-monitoring.yaml          # Prometheus ServiceMonitor
    ├── 12-monitoring-cluster-values.yaml
    └── 13-monitoring-cluster.yaml  # Prometheus/Grafana Stack
```

---

## 🔄 CI/CD 파이프라인 흐름

```
┌─────────────────────────────────────────────────────────────────┐
│                         CI (Jenkins)                             │
├─────────────────────────────────────────────────────────────────┤
│  1. 개발자가 spring-petclinic Repo에 코드 Push                   │
│  2. GitHub Webhook → Jenkins 트리거                              │
│  3. Jenkins: Maven Build → Docker Build → ECR Push              │
│  4. Jenkins: 이 Repo의 kustomization.yaml 이미지 태그 업데이트    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                         CD (ArgoCD)                              │
├─────────────────────────────────────────────────────────────────┤
│  5. ArgoCD: 1분마다 Polling → 변경 감지                          │
│  6. ArgoCD: EKS에 자동 배포 (Sync)                               │
│  7. External Secrets: AWS Secrets Manager에서 DB 정보 동기화     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔐 External Secrets 구성

AWS Secrets Manager와 연동하여 DB 자격 증명을 자동으로 Kubernetes Secret으로 동기화합니다.

### 동기화 흐름

```
AWS Secrets Manager          External Secrets Operator         Kubernetes
┌─────────────────┐         ┌─────────────────────────┐       ┌──────────────┐
│ petclinic-kr/db │────────▶│   ClusterSecretStore   │──────▶│ petclinic-   │
│                 │  IRSA   │   ExternalSecret       │ Sync  │ db-secret    │
│ - DB_URL        │         │                        │       │              │
│ - DB_USERNAME   │         │   refreshInterval: 1h  │       │ (자동 생성)   │
│ - DB_PASSWORD   │         └─────────────────────────┘       └──────────────┘
└─────────────────┘
```

### Sync Wave 순서

| 순서 | 파일 | Sync Wave | 설명 |
|------|------|-----------|------|
| 1 | `08-cluster-secret-store.yaml` | -1 | AWS Secrets Manager 연결 설정 |
| 2 | `09-external-secret.yaml` | 0 | DB Secret 동기화 설정 |
| 3 | 기타 리소스 | 0 | Deployments, Services 등 |

### 상태 확인 명령어

```bash
# ClusterSecretStore 상태
kubectl get clustersecretstore

# ExternalSecret 상태
kubectl get externalsecret -n petclinic

# 생성된 Secret 확인
kubectl get secret petclinic-db-secret -n petclinic -o yaml
```

---

## 📦 마이크로서비스 구성

| 서비스 | 포트 | 설명 |
|--------|------|------|
| config-server | 8888 | Spring Cloud Config Server |
| discovery-server | 8761 | Eureka Service Discovery |
| customers-service | 8081 | 고객 정보 관리 |
| visits-service | 8082 | 방문 기록 관리 |
| vets-service | 8083 | 수의사 정보 관리 |
| api-gateway | 8080 | API Gateway (외부 진입점) |
| admin-server | 9090 | Spring Boot Admin |

---

## 🚀 사용법

### 매니페스트 확인 (Dry-run)

```bash
kubectl kustomize .
```

### 수동 배포 (테스트용)

```bash
kubectl apply -k .
```

### 특정 서비스만 배포

```bash
kubectl apply -f manifests/06-api-gateway.yaml
```

### ArgoCD로 Sync

```bash
argocd app sync petclinic
```

### ArgoCD edit
```
kubectl edit configmap argocd-cm -n argocd
```

---

## 🏷️ 이미지 태그 업데이트

Jenkins가 자동으로 업데이트하지만, 수동으로 하려면:

### 방법 1: kustomize CLI

```bash
kustomize edit set image \
  springcommunity/spring-petclinic-api-gateway=946775837287.dkr.ecr.ap-northeast-2.amazonaws.com/petclinic-msa/petclinic-api-gateway:2
```

### 방법 2: 직접 수정

`kustomization.yaml`의 `images` 섹션:

```yaml
images:
  - name: springcommunity/spring-petclinic-api-gateway
    newName: 946775837287.dkr.ecr.ap-northeast-2.amazonaws.com/petclinic-msa/petclinic-api-gateway
    newTag: "2"  # ← 이 부분 변경
```

---

## ⏪ 롤백

### Git Revert (권장)

```bash
git revert HEAD
git push
```

ArgoCD가 자동으로 이전 버전으로 배포합니다.

### ArgoCD CLI

```bash
# 히스토리 확인
argocd app history petclinic

# 특정 버전으로 롤백
argocd app rollback petclinic <REVISION>
```

---

## 📊 상태 확인 명령어

```bash
# Pod 상태
kubectl get pods -n petclinic

# Service 상태
kubectl get svc -n petclinic

# Ingress (ALB DNS 확인)
kubectl get ingress -n petclinic

# ArgoCD Application 상태
kubectl get applications -n argocd

# External Secrets 상태
kubectl get clustersecretstore
kubectl get externalsecret -n petclinic
```

---

## ⚠️ 사전 요구사항

이 GitOps Repo가 정상 동작하려면 다음이 필요합니다:

| 구성요소 | 설치 위치 | 설명 |
|----------|----------|------|
| AWS Load Balancer Controller | EKS | Ingress ALB 생성 |
| External Secrets Operator | EKS | Secret 동기화 |
| ArgoCD | EKS | GitOps CD |
| AWS Secrets Manager | AWS | DB 자격 증명 저장 |
| IRSA Role | AWS/EKS | External Secrets 권한 |

> 💡 위 구성요소는 Terraform (`infra-terraform-jenkins`)으로 자동 설치됩니다.

---

## 🔗 관련 Repository

| Repository | 설명 |
|------------|------|
| **spring-petclinic** | 소스코드 + Jenkinsfile (CI) |
| **infra-terraform-jenkins** | Terraform 인프라 코드 (IaC) |
| **petclinic-gitops** | K8s 매니페스트 (CD) ← 현재 Repo |

---

## 📝 주의사항

1. **External Secrets**: `remoteRef.key`가 Terraform의 `project_name`과 일치해야 함 (현재: `petclinic-kr/db`)

2. **이미지 태그**: Jenkins가 자동 업데이트하므로 수동 변경 시 충돌 주의

3. **Sync Wave**: ClusterSecretStore(-1) → ExternalSecret(0) 순서로 생성됨

4. **모니터링**: 11~13번 파일은 Prometheus/Grafana Stack 설치 시 필요
