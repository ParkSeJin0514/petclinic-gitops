# 🐾 PetClinic GitOps

ArgoCD 기반 PetClinic 애플리케이션 배포 매니페스트 (Kustomize)

## 🏛️ 아키텍처

```
ArgoCD (platform-gitops)
    │
    └── petclinic-app.yaml
            │
            └── [Sync Wave 15]  ← Platform 컴포넌트 후 배포
                    │
                    └── petclinic-gitops/
                            ├── manifests/     # K8s 리소스
                            └── kustomization.yaml
```

## 📁 디렉토리 구조

```
├── kustomization.yaml          # Kustomize 설정 (이미지 태그 관리)
└── manifests/
    ├── 00-namespace.yaml       # petclinic 네임스페이스
    ├── 01-config-server.yaml   # Spring Cloud Config
    ├── 02-discovery-server.yaml # Eureka (K8s에서는 선택적)
    ├── 03-customers-service.yaml
    ├── 04-visits-service.yaml
    ├── 05-vets-service.yaml
    ├── 06-api-gateway.yaml     # 외부 트래픽 진입점
    ├── 07-admin-server.yaml    # Spring Boot Admin
    ├── 08-cluster-secret-store.yaml  # External Secrets (AWS SM 연동)
    ├── 09-external-secret.yaml       # DB 비밀번호 자동 주입
    ├── 10-ingress.yaml               # ALB Ingress
    ├── 11-monitoring.yaml            # Prometheus + Grafana (앱 레벨)
    ├── 12-monitoring-cluster-values.yaml
    └── 13-monitoring-cluster.yaml    # 클러스터 모니터링
```

## 🔄 배포 흐름

```
petclinic-dev (소스)
      │
      │ Push
      ▼
GitHub Actions CI
      │
      ├─ Maven Build (변경된 서비스만)
      ├─ Docker Build & ECR Push
      └─ GitOps 업데이트 (yq로 태그 수정)
              │
              ▼
petclinic-gitops (이 저장소)
      │
      │ ArgoCD 감지
      ▼
EKS Cluster 배포
```

## ⚡ Sync Wave 15

이 애플리케이션은 **Sync Wave 15**로 배포됩니다.

```yaml
# platform-gitops/apps/petclinic-app.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "15"
```

**이유:**
- EKS 클러스터 생성 직후 VPC CNI의 IP 풀 준비에 30초~2분 소요
- Platform 컴포넌트(ALB Controller, Karpenter 등) 설치 완료 후 배포
- `failed to assign an IP address` 에러 방지

## 🚀 배포 방법

### ArgoCD 자동 배포 (권장)
platform-gitops의 `petclinic-app.yaml`에서 이 저장소를 참조하여 자동 배포

### 수동 배포
```bash
# Kustomize 미리보기
kubectl kustomize .

# 배포
kubectl apply -k .

# 삭제
kubectl delete -k .
```

### ArgoCD 수동 동기화
```bash
argocd app sync petclinic
argocd app get petclinic
```

## ⚙️ 주요 기능

| 기능 | 설명 | 매니페스트 |
|------|------|-----------|
| External Secrets | AWS Secrets Manager에서 DB 비밀번호 자동 주입 | 08, 09 |
| ALB Ingress | AWS Load Balancer Controller로 ALB 생성 | 10 |
| 모니터링 | Prometheus + Grafana (앱/클러스터 레벨) | 11-13 |
| RDS 연동 | MySQL 8.0 (platform-dev에서 생성) | 03-05 |

## 🏷️ 이미지 태그 변경

### 자동 (CI/CD)
`petclinic-dev`에서 Push 시 GitHub Actions가 자동으로 `kustomization.yaml` 업데이트

### 수동
```yaml
# kustomization.yaml
images:
  - name: petclinic-config-server
    newTag: "abc123"  # Git SHA 또는 버전
  - name: petclinic-api-gateway
    newTag: "def456"
```

## 🔐 시크릿 관리

### External Secrets 연동
```yaml
# 09-external-secret.yaml
spec:
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: petclinic-db-secret
  data:
    - secretKey: password
      remoteRef:
        key: petclinic/db    # AWS Secrets Manager 키
        property: password
```

### AWS Secrets Manager에 비밀번호 저장
```bash
aws secretsmanager create-secret \
  --name petclinic/db \
  --secret-string '{"password":"your-db-password"}'
```

## 📊 모니터링

### Prometheus 메트릭
- JVM 메트릭 (Heap, GC, Threads)
- HTTP 요청 메트릭 (지연시간, 에러율)
- Custom 비즈니스 메트릭

### Grafana 대시보드
- 애플리케이션 레벨: 각 서비스별 상태
- 클러스터 레벨: 노드, Pod, 리소스 사용량

### 접속
```bash
# Grafana 포트포워딩
kubectl port-forward svc/grafana 3000:80 -n petclinic

# 브라우저: http://localhost:3000
# 기본 계정: admin / admin
```

## 🔧 트러블슈팅

### Pod가 ContainerCreating에서 멈춤

**증상:**
```
failed to assign an IP address to container
```

**원인:** VPC CNI IP 풀 준비 미완료 (클러스터 초기화 직후)

**해결:**
```bash
kubectl delete pod <pod-name> -n petclinic
# 재생성 시 IP 할당 성공
```

### DB 연결 실패

**확인:**
```bash
# Secret 확인
kubectl get secret petclinic-db-secret -n petclinic -o yaml

# External Secret 상태
kubectl get externalsecret -n petclinic

# RDS 엔드포인트 확인 (ConfigMap)
kubectl get configmap petclinic-config -n petclinic -o yaml
```

## 🔗 연관 저장소

| 저장소 | 설명 |
|--------|------|
| **petclinic-dev** | 소스 코드 + CI/CD (GitHub Actions) |
| **platform-gitops** | 플랫폼 컴포넌트 (ALB Controller, Karpenter 등) |
| **platform-dev** | Terraform 인프라 (EKS, RDS, VPC) |