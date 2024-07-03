---
title: "[EKS] AWS Auth 사라진다"
categories:
  - eks
tags:
  - [aws-auth]
toc: true
toc_sticky: true
date: 2024-07-02
last_modified_at: 2024-07-03
---

![aws-eks](https://github.com/may-30/may-30.github.io/assets/155306250/9152a129-69af-4e72-8cd8-9f41524e0681)

# 0. 작성 취지

![1](https://github.com/may-30/may-30.github.io/assets/155306250/e61a91a8-2062-418e-9302-6d73cc17d57c)

[eks 공식 문서](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/auth-configmap.html)에서 위 사진과 같이 `aws-auth`는 더 이상 사용되지 않는다고 언급한다.

**"그럼 관리를 어떻게...?"** 하는 궁금증에서 출발해서 작성하게 되었다.

# 1. aws-auth

**`aws-auth`을 간단하게 말해보자면 aws의 권한(`iam`)과 k8s의 권한(`role`, `clusterrole`)을 서로 연결시켜주는 것이다.**

이러한 역할자가 필요한 이유는 aws와 k8s는 서로 다른 영역이기 때문에 aws에서는 k8s의 영역에서도 aws의 iam으로 권한 관리를 할 수 있도록 만들어 놓는 시스템이 `aws-auth`이다.

eks에서 역할 기반, 사용자 기반의 접근 제어를 위해서 `aws-auth`라는 `kube-system` namespace의 configmap을 통해 연결시킨다.

```yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:masters
      rolearn: arn:aws:iam::{ACCOUNT_ID}:role/AWSReservedSSO_{PERMISSION_SET_NAME}_{RANDOM_ID}
      username: {USERNAME}
  mapUsers: |
    - groups:
      - system:masters
      userarn: arn:aws:iam::{ACCOUNT_ID}:user/{IAM_USER}
      username: {USERNAME}
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
```

위 yaml 파일은 `aws-auth`의 예시이다.

역할 기반으로 접근할 경우에는 `mapRoles` 영역에, 사용자 기반으로 접근할 경우에는 `mapUsers` 영역에 기재하면 된다.

자 그렇다면 직접 `aws-auth`를 테스트 해보자.

`aws-auth`에서 **특정 aws iam user**에 대해서 **k8s의 모든 namespace에 대해 읽기 권한** 만을 **연결**하고 싶다면 어떻게 해야 할까?

아래 3가지가 필요하다.

1. 특정 aws iam user : **aws iam policy - aws iam user 간 연결**
2. k8s의 모든 namespace에 대한 읽기 권한 : **clusterrole, clusterrolebinding**
3. 연결 : **aws-auth**

테스트는 위 순서대로 진행한다.

## 1-1. iam policy, iam user 생성

먼저 aws의 권한부터 생성한다.

### 1-1-1. iam policy 생성

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:ListClusters"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:AccessKubernetesApi"
      ],
      "Resource": [
        "arn:aws:eks:ap-northeast-2:{ACCOUNT_ID}:cluster/{EKS_CLUSTER_NAME}"
      ]
    }
  ]
}
```

위 json 파일과 같이 정책을 담은 json 파일을 생성한다.

(**❗️ 해당 정책은 다른 aws 서비스는 사용하지 못하고 오직 클러스터를 접근하기 위한 정말 최소한의 권한만 들어가있다는 점을 참고한다.**)

```bash
aws iam create-policy \
--policy-name test-policy-eks-readonly \
--policy-document file://eks-readonly.json
```

aws cli로 간단하게 iam policy를 생성한다.

### 1-1-2. iam user 생성

```bash
aws iam create-user \
--username test-eks-readonly
```

### 1-1-3. iam user와 iam role 연결

```bash
aws iam attach-user-policy \
--user-name test-eks-readonly \
--policy-arn {test-policy-eks-readonly_ARN}
```

![2](https://github.com/may-30/may-30.github.io/assets/155306250/f395f9af-66cf-496f-9f2e-d8b87e85600f)

위 사진을 통해서 성공적으로 연결된 모습을 확인할 수 있다.

## 1-2. clusterrole, clusterrolebinding 생성

그 다음은 k8s의 권한을 생성한다.

테스트에서는 k8s의 모든 namespace에 대해 읽기 권한을 부여할 것이기 때문에 `clusterrole`을 생성한다.

특정 namespace에 대한 권한이라면 `role`을 생성해야 한다.

### 1-2-1. clusterrole 생성

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-only
rules:
  - apiGroups: ['*']
    resources: ['*']
    verbs: ['get', 'list', 'watch']
```

clusterrole.yaml 파일의 모습이다.

```bash
kubectl apply -f clusterrole.yaml
```

### 1-2-2. clusterrolebinding 생성

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-only
roleRef:
  kind: ClusterRole
  name: read-only
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: Group
    name: read-only
    apiGroup: rbac.authorization.k8s.io
```

clusterrolebinding.yaml 파일의 모습이다.

```bash
kubectl apply -f clusterrolebinding.yaml
```

## 1-3. aws-auth 수정

```bash
kubectl -n kube-system edit cm aws-auth
```

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  mapRoles: |
    null
  mapUsers: |
    - groups:
      - read-only
      userarn: arn:aws:iam::{ACCOUNT_ID}:user/test-eks-readonly
      username: test-eks-readonly
kind: ConfigMap
```

위 파일과 같이 `aws-auth`를 수정하면 된다.

(**❗️ metadata는 불필요한 정보라 삭제함.**)

## 1-4. 접근 테스트

```bash
aws eks update-kubeconfig --profile {test-eks-readonly_PROIFLE} --name {EKS_CLUSTER_NAME}
```

![3](https://github.com/may-30/may-30.github.io/assets/155306250/c0c49e41-4a0b-49ee-9f3b-73dc119432f3)

![4](https://github.com/may-30/may-30.github.io/assets/155306250/19c73ae2-4358-4382-bd15-ebcc7b87bcb7)

조회는 잘 되지만 생성하거나 삭제하는 명령어는 forbidden 되는 모습을 확인할 수 있다.

그렇다면 이제 `aws-auth`를 중단하게 되면 eks의 aws 권한과 k8s 권한을 어떻게 연결시켜주는 것일까?

---

# 2. EKS 액세스 항목

[23년 12월](https://aws.amazon.com/ko/about-aws/whats-new/2023/12/amazon-eks-controls-iam-cluster-access-management/), aws는 eks 액세스 항목에 대해 개선을 하게 된다.

`eks 액세스 항목`은 보다 간편하게 aws console ui 상에서 aws 권한과 k8s 권한을 연결할 수 있도록 구현한 항목이다.

eks 액세스 항목 테스트는 아래 2가지로 진행된다.

1. eks 액세스 항목에서 사전 정의된 항목으로 테스트.
2. clusterrole, clusterrolebinding, group으로 정의된 read-only를 eks 액세스 항목에서 테스트.

우선 테스트를 위해 `aws-auth`를 아래와 같이 정리해준다.

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::{ACCOUNT_ID}:role/{NODE_ROLE}
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - null
kind: ConfigMap
```

(초기상태로 돌려주면 된다.)
(**❗️ metadata는 불필요한 정보라 삭제함.**)

![5](https://github.com/may-30/may-30.github.io/assets/155306250/2fdccc99-6c02-474b-a3e7-d0e23f96ce63)

이후 kubectl 명령어로 아무런 동작을 취하면 위 사진과 같이 에러가 발생하게 되는데 정상이다.

## 2-1. eks 액세스 항목에서 사전 정의된 항목으로 권한 테스트

### 2-1-1. 사전 정의된 eks 액세스 항목 종류

![6](https://github.com/may-30/may-30.github.io/assets/155306250/87489425-82d4-4c86-ab00-f00fcefacd1d)

사전 정의된 eks 액세스 항목 종류는 총 5가지이며 각각의 세세한 항목들은 [링크](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/access-policies.html#access-policy-permissions)를 통해서 확인할 수 있다.

여기서 진행할 테스트는 `AmazonEksViewPolicy`이다.

### 2-1-2. eks 액세스 항목 생성

![7](https://github.com/may-30/may-30.github.io/assets/155306250/aecc1e0a-0205-4c4a-9e59-577b3d73f62f)

액세스 항목 생성으로 들어간다.

![8](https://github.com/may-30/may-30.github.io/assets/155306250/5beb1045-3598-4609-80a7-d07f3e6178af)

iam 보안 주체는 역할, 사용자를 선택할 수 있다.

다만, iam 사용자를 선택하면 유형은 표준으로 잡아주어야 한다.

![9](https://github.com/may-30/may-30.github.io/assets/155306250/b6fc3403-7fd1-4a95-b3c3-68f905e2247e)

사용자 이름의 경우에는 [eks 공식 문서](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/access-entries.html) 상 특별한 경우가 아니라면 별도로 지정하지 않을 것을 권장한다고 되어 있다.

![10](https://github.com/may-30/may-30.github.io/assets/155306250/4c28e58d-5cb4-46d1-8ddd-e59919cd6a38)

이번에는 `kube-system`에 대해서만 읽을 수 있는 권한을 생성한다.
(**❗️ 반드시 정책 추가 버튼을 클릭해야 한다.**)

![11](https://github.com/may-30/may-30.github.io/assets/155306250/d9831cb3-6d2f-4a04-b721-7bccbae4e235)

생성된 모습을 확인할 수 있다.

### 2-1-3. eks 액세스 항목 테스트

![12](https://github.com/may-30/may-30.github.io/assets/155306250/a3ddcd6b-2501-4889-bdb3-67ed301c48da)

모든 namespace에 대해서 조회하는 명령 - forbidden : 정상

kube-system namespace에 대해서 조회하는 명령 - ok : 정상

kube-system namespace에 대해서 삭제하는 명령 - forbidden : 정상

## 2-2. clusterrole, clusterrolebinding, group으로 정의된 read-only를 EKS 액세스 항목에서 테스트.

이번 테스트는 aws-auth와 동일한 결과를 예상하지만 aws-auth에서 행해야 할 작업을 aws console ui에서만 하는 테스트가 될 것이다.

(**❗️ 테스트를 시작하기에 앞서 기존에 만들어 둔 EKS 액세스 항목을 삭제해주어야 한다.**)

### 2-2-1. eks 액세스 항목 생성

![13](https://github.com/may-30/may-30.github.io/assets/155306250/a235f07f-973c-4f9a-b942-887cfd1d922e)

기존에 생성해두었던 `read-only` group을 연결한다.

![14](https://github.com/may-30/may-30.github.io/assets/155306250/1d4878c1-93be-4ae2-89e8-c4ae5ed561d0)

이번에는 중요한 점이 **"2단계 : 액세스 정책 추가"** 단계에서 아무런 정책을 기재하지 않는다는 점이다.

![15](https://github.com/may-30/may-30.github.io/assets/155306250/a6c34cdd-b22e-4790-9bac-147259d568da)

위 사진과 같이 정상적으로 생긴 것을 확인할 수 있다.

### 2-2-2. eks 액세스 항목 테스트

![16](https://github.com/may-30/may-30.github.io/assets/155306250/1b169c5d-5e3a-4721-a707-a69dd1458b70)

`aws-auth` 테스트와 동일한 결과가 출력되는 것을 확인할 수 있다.

# 3. 마치며

`aws-auth` 대신 eks 액세스 항목이 출시되었다는 소식은 오래 전부터 들었지만 실제 적용은 하지 않고 있고 **eks 1.30 버전**에서도 `aws-auth`를 사용 중이다.

확실히 eks 액세스 항목은 편리함이 뛰어나지만 직접 명령어와 수정을 터미널 상에서 수행하지 않아서 그런지 뭔가 자꾸 추상적인 작업인 것처럼 느껴진다.

아무튼 공식 문서 상에서도 더 이상 사용되지 않는다고 못을 박아놓은 상태이니 곧 사라지지 않을까 싶어서 포스팅해보았다.