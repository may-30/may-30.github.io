---
title: '[EKS] Loki Stack 설치'
categories:
  - eks
tags:
  - [monitoring, loki stack]
toc: true
toc_sticky: true
date: 2024-07-30
last_modified_at: 2024-07-30
---

![aws-eks](https://github.com/user-attachments/assets/ca9e940e-4d8f-48e0-9734-21fea42e026d){: .align-center}

# 0. 작성 취지

EKS 환경에 Loki Stack을 helm으로 설치하는 과정을 작성하려고 한다.

설치하려고 하는 Loki Stack은 PLG 구조로 promtail, loki, grafana가 동시에 동작하는 형태이다.

![1](https://github.com/user-attachments/assets/24eff6c0-653e-4494-8d37-8422d9fc719f){: .align-center}

자세히 그림으로 나타내보면 위와 같이 구성되어 있으며 각각은 아래와 같이 서로 다르게 구성되어 있다.

- promtail : `daemonset`
- loki : `statefulset`
- grafana: `deployment`

이제 어떻게 구성할 수 있는지 본격적으로 실습을 하기 전에 테스트 해보면서 겪었던 중요한 사실들을 아래에 공유하려고 한다.

<div class="notice--primary" markdown="1">
Loki - EKS Pod Identity 사용하지 못 함.

- Loki는 S3를 저장소로 사용하고 있는데 이를 EKS Pod Identity로 해결하려 했으나 OIDC 밖에 적용되지 않는 것을 테스트로 확인함.

Grafana Image Tag - 가장 최근 버전 사용하지 못 함.

- Grafana에서 log volume으로 가시성 있게 보여주는 대시보드가 존재하는데 이는 가장 최근 이미지 버전을 사용하면 계속해서 에러를 뿜는 것을 테스트로 확인함.
</div>

Loki Stack 설치 순서는 아래와 같이 작성된다.

1. iam 단계
2. helm 단계
3. 외부 노출 단계

# 1. iam 단계

## 1-1. iam policy 생성

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::<<loki가 사용할 버킷명>>",
        "arn:aws:s3:::<<loki가 사용할 버킷명>>/*"
      ]
    }
  ]
}
```

위와 같이 iam policy를 생성해주면 된다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*"],
      "Resource": [
        "arn:aws:s3:::<<loki가 사용할 버킷명>>",
        "arn:aws:s3:::<<loki가 사용할 버킷명>>/*"
      ]
    }
  ]
}
```

경우에 따라서 위와 같이 s3에 대한 모든 권한을 부여해도 되지만 최소 권한 원칙에 위배되는 행동이기 때문에 테스트 목적에만 적용하는 것을 권장한다.

## 1-2. iam role 생성

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::{aws account 계정번호 12자리}:oidc-provider/oidc.eks.{리전 코드}.amazonaws.com/id/{oidc 고유 번호}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.{리전 코드}.amazonaws.com/id/{oidc 고유 번호}:aud": "sts.amazonaws.com",
          "oidc.eks.{리전 코드}.amazonaws.com/id/{oidc 고유 번호}:sub": "system:serviceaccount:{loki가 생성될 네임스페이스}:{loki가 사용할 serviceaccount 이름}"
        }
      }
    }
  ]
}
```

iam role 생성의 경우에는 전에 생성해 놓은 iam policy를 선택하고 `trust relationships`에 위와 같이 작성해두도록 한다.

# 2. helm 단계

Loki Stack의 경우에는 수정 및 추가되는 양이 너무 많아서 각 주체별로 나눠서 작성하도록 하겠다.

## 2-1. promtail

promtail은 추가하거나 변경한 부분은 없다.

## 2-2. loki

```yaml
loki:
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
  commonConfig:
    path_prefix: /var/loki
    replication_factor: 1
  compactor:
    shared_store: s3
    retention_enabled: true
    retention_delete_worker_count: 150
    apply_retention_interval: 1h
    compaction_interval: 5m
    working_directory: /data/retention
    delete_request_store: s3
  config:
    schema_config:
      configs:
        - from: 2024-07-31
          index:
            period: 24h
            prefix: loki_index_
          object_store: s3
          schema: v12
          store: tsdb
    storage_config:
      aws:
        region: ap-northeast-2
        bucketnames: # loki가 사용할 버킷명
        s3forcepathstyle: false
      tsdb_shipper:
        shared_store: s3
        active_index_directory: /data/tsdb-index
        cache_location: /data/tsdb-cache
  limit_config:
    retention_period: 720h
  serviceAccount:
    create: true
    name: # loki가 사용할 serviceaccount명
    annotations:
      eks.amazonaws.com/role-arn: # loki가 사용할 iam role arn
```

`node-type: mgmt`라는 node label이 있는 node에만 생성되도록 `nodeSelector`, `nodeAffinity` 설정을 걸어두었다.

양이 너무 방대해서 하나하나의 옵션은 참고했던 공식 홈페이지 링크로 대체한다.

- [tsdb](https://grafana.com/docs/loki/latest/operations/storage/tsdb/)
- [log retention](https://grafana.com/docs/loki/latest/operations/storage/retention/)

주의할 점으로 `loki.config.schema_config.configs[0].schema`의 경우 공식 홈페이지에는 `v13`으로 나와있지만 현재 날짜(24.07.30)로 테스트 해보면 **`invalid schema version`** 이라고 하면서 정상적으로 설치가 되지 않는다.

## 2-3. grafana

```yaml
grafana:
  enabled: true
  image:
    tag: 10.2.5
    users:
      default_theme: dark
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
```

`node-type: mgmt`라는 node label이 있는 node에만 생성되도록 `nodeSelector`, `nodeAffinity` 설정을 걸어두었다.

### 2-3-1. grafana.users.default_theme

기본적으로 grafana에서 사용하는 테마는 light로 설정되어 있는데 이를 dark 테마로 변경할 수 있다.

# 3. 외부 노출 단계

Loki Stack에서 외부로 노출해야 하는 Service는 `Grafana`이다.

## 3-1. Health Check 경로 변경

![2](https://github.com/user-attachments/assets/ab01dd58-85cd-4774-9601-1d8486a160e8){: .align-center}

`/api/health`로 변경했다.

![3](https://github.com/user-attachments/assets/c21e7769-6262-435c-824b-945c9b68fb1e){: .align-center}

생성된 Grafana의 Probe 설정 또한 모두 `/api.health`으로 설정되어 있는 것을 확인할 수 있다.

## 3-2. targetgroupbindings

```yaml
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: grafana-tgb
  namespace: monitoring
spec:
  serviceRef:
    name: loki-grafana
    port: 80
  targetGroupARN: # Target Group ARN
  vpcID: # VPC ID
```

targetgroupbinding을 통해서 Target Group과 K8s의 Service를 연결시켜주고 ALB에 등록하면 된다.

targetgroupbinding은 [링크](https://may-30.github.io/eks/aws-lb-controller-method-2/#1-2-ingress--alb-%EB%8F%85%EB%A6%BD%EC%A0%81%EC%9C%BC%EB%A1%9C-%EC%83%9D%EC%84%B1)에서 더 자세하게 볼 수 있다.

![4](https://github.com/user-attachments/assets/d0a57bc1-00f5-41f8-be07-c2bc7d6bc21d){: .align-center}

targetgroupbindings까지 생성을 완료하면 외부에서 정상적으로 접속할 수 있는 모습을 확인할 수 있다.

# 4. 마치며

Loki Stack을 helm으로 설치하면서 `nodeSelector`, `nodeAffinity`, `targetgroupbindings` 뿐만 아니라 Promtail, Loki, Grafana의 고유 설정까지도 활용해 보았다.

Loki Stack으로도 테스트 해보면서 helm 파일에 내용이 많이 추가될 것으로 보이지만 우선 설치까지는 무난하게 완료했다.
