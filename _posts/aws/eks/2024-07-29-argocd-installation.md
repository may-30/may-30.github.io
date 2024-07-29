---
title: '[EKS] ArgoCD 설치'
categories:
  - eks
tags:
  - [cicd, argocd]
toc: true
toc_sticky: true
date: 2024-07-29
last_modified_at: 2024-07-29
---

![aws-eks](https://github.com/user-attachments/assets/e507cbd4-dd54-4f89-9979-481969113fbd){: .align-center}

# 0. 작성 취지

EKS 환경에 ArgoCD를 helm으로 설치하는 과정을 작성하려고 한다.

해당 글의 핵심은 `nodeSelector`, `nodeAffinity`, `targetgroupbindings`이 될 것 같다.

순서는 아래와 같이 작성된다.

1. helm 단계
2. 외부 노출 단계

# 1. helm 단계

```bash
helm repo add argo https://argoproj.github.io/argo-helm

helm show values argo/argo-cd > argocd.yaml
```

위 명령어로 ArgoCD helm 단계 준비를 한다.

```yaml
global:
  domain: argocd.may30.xyz
  nodeSelector:
    node-type: mgmt
  affinity:
    podAntiAffinity: soft
    nodeAffinity:
      type: hard
      matchExpressions:
        - key: node-type
          operator: In
          values:
            - mgmt
configs:
  cm:
    timeout.reconciliation: 60s
  params:
    server.insecure: true
repoServer:
  autoscaling:
    enabled: true
```

## 1-1. global 부분

### 1-1-1. domain

ArgoCD에 접속할 도메인을 기입한다.

### 1-1-2. nodeSelector, affinity

현재 하고 있는 테스트는 node에 label을 적용해서 해당 node에만 생성되도록 설정해두는 것이다.

때문에 key-value가 `node-type` - `mgmt`로 되어 있는 node에 Jenkins가 생성되도록 설정해두었다.

## 1-2. configs 부분

### 1-2-1. cm

ArgoCD는 Source에 해당하는 Git Repsoitory를 주기적으로 Sync해서 변경되는 포인트가 발생하면 자동으로 Update가 되는 구조인데 기본적으로 180초(3분)이다.

이를 짧게 가져가기 위해서 60초(1분)으로 설정했다.

### 1-2-2. params

ArgoCD는 자체적으로 인증서를 가지고 있다.

때문에 외부에서 들어오는 트래픽에 대해서 자동으로 Redirect가 처리되어 자신만의 인증서로 인증하려고 하는데 이를 방지하고자 true로 해주었다.

## 1-3. repoServer 부분

repoServer에 한하여 autoscaling이 되도록 구성하였고 HPA로 동작하기 때문에 `metrics-server`는 필수로 설치되어 있어야 한다.

# 2. 외부 노출 단계

## 2-1. Health Check 경로 변경

![1](https://github.com/user-attachments/assets/4892c0b9-703e-4aae-b7c8-e92d889f5a51){: .align-center}

`/healthz`로 변경해주었다.

![2](https://github.com/user-attachments/assets/dfa08b9e-615f-4e2d-a418-cd9b422a4fc9){: .align-center}

생성된 ArgoCD의 Probe 설정을 보면 `/healthz`로 되어 있는 것을 확인할 수 있다.

## 2-2. targetgroupbindings

```yaml
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: argocd-tgb
  namespace: argo
spec:
  serviceRef:
    name: argocd-server
    port: 80
  targetGroupARN: # TARGETGROUP ARN
  vpcID: # VPC ID
```

targetgroupbinding을 통해서 Target Group과 K8s의 Service를 연결시켜주고 ALB에 등록하면 된다.

targetgroupbinding은 [링크](https://may-30.github.io/eks/aws-lb-controller-method-2/#1-2-ingress--alb-%EB%8F%85%EB%A6%BD%EC%A0%81%EC%9C%BC%EB%A1%9C-%EC%83%9D%EC%84%B1)에서 더 자세하게 볼 수 있다.

![3](https://github.com/user-attachments/assets/f21bc905-59b5-444e-b664-86d65f54c564){: .align-center}

targetgroupbindings까지 생성을 완료하면 외부에서 정상적으로 접속할 수 있는 모습을 확인할 수 있다.

# 3. 마치며

우선, ArgoCD 설치에 대해 알아보았다.

ArgoCD는 운영 환경에서 사용할 경우라면 직접 화면에서 수정해주어야 할 부분들이 상당히 많다.

하나씩 테스트 해보면서 알아가보면 좋을 듯 하다.