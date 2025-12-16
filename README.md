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
│   │   └── external-secret.yaml    # petclinic-kr/db
│   │
│   └── gcp/                        # GCP 환경
│       ├── kustomization.yaml      # Artifact Registry 이미지
│       ├── cluster-secret-store.yaml # GCP Secret Manager
│       ├── external-secret.yaml    # petclinic-dr-db-credentials
│       └── ingress-patch.yaml      # GKE Ingress 패치
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
