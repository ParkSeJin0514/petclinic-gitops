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
│       ├── 08-hpa.yaml             # HPA (Horizontal Pod Autoscaler)
│       ├── 10-ingress.yaml         # Ingress (host: psj0514.site)
│       ├── 11-app-monitoring.yaml  # PetClinic 앱 모니터링 (Prometheus + Grafana)
│       └── 12-cluster-monitoring.yaml # 클러스터 모니터링 (kube-prometheus-stack)
│
├── overlays/
│   ├── aws/                        # AWS 환경
│   │   ├── kustomization.yaml      # ECR 이미지 + AWS 태그
│   │   ├── external-secret.yaml    # petclinic-kr/db (ClusterSecretStore는 platform-gitops에서 관리)
│   │   └── karpenter-node-selector-patch.yaml # Karpenter 노드 스케줄링
│   │
│   └── gcp/                        # GCP 환경
│       ├── kustomization.yaml      # Artifact Registry 이미지
│       ├── cluster-secret-store.yaml # GCP Secret Manager (Workload Identity)
│       ├── external-secret.yaml    # petclinic-dr-db-credentials
│       ├── monitoring-ingress.yaml # 앱 모니터링 Ingress (Grafana + Prometheus)
│       ├── cluster-monitoring-ingress.yaml # 클러스터 모니터링 통합 Ingress
│       ├── cluster-monitoring-backend-config.yaml # 클러스터 모니터링 Health Check
│       ├── petclinic-ingress-patch.yaml # base ingress → GKE Ingress 패치
│       ├── backend-config.yaml     # GCP Health Check 설정
│       └── service-patch.yaml      # Service에 BackendConfig 연결
```

## ☁️ Multi-Cloud 지원

| 항목 | AWS (Primary) | GCP (DR) |
|------|---------------|----------|
| **역할** | 주 운영 환경 | 재해복구 (DR) 환경 |
| **Container Registry** | ECR | Artifact Registry |
| **Secrets** | AWS Secrets Manager | GCP Secret Manager |
| **Ingress** | ALB Controller | GKE Ingress (GCE) |
| **인증** | IRSA | Workload Identity |
| **클러스터 모니터링** | kube-prometheus-stack | kube-prometheus-stack |
| **ArgoCD Path** | `overlays/aws` | `overlays/gcp` |

> **Note**: AWS가 Primary CSP이고, GCP는 DR(Disaster Recovery) 보조 환경입니다.

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
kubectl describe ingress monitoring-ingress -n petclinic | grep -i backend
```

## 🌐 GKE Ingress 구성 (GCP)

GCP 환경에서는 GKE Ingress (GCE Load Balancer)를 사용합니다.

### Ingress 구성 파일

| 파일 | 유형 | 설명 |
|------|------|------|
| `petclinic-ingress-patch.yaml` | Patch | base의 ALB Ingress를 GKE Ingress로 변환 |
| `monitoring-ingress.yaml` | Resource | Grafana + Prometheus 전용 Ingress (새 리소스) |
| `delete-separate-ingress.yaml` | Patch | base의 불필요한 개별 Ingress 삭제 |

### petclinic-ingress (패치)

```yaml
# base의 petclinic-ingress를 GKE용으로 패치
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: petclinic-ingress
  annotations:
    kubernetes.io/ingress.class: gce
spec:
  ingressClassName: gce
  rules:
    - http:
        paths:
          - path: /
            backend:
              service:
                name: api-gateway
                port:
                  number: 8080
          - path: /admin
            backend:
              service:
                name: admin-server
                port:
                  number: 9090
```

### monitoring-ingress (앱 모니터링)

```yaml
# PetClinic 앱 모니터링 Ingress (11-app-monitoring.yaml용)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: monitoring-ingress
  annotations:
    kubernetes.io/ingress.class: gce
spec:
  ingressClassName: gce
  rules:
    - http:
        paths:
          - path: /
            backend:
              service:
                name: grafana-server
                port:
                  number: 3000
          - path: /prometheus
            backend:
              service:
                name: prometheus-server
                port:
                  number: 9090
```

### cluster-monitoring-ingress (클러스터 모니터링 통합)

```yaml
# kube-prometheus-stack 통합 Ingress (Grafana + Prometheus)
# GCP에서는 2개 Ingress를 1개로 통합하여 LB 비용 절감
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cluster-monitoring-ingress
  annotations:
    kubernetes.io/ingress.class: gce
    cloud.google.com/backend-config: '{"ports": {"3000": "grafana-cluster-backend-config", "9090": "prometheus-cluster-backend-config"}}'
spec:
  ingressClassName: gce
  rules:
    - http:
        paths:
          - path: /              # Grafana
            backend:
              service:
                name: grafana-server
                port:
                  number: 3000
          - path: /prometheus    # Prometheus
            backend:
              service:
                name: prometheus-server
                port:
                  number: 9090
```

**접속 URL:**
- `http://<LB_IP>/` → Grafana 대시보드
- `http://<LB_IP>/prometheus` → Prometheus UI

### GKE Ingress 확인

```bash
# Ingress 목록 확인
kubectl get ingress -n petclinic

# External IP 확인 (프로비저닝에 3-5분 소요)
kubectl get ingress petclinic-ingress -n petclinic -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Ingress 상세 정보
kubectl describe ingress petclinic-ingress -n petclinic
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

## 📊 모니터링 구성

두 가지 레벨의 모니터링 시스템이 있습니다.

### 모니터링 파일 구조

| 파일 | Namespace | 목적 | 구성요소 |
|------|-----------|------|----------|
| `11-app-monitoring.yaml` | petclinic | PetClinic 앱 모니터링 | Prometheus + Grafana (직접 배포) |
| `12-cluster-monitoring.yaml` | monitoring | K8s 클러스터 모니터링 | kube-prometheus-stack (Helm) |

### 11-app-monitoring.yaml (애플리케이션 레벨)

PetClinic MSA 서비스들의 메트릭을 수집합니다.

```
PetClinic 서비스들 ──(/actuator/prometheus)──▶ Prometheus (petclinic) ──▶ Grafana
```

**수집 대상:**
- config-server, discovery-server
- customers-service, visits-service, vets-service
- api-gateway, admin-server

**Ingress:**
- AWS: `grafana-ingress`, `prometheus-ingress` (ALB)
- GCP: `monitoring-ingress` (GKE Ingress)

### 12-cluster-monitoring.yaml (클러스터 레벨)

Kubernetes 클러스터 전체의 메트릭을 수집합니다.

```
K8s 클러스터 ──▶ kube-prometheus-stack (monitoring) ──▶ Grafana/Prometheus/AlertManager
```

**수집 대상:**
- 노드 CPU/Memory/Disk
- Pod 상태, 재시작
- kube-apiserver, kubelet, CoreDNS 등

**Ingress (LB 연결):**

| Ingress | AWS | GCP |
|---------|-----|-----|
| `cluster-grafana-ingress` | ALB (group) | ❌ 삭제 (통합) |
| `cluster-prometheus-ingress` | ALB (group) | ❌ 삭제 (통합) |
| `cluster-monitoring-ingress` | - | GCE LB (통합) |
| `cluster-alertmanager-ingress` | ❌ 삭제 | ❌ 삭제 |

> **Note**: GCP에서는 Grafana + Prometheus를 하나의 Ingress로 통합
> - `/` → Grafana
> - `/prometheus` → Prometheus

**배포 방식:**

| 환경 | 방식 | 위치 |
|------|------|------|
| AWS (Primary) | ArgoCD + Kustomize | `petclinic-gitops/base/` + `overlays/aws/` |
| GCP (DR) | ArgoCD + Kustomize | `petclinic-gitops/base/` + `overlays/gcp/` |

> **Note**: 클러스터 모니터링(kube-prometheus-stack)은 `petclinic-gitops` 저장소에서 ArgoCD로 관리됩니다.
> - AWS: base의 ALB Ingress 사용 (Ingress Group으로 하나의 ALB 공유)
> - GCP: Kustomize JSON Patch로 ALB → GKE Ingress 변환

---

## 📈 HPA (Horizontal Pod Autoscaler)

트래픽 증가 시 자동으로 Pod 수를 조절합니다. Karpenter와 연동하여 노드도 자동 확장됩니다.

### 동작 흐름

```
트래픽 증가
    ↓
HPA: "CPU 70% 초과! Pod 2개 → 5개로 증가"
    ↓
새 Pod 3개 Pending (노드 리소스 부족)
    ↓
Karpenter: "Pending Pod 감지! 새 노드 프로비저닝"
    ↓
새 노드 Ready → Pod 스케줄링 완료
```

### HPA 적용 대상 (base/manifests/08-hpa.yaml)

| 서비스 | minReplicas | maxReplicas | CPU 임계값 |
|--------|-------------|-------------|------------|
| api-gateway | 2 | 4 | 70% |
| customers-service | 2 | 4 | 70% |
| visits-service | 2 | 4 | 70% |
| vets-service | 2 | 4 | 70% |

> **참고**: maxReplicas를 4로 제한하여 /24 서브넷 IP 고갈 방지

### HPA 미적용 서비스

| 서비스 | 이유 |
|--------|------|
| config-server | 시작 시에만 사용 (트래픽 적음) |
| discovery-server | 내부 서비스 등록용 |
| admin-server | 관리 도구 |
| prometheus-server | 모니터링 |
| grafana-server | 대시보드 |

### 스케일링 정책

**Scale Up (확장)**
- 안정화 대기 시간: 60초
- 최대 100% 증가 또는 2개 Pod 추가 (15초마다)

**Scale Down (축소)**
- 안정화 대기 시간: 300초 (5분 대기)
- 최대 50% 감소 (60초마다)

### ArgoCD ignoreDifferences 설정 (필수!)

HPA와 ArgoCD를 함께 사용할 때, ArgoCD의 `selfHeal: true` 설정이 HPA가 변경한 replicas를 Git 상태로 되돌려버리는 문제가 발생합니다.

**문제 증상:**
```
Pod 생성 → 몇 초 후 Terminating → 다시 생성 → 반복 (플래핑)
```

**원인:**
1. HPA가 CPU 초과 감지 → replicas 4로 스케일업
2. ArgoCD가 Git과 비교 → "Git에는 replicas가 없는데?"
3. ArgoCD selfHeal이 Git 상태로 복원 → Pod 삭제
4. HPA가 다시 스케일업 시도 → 무한 반복

**해결책:** ArgoCD Application에 `ignoreDifferences` 추가

```yaml
# platform-gitops/aws/apps/petclinic-app.yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

이 설정으로 ArgoCD가 Deployment의 replicas 필드를 무시하고, HPA가 자유롭게 스케일링할 수 있습니다.

### 확인 방법

```bash
# HPA 상태 확인
kubectl get hpa -n petclinic

# HPA 상세 정보
kubectl describe hpa api-gateway-hpa -n petclinic

# 현재 Pod 수와 메트릭 확인
kubectl top pods -n petclinic
```

## 🩺 Health Check (Probe) 설정

부하 시 Pod 재시작을 방지하기 위해 Probe timeout을 여유있게 설정합니다.

### Probe 설정 (4개 서비스 동일)

| Probe | 항목 | 값 | 설명 |
|-------|------|-----|------|
| **Liveness** | initialDelaySeconds | 200 | 앱 시작 대기 |
| | periodSeconds | 10 | 체크 주기 |
| | timeoutSeconds | 10 | 응답 대기 |
| | failureThreshold | 20 | 실패 허용 횟수 |
| **Readiness** | initialDelaySeconds | 120 | 앱 시작 대기 |
| | periodSeconds | 10 | 체크 주기 |
| | timeoutSeconds | 10 | 응답 대기 |
| | failureThreshold | 10 | 실패 허용 횟수 |

> **참고**: timeout을 10초로 설정하여 부하 시에도 health check 실패를 방지

---

## 🔧 트러블슈팅

### HPA에서 메트릭이 `<unknown>`으로 표시됨

**원인**: Metrics Server가 설치되지 않음 (EKS는 기본 미설치)

**확인**:
```bash
# Metrics Server Pod 확인
kubectl get pods -n kube-system | grep metrics

# metrics API 동작 확인
kubectl top nodes
```

**해결**: Metrics Server는 `platform-gitops-last`에서 ArgoCD로 관리됨
```bash
# metrics-server Application 확인
kubectl get application -n argocd | grep metrics

# 설치 후 HPA 메트릭 확인 (1-2분 대기)
kubectl get hpa -n petclinic
```

### GKE에서 ImagePullBackOff

**원인**: GKE 서비스 계정에 Artifact Registry 읽기 권한 없음

**해결**:
```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:SA@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"
```

### HPA 메모리 사용률이 100% 초과

**원인**: Java 애플리케이션의 힙 메모리가 컨테이너 메모리 limit을 초과

**증상**:
```bash
$ kubectl get hpa -n petclinic
NAME                     REFERENCE                       TARGETS           MINPODS   MAXPODS
customers-service-hpa    Deployment/customers-service    101%/80%, 15%/70%   2         8
```

**해결**: 메모리 limit 증가 + JAVA_OPTS로 힙 메모리 제한

```yaml
# base/manifests/0X-service.yaml
resources:
  requests:
    cpu: 100m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 768Mi

env:
  - name: JAVA_OPTS
    value: "-Xmx512m -Xms256m"  # 힙 메모리를 limit의 70%로 제한
```

**적용된 서비스**:

| 서비스 | CPU Req/Limit | Memory Req/Limit | JAVA_OPTS |
|--------|---------------|------------------|-----------|
| customers-service | 100m / 500m | 512Mi / 768Mi | -Xmx512m -Xms256m |
| visits-service | 100m / 500m | 512Mi / 768Mi | -Xmx512m -Xms256m |
| vets-service | 100m / 500m | 512Mi / 768Mi | -Xmx512m -Xms256m |
| api-gateway | 100m / 500m | 512Mi / 768Mi | -Xmx512m -Xms256m |

**확인**:
```bash
# HPA 메모리 사용률 확인 (70-80% 정상)
kubectl get hpa -n petclinic

# Pod 실제 메모리 사용량 확인
kubectl top pods -n petclinic

# Pod 재시작 후 적용 확인
kubectl rollout restart deployment -n petclinic --all
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
| **platform-gitops-last** | 플랫폼 컴포넌트 (ArgoCD, External Secrets 등) |
| **platform-dev-last** | Terraform 인프라 (EKS, GKE, VPC 등) |
