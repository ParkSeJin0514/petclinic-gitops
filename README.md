# 🐾 PetClinic GitOps

ArgoCD 기반 PetClinic 애플리케이션 배포 매니페스트 (Kustomize + Multi-Cloud Overlay)

## 🏛️ 아키텍처

```
                    petclinic-dev (소스 코드)
                           │
                           │ Push
                           ▼
                    GitHub Actions CI
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           ▼               ▼               ▼
      Maven Build    Docker Build    GitOps Update
                           │
           ┌───────────────┴───────────────┐
           │                               │
           ▼                               ▼
    AWS ECR Push                 GCP Artifact Registry Push
           │                               │
           └───────────────┬───────────────┘
                           │
                           ▼
              petclinic-gitops (이 저장소)
                           │
           ┌───────────────┴───────────────┐
           │                               │
           ▼                               ▼
   overlays/aws/                   overlays/gcp/
   (EKS + ECR)                     (GKE + AR)
           │                               │
           ▼                               ▼
    ArgoCD (AWS)                   ArgoCD (GCP)
           │                               │
           ▼                               ▼
      EKS 배포                        GKE 배포
```

## 📁 디렉토리 구조

```
petclinic-gitops/
├── kustomization.yaml              # 루트 (기본: overlays/aws 참조)
├── base/                           # 공통 리소스
│   ├── kustomization.yaml
│   └── manifests/
│       ├── 00-namespace.yaml       # petclinic 네임스페이스
│       ├── 01-config-server.yaml   # Spring Cloud Config
│       ├── 02-discovery-server.yaml # Eureka
│       ├── 03-customers-service.yaml
│       ├── 04-visits-service.yaml
│       ├── 05-vets-service.yaml
│       ├── 06-api-gateway.yaml     # 외부 트래픽 진입점
│       ├── 07-admin-server.yaml    # Spring Boot Admin
│       ├── 10-ingress.yaml         # Ingress (base)
│       ├── 11-monitoring.yaml      # Prometheus + Grafana
│       ├── 12-monitoring-cluster-values.yaml
│       └── 13-monitoring-cluster.yaml
│
├── overlays/
│   ├── aws/                        # AWS 환경
│   │   ├── kustomization.yaml      # ECR 이미지 + AWS 태그
│   │   ├── cluster-secret-store.yaml # AWS Secrets Manager
│   │   ├── external-secret.yaml    # petclinic-kr/db
│   │   └── karpenter-node-selector-patch.yaml # Karpenter 노드 스케줄링
│   │
│   └── gcp/                        # GCP 환경
│       ├── kustomization.yaml      # Artifact Registry 이미지
│       ├── cluster-secret-store.yaml # GCP Secret Manager
│       ├── external-secret.yaml    # petclinic-dr-db-credentials
│       ├── ingress-patch.yaml      # GKE Ingress 패치
│       ├── backend-config.yaml     # GCP Health Check 설정
│       └── service-patch.yaml      # Service에 BackendConfig 연결
```

## ☁️ Multi-Cloud 지원

| 항목 | AWS (Primary) | GCP (DR) |
|------|---------------|----------|
| **Container Registry** | ECR | Artifact Registry |
| **Secrets** | AWS Secrets Manager | GCP Secret Manager |
| **Ingress** | ALB Controller | GKE Ingress (GCE) |
| **인증** | IRSA | Workload Identity |
| **ArgoCD Path** | `overlays/aws` | `overlays/gcp` |

## 🐳 이미지 레지스트리

### AWS ECR
```
946775837287.dkr.ecr.ap-northeast-2.amazonaws.com/petclinic-msa/petclinic-*
```

### GCP Artifact Registry
```
asia-northeast3-docker.pkg.dev/kdt2-final-project-t1/petclinic-msa/petclinic-*
```

## ⚙️ ArgoCD Application 설정

### AWS (EKS)
```yaml
spec:
  source:
    repoURL: https://github.com/ParkSeJin0514/petclinic-gitops.git
    path: overlays/aws        # AWS overlay 사용
    targetRevision: main
```

### GCP (GKE)
```yaml
spec:
  source:
    repoURL: https://github.com/ParkSeJin0514/petclinic-gitops.git
    path: overlays/gcp        # GCP overlay 사용
    targetRevision: main
```

## 🔄 CI/CD 파이프라인

`petclinic-dev`에서 Push 시 자동 실행:

1. **Maven Build** - 변경된 서비스만 빌드
2. **Docker Build** - 멀티 플랫폼 이미지 생성
3. **ECR Push** - AWS Container Registry
4. **Artifact Registry Push** - GCP Container Registry
5. **GitOps Update** - AWS/GCP overlay 모두 태그 업데이트
6. **ArgoCD Sync** - 양쪽 클러스터 자동 배포

## 🔐 External Secrets 설정

### AWS (overlays/aws/)
```yaml
# cluster-secret-store.yaml
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-northeast-2
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

### GCP (overlays/gcp/)
```yaml
# cluster-secret-store.yaml
spec:
  provider:
    gcpsm:
      projectID: kdt2-final-project-t1
      auth:
        workloadIdentity:
          clusterLocation: asia-northeast3
          clusterName: petclinic-dr-gke
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

## 🚀 수동 배포

```bash
# AWS 환경 배포
kubectl apply -k overlays/aws

# GCP 환경 배포
kubectl apply -k overlays/gcp

# 미리보기
kubectl kustomize overlays/aws
kubectl kustomize overlays/gcp
```

## 🏷️ 이미지 태그 수동 변경

```yaml
# overlays/aws/kustomization.yaml 또는 overlays/gcp/kustomization.yaml
images:
  - name: springcommunity/spring-petclinic-config-server
    newName: <registry>/petclinic-config-server
    newTag: "9"  # 태그 변경
```

## 🔧 GCP BackendConfig (Health Check)

GCP GCE Ingress는 기본적으로 `/` 경로로 Health Check를 수행합니다.
Grafana, Prometheus 등은 별도의 Health Check 경로가 필요하므로 **BackendConfig**를 사용합니다.

### BackendConfig 구성 (overlays/gcp/backend-config.yaml)

| 서비스 | Health Check Path | Port |
|--------|------------------|------|
| Grafana | `/api/health` | 3000 |
| Prometheus | `/-/healthy` | 9090 |
| API Gateway | `/actuator/health` | 8080 |

### Service 연결 (overlays/gcp/service-patch.yaml)

```yaml
metadata:
  annotations:
    cloud.google.com/backend-config: '{"default": "grafana-backend-config"}'
```

Service에 위 annotation을 추가하면 GCP가 BackendConfig의 Health Check 설정을 사용합니다.

### 확인 방법

```bash
# BackendConfig 확인
kubectl get backendconfig -n petclinic

# Service annotation 확인
kubectl get svc grafana-server -n petclinic -o jsonpath='{.metadata.annotations}'

# Ingress Backend 상태 확인
kubectl describe ingress grafana-ingress -n petclinic | grep -i backend
```

## 🚀 Karpenter 노드 스케줄링 (AWS)

AWS EKS 환경에서 PetClinic 워크로드가 Karpenter가 프로비저닝한 노드에만 스케줄링되도록 설정되어 있습니다.

### 동작 방식

```
Karpenter NodePool (general)
├── 노드 라벨: managed-by: karpenter
└── 노드 라벨: node-role: workload
                    ↓
PetClinic Deployments (patch 적용)
└── nodeSelector: managed-by: karpenter
                    ↓
Pod들이 Karpenter 노드에만 스케줄링됨
(Managed Node Group 제외)
```

### 적용 대상 (overlays/aws/karpenter-node-selector-patch.yaml)

| Deployment | 설명 |
|------------|------|
| config-server | Spring Cloud Config Server |
| discovery-server | Eureka Discovery Server |
| customers-service | 고객 정보 서비스 |
| visits-service | 방문 기록 서비스 |
| vets-service | 수의사 정보 서비스 |
| api-gateway | API Gateway |
| admin-server | Spring Boot Admin |
| prometheus-server | 메트릭 수집 |
| grafana-server | 대시보드 |

### Karpenter NodePool 설정 (platform-gitops)

```yaml
# NodePool에서 정의된 노드 라벨
template:
  metadata:
    labels:
      node-role: workload
      managed-by: karpenter
```

### 확인 방법

```bash
# Karpenter 노드 확인
kubectl get nodes -l managed-by=karpenter

# Pod가 Karpenter 노드에서 실행 중인지 확인
kubectl get pods -n petclinic -o wide

# 특정 Pod의 노드 정보 확인
kubectl get pod <pod-name> -n petclinic -o jsonpath='{.spec.nodeName}'
```

### 기존 Pod 마이그레이션

patch 적용 후 기존 Managed Node Group에서 실행 중인 Pod들을 Karpenter 노드로 이동시키려면:

```bash
# 모든 Deployment 재시작
kubectl rollout restart deployment -n petclinic --all

# 또는 개별 Deployment 재시작
kubectl rollout restart deployment/<deployment-name> -n petclinic
```

## 🔧 트러블슈팅

### GKE에서 ImagePullBackOff

**원인**: GKE 서비스 계정에 Artifact Registry 읽기 권한 없음

**해결**:
```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:SA@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"
```

### External Secret 실패

**확인**:
```bash
kubectl get clustersecretstore
kubectl get externalsecret -n petclinic
kubectl describe externalsecret petclinic-db-secret -n petclinic
```

## 🔗 관련 저장소

| 저장소 | 설명 |
|--------|------|
| **petclinic-dev** | 소스 코드 + CI/CD (GitHub Actions) |
| **platform-gitops-test** | 플랫폼 컴포넌트 (ArgoCD, External Secrets 등) |
| **platform-dev-test** | Terraform 인프라 (EKS, GKE, VPC 등) |
