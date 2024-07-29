---
title: '[EKS] Jenkins 설치'
categories:
  - eks
tags:
  - [cicd, jenkins]
toc: true
toc_sticky: true
date: 2024-07-29
last_modified_at: 2024-07-29
---

![aws-eks](https://github.com/user-attachments/assets/eba746ec-7f6e-406d-8050-8f6aa5152ee2){: .align-center}

# 0. 작성 취지

EKS 환경에 Jenkins를 helm으로 설치하는 과정을 작성하려고 한다.

설치하려고 하는 Jenkins는 controller(master), agent(slave) 구조로 설치하려고 한다.

해당 구조의 장점은 Jenkins의 Job이 실행될 때마다 Job 별로 Jenkins agent pod가 생성되어 controller에 부담을 주지 않는 구조이다.

뿐만 아니라 Job이 종료되면 Jenkins agent 또한 종료되기 때문에 불필요한 리소스를 줄일 수 있다는 장점이 있다.

해당 글의 핵심은 `nodeSelector`, `nodeAffinity`, `targetgroupbindings`이 될 것 같다.

순서는 아래와 같이 작성된다.

1. 스토리지 단계 (선택사항)
2. helm 단계
3. 외부 노출 단계

# 1. 스토리지 단계 (선택사항)

Jenkins의 경우에는 스토리지가 선택 사항으로 빠져도 되지만 나의 경우에는 `workspace` 경로에 존재하는 프로젝트들을 전부 보관하고 싶어서 스토리지를 생성하려고 한다.

기본적으로 helm values 파일에서 Jenkins의 스토리지를 직접 생성해주지 않는다.

그래서 EBS volume을 Static Provisioning으로 생성해주었지만 EFS로 생성해도 무방하다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: jenkins
spec:
  storageClassName: ''
  volumeName: jenkins-pv
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 50Gi
  csi:
    driver: ebs.csi.aws.com
    fsType: ext4
    volumeHandle: # EBS Volume ID
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.ebs.csi.aws.com/zone
              operator: In
              values:
                - ap-northeast-2a
```

Static Provisioning에 대한 실습 내용은 [링크](https://may-30.github.io/eks/amazon-csi-driver/#2-1-%EC%A0%95%EC%A0%81-%ED%94%84%EB%A1%9C%EB%B9%84%EC%A0%80%EB%8B%9D)에 존재하니 참고하면 좋을 듯 하다.

# 2. helm 단계

```bash
helm repo add jenkins https://charts.jenkins.io/

helm show values jenkins/jenkins > jenkins.yaml
```

위 명령어로 Jenkins helm 단계 준비를 한다.

```yaml
controller:
  jenkinsUrl: 'https://jenkins.may30.xyz'
  nodeSelector:
    node-type: mgmt
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-type
                operator: In
                values:
                  - mgmt
agent:
  volumes:
    - type: PVC
      claimName: jenkins-pvc
      mountPath: /var/jenkins_home
      readOnly: false
  nodeSelector:
    node-type: mgmt
  additionalContainers:
    - sideContainerName: dind
      image:
        repository: bentolor/docker-dind-awscli
        tag: dind
      command: dockerd-entrypoint.sh
      args: ''
      privileged: true
      resources:
        requests:
          cpu: 500m
          memory: 1Gi
        limits:
          cpu: 1
          memory: 2Gi
persistence:
  existingClaim: 'jenkins-pvc'
  size: '50Gi'
```

## 2-1. controller 부분

### 2-1-1. jenkinsUrl

해당 부분을 기입하지 않으면 Jenkins를 띄워서 웹으로 접근했을 때 **"역방향 프록시 설정이 잘못되었습니다."** 라는 문구가 출력된다.

![1](https://github.com/user-attachments/assets/718b039c-a8bb-4590-8fff-dd01be9f9878){: .align-center}

그 이유는 위 사진처럼 Jenkins URL에 접속한 도메인이 작성되어 있어야 하는데 helm 기본 values로 설치하면 기본값이 들어가서 서로 다른 정보를 가리키고 있기 떄문이다.

### 2-1-2. nodeSelector, affinity

현재 하고 있는 테스트는 node에 label을 적용해서 해당 node에만 생성되도록 설정해두는 것이다.

때문에 key-value가 `node-type` - `mgmt`로 되어 있는 node에 Jenkins가 생성되도록 설정해두었다.

## 2-2. agent 부분

### 2-2-1. volumes

Job이 실행될 때마다 생성되는 Jenkins agent에서 생성한 프로젝트 별로 workspace가 생성되는데 그걸 보관하기 위해 EBS volume을 연결한다.

### 2-2-2. nodeSelector

현재 하고 있는 테스트는 node에 label을 적용해서 해당 node에만 생성되도록 설정해두는 것이다.

때문에 key-value가 `node-type` - `mgmt`로 되어 있는 node에 Jenkins가 생성되도록 설정해두었다.

### 2-2-3. additionalContainers

K8s에서 1.20 버전 이상부터 docker의 컨테이너 런타임 지원 중단에 따라 Jenkins 또한 Jenkins Pod에서 직접 docker 런타임을 사용할 수 없게 되었다.

그래서 DinD(Docker in Docker) 방식으로 적용해야하며 사용한 이미지는 추후 ECR push 까지 고려한 docker image이다.

## 2-3. persistence 부분

사전에 생성해두었던 EBS volume을 Jenkins controller에서 사용할 수 있도록 선언해준다.

정확하게는 Jenkins agent가 workspace를 만드는 것이기 떄문에 해당 workspace를 controller에서도 지속적으로 가지고 있게끔 구성한다.

# 3. 외부 노출 단계

## 3-1. Health Check 경로 변경

![2](https://github.com/user-attachments/assets/ea210ec7-7c0d-45cc-86f2-1d1514c238d5){: .align-center}

`/login`으로 변경해주었으며 targetgroup의 기본값인 `/` 으로 설정해두면 403 Forbidden이 발생한다.

![3](https://github.com/user-attachments/assets/de876852-3dcb-4696-ab52-2cfa483dad10){: .align-center}

생성된 Jenkins의 Probe 설정 또한 모두 `/login`으로 설정되어 있는 것을 확인할 수 있다.

## 3-2. targetgroupbindings

```yaml
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: jenkins-tgb
  namespace: jenkins
spec:
  serviceRef:
    name: jenkins
    port: 8080
  targetGroupARN: # TARGETGROUP ARN
  vpcID: # VPC ID
```

targetgroupbinding을 통해서 Target Group과 K8s의 Service를 연결시켜주고 ALB에 등록하면 된다.

targetgroupbinding은 [링크](https://may-30.github.io/eks/aws-lb-controller-method-2/#1-2-ingress--alb-%EB%8F%85%EB%A6%BD%EC%A0%81%EC%9C%BC%EB%A1%9C-%EC%83%9D%EC%84%B1)에서 더 자세하게 볼 수 있다.

![4](https://github.com/user-attachments/assets/e3f13104-7ca3-444c-b6cb-96169d7853bb){: .align-center}

targetgroupbindings까지 생성을 완료하면 외부에서 정상적으로 접속할 수 있는 모습을 확인할 수 있다.

# 4. 마치며

Jenkins를 helm으로 설치하면서 `nodeSelector`, `nodeAffinity`, `targetgroupbindings`을 활용해 보았다.

Jenkins로도 테스트 해보면서 helm 파일에 내용이 많이 추가될 것으로 보이지만 우선 설치까지는 무난하게 완료했다.