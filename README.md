# KYEOL Platform GitOps

> Kubernetes 플랫폼 컴포넌트 Helm Charts GitOps 레포지토리
> AWS Load Balancer Controller, ExternalDNS, Fluent Bit, Metrics Server 관리

---

## 개요

KYEOL 프로젝트의 EKS 클러스터에 필수 플랫폼 컴포넌트를 Helm Charts로 배포합니다. ArgoCD를 통해 GitOps 방식으로 관리하며, 환경별 독립 구성을 지원합니다.

### 관리 대상 컴포넌트

- **AWS Load Balancer Controller**: Kubernetes Ingress 리소스를 ALB로 프로비저닝
- **ExternalDNS**: Kubernetes Service/Ingress를 Route53에 자동 등록
- **Fluent Bit**: 컨테이너 로그를 CloudWatch Logs로 전송
- **Metrics Server**: HPA (Horizontal Pod Autoscaler)를 위한 메트릭 수집

---

## 디렉토리 구조

```
kyeol-platform-gitops/
├── clusters/                       # 클러스터별 Helm Values
│   ├── common/
│   │   └── values/
│   │       └── fluent-bit.values.yaml  # 공통 설정
│   ├── dev/
│   │   ├── addons/
│   │   │   ├── aws-load-balancer-controller/
│   │   │   │   └── kustomization.yaml
│   │   │   ├── external-dns/
│   │   │   └── metrics-server/
│   │   └── values/
│   │       ├── aws-load-balancer-controller.values.yaml
│   │       ├── external-dns.values.yaml
│   │       ├── fluent-bit.values.yaml
│   │       └── metrics-server.values.yaml
│   ├── mgmt/
│   │   └── values/
│   │       └── fluent-bit.values.yaml  # MGMT는 ALB Controller 없음
│   ├── stage/
│   │   ├── addons/
│   │   └── values/
│   └── prod/
│       ├── addons/
│       ├── patches/
│       │   └── alb-readiness-probe-patch.yaml
│       └── values/
├── common/
│   └── namespaces/
│       └── kyeol-platform.yaml
├── argocd/                         # ArgoCD Application 정의
│   ├── app-of-apps/
│   │   ├── projects/
│   │   │   ├── kyeol-app-project.yaml
│   │   │   └── kyeol-platform-project.yaml
│   │   └── root-app.yaml
│   ├── applications/
│   │   ├── dev-aws-load-balancer-controller.yaml
│   │   ├── dev-external-dns.yaml
│   │   ├── dev-fluent-bit.yaml
│   │   ├── mgmt-fluent-bit.yaml
│   │   ├── stage-aws-load-balancer-controller.yaml
│   │   ├── stage-external-dns.yaml
│   │   ├── stage-fluent-bit.yaml
│   │   ├── prod-aws-load-balancer-controller.yaml
│   │   ├── prod-external-dns.yaml
│   │   └── prod-fluent-bit.yaml
│   └── bootstrap/
│       ├── kustomization.yaml
│       └── namespace.yaml
└── README.md
```

---

## 핵심 원칙

### ✅ 해야 할 것

1. **Terraform에서 IRSA 생성 후 Helm 배포**
   - AWS Load Balancer Controller, ExternalDNS, Fluent Bit 모두 IRSA 필요
   - Terraform output에서 Role ARN 확인 후 values.yaml에 설정

2. **환경별 독립 구성**
   - MGMT, DEV, STAGE, PROD 각각 독립된 Helm Release
   - 환경별 values 파일 분리

3. **ArgoCD로 자동 동기화**
   - Git Push 시 자동으로 Helm 차트 업데이트
   - 선언적 배포 (Helm install/upgrade 수동 실행 최소화)

4. **Helm Repository 사용**
   - AWS Load Balancer Controller: `https://aws.github.io/eks-charts`
   - ExternalDNS: `https://kubernetes-sigs.github.io/external-dns`
   - Fluent Bit: `https://fluent.github.io/helm-charts`
   - Metrics Server: `https://kubernetes-sigs.github.io/metrics-server`

### ❌ 하지 말아야 할 것

1. **IRSA 없이 Helm 배포 금지**
   - Access Key 사용 금지

2. **ALB Controller 삭제 주의**
   - ALB Controller 삭제 시 모든 Ingress ALB도 삭제됨
   - 먼저 애플리케이션 Ingress 제거 후 ALB Controller 삭제

3. **환경별 values 혼용 금지**
   - DEV values를 PROD에 복사 금지 (IRSA Role ARN 다름)

---

## 환경별 구성

| 환경 | ALB Controller | ExternalDNS | Fluent Bit | Metrics Server |
|------|----------------|-------------|------------|----------------|
| **MGMT** | ❌ | ❌ | ✅ | ❌ |
| **DEV** | ✅ | ✅ | ✅ | ✅ |
| **STAGE** | ✅ | ✅ | ✅ | ✅ |
| **PROD** | ✅ | ✅ | ✅ | ✅ |

---

## 빠른 시작

### 사전 요구사항

- Terraform으로 EKS 클러스터 및 IRSA 생성 완료
- kubectl 설정 완료
- Helm 3.x 설치

### Terraform Output 확인

```bash
# DEV 환경 IRSA Role ARN 확인
cd kyeol-infra-terraform/envs/dev
terraform output alb_controller_role_arn
terraform output external_dns_role_arn
terraform output fluent_bit_role_arn
```

### Helm 차트 배포

#### 수동 배포 (테스트용)

```bash
# DEV 환경 kubectl context 설정
kubectl config use-context kyeol-dev-eks

# AWS Load Balancer Controller 설치
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  -f clusters/dev/values/aws-load-balancer-controller.values.yaml

# ExternalDNS 설치
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns
helm repo update

helm upgrade --install external-dns external-dns/external-dns \
  -n kube-system \
  -f clusters/dev/values/external-dns.values.yaml

# Fluent Bit 설치
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

helm upgrade --install fluent-bit fluent/fluent-bit \
  -n kube-system \
  -f clusters/dev/values/fluent-bit.values.yaml

# Metrics Server 설치
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server
helm repo update

helm upgrade --install metrics-server metrics-server/metrics-server \
  -n kube-system \
  -f clusters/dev/values/metrics-server.values.yaml
```

#### ArgoCD 자동 배포 (권장)

```bash
# ArgoCD에서 Application 생성
kubectl apply -f argocd/applications/dev-aws-load-balancer-controller.yaml
kubectl apply -f argocd/applications/dev-external-dns.yaml
kubectl apply -f argocd/applications/dev-fluent-bit.yaml

# 동기화 확인
argocd app list
argocd app get aws-load-balancer-controller-dev
```

---

## 주요 작업

### 1. 새 환경 추가

```bash
# 1. 기존 환경 복사
cp -r clusters/dev clusters/qa

# 2. values 파일에서 IRSA Role ARN 변경
vim clusters/qa/values/aws-load-balancer-controller.values.yaml
# serviceAccount.annotations.eks.amazonaws.com/role-arn: <QA_ROLE_ARN>

# 3. ArgoCD Application 생성
cp argocd/applications/dev-aws-load-balancer-controller.yaml \
   argocd/applications/qa-aws-load-balancer-controller.yaml

vim argocd/applications/qa-aws-load-balancer-controller.yaml
# name, destination.server, path 변경
```

### 2. Helm 차트 버전 업그레이드

```yaml
# argocd/applications/dev-aws-load-balancer-controller.yaml
spec:
  sources:
    - repoURL: https://aws.github.io/eks-charts
      chart: aws-load-balancer-controller
      targetRevision: 1.7.0  # 1.6.2 → 1.7.0
```

```bash
git add argocd/applications/dev-aws-load-balancer-controller.yaml
git commit -m "chore: Upgrade ALB Controller to 1.7.0 in dev"
git push origin main

# ArgoCD가 자동으로 감지 및 Sync
```

### 3. Values 설정 변경

```yaml
# clusters/prod/values/aws-load-balancer-controller.values.yaml
resources:
  limits:
    cpu: 500m      # 200m → 500m
    memory: 512Mi  # 256Mi → 512Mi
  requests:
    cpu: 200m      # 100m → 200m
    memory: 256Mi  # 128Mi → 256Mi
```

```bash
git add clusters/prod/values/aws-load-balancer-controller.values.yaml
git commit -m "scale: Increase ALB Controller resources in prod"
git push origin main
```

---

## 컴포넌트 상세

### AWS Load Balancer Controller

**역할**: Kubernetes Ingress → AWS ALB 자동 생성

**핵심 설정**:
```yaml
# clusters/dev/values/aws-load-balancer-controller.values.yaml
clusterName: kyeol-dev-eks
serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/kyeol-dev-alb-controller
region: ap-northeast-2
vpcId: vpc-xxxxxxxxx
```

**확인**:
```bash
# Pod 상태
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# Ingress 생성 시 ALB 자동 생성 확인
kubectl get ingress -A
aws elbv2 describe-load-balancers --region ap-northeast-2
```

### ExternalDNS

**역할**: Kubernetes Service/Ingress → Route53 DNS 레코드 자동 생성

**핵심 설정**:
```yaml
# clusters/dev/values/external-dns.values.yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/kyeol-dev-external-dns
provider: aws
domainFilters:
  - dev.kyeol.com
txtOwnerId: kyeol-dev-eks
policy: sync  # upsert-only (생성만), sync (생성+삭제)
```

**확인**:
```bash
# Pod 상태
kubectl get pods -n kube-system -l app.kubernetes.io/name=external-dns

# Route53 레코드 확인
aws route53 list-resource-record-sets --hosted-zone-id Z1234567890ABC
```

### Fluent Bit

**역할**: 컨테이너 로그 → CloudWatch Logs 전송

**핵심 설정**:
```yaml
# clusters/dev/values/fluent-bit.values.yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/kyeol-dev-fluent-bit
config:
  outputs: |
    [OUTPUT]
        Name cloudwatch_logs
        Match *
        region ap-northeast-2
        log_group_name /aws/eks/kyeol-dev-eks/cluster
        auto_create_group true
```

**확인**:
```bash
# Pod 상태
kubectl get pods -n kube-system -l app.kubernetes.io/name=fluent-bit

# CloudWatch Logs 확인
aws logs describe-log-groups --log-group-name-prefix /aws/eks/kyeol-dev-eks
```

### Metrics Server

**역할**: Pod/Node 메트릭 수집 (HPA용)

**핵심 설정**:
```yaml
# clusters/dev/values/metrics-server.values.yaml
args:
  - --kubelet-preferred-address-types=InternalIP
  - --kubelet-use-node-status-port
  - --metric-resolution=15s
```

**확인**:
```bash
# Pod 상태
kubectl get pods -n kube-system -l app.kubernetes.io/name=metrics-server

# 메트릭 조회
kubectl top nodes
kubectl top pods -A
```

---

## 트러블슈팅

### ALB Controller Pod CrashLoopBackOff

**증상**: ALB Controller Pod가 시작 실패

**확인**:
```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

**일반적인 원인**:
- IRSA Role ARN 잘못 설정
- VPC ID 잘못 설정
- Cluster Name 불일치

**해결**:
```bash
# Terraform output 확인
cd kyeol-infra-terraform/envs/dev
terraform output alb_controller_role_arn
terraform output vpc_id
terraform output eks_cluster_name

# values.yaml 수정 후 재배포
helm upgrade aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  -f clusters/dev/values/aws-load-balancer-controller.values.yaml
```

### Ingress 생성했지만 ALB 미생성

**증상**: Ingress 리소스 생성했지만 ALB가 생성되지 않음

**확인**:
```bash
kubectl describe ingress <ingress-name> -n <namespace>
```

**일반적인 원인**:
- Ingress Class 어노테이션 누락
- ALB Controller Pod 미실행

**해결**:
```yaml
# Ingress에 필수 어노테이션 추가
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
```

### ExternalDNS가 Route53 레코드 생성 안 함

**증상**: Ingress/Service 생성했지만 DNS 레코드 미생성

**확인**:
```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=external-dns
```

**일반적인 원인**:
- IRSA 권한 부족
- domainFilters 불일치
- Hosted Zone ID 잘못 설정

**해결**:
```bash
# IAM Policy 확인
aws iam get-role-policy --role-name kyeol-dev-external-dns --policy-name ExternalDNSPolicy

# values.yaml 확인
cat clusters/dev/values/external-dns.values.yaml
```

---

## 모범 사례

### 1. GitOps 우선

수동 Helm 명령 사용을 최소화하고 ArgoCD를 통한 자동 배포를 사용합니다.

```bash
# ❌ 잘못된 방법
helm upgrade aws-load-balancer-controller ...

# ✅ 올바른 방법
# 1. values.yaml 수정
# 2. Git Commit & Push
# 3. ArgoCD 자동 Sync 대기
```

### 2. 환경별 독립성

각 환경은 독립된 Helm Release로 관리합니다.

```bash
# ✅ 올바른 방법
helm list -A | grep aws-load-balancer-controller
# aws-load-balancer-controller-dev
# aws-load-balancer-controller-stage
# aws-load-balancer-controller-prod
```

### 3. IRSA 우선

모든 플랫폼 컴포넌트는 IRSA를 통해 AWS 서비스에 접근합니다.

```yaml
# ✅ 올바른 방법
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::...

# ❌ 잘못된 방법
env:
  - name: AWS_ACCESS_KEY_ID
    value: AKIAIOSFODNN7EXAMPLE
```

---

## 배포 전 체크리스트

- [ ] Terraform으로 IRSA 생성 완료
- [ ] Terraform output에서 Role ARN 확인
- [ ] values.yaml에 정확한 ARN 설정
- [ ] clusterName, vpcId, region 설정 확인
- [ ] Helm 차트 버전 호환성 확인
- [ ] STAGE 환경에서 충분히 테스트 (PROD 배포 시)

---

## 다른 레포지토리와의 관계

| 레포지토리 | 관계 |
|-----------|------|
| kyeol-infra-terraform | 이 레포 실행 전 EKS + IRSA 생성 필요 |
| kyeol-app-gitops | 이 레포 실행 후 애플리케이션 배포 가능 |

---

## 관련 문서

- **플랫폼 운영 가이드**: [kyeol-docs/runbook/runbook-platform.md](../kyeol-docs/runbook/runbook-platform.md)
- **인프라 구성**: [kyeol-docs/runbook/runbook-infra.md](../kyeol-docs/runbook/runbook-infra.md)
- **장애 대응**: [kyeol-docs/troubleshooting.md](../kyeol-docs/troubleshooting.md)

---

**마지막 업데이트**: 2026-01-21
**레포지토리**: https://github.com/MSP-Team3/kyeol-platform-gitops
