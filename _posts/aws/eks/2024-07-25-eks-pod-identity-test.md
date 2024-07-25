---
title: "[EKS] EKS Pod Identity 테스트 해보기"
categories:
  - eks
tags:
  - [eks auth, eks pod identity]
toc: true
toc_sticky: true
date: 2024-07-25
last_modified_at: 2024-07-25
---

![aws-eks](https://github.com/user-attachments/assets/465110c7-b76d-4f0d-a0f5-923b711cc5ea){: .align-center}

# 0. 작성 취지

이번 포스팅은 EKS Pod Identity를 사용해서 Pod에 IAM Role을 부여하는 테스트이다.

EKS Pod Identity의 경우에는 [23년 11월](https://aws.amazon.com/ko/about-aws/whats-new/2023/11/amazon-eks-pod-identity/)에 공식적으로 출시한 EKS Pod의 권한 부여 기능이다.

기존에는 IRSA(IAM Role for Service Accounts)라고 해서 K8s Service Account에 IAM Role을 Annotations로 작성해서 K8s의 권한과 AWS의 권한을 이어주는 역할을 했었다.

대신 IRSA의 경우 사용하기 위한 전제조건이 OIDC(OpenID Connect)가 별도로 구성되어야 했다.

EKS Pod Identity는 IRSA의 불편함을 해소하고자 구성된 것이며 가장 큰 차이점으로 OIDC를 생성하지 않고도 K8s Service Account와 IAM Role을 연동할 수 있으며 별도의 Annotations 또한 작성할 필요가 없다.

IRSA의 경우 관리를 위해서 CLI를 통해 확인해야 하는 번거로움이 있었다면 EKS Pod Identity는 GUI를 통해서 간편하게 확인할 수 있다.

아래 간략하게 정리해보았다.

|             |                                               EKS Pod Identity                                               |                                            IRSA                                             |
| :---------: | :----------------------------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: |
|    관리     |                                              AWS Console (GUI)                                               |                                        kubectl (CLI)                                        |
|  OIDC 필요  |                                                      X                                                       |                                              O                                              |
| 역할 확장성 | `pods.eks.amazonaws.com` 를 IAM Role의 Trust Policy에 기입하면 새로운 EKS 클러스터에 동일하게 매핑해주면 됨. | EKS 클러스터 마다 OIDC를 별도로 생성해야 하므로 IAM Role의 Trust Policy를 업데이트 해야 함. |

테스트는 아래와 같이 진행된다.

1. Pod Identity로 ALB Controller 설치 (feat. helm)

---

# 1. Pod Identity로 ALB Controller 설치 (feat. helm)

직접 helm으로 설치하는 ALB Controller 속에서 IRSA 방식이 아닌 Pod Identity로 설치하는 방식을 알아보자.

이번 방식 하나만 잘 숙지하면 다른 Addon을 설치할 때 동일한 방식으로 설치하면 된다.

## 1-1. IAM 영역

### 1-1-1. IAM Policy 생성

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json
```

![1](https://github.com/user-attachments/assets/4a6776b7-e387-4cf4-b688-5b7e07b4fb0b){: .align-center}

`curl` 명령어를 통해서 alb controller에서 필요한 IAM Policy JSON 파일을 다운로드 받고 IAM Policy를 생성한다.

### 1-1-2. IAM Role 생성

![2](https://github.com/user-attachments/assets/30f07991-573e-4f3d-97ca-89ac6d4dbca8){: .align-center}

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "pods.eks.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ]
        }
    ]
}
```

IAM Poicy를 생성한 뒤에 IAM Role을 생성하면 되는데 위 사진처럼 Trust Relationships(신뢰 관계)를 설정해주어야 한다.

## 1-2. EKS Pod Identity 설정

### 1-2-1. K8s Service Account 생성

```bash
kubectl -n kube-system create sa aws-load-balancer-controller-sa
```

alb controller에서 사용할 Service Account를 생성해야 하며 위 명령어로 생성할 수 있다.

alb controller의 경우 `kube-system` Namespace에 생성되는 것이 기본이며 Service Account의 이름은 마음대로 설정할 수 있다.

![3](https://github.com/user-attachments/assets/a3a4f7e5-92a3-4b8a-9c74-fca22fff1cd6){: .align-center}

그 이후에 EKS Console로 들어와서 Pod Identity associations에서 생성 버튼을 클릭한다.

![4](https://github.com/user-attachments/assets/295af5c2-5f54-4927-b738-fc8bb42f5487){: .align-center}

전에 만들어 놓은 IAM Role을 선택하고 연결할 Service Account의 Namespace와 Service Account를 선택한다.

![5](https://github.com/user-attachments/assets/80e066f1-2b26-4211-bb5d-1976df45da85){: .align-center}

위 사진처럼 정상적으로 생성된 모습을 확인할 수 있다.

## 1-2. Helm 영역

### 1-2-1. helm repo 설치 및 기본값 다운로드

```bash
helm repo add aws https://aws.github.io/eks-charts

helm show values aws/aws-load-balancer-controller > alb-controller.yaml
```

alb controller에서 제공하는 helm `values.yaml` 파일을 다운로드 받는다.

### 1-2-2. alb-controller.yaml 파일 수정

```yaml
serviceAccount:
    name: aws-load-balancer-controller-sa

clusterName: dev-eks-cluster

region: ap-northeast-2

vpcId: # VPC ID
```

위와 같이 `values.yaml`(`alb-controller.yaml`) 파일을 수정한다.

```bash
helm -n kube-system install aws-load-balancer-controller aws/aws-load-balancer-controller -f alb-controller.yaml
```

위 명령어를 통해서 alb controller를 helm으로 생성한다.

## 1-3. Ingress 생성 테스트

### 1-3-1. application.yaml

```yaml
# test-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
  namespace: test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: test-app
        image: # IMAGE URI
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: test-app-svc
  namespace: test
spec:
  selector:
    app: test-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: test-app-tgb
  namespace: test
spec:
  serviceRef:
    name: test-app-svc
    port: 80
  targetGroupARN: # TARGETGROUP ARN
```

ingress를 생성하는 테스트이지만 ingress의 경우 alb와 독립적으로 가져가기 위해서 별도로 생성했다.

해당 부분을 정확하게 모르겠다면 [링크](https://may-30.github.io/eks/aws-lb-controller-method-2/#1-2-ingress--alb-%EB%8F%85%EB%A6%BD%EC%A0%81%EC%9C%BC%EB%A1%9C-%EC%83%9D%EC%84%B1)를 참고하면 된다.

```bash
kubectl apply -f test-app.yaml
```

### 1-3-2. 접속 테스트

![6](https://github.com/user-attachments/assets/3ba3ffcc-5564-400a-85cb-02a323162b4b){: .align-center}

정상적으로 접속할 수 있는 것을 확인할 수 있다.

---

# 2. 마치며

IRSA는 OIDC를 사용해야 하기 때문에 이해해야 하는 깊이와 구성의 어려움이 있었지만 최근에 출시된 EKS Pod Identity는 구성 자체도 어렵지 않고 AWS Console에서 관리하기도 편리하기에 현업에서 바로 적용시켜볼 가치가 있다고 본다.
