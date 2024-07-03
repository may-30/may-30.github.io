---
title: "[EKS] eks nodegroup join (feat. NO nat gateway)"
categories:
  - eks
tags:
  - [private eks]
toc: true
toc_sticky: true
date: 2024-07-03
last_modified_at: 2024-07-03
---

![aws-eks](https://github.com/may-30/may-30.github.io/assets/155306250/9152a129-69af-4e72-8cd8-9f41524e0681){: .align-center}

# 0. 작성 취지

eks 구조를 조금 깊이 있게 보던 중 eks nodegroup join과 관련해서 궁금중이 생겼다.

**"eks nodegroup 서브넷 라우팅 정보에 `0.0.0.0/0 -> nat gateway`를 제거하면 무슨 일이 벌어질까?"**

eks nodegroup에는 보통 application이 동작하고 있다 보니 외부 통신이 이루어져야 하는 경우가 많아서 아무런 의심없이 `0.0.0.0/0 -> nat gateway`로 보내주었다.

하지만 보안 정책이 엄격한 회사라면 당연히 적용할 것이고 나처럼 테스트를 좋아하는 사람이라면 nat gateway 없이 eks nodegroup을 join에 대해 도전해보고 싶어졌다.

테스트는 아래 순서로 진행한다.

1. **"eks nodegroup 서브넷 라우팅 nat gateway 제거"**를 통해서 join이 정상적으로 이루어지는지 확인해본다.
2. **"TroubleShooting"**을 통해서 해결 과정을 본다.

# 1. eks nodegroup 서브넷 라우팅 nat gateway 제거

## 1-1. eks nodegroup 생성

![1](https://github.com/may-30/may-30.github.io/assets/155306250/0a9c244e-279f-45b1-88f8-6d4b66874bdb){: .align-center}

기존 `0.0.0.0/0 -> nat gateway`를 제거해서 오직 local로만 통신되도록 구성했다.

![2](https://github.com/may-30/may-30.github.io/assets/155306250/1619ce5f-2e22-469e-b842-41d57d05a1c4){: .align-center}

그리고 바로 eks nodegroup을 생성하고 오랜 시간 뒤에 생성 실패를 경험했다.

## 1-2. log 분석

eks nodegroup에서 사용되는 ami는 aws에서 자체적으로 제공해주는 eks에 특화된 ami(eks optimized ami)이며 버전 관리는 [링크](https://awslabs.github.io/amazon-eks-ami/)에서 확인할 수 있다.

우선 eks 관리형 서버이자 bastion 역할을 하는 서버로 들어가서 eks nodegroup에 ssh로 접속하여 `cloud-init-output.log`를 분석할 것이다.

`cloud-init-output.log`를 분석하는 이유는 eks optimized ami는 인스턴스가 시작되자마자 `cloud-init`을 실행하게 설정되어 있는데 그와 관련된 로그가 `cloud-init-output.log`로 출력되기 때문이다.

### 1-2-1. cloud-init-output.log 분석

![3](https://github.com/may-30/may-30.github.io/assets/155306250/bf2e483f-0bb6-4e5a-9e78-5b2d97926bb6){: .align-center}

eks 관리형 서버 -> eks nodegroup으로 ssh 접속해서 `cloud-init-output.log`를 확인한 결과이다.

`/etc/eks/bootstrap.sh`을 동작시키는 것을 알 수 있다.

(**❗️ 테스트 후 삭제된 eks cluster 입니다.**)

자세히 보면 빨간색 네모 박스에는 **"`ec2.ap-northeast-2.amazonaws.com`에 대한 엔드포인트를 찾을 수 없다."** 라고 써져 있다.

![4](https://github.com/may-30/may-30.github.io/assets/155306250/9444401f-3aac-461d-9a80-82534f500c1d){: .align-center}

그 이후에도 계속해서 `ec2.ap-northeast-2.amazonaws.com`에 대한 엔드포인트를 찾고 실패하고를 반복하는 로그를 확인할 수 있다.

그렇다면 `ec2.ap-northeast-2.amazonaws.com`은 어디에 있을까?

![5](https://github.com/may-30/may-30.github.io/assets/155306250/d8a3e65d-4f30-49ed-b0b0-4a6265917377){: .align-center}

```bash
curl -v https://ec2.ap-northeast-2.amazonaws.com
```

`nslookup` 명령어가 기본적으로 설치되어 있지 않기 때문에 `curl -v`로 검색했더니 공인 ip(`52.95.191.73`)으로 나오고 해당 공인 ip로 질의하려는 모습을 확인할 수 있다.

이렇게 되면 eks nodegroup 서브넷에는 `local` 라우팅 밖에 없기 때문에 당연히 `ec2 api`에 대한 호출을 할 수 없음으로 발생하는 에러이다.

### 1-2-2. /etc/eks 디렉터리 분석

그렇다면 `/etc/eks` 디렉터리에는 무엇이 존재할까?

![6](https://github.com/may-30/may-30.github.io/assets/155306250/c73a6a7c-867a-4375-ba90-c352005f71bd){: .align-center}

아주 다양한 파일과 디렉터리를 확인할 수 있는데 특히 눈에 띄는 것은 `bootstrap.sh`과 `get-ecr-uri.sh`이다.

### 1-2-3. /etc/eks/get-ecr-uri.sh 분석

![7](https://github.com/may-30/may-30.github.io/assets/155306250/b4721f68-3de0-4f78-b73c-bea53aa234d9){: .align-center}

파일 이름에서 유추할 수 있듯이 `ecr api`를 호출하는 것으로 보이는데 dkr로 호출하고 있음을 확인할 수 있다.

![8](https://github.com/may-30/may-30.github.io/assets/155306250/c9c54bdb-d8c4-448c-ae29-29a1745a856a){: .align-center}

```bash
curl -v https://602401143452.dkr.ecr.ap-northeast-2.amazonaws.com
```

마찬가지로 공인 ip가 출력되며 `dkr ecr api`에 대한 호출을 할 수 없을 것이라 에러가 발생할 예정이다.

(**❗️ 602401143452의 경우 ap-northeast-2이면 할당되는 aws 자체의 계정 account id 
입니다.**)

### 1-2-4. /etc/eks/bootstrap.sh 분석

![9](https://github.com/may-30/may-30.github.io/assets/155306250/31d53c46-2601-4757-b282-c3a0554c3014){: .align-center}

`get-ecr-uri.sh`을 동작시키는 부분을 확인할 수 있다.

![10](https://github.com/may-30/may-30.github.io/assets/155306250/32e2ad5c-c5cf-411d-b275-cf1da22de810){: .align-center}

`aws eks` 관련 명령어를 실행시키는 것을 확인할 수 있다.

![11](https://github.com/may-30/may-30.github.io/assets/155306250/6c81b570-8163-40c4-a768-9763bbd657ba){: .align-center}

```bash
curl -v https://eks.ap-northeast-2.amazonaws.com
```

마찬가지로 공인 ip가 출력되는 모습을 확인할 수 있다.

# 2. TroubleShooting

## 2-1. TroubleShooting #1

aws에서 공인 ip를 사용하지 않고 private 통신을 가능하게 하는 작업은 `vpc endpoint`를 생성하는 작업이다.

`vpc endpoint`는 동일한 host에 대해서 사설 통신을 할 수 있도록 별도의 통로를 만들어주는 서비스이다.

현재 필요하다고 파악된 `vpc endpoint`를 유추해보았을 때 총 3가지이다.

1. ec2 api
2. dkr ecr api
3. eks api

### 2-1-1. vpc endpoint 생성

![12](https://github.com/may-30/may-30.github.io/assets/155306250/fe43d228-375f-4646-b0cf-66154bfbec35){: .align-center}

`vpc endpoint`를 생성한다.

![13](https://github.com/may-30/may-30.github.io/assets/155306250/e7287333-be50-4964-a758-b70f63d3f3c3){: .align-center}

(**❗️ 참고로 현재 제가 구성한 eks cluster, nodegroup은 security group의 outbound까지 모두 제어하고 있습니다.**)

(**❗️ 만약 저와 같이 security group의 outbound까지 모두 제어하고 있다면 eks nodegroup security group의 outbound에 vpc endpoint의 security group을 추가해주어야 합니다.**)

### 2-1-2. vpc endpoint 테스트

![14](https://github.com/may-30/may-30.github.io/assets/155306250/914ece04-2cd3-4c1f-abd6-3cb5b9bea3e7){: .align-center}

위 사진에서 확인할 수 있듯이 사설 ip를 반환하는 것을 확인할 수 있다.

(**❗️ 사진은 dkr ecr endpoint만 확인했지만 ec2, eks 모두 사설 ip를 반환합니다.**)

### 2-1-3. eks nodegroup 재생성

사실 `/etc/eks/bootstrap.sh`만 재실행해도 문제는 없지만 보다 깔끔하게 해결하기 위해서 기존에 구성되어 있던 eks nodegroup을 삭제하고 다시 생성했다.

![15](https://github.com/may-30/may-30.github.io/assets/155306250/963d201c-3352-4343-a3d6-baa4e87f7df7){: .align-center}

![16](https://github.com/may-30/may-30.github.io/assets/155306250/2a34f5ec-8ae5-4c55-86dc-f3ca2f9e569c){: .align-center}

`/var/log/cloud-init-output.log` 자체는 에러가 발생하지 않지만 nodegroup의 핵심 `kubelet`은 dead 상태이며 join은 여전히 되지 않는다.

## 2-2. TroubleShooting #2

### 2-2-1. journalctl 분석

![17](https://github.com/may-30/may-30.github.io/assets/155306250/f42ddaaa-e528-4b76-aa02-69fe5347952b){: .align-center}

```bash
journalctl -xe --no-pager
```

`kubelet` 자체가 dead 상태를 확인하고 시스템의 로그를 확인해본다.

(**❗️ 많은 로그를 확인하기 위해서 `--no-pager` 옵션을 사용했습니다.**)

![18](https://github.com/may-30/may-30.github.io/assets/155306250/40e1c878-1e9d-4344-9921-d67a322be459){: .align-center}

`/etc/eks/containerd/pull-sandbox-image.sh`가 동작해야 한다는 점을 알게 되었다.

`api.ecr.ap-northeast-2.amazonaws.com`에 해당하는 `vpc endpoint`가 필요하다는 것을 알게 되었다.

### 2-2-2. 추가 vpc endpoint 생성

![19](https://github.com/may-30/may-30.github.io/assets/155306250/7bf94c53-47a2-4bc6-be8e-85713c979817){: .align-center}

(**❗️ 추가로 생성된 vpc endpoint에 대해서는 테스트를 별도로 진행했지만 기재하지 않았습니다.**)

### 2-2-3. eks nodegroup 재생성

![20](https://github.com/may-30/may-30.github.io/assets/155306250/252842d9-1849-4c70-9bb1-a280d717637b){: .align-center}

이번에도 마찬가지로 `kubelet`은 dead 상태이다.

## 2-3. TroubleShooting #3

### 2-3-1. journalctl 분석

![21](https://github.com/may-30/may-30.github.io/assets/155306250/ff328ccc-c3d5-4d12-9d2c-0903dd7a877f){: .align-center}

이번에는 `s3 vpc endpoint` 문제이다.

### 2-3-2. 추가 vpc endpoint 생성

s3의 vpc endpoint는 총 2가지 종류가 있다.

1. interface endpoint
2. gateway endpoint

![22](https://github.com/may-30/may-30.github.io/assets/155306250/c9803c2a-6986-49c7-a580-58408411785c){: .align-center}

`journalctl` 분석 결과에서 s3 호출이 interface endpoint 형태가 아니기 때문에 gateway endpoint를 생성해주어야 한다.

![23](https://github.com/may-30/may-30.github.io/assets/155306250/df645473-f291-48da-b9dd-1fa41ccb2160){: .align-center}

(**❗️ 참고로 현재 제가 구성한 eks cluster, nodegroup은 security group의 outbound까지 모두 제어하고 있습니다.**)

(**❗️ 만약 저와 같이 security group의 outbound까지 모두 제어하고 있다면 eks nodegroup security group의 outbound에 vpc endpoint의 security group을 추가해주어야 합니다.**)

### 2-3-3. eks nodegroup 재생성

![24](https://github.com/may-30/may-30.github.io/assets/155306250/7dba7505-eaa6-48e7-b7e8-5969a7078997){: .align-center}

드디어 정상적으로 eks nodegroup이 생성되었다.

![25](https://github.com/may-30/may-30.github.io/assets/155306250/07a97c87-c403-4702-9645-0bf5a42a89cf){: .align-center}

eks 관리 서버에서 kubectl을 입력하면 정상적으로 pod들이 running인 상태로 출력된다.

# 3. 마치며

이처럼 eks nodegroup을 nat gateway 없이 생성하는 것을 `eks fully private cluster` 라고 한다.

`eks fully private cluster`를 구성할 때 필요한 엔드포인트는 **5가지**이다.

1. eks (interface) - `/etc/eks/bootstrap.sh`에서 aws cli를 사용할 때 필요함.
2. ec2 (interface) - `aws-cloud-provider`와 통합하기 위해 필요함.
3. dkr.ecr (interface) - `ecr` image를 가져올 때 필요함.
4. api.ecr (interface) - `ecr` image를 가져올 때 필요함.
5. s3 (gateway) - `ecr` image를 가져올 때 필요함.