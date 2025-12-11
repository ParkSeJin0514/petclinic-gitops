# PetClinic GitOps

ArgoCD 기반 PetClinic 애플리케이션 배포 매니페스트 (Kustomize)

## 아키텍처

```
ArgoCD
    │
    └── petclinic-app (Application)
            │
            └── petclinic-gitops/
                    ├── manifests/     # K8s 리소스
                    └── kustomization.yaml
```

## 디렉토리 구조

```
├── kustomization.yaml          # Kustomize 설정
└── manifests/
    ├── 00-namespace.yaml       # petclinic 네임스페이스
    ├── 01-config-server.yaml
    ├── 02-discovery-server.yaml
    ├── 03-customers-service.yaml
    ├── 04-visits-service.yaml
    ├── 05-vets-service.yaml
    ├── 06-api-gateway.yaml
    ├── 07-admin-server.yaml
    ├── 08-cluster-secret-store.yaml  # External Secrets
    ├── 09-external-secret.yaml       # DB 비밀번호
    ├── 10-ingress.yaml               # ALB Ingress
    ├── 11-monitoring.yaml            # Prometheus + Grafana
    ├── 12-monitoring-cluster-values.yaml
    └── 13-monitoring-cluster.yaml
```

## 배포 방법

### ArgoCD 자동 배포
platform-gitops의 `petclinic-app.yaml`에서 이 저장소를 참조

### 수동 배포
```bash
# Kustomize 미리보기
kubectl kustomize .

# 배포
kubectl apply -k .

# 삭제
kubectl delete -k .
```

## 주요 기능

| 기능 | 설명 |
|------|------|
| External Secrets | AWS Secrets Manager에서 DB 비밀번호 자동 주입 |
| ALB Ingress | AWS Load Balancer Controller로 ALB 생성 |
| 모니터링 | Prometheus + Grafana (앱/클러스터 레벨) |

## 이미지 태그 변경

`kustomization.yaml` 수정:
```yaml
images:
  - name: petclinic-config-server
    newTag: "2.0"  # 변경할 태그
```

## 연관 저장소

- **petclinic-dev**: 소스 코드 + CI/CD (Jenkins)
- **platform-gitops**: 플랫폼 컴포넌트 (ALB Controller, EFS CSI 등)
