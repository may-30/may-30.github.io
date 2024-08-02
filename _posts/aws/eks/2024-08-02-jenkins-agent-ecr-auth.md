---
title: '[EKS] Jenkins Agent에 ECR 권한이 필요하다'
categories:
  - eks
tags:
  - [cicd, jenkins]
toc: true
toc_sticky: true
date: 2024-08-02
last_modified_at: 2024-08-02
---

![aws-eks](https://github.com/user-attachments/assets/5363d72b-369c-4af6-94e8-40f0c8eaec57){: .align-center}

# 0. 작성 취지

Jenkins Pipeline을 테스트 중에 있다가 ECR로 docker image를 push 해야 하는 로직이 구현되어야 했다.

그치만 내가 구성한 Jenkins는 controller, agent 기반으로 동작하는 것이라서 Jenkins Job이 동작할 때마다 Jenkins Agent가 생성되었다가 Job이 끝나면 종료되는 구조이다.

그렇기 때문에 Jenkins Agent만의 Pod에 별도로 권한을 부여하는 방식을 찾아보았고 Jenkins Agent Pod에 ServiceAccount를 부여하는 방법을 알게 되었다.

테스트는 아래 순서로 진행된다.

1. IAM 생성
2. Jenkins Helm 설정
3. EKS Pod Identity 설정
4. Jenkins 설정

# 1. IAM 생성

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:CompleteLayerUpload",
                "ecr:GetAuthorizationToken",
                "ecr:UploadLayerPart",
                "ecr:InitiateLayerUpload",
                "ecr:BatchCheckLayerAvailability",
                "ecr:PutImage"
            ],
            "Resource": "*"
        }
    ]
}
```

위 json 파일은 iam policy이다.

Jenkins의 특성상 모든 ECR에 접근해서 Push 할 수 있어야 한다는 생각을 가지고 있기 때문에 위와 같은 정책을 적용했다.

물론, Jenkins Agent 마다 생성해서 각 Agent에서 접근할 수 있는 서비스의 ECR 별로 권한을 상세하게 가져가는 것도 올바른 방법이다.

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

신뢰 관계는 EKS Pod Identity 방식을 사용할 것이므로 위와 같이 구성한다.

# 2. Jenkins Helm 설정

```yaml
serviceAccountAgent:
  create: true
  name: jenkins-agent-sa
```

Jenkins의 Agent에서 사용할 ServiceAccount를 정의하는 부분이 존재한다.

EKS Pod Identity를 사용할 것이기 때문에 위처럼 아주 간단하게 ServiceAccount만 구성하면 되고 위처럼 작성해주면 Jenkins Agent가 생성될 때 자동으로 ServiceAccount가 매핑되서 동작할 것이다.

```bash
helm -n jenkins upgrade jenkins jenkins/jenkins -f jenkins.yaml
```

위 명령어를 통해서 Jenkins의 values를 적용시킨다.

# 3. EKS Pod Identity 설정

![1](https://github.com/user-attachments/assets/f0fba1c7-bd22-42a5-801a-6b6ab8973d87){: .align-center}

위와 같이 EKS Pod Identity를 적용해주면 된다.

EKS Pod Identity 적용법을 모르겠다면 [링크](https://may-30.github.io/eks/eks-pod-identity-test/)를 참고해보길 바란다.

# 4. Jenkins 테스트

본격적인 준비는 마쳤다.

이제 Jenkins에서 정말 해당 Role을 잘 사용할 수 있는지를 증명하면 된다.

그러기 위해서는 간단한 Pipeline을 생성할 것이고 거기서 테스트를 직접할 것이다.

## 4-1. Pipeline 생성

```groovy
pipeline {
    agent any

    stages {
      stage ("Checkout Application Git Branch") {
        steps {
          git branch: "main",
              credentialsId: "root",
              url: "https://gitlab.may30.xyz/root/test-app.git"
        }
        post {
          failure {
            echo "❌ Checkout Failed!"
          }
          success {
            echo "✅ Checkout Success!"
          }
        }
      }
      
    stage ("Check ECR Auth") {
      steps {
        container("dind") {
          sh "aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin {ECR URI 주소}"
        }
      }
    }
  }
}
```

위 Jenkinsfile 양식을 참고해서 Jenkins Pipeline을 생성하면 된다.

여기서 주목할 것은 `container("dind")` 이다.

현재 Jenkins Agent Pod에는 총 2개의 Container가 동작한다.

1. dind
2. jnpl

2가지 중 특히 `dind`는 K8s에서 더이상 docker 컨테이너 런타임 지원 중단으로 인해 Jenkins도 마찬가지로 docker 런타임을 제외시켰고 때문에 docker 관련 container를 별도로 동작시켜야 했다.

Jenkins Agent의 기본으로 동작하는 컨테이너는 `jnpl` 컨테이너이고 해당 컨테이너에는 docker 명령어가 존재하지 않는다.

떄문에 docker 명령어를 동작시키기 위해서는 docker 명령어가 들어있는 `dind` 컨테이너를 지정해서 동작시켜주어야 한다.

## 4-2. Jenkins 테스트

![2](https://github.com/user-attachments/assets/98fece08-fa51-46f7-bc4d-2125b503fcff){: .align-center}

![3](https://github.com/user-attachments/assets/a343f2b1-d2cb-40d9-a6fd-d95a634a65ca){: .align-center}

Jenkins에서 빌드를 누르면 Jenkins Job이 생성되고 Jenkins Job이 생성되면 Jenkins Agent가 동작하는 방식이다.

해당 Jenkins Agent를 분석하면 ServiceAccount와 EKS Pod Identity가 정상적으로 동작한다는 것을 알 수 있다.

![4](https://github.com/user-attachments/assets/79920d5f-3652-49af-937c-699d67627da4){: .align-center}

성공적으로 ECR에 login을 한 결과를 토대로 Jenkins Agent에 IAM 권한을 부여한 것이 증명되었다.

# 5. 마치며

확실히 IRSA 방식보다 EKS Pod Identity 방식이 훨씬 간편하다는 것을 느끼게 되었다.