---
title: "[EKS] amazon csi driver 역할과 테스트"
categories:
  - eks
tags:
  - [csi driver]
toc: true
toc_sticky: true
date: 2024-07-04
last_modified_at: 2024-07-04
---

![aws-eks](https://github.com/may-30/may-30.github.io/assets/155306250/9152a129-69af-4e72-8cd8-9f41524e0681){: .align-center}

# 0. 작성 취지

eks에서 스토리지와 연동하려면 설치해야 하는 `amazon csi driver`에 대해 알아보려고 한다.

`amazon csi driver`는 크게 정적 프로비저닝과 동적 프로비저닝이 존재하는데 각각 테스트 또한 진행하려고 한다.

# 1. amazon csi driver 동작 원리

## 1-1. csi는 무엇인가?

csi는 `container storage interface`의 약자이다.

csi가 필요한 이유는 k8s의 역사를 조금 알아야 하는데 바로 밑에서 얘기하겠다.

### 1-1-1. k8s csi 도입 전

csi 도입 전의 k8s는 직접 스토리지를 관리할 수 있도록 `storage provisioner`를 k8s 자체적으로 가지고 있었다.

하지만 이 방식의 최대 단점은 바로 다양한 storage vender 회사의 기능이 업데이트 될 때마다 k8s에서도 업데이트 된 기능을 맞추어야 하기 때문에 자체 `storage provisioner`를 수정해서 k8s 자체 버전을 업데이트 해야 한다는 점이었다.

aws만 하더라도 ebs, efs, fsx, s3까지 총 4가지인데 azure, gcp 등 cloud vender 뿐만 아니라 다양한 storage vender 회사까지 각기 다르게 업데이트를 한다면 얼마나 자주 k8s의 버전이 릴리즈 되겠는가?

이러한 점을 해결하고자 고안된 것이 csi이다.

### 1-1-2. k8s csi 도입 후

csi는 여러 storage vender가 컨테이너 오케스트레이션 시스템에서 동작할 수 있도록 k8s와 스토리지 사이에 위치한 표준 인터페이스를 의미한다.

## 1-2. amazon csi driver

amazon csi driver는 eks가 아래 aws storage와 연동을 진행할 때 필요한 addon이다.

1. 블록 스토리지 : EBS (amazon ebs csi driver)
2. 파일 스토리지 : EFS (amazon efs csi driver), FSx (amazon fsx csi driver)
3. 객체 스토리지 : S3 (mountpoint for amazon s3 csi driver)

amazon csi driver를 설치하면 아래 사진의 빨간색 네모 박스와 같이 핵심 컴포넌트 2가지가 설치된다.

![1](https://github.com/may-30/may-30.github.io/assets/155306250/482c0074-50f5-4963-805f-5e6453339613){: .align-center}

> 사진 출처 : https://www.oreilly.com/library/view/production-kubernetes/9781492092292/ch04.html

1. csi controller
2. csi node

### 1-2-1. csi controller

csi controller는 `deployment` 형태로 동작한다.

csi controller는 `kube-apisever`를 지속적으로 바라보고 있다가 볼륨 관련 요청(`storage class`, `persistent volume claim`, `persistent volume`)이 발생하면 `aws api`를 통해서 aws storage를 생성하는 역할을 하게 된다.

(**❗️ kube-apiserver는 kubectl 명령어 뿐만 아니라 k8s의 모든 컴포넌트들이 다른 컴포넌트들과 통신하기 위해 무조건 거쳐야 하는 창구입니다.**)

csi controller가 `aws api`를 통해서 aws storage를 어떻게 생성하는 것인지에 대해 궁금할 수 있는데 3가지를 중점적으로 살펴보면 된다.

1. csi controller deployment 혹은 pod에 선언되어 있는 service account 확인

![2](https://github.com/may-30/may-30.github.io/assets/155306250/aa98bfbf-d99f-49fe-926d-2cdc12cf9c23){: .align-center}

2. csi controller가 사용하고 있는 service account의 annotations

![3](https://github.com/may-30/may-30.github.io/assets/155306250/a8a46d09-ff14-4d67-bd73-f350234986f5){: .align-center}

3. annotations에 적혀 있는 iam role과 iam policy

![4](https://github.com/may-30/may-30.github.io/assets/155306250/1b117c41-a58c-4813-97e2-f902b54d0bf8){: .align-center}

![5](https://github.com/may-30/may-30.github.io/assets/155306250/c1a42a42-6815-4894-88b1-4a2cfb8b6183){: .align-center}

### 1-2-2. csi node

csi node는 각 node에 1개씩 설치되는 `daemonset` 형태로 동작한다.

csi node는 csi controller의 `aws api`를 통해 생성된 aws storage를 직접 pod에 마운트하여 연결하는 작업을 하게 된다.

# 2. amazon csi driver 프로비저닝

## 2-1. 정적 프로비저닝

정적 프로비저닝에 필요한 컴포넌트는 `ebs volume`, `persistent volume claim`, `persistent volume` 3가지이다.

### 2-1-1. ebs volume 생성

![6](https://github.com/may-30/may-30.github.io/assets/155306250/b65a7811-aac9-44a6-82ae-62b98df91cf9){: .align-center}

(**❗️ 테스트 후 삭제된 ebs volume입니다.**)

### 2-1-2. storage.yaml 파일 생성 및 리뷰

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-eks-static-pvc
spec:
  storageClassName: ""
  volumeName: test-eks-static-pv
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-eks-static-pv
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Gi
  csi:
    driver: ebs.csi.aws.com
    fsType: ext4
    volumeHandle: vol-04227c5e0580b1cce
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.ebs.csi.aws.com/zone
              operator: In
              values:
                - ap-northeast-2a
```

`nodeSelectorTerms`의 zone 부분은 [링크](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/examples/kubernetes/static-provisioning/README.md#usage)에서 ebs volume이 생성되어 있는 가용 영역을 작성하라고 되어 있다.

### 2-1-3. pod 생성 리뷰

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-eks-static-ebs-app
spec:
  containers:
  - name: app
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 5; done"]
    volumeMounts:
    - name: test-eks-static-ebs-mount
      mountPath: /data
  volumes:
  - name: test-eks-static-ebs-mount
    persistentVolumeClaim:
      claimName: test-eks-static-pvc
```

### 2-1-4. 확인

![7](https://github.com/may-30/may-30.github.io/assets/155306250/d7d9e95a-a99f-4db2-a526-0581ba1b2c1a){: .align-center}

`pvc`와 `pv`가 `Bound`로 마운트 되어 있는 모습을 확인할 수 있다.

![8](https://github.com/may-30/may-30.github.io/assets/155306250/b2a50c50-22c0-47be-a3da-b50dda69cbda){: .align-center}

직접 pod에 접속해보면 `/data` 디렉터리에 out.txt로 잘 쌓이는 모습을 확인할 수 있다.

### 2-1-5. 사이즈 확장 테스트

![9](https://github.com/may-30/may-30.github.io/assets/155306250/7fa7e297-d126-43e2-9dfb-08528e68644c){: .align-center}

정적 프로비저닝으로 생성한 ebs는 사이즈 확장이 불가능하며 오직 동적 프로비저닝만 가능하다는 문구가 출력된다.

## 2-2. 동적 프로비저닝

동적 프로비저닝에 필요한 컴포넌트는 `storage class`, `persistent volume claim` 2가지 이다.

### 2-2-1. storage.yaml 파일 생성 및 리뷰

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  csi.storage.k8s.io/fstype: xfs
  type: gp3
  iopsPerGB: '3000'
  encrypted: 'true'
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-eks-dynamic-pvc
spec:
  storageClassName: 'gp3'
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

`volumeBindingMode: WatiForFirstConsumer`: pv 생성 요청이 들어오기 전까지 pv를 생성하지 않겠다는 옵션이다.

`allowVolumeExpansion: true` : 동적 프로비저닝으로 생성된 ebs volume에 한하여 자동으로 용량을 증설시키겠다는 옵션이다.

### 2-2-2. pod 생성 및 리뷰

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-eks-dynamic-ebs-app
spec:
  containers:
  - name: app
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 5; done"]
    volumeMounts:
    - name: test-eks-dynamic-ebs-mount
      mountPath: /data
  volumes:
  - name: test-eks-dynamic-ebs-mount
    persistentVolumeClaim:
      claimName: test-eks-dynamic-pvc
```

### 2-2-3. 확인

![10](https://github.com/may-30/may-30.github.io/assets/155306250/a044a1fd-bab2-4d8d-acf6-cc5bd61dc8b8){: .align-center}

`storage.yaml` 파일만 apply 시킨 경우에는 `pv`가 별도로 매핑되지 않기 떄문에 `pvc`가 무한적인 `Pending` 상태에 빠지게 된다.

![11](https://github.com/may-30/may-30.github.io/assets/155306250/1ab11a1a-ad50-4ff5-89b7-e83748342c1c){: .align-center}

`pod` 생성 후에 자동으로 `pv`가 `Bound`된 모습을 확인할 수 있다.

![12](https://github.com/may-30/may-30.github.io/assets/155306250/3c3b2a6f-3290-404d-8b73-a269ed3798f8){: .align-center}

정적 프로비저닝과 마찬가지로 정상적으로 쌓이는 모습을 확인할 수 있다.

### 2-2-4. 사이즈 확장 테스트

![13](https://github.com/may-30/may-30.github.io/assets/155306250/2c21ca1c-8160-407f-ab57-3bf029682211){: .align-center}

![14](https://github.com/may-30/may-30.github.io/assets/155306250/4abe33ea-eded-441d-bac9-221ac1033de4){: .align-center}

`ebs csi controller` 안에 있는 컨테이너들 중 하나인 resize를 담당하는 컨테이너인 `csi-resizer`의 로그를 분석한다.

로그를 분석하면 정상적으로 사이즈를 증설했다는 이벤트를 확인할 수 있다.

(**❗️ 참고로 ebs의 경우 사이즈 확장은 되지만 축소는 불가합니다.**)

![15](https://github.com/may-30/may-30.github.io/assets/155306250/42ceff4c-d494-44e5-b092-a0ea82a79839){: .align-center}

![16](https://github.com/may-30/may-30.github.io/assets/155306250/fe238853-c919-49d0-a5a6-bb4310a9575a){: .align-center}

기존 10GiB에서 20GiB로 증설된 모습을 확인할 수 있다.

(**❗️ 테스트 후 삭제된 ebs volume입니다.**)

# 3. 마치며

실제 테스트는 ebs로만 했지만 efs, s3까지 모두 가능하다.

동적 프로비저닝이 실제 스토리지를 수동으로 생성하지 않아도 된다는 장점이 있지만 프로젝트를 경험하게 되면 장점 보단 단점으로 다가오게 된다.

이유는 동적 프로비저닝의 경우에는 pvc와 pv에 따라서 개별적인 디렉터리를 마운트하기 때문이다.

예를 들어 image 파일과 같은 정적 파일을 동일한 디렉토리를 마운트해야 하는 efs의 경우에는 동적 프로비저닝의 경우 서로 다른 디렉터리를 마운트해서 불러오지 못하는 경우가 있기 때문에 정적 프로비저닝을 선호하게 되는 경험을 할 수 있다.