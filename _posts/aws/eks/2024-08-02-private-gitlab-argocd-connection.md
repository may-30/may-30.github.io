---
title: '[EKS] Private Gitlab과 ArgoCD 연동'
categories:
  - eks
tags:
  - [cicd, gitlab, argocd]
toc: true
toc_sticky: true
date: 2024-08-02
last_modified_at: 2024-08-02
---

![eks](https://github.com/user-attachments/assets/186679d7-82b9-41a6-b8cf-c60331dbcfbc){: .align-center}

# 0. 작성 취지

Private Gitlab과 ArgoCD 연동하는 테스트를 하려고 한다.

보통 Private Gitlab과 ArgoCD 연동을 ssh key 방식으로 연결하는 것으로 많이 보이는데 해당 방식이 나에겐 적합하지 않아보였다.

**"22번 포트가 열려야 하고 해당 포트에 대한 로드밸런서의 리스너를 생성해주어야 하는데 과연 적합한 방식일까?"** 하는 생각이 들었다.

(**❗️ 물론 22번 포트는 포트 포워딩을 통해서 다른 포트로 오픈할 수 있습니다.**)

그래서 테스트를 하면서 찾아보다가 HTTPS로도 연동할 수 있는 방법을 찾게되었다.

가장 중요한 것은 아래와 같다.

<div class="notice--primary" markdown="1">

Private Gitlab의 Deploy Token 생성

- Personal Access Token, Project Access Token이 아닌 Deploy Token이라는 점을 인지해야 한다.

Route53 Private Hosted Zone 생성

- ArgoCD에서 Gitlab의 도메인을 찾을 수 있도록 사설 도메인을 만들어주어야 한다.

</div>

테스트 실습은 아래와 같이 진행하며 Gitlab과 ArgoCD는 사전에 설치되어 있는 것으로 가정한다.

1. Private Gitlab Deploy Token 생성
2. Private Load Balancer 생성 및 연동
3. Route53 Private Hosted Zone 생성
4. ArgoCD 연동

# 1. Private Gitlab Deploy Token 생성

![1](https://github.com/user-attachments/assets/5cad5577-4d54-4a6b-9976-b40854e21a88){: .align-center}

Gitlab에서 생성된 프로젝트에서 **`Settings` > `Repository` > `Deploy tokens`** 순으로 들어간다.

![2](https://github.com/user-attachments/assets/a300bb6c-d562-475b-be41-6325a9cb64bb){: .align-center}

Deploy tokens에서 위 사진과 같이 설정해주면 되는데 항목은 각각 아래와 같다.

|      항목       |                                                   설명                                                    |
| :-------------: | :-------------------------------------------------------------------------------------------------------: |
|      Name       |                                   Gitlab에서 표시되는 Deploy token 이름                                   |
| Expiration date |                         Deploy token 만료 날짜 (기입하지 않으면 무제한으로 설정)                          |
|    Username     | ArgoCD에서 기입할 username 이름 (설정하지 않으면 기본적으로 `gitlab+deploy-token-{n}` 형식으로 자동 생성) |

Deploy token은 권한 자체가 `read_repository` 권한 밖에 없지만 혹시 모를 노출 사고를 대비해서 만료 날짜를 설정해주는 것이 좋지만 테스트이기 때문에 작성하지 않았다.

![3](https://github.com/user-attachments/assets/a23e159a-f9ea-491b-9ca9-2fb13d36df2a){: .align-center}

정상적으로 생성이 완료된 모습을 확인할 수 있으며 `Expires` 영역은 **`Never`**로 설정되어 있는 모습을 확인할 수 있다.

# 2. Private Load Balancer 생성 및 연동

Private Load Balancer를 생성하는 이유는 Route 53 Private Hosted Zone에 매핑시켜주기 위함이다.

이유는 ArgoCD 자체에서 Gitlab이 사용 중인 도메인인 `gitlab.may30.xyz` 자체를 모르기 때문이다.

![4](https://github.com/user-attachments/assets/4cf6c80b-5085-4efd-875d-e303d7bc54a9){: .align-center}

우와 같이 Internal인 Load Balancer를 생성한다.

```yaml
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: gitlab-pri-tgb
  namespace: gitlab
spec:
  serviceRef:
    name: gitlab-webservice-default
    port: 8181
  targetGroupARN: # TARGET GROUP ARN
  vpcID: # VPC ID
```

나의 경우에는 ALB를 분리해서 Target Group으로 바인딩하는 것을 고수하고 있기 떄문에 위와 같은 yaml 파일로 `targetgroupbindings`를 생성한다.

![5](https://github.com/user-attachments/assets/161274bc-9937-4107-a08b-6f1f3906244e){: .align-center}

참고로 내가 생성해놓은 모든 `targetgroupbindings`이다.

# 3. Route53 Private Hosted Zone 생성

Private Load Balancer로 생성해주더라도 아직 ArgoCD는 `gitlab.may30.xyz`라는 도메인을 찾을 수 없다.

그렇기 떄문에 Route53의 Private Hosted Zone을 생성해서 해당 도메인에 대해 Private Load Balancer를 연결해준다.

![6](https://github.com/user-attachments/assets/a6774db0-b42c-426e-aa6c-b08d32706c1a){: .align-center}

위 사진과 같이 최종 모습으로 생성해주면 된다.

# 4. ArgoCD 연동

이제 ArgoCD에서 연동함으로써 정상적으로 동작하는지에 대한 증명을 할 시간이다.

![7](https://github.com/user-attachments/assets/12eba6b5-1a77-4c35-99ef-60aea7268169){: .align-center}

위 사진과 같이 ArgoCD에 접속해서 **`Settings` > `Connect Repo`**로 들어간다.

![8](https://github.com/user-attachments/assets/1a46c7ff-ee12-46a0-8a85-ee3776cc29af){: .align-center}

Connection method는 `HTTPS`로 맞춰준다.

Repository URL의 경우 `Gitlab에 만들어놓은 프로젝트의 URL`로 작성한다.

Username, Password의 경우 `Gitlab에서 생성한 Deploy tokens`로 작성한다.

![9](https://github.com/user-attachments/assets/338aaffa-c965-47e4-a4e4-bde76918fe7c){: .align-center}

성공적으로 연동이 된 모습을 확인할 수 있다.

# 5. 마치며

Private Gitlab과 ArgoCD와 연동은 성공적으로 마쳤다.

이제 연결된 Gitlab으로 ArgoCD Application을 생성하면 변경점에 대해서 자동으로 Sync를 맞출 수 있을 것이다.
