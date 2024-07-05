---
title: "[EKS] aws load balancer controller를 활용한 ingress 생성 방안 2가지"
categories:
  - eks
tags:
  - [aws load balancer controller]
toc: true
toc_sticky: true
date: 2024-07-05
last_modified_at: 2024-07-05
---

![aws-eks](https://github.com/may-30/may-30.github.io/assets/155306250/9152a129-69af-4e72-8cd8-9f41524e0681){: .align-center}

# 0. 작성 취지

`aws load balancer controller`를 사용해서 ingress에 annotations를 기입해서 생성하면 alb가 생성되고, service로 annotations를 기입해서 생성하면 nlb가 생성된다.

나도 여태까지는 이러한 구성이 전혀 문제가 없다고 생각했다.

하지만 aws summit seoul 2024에 직접 참여해서 [GS 리테일 Amazon EKS의 모든 것: 무중단 운영부터 배포 자동화까지](https://kr-resources.awscloud.com/aws-summit-seoul-2024-day-1-track-6)를 듣고 나니까 **"k8s의 자원인 ingress, service를 무조건 alb, nlb와 연결시키는 것이 마냥 좋은 것은 아니구나"** 라는 깨달음을 얻게 되었다.

그 이유는 eks 무중단 업그레이드 때문이다.

eks 버전 업그레이드는 3개월마다 자주 있는 이벤트이지만 자주 있는 이벤트임에도 불구하고 업그레이드 난이도는 경우에 따라서 최상이다.

특히 24시간 365일 장애 없이 동작해야 하는 경우에는 더더욱이 그렇다.

기존에는 `route53`의 기중치 기반을 활용해서 eks blue/green 버전 업그레이드를 진행했더라면 이제는 `alb` 자체를 독립적으로 두기 때문에 `alb listener` 가중치 기반으로 eks blue/green 버전 업그레이드를 진행할 수 있게 된다는 소리이다.

깨달음을 얻은 김에 직접 테스트 해보고 적용해보려고 한다.

# 1. 테스트

(**❗️ 사전 조건으로 `aws load balancer controller`가 반드시 설치되어 있어야 합니다.**)

## 1-1. ingress + alb 종속적으로 생성

해당 테스트는 ingress를 생성하면 alb가 자동으로 생성되는 테스트이다.

### 1-1-1. k8s 자원 생성

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: test
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: test
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: test
  annotations:
    alb.ingress.kubernetes.io/load-balancer-name: ALB_NAME
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/subnets: "SUBNET_NAME, SUBNET_NAME"
    alb.ingress.kubernetes.io/security-groups: SECURITY_GROUP_ID
    alb.ingress.kubernetes.io/tags: Name=ALB_NAME
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: ACM_ARN
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
```

`deployment`, `service`, `ingress` 모두 생성되는 예제이다.

(**❗️ namespace는 test에 생성되는 예제입니다.**)

![1](https://github.com/may-30/may-30.github.io/assets/155306250/3e3ad72a-bf3e-48a1-9d85-121eaa6a7be7){: .align-center}

실제 프로비저닝이 정상적으로 이루어진 모습을 확인할 수 있다.

### 1-1-2. alb 자원 확인

![2](https://github.com/may-30/may-30.github.io/assets/155306250/7487b4c0-472f-4447-b069-3c4389c4461a){: .align-center}

실제 alb가 정상적으로 생성되어 있는 화면이다.

사진 하단에 나와있는 `resource map`에서도 `target group`이 모두 `healthy`로 출력되는 것을 확인할 수 있다.

### 1-1-4. 도메인 등록

![3](https://github.com/may-30/may-30.github.io/assets/155306250/571754e8-068b-404e-b51c-f5f5ebdf567e){: .align-center}

route53에 CNAME으로 등록해준다.

![4](https://github.com/may-30/may-30.github.io/assets/155306250/0d13d241-2280-4cee-aff7-0558608b8b2d){: .align-center}

(**❗️ [링크](https://docs.aws.amazon.com/ko_kr/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)를 참고하면 alb와 route53이 같은 aws account라면 a 레코드로 등록하는 것이 비용 측면에서 유리합니다.**)

### 1-1-3. 동작 확인

![5](https://github.com/may-30/may-30.github.io/assets/155306250/140a14ab-bef5-4135-8cbd-5455bb9bf7c0){: .align-center}

```bash
curl -I -L http://alb-integrated.may30.xyz
```

위 명령어를 통해서 정상적인 redirect가 되었는지 확인할 수 있고 웹 브라우저에 도메인을 입력하면 자동으로 http -> https redirect을 확인할 수 있다.

다음 테스트를 위해서 리소스를 정리해준다.

## 1-2. ingress / alb 독립적으로 생성

(**❗️ 사전 조건으로 `aws load balancer controller`가 반드시 설치되어 있어야 합니다.**)

### 1-2-1. alb 자원 생성

![6](https://github.com/may-30/may-30.github.io/assets/155306250/746347a0-a470-484b-8071-4626959c18dd){: .align-center}

위 사진과 같이 alb를 직접 console에서 생성해준다.

(**❗️ 테스트 후 삭제된 alb입니다.**)

![7](https://github.com/may-30/may-30.github.io/assets/155306250/74db0caa-fb0c-4275-aed4-4bd926628c32){: .align-center}

(**❗️ alb를 독립적으로 생성할 시 target group은 임시로 생성해서 바인딩해주어야 하며 alb 생성 완료 후 삭제해주시면 됩니다.**)

### 1-2-2. k8s 자원 생성

![8](https://github.com/may-30/may-30.github.io/assets/155306250/30938cdb-bb27-4058-9310-7476b29fa955){: .align-center}

우선 위 사진과 같이 비어있는 target group을 생성해주어야 한다.

(**❗️ protocol, port가 있는 빨간색 네모 박스 확인할 수 있듯이 실제 alb에서 listener에 연결될 port를 기입한다.**)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: test
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: test
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: nginx-tgb
  namespace: test
spec:
  serviceRef:
    name: nginx-svc
    port: 80
  targetGroupARN: TARGETGROUP_ARN
```

![9](https://github.com/may-30/may-30.github.io/assets/155306250/ebda52a8-a1f2-432c-9db9-5f1244abc250){: .align-center}

![10](https://github.com/may-30/may-30.github.io/assets/155306250/56bbdb19-4270-4119-94d3-0831f4b3472d){: .align-center}

비어있던 target group에 성공적으로 연결된 모습을 확인할 수 있다.

### 1-2-3. alb redirect listener 추가

![11](https://github.com/may-30/may-30.github.io/assets/155306250/5fa485bd-190d-49cc-a82b-a696b23e71b9){: .align-center}

http -> https redirect 기능 구현을 위해서 listener를 추가한다.

### 1-2-4. 도메인 등록

![12](https://github.com/may-30/may-30.github.io/assets/155306250/b8b1c797-72bd-4643-93ae-d0bc281ba441){: .align-center}

마찬가지로 route53에 CNAME으로 등록해준다.

### 1-2-5. 동작 확인

![13](https://github.com/may-30/may-30.github.io/assets/155306250/3d0ebbd2-36d9-490b-9d6e-7067c4a9a61e){: .align-center}

```bash
curl -I -L http://alb-separated.may30.xyz
```

정상적으로 redirect가 되는 모습을 확인할 수 있다.

# 2. 마치며

ingress와 alb 자원을 분리시키는 테스트를 해보았다.

ingress와 alb 자원을 분리시키면 eks 버전 업그레이드에 상당한 장점으로 작용될 것으로 보인다.

왜 그렇게 생각하는지에 대해서 조만간 eks 버전 업그레이드의 방안 내용을 기술한 블로그를 올리도록 하겠다.