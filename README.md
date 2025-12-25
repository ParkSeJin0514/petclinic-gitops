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
│   │   └── cluster-monitoring-ingress.yaml  # kube-prometheus-stack ALB Ingress
│   └── gcp/                        # GCP 환경 (AR + Workload Identity)
│       ├── cluster-monitoring-ingress.yaml       # kube-prometheus-stack GCE Ingress
│       └── cluster-monitoring-backend-config.yaml # GKE Health Check 설정
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
| API Gateway | `/actuator/health` | 8080 |
| Admin Server | `/actuator/health` | 9090 |
| Grafana (kube-prometheus-stack) | `/api/health` | 80 |
| Prometheus (kube-prometheus-stack) | `/prometheus/-/healthy` | 9090 |

### 🔗 NEG (Network Endpoint Group)

GCE Ingress에서 BackendConfig Health Check가 올바르게 적용되려면 **NEG annotation**이 필수입니다.

```yaml
# overlays/gcp/service-patch.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  annotations:
    cloud.google.com/backend-config: '{"default": "api-gateway-backend-config"}'
    cloud.google.com/neg: '{"ingress": true}'
spec:
  type: NodePort
```

**NEG 타입 비교:**

| 항목 | `{"ingress": true}` (현재 사용) | `{"exposed_ports": {...}}` |
|------|-------------------------------|----------------------------|
| NEG 이름 | 자동 생성 (랜덤 해시) | 고정 이름 |
| 클러스터 재생성 시 | 새 NEG 생성 | cluster-uid 충돌로 실패 |
| 외부 LB 연동 | 매번 재연결 필요 | 매번 재연결 필요 |
| 권장 | **권장** | 비권장 (cluster-uid 문제) |

> **중요**: NEG는 cluster-uid를 description에 포함하므로, 고정 이름(`exposed_ports`)을 사용해도 클러스터 재생성 시 기존 NEG를 재사용할 수 없습니다. 따라서 `{"ingress": true}`를 사용하는 것이 더 단순합니다.

**Instance Group vs NEG:**

| 항목 | Instance Group | NEG |
|------|----------------|-----|
| 트래픽 경로 | LB → Node → Pod | **LB → Pod (직접)** |
| Health Check | Node 레벨 | **Pod 레벨** |
| 성능 | 보통 | **더 빠름** |
| GCP 권장 | 레거시 | **권장** |

> **중요**: NEG annotation 추가 후 Ingress를 삭제/재생성해야 NEG 백엔드로 전환됩니다.

## 📊 모니터링 구성

### Cluster Monitoring (kube-prometheus-stack)

- **Helm 설치**: Terraform compute 모듈에서 자동 설치
- **Ingress 관리**: petclinic-gitops에서 통합 관리 (이 저장소)

| 항목 | AWS | GCP |
|------|-----|-----|
| Namespace | `petclinic` | `petclinic` |
| Ingress 파일 | `overlays/aws/cluster-monitoring-ingress.yaml` | `overlays/gcp/cluster-monitoring-ingress.yaml` |
| Ingress Class | ALB | GCE |
| Grafana URL | `http://<ALB>/` | `http://<GCE-LB>/` |
| Prometheus URL | `http://<ALB>/prometheus` | `http://<GCE-LB>/prometheus` |

> **참고**: Terraform은 Helm Chart만 설치하고, 모든 Ingress는 이 저장소에서 GitOps로 관리합니다.

### Application Monitoring

| 파일 | Namespace | 목적 |
|------|-----------|------|
| `11-app-monitoring.yaml` | petclinic | PetClinic 서비스 메트릭 수집 (ServiceMonitor) |

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

### GCE Ingress 502 Bad Gateway
- **원인**: BackendConfig Health Check가 적용되지 않음 (Instance Group 사용 시)
- **확인**:
  ```bash
  kubectl get ingress petclinic-ingress -n petclinic -o jsonpath='{.metadata.annotations.ingress\.kubernetes\.io/backends}' | python3 -m json.tool
  ```
- **해결**:
  1. Service에 NEG annotation 추가: `cloud.google.com/neg: '{"ingress": true}'`
  2. Ingress 삭제 후 재생성: `kubectl delete ingress petclinic-ingress -n petclinic`
  3. 백엔드가 `k8s1-xxx-...` 형태로 바뀌고 HEALTHY가 되면 정상

### GCE Ingress UNHEALTHY 백엔드
- **원인**: Health Check 경로 불일치
- **확인**: BackendConfig의 `requestPath`가 실제 서비스의 health endpoint와 일치하는지 확인
- **해결**: BackendConfig 수정 후 Ingress 재생성

### HPA 메트릭이 `<unknown>` 표시
- **원인**: Metrics Server 미설치 (EKS 기본 미설치)
- **확인**: `kubectl get pods -n kube-system | grep metrics`

### GKE ImagePullBackOff
- **원인**: GKE 서비스 계정에 Artifact Registry 읽기 권한 없음
- **해결**: `gcloud projects add-iam-policy-binding` 으로 권한 추가

### External Secret 실패
- **확인**: `kubectl describe externalsecret petclinic-db-secret -n petclinic`

### 외부 LB (psj0514-static-lb)와 GKE 연동

수동으로 생성한 외부 LB를 GKE 서비스와 연동합니다. GKE Ingress를 사용하지 않고 외부 LB만 사용합니다.

**현재 구성:**
- petclinic-ingress: 삭제됨 (외부 LB 사용)
- NEG annotation: `{"ingress": true}` (자동 이름 생성)

**클러스터 재생성 후 작업:**

1. ArgoCD Sync로 Service 배포 (NEG 자동 생성됨)
2. NEG 이름 확인:
   ```bash
   gcloud compute network-endpoint-groups list \
     --filter="name~k8s1.*petclinic.*api-gateway" \
     --project=kdt2-final-project-t1
   ```
3. GCP 콘솔에서 Load Balancer 백엔드 서비스에 NEG 연결
   - 백엔드 유형: `영역별 네트워크 엔드포인트 그룹`
   - NEG 선택: `k8s1-...-petclinic-api-gateway-...` (SIZE > 0인 것)
   - Zone: Pod가 있는 zone 선택

**NEG 상태 확인:**

```bash
# NEG 목록 및 엔드포인트 수 확인
gcloud compute network-endpoint-groups list \
  --filter="name~k8s1.*petclinic" \
  --project=kdt2-final-project-t1

# 백엔드 서비스 Health 상태 확인
gcloud compute backend-services get-health petclinic-gke-backend --global \
  --project=kdt2-final-project-t1
```

**주의사항:**
- 클러스터 삭제 시 NEG가 Load Balancer에 연결되어 있으면 삭제 실패
- 삭제 전 백엔드 서비스에서 NEG 연결 해제 필요
- 클러스터 재생성 후 NEG 이름이 변경되므로 백엔드 서비스 재연결 필수

## 🔗 관련 저장소

| 저장소 | 설명 |
|--------|------|
| **petclinic-dev** | 소스 코드 + CI/CD |
| **platform-gitops-last** | 플랫폼 컴포넌트 (ArgoCD, External Secrets 등) |
| **platform-dev-last** | Terraform 인프라 (EKS, GKE, VPC 등) |
