---
title: '[EKS] Private Gitlab과 Jenkins 연동'
categories:
  - eks
tags:
  - [cicd, gitlab, jenkins]
toc: true
toc_sticky: true
date: 2024-08-02
last_modified_at: 2024-08-02
---

![aws-eks](https://github.com/user-attachments/assets/25f9b81e-551e-476d-bd53-86819a8e64d9){: .align-center}

# 0. 작성 취지

Private Gitlab과 Jenkins 연동하는 테스트를 하려고 한다.

보통 Private Gitlab과 Jenkins 연동을 ssh key 방식으로 연결하는 것으로 많이 보이는데 해당 방식이 나에겐 적합하지 않아보였다.

**"22번 포트가 열려야 하고 해당 포트에 대한 로드밸런서의 리스너를 생성해주어야 하는데 과연 적합한 방식일까?"** 하는 생각이 들었다.

(**❗️ 물론 22번 포트는 포트 포워딩을 통해서 다른 포트로 오픈할 수 있습니다.**)

그래서 테스트를 하면서 찾아보다가 HTTPS로도 연동할 수 있는 방법을 찾게되었다.

가장 중요한 것은 아래와 같다.

<div class="notice--primary" markdown="1">

Private Gitlab의 Personal Access Token 생성

- Jenkins의 Gitlab Plugin을 통해 Gitlab과 Jenkins 연결을 위해서 Personal Access Token을 별도로 생성해야 한다.

Jenkins의 Credentials 생성

- Gitlab의 Personal Access Token을 Credentials로 생성한다. (Gitlab - Jenkins 연동)
- Gitlab의 User ID, PW 정보를 Credentials로 생성한다. (Jenkins에서 Gitlab Clone 용도)

</div>

테스트 실습은 아래와 같이 진행하며 Gitlab과 Jenkins는 사전에 설치되어 있는 것으로 가정한다.

또한 Gitlab의 Internal Load Balancer 설정은 [Private Gitlab - ArgoCD 연동](https://may-30.github.io/eks/private-gitlab-argocd-connection/)에서 이미 진행했기 때문에 생략한다.

추가로 아주 간단한 Pipeline으로 구성해서 Jenkinsfile까지 구성해보도록 한다.

1. Private Gitlab Personal Access Token 생성
2. Jenkins Plugin 다운로드
3. Jenkins Credentials 생성 및 설정
4. Jenkins Pipeline 생성

# 1. Private Gitlab Personal Access Token 생성

![1](https://github.com/user-attachments/assets/683f92d9-798e-44f7-b576-8f8243cd0d7c){: .align-center}

위 사진처럼 Personal Access Token을 생성한다.

나의 경우에는 jenkins라는 이름을 가진 Token을 생성해주었다.

# 2. Jenkins Plugin 다운로드

생성된 Jenkins에서 추가로 다운로드 받아야 할 Plugin은 `gitlab-plugin`과 `pipeline-stage-view`이다.

`gitlab-plugin`의 경우에는 jenkins가 gitlab과 연동할 수 있도록 구성할 수 있는 Plugin이다.

`pipeline-stage-view`의 경우에는 jenkins pipeline 구성 시 각 단계마다 어떻게 진행되는지에 대해 가시성있게 보여주는 Plugin이다.

## 2-1. 수동으로 받는 방법

![2](https://github.com/user-attachments/assets/6c1c4eae-69e4-4825-be1f-aced4b5e3cfa){: .align-center}

![3](https://github.com/user-attachments/assets/25a3b479-f886-4926-b089-689a79de9034){: .align-center}

Jenkins Plugin의 경우 위 사진들처럼 이미 생성된 Jenkins에 설치하는 것도 좋은 방법이다.

## 2-2. helm으로 받는 방법

하지만 helm으로 설치되어 있어서 pod 형태로 띄워져 있는 Jenkins라면 과연 이 방식이 올바른 방법인가? 하는 의구심이 든다.

왜냐하면 pod가 예기치 못한 상황에서 재시작이 되거나 다시 설치해야 하는 경우가 발생한다면 전부다 다시 손으로 설치해야 하기 때문이다.

```yaml
controller:
  installPlugins:
    - pipeline-stage-view:2.34
    - gitlab-plugin:1.8.1
```

이를 방지하고자 Jenkins의 helm에서 사용하는 yaml 파일에 위의 내용을 추가해주면 된다.

이렇게 추가한다면 `helm upgrade`, `helm install` 명령어에도 해당 plugin을 포함해서 설치하게끔 구성된다.

# 3. Jenkins Credentials 생성 및 설정

## 3-1. Gitlab Personal Access Token 등록

Jenkins와 Gitlab을 연동할 때 필요한 Token이며 Connection에 반드시 필요한 Credentials이다.

![4](https://github.com/user-attachments/assets/c5869d4a-991a-47c4-8328-11f7d801dabe){: .align-center}

![5](https://github.com/user-attachments/assets/7d050acb-1108-45f1-b311-77e11255f8fe){: .align-center}

위 순서대로 접근해서 Credentials를 생성할 수 있는 페이지에 접근한다.

![6](https://github.com/user-attachments/assets/1c11de37-f25b-40f4-a8f1-4086a2fbe9a8){: .align-center}

위 사진처럼 ID, Description은 Jenkins에서만 식별 용도로 생성하는 정보이기 때문에 원하는대로 입력해도 상관없다.

**하지만 API Token의 경우에는 사전에 발급 받은 Personal Access Token을 작성해야 한다.**

## 3-2. Gitlab User ID / Password 등록

Jenkins에서 Gitlab의 Repository를 clone 할 때 필요한 Credentials이다.

![7](https://github.com/user-attachments/assets/bc27bef8-e4ea-43e4-b281-a008a790df60){: .align-center}

Personal Access Token을 Credentials로 생성할 때와 동일하게 접근한다.

![8](https://github.com/user-attachments/assets/a5c7dc90-04e8-411b-affb-209f6d6c40b4){: .align-center}

ID, Description은 Personal Access Token과 마찬가지로 Jenkins에서만 식별 용도로 생성하는 정보이기 때문에 원하는대로 입력해도 상관없다.

**하지만 Username, Password의 경우에는 실제 Gitlab에 로그인하는 계정 정보를 작성해야 한다.**

## 3-3. Gitlab - Jenkins Connection

Gitlab의 정보를 Jenkins에서 사용할 수 있도록 Credentials로 등록했으니 실제 사용해서 Gitlab과 Jenkins의 연동을 진행할 것이다.

![9](https://github.com/user-attachments/assets/9f16249d-61db-475c-8f17-300dd7c1e3ba){: .align-center}

위 사진의 경로로 들어간다.

![10](https://github.com/user-attachments/assets/3d7b145c-3985-4ab9-b883-1a0094c473aa){: .align-center}

Gitlab의 도메인 정보가 올바르게 들어가 있어야 하고 Credentials 정보는 Gitlab API Token 정보인 Gitlab Personal Access Token 정보를 선택해주어야 한다.

그 이후에 Test Connection을 진행하면 위 사진처럼 Success가 떠야 성공적으로 연동되었다고 볼 수 있다.

# 4. Jenkins Pipeline 생성

이제 간단하게 Jenkins Pipeline을 생성해서 Gitlab의 정보를 가져올 수 있는지 직접 테스트를 해볼 것이다.

## 4-1. Pipeline 생성

![11](https://github.com/user-attachments/assets/1167cc91-53b3-4957-90bc-cd4e7f0d7f1b){: .align-center}

위 사진과 같이 Jenkins Pipeline을 생성한다.

![12](https://github.com/user-attachments/assets/ef5ce5d5-262b-42b9-9ab2-eb75d1527315){: .align-center}

![13](https://github.com/user-attachments/assets/45b8dc1d-b061-4f0b-8169-7e448cd526bd){: .align-center}

위 사진 순서대로 생성을 진행하며 사전에 연결해둔 gitlab connection은 자동으로 선택되어 있는 것을 확인할 수 있다.

```groovy
pipeline {
    agent any
    
    stages {
        stage ("Checkout Application Git Branch") {
            steps {
                git branch: "main",
                    credentialsId: "root",
                    url: "https://gitlab.may30.xyz/root/gitlab-conn.git"
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
        
        stage ("Check Directory") {
            steps {
                sh "ls -al"
            }
        }
    }
}
```

Jenkinsfile은 위와 같이 구성되어 있다.

## 4-2. Build 및 로그 확인

![14](https://github.com/user-attachments/assets/2388ea22-dd6b-4286-92e7-765ffa9f1bf7){: .align-center}

빌드를 하게 되면 Jenkins는 controller, agent 구조이기 때문에 job이 생성될 때마다 agent가 하나 동작하는 것을 확인할 수 있다.

![15](https://github.com/user-attachments/assets/a8e18ad1-c69f-4029-acd7-d9ed87cff47e){: .align-center}

정상적으로 동작한 모습을 확인할 수 있다.

# 5. 마치며

Private Gitlab과 Jenkins 간의 연동은 성공적으로 마쳤다.

이제 연결된 Gitlab으로 Jenkins에서 Application의 빌드와 배포까지 진행할 수도 있지만 K8s 환경에서는 Jenkins에서는 빌드까지만 진행하고 배포는 ArgoCD에서 진행하도록 구성한다.