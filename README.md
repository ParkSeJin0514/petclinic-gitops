# 🐾 PetClinic GitOps

ArgoCD 기반 PetClinic 애플리케이션 배포 매니페스트 (Kustomize + Multi-Cloud Overlay)

## 🏗️ 아키텍처

```
petclinic-dev (소스코드) → GitHub Actions CI → petclinic-gitops (이 저장소)
                                                     │
                     ┌───────────────────────────────┼───────────────────────────────┐
                     │                               │                               │
              overlays/aws                    overlays/gcp                     base/
              (EKS + ECR)                    (GKE + AR)                    (공통 리소스)
                     │                               │
              ArgoCD (AWS)                   ArgoCD (GCP)
                     │                               │
               EKS 배포                        GKE 배포
```

## 📁 디렉토리 구조

```
petclinic-gitops/
├── base/                           # 공통 리소스
│   ├── kustomization.yaml
│   └── manifests/
│       ├── 00-namespace.yaml       # petclinic 네임스페이스
│       ├── 01-config-server.yaml   # Spring Cloud Config
│       ├── 02-discovery-server.yaml # Eureka
│       ├── 03~05-*-service.yaml    # customers, visits, vets
│       ├── 06-api-gateway.yaml     # 외부 트래픽 진입점
│       ├── 07-admin-server.yaml    # Spring Boot Admin
│       ├── 08-hpa.yaml             # HPA
│       ├── 10-ingress.yaml         # Ingress (host: psj0514.site)
│       └── 11-app-monitoring.yaml  # PetClinic 앱 모니터링
│
├── overlays/
│   ├── aws/                        # AWS 환경 (ECR + IRSA)
│   │   └── cluster-monitoring-ingress.yaml  # kube-prometheus-stack Ingress (petclinic ns)
│   └── gcp/                        # GCP 환경 (AR + Workload Identity)
│       ├── cluster-monitoring-ingress.yaml       # kube-prometheus-stack Ingress (monitoring ns)
│       └── cluster-monitoring-backend-config.yaml # GKE Health Check 설정 (monitoring ns)
```

## ☁️ Multi-Cloud 지원

| 항목 | AWS (Primary) | GCP (DR) |
|------|---------------|----------|
| Container Registry | ECR | Artifact Registry |
| Secrets | AWS Secrets Manager | GCP Secret Manager |
| Ingress | ALB Controller | GKE Ingress (GCE) |
| 인증 | IRSA | Workload Identity |
| ArgoCD Path | `overlays/aws` | `overlays/gcp` |

## 🔧 Kustomize Overlay 패턴

Base에 정의된 공통 리소스를 각 클라우드 환경에 맞게 오버라이드합니다.

### 오버라이드 예시 (GCP)

```yaml
# overlays/gcp/kustomization.yaml
resources:
  - ../../base
  - cluster-secret-store.yaml        # GCP 전용 리소스 추가
patches:
  - path: petclinic-ingress-patch.yaml  # ALB → GKE Ingress 변환
  - target:                           # 불필요한 Ingress 삭제
      kind: Ingress
      name: grafana-ingress
    patch: |
      $patch: delete
images:
  - name: springcommunity/spring-petclinic-*
    newName: asia-northeast3-docker.pkg.dev/.../petclinic-*  # 이미지 교체
```

## 🌐 GCP Ingress 구성

GCP에서는 여러 Ingress를 통합하여 LB 비용을 절감합니다.

| Ingress | 용도 | 경로 |
|---------|------|------|
| `petclinic-ingress` | 메인 앱 | `/` → api-gateway, `/admin` → admin-server |
| `monitoring-ingress` | 앱 모니터링 | `/` → Grafana, `/prometheus` → Prometheus |
| `cluster-monitoring-ingress` | 클러스터 모니터링 | `/` → Grafana, `/prometheus` → Prometheus |

### 🏥 BackendConfig (Health Check)

GKE Ingress는 기본 `/` 경로로 Health Check를 수행하므로 BackendConfig로 별도 설정합니다.

| 서비스 | Health Check Path | Port |
|--------|------------------|------|
| Grafana (kube-prometheus-stack) | `/api/health` | 80 |
| Prometheus (kube-prometheus-stack) | `/prometheus/-/healthy` | 9090 |
| API Gateway | `/actuator/health` | 8080 |

## 📊 모니터링 구성

| 파일 | Namespace | 목적 |
|------|-----------|------|
| `11-app-monitoring.yaml` | petclinic | PetClinic 서비스 메트릭 수집 |
| `cluster-monitoring-ingress.yaml` (AWS) | petclinic | kube-prometheus-stack Ingress |
| `cluster-monitoring-ingress.yaml` (GCP) | monitoring | kube-prometheus-stack Ingress |
| `cluster-monitoring-backend-config.yaml` | monitoring | GKE Health Check 설정 (GCP only) |

> **참고**:
> - **AWS**: kube-prometheus-stack과 Ingress 모두 `petclinic` namespace에 설치
> - **GCP**: kube-prometheus-stack과 Ingress 모두 `monitoring` namespace에 설치

## ⚖️ HPA (Horizontal Pod Autoscaler)

| 서비스 | minReplicas | maxReplicas | CPU 임계값 |
|--------|-------------|-------------|------------|
| api-gateway | 2 | 4 | 70% |
| customers-service | 2 | 4 | 70% |
| visits-service | 2 | 4 | 70% |
| vets-service | 2 | 4 | 70% |

> **참고**: maxReplicas를 4로 제한하여 /24 서브넷 IP 고갈 방지

### ArgoCD ignoreDifferences 설정

HPA와 ArgoCD selfHeal 충돌 방지:

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

## 🚀 Karpenter 노드 스케줄링 (AWS)

AWS에서 PetClinic 워크로드가 Karpenter 노드에만 스케줄링되도록 설정:

```yaml
# overlays/aws/karpenter-node-selector-patch.yaml
nodeSelector:
  managed-by: karpenter
```

## 📦 수동 배포

```bash
# AWS/GCP 환경 배포
kubectl apply -k overlays/aws
kubectl apply -k overlays/gcp

# 미리보기
kubectl kustomize overlays/aws
kubectl kustomize overlays/gcp
```

## 🔍 트러블슈팅

### HPA 메트릭이 `<unknown>` 표시
- **원인**: Metrics Server 미설치 (EKS 기본 미설치)
- **확인**: `kubectl get pods -n kube-system | grep metrics`

### GKE ImagePullBackOff
- **원인**: GKE 서비스 계정에 Artifact Registry 읽기 권한 없음
- **해결**: `gcloud projects add-iam-policy-binding` 으로 권한 추가

### External Secret 실패
- **확인**: `kubectl describe externalsecret petclinic-db-secret -n petclinic`

## 🔗 관련 저장소

| 저장소 | 설명 |
|--------|------|
| **petclinic-dev** | 소스 코드 + CI/CD |
| **platform-gitops-last** | 플랫폼 컴포넌트 (ArgoCD, External Secrets 등) |
| **platform-dev-last** | Terraform 인프라 (EKS, GKE, VPC 등) |
