---
title: "[EKS] ingress nginx를 활용한 redirect 처리 방법"
categories:
  - eks
tags:
  - [ingress nginx]
toc: true
toc_sticky: true
date: 2024-07-08
last_modified_at: 2024-07-08
---

![aws-eks](https://github.com/may-30/may-30.github.io/assets/155306250/9152a129-69af-4e72-8cd8-9f41524e0681){: .align-center}

# 0. 작성 취지

ingress nginx controller를 사용하면서 redirect를 구현하면서 예상하지 못했던 문제점들을 겪게 되었다.

많은 문제점들 중에서 제대로 동작하는 아래 2가지의 케이스를 공유할 것이다.

1. 오직 443 포트로만 들어오는 ingress nginx
2. 사설 인증서를 tls secret로 등록해서 사용하는 ingress nginx

<div class="notice--primary" markdown="1">
그리고 가장 중요한 점은 단순하게 ingress nginx controller만 설치하면 nlb의 속성을 전부 활용하지 못한다.

따라서 aws load balancer controller를 설치하면 같이 생성되는 crd를 활용해야 한다.

[링크](https://github.com/kubernetes/ingress-nginx/issues/8026)를 보면 관련 설명들이 나오며 가장 중요한 포인트는 아래 2가지이다.

1. ingress nginx는 aws nlb의 속성을 제대로 사용하지 못함.

2. aws nlb의 속성을 활용하고 싶다면 aws laod balancer controller를 설치.
</div>

아 그리고 나의 경우 `helm`으로 설치했다.

# 1. 오직 443 포트로만 들어오는 ingress nginx

![1](https://github.com/may-30/may-30.github.io/assets/155306250/f70d0a53-396f-4fc1-a53c-f6784d769cb8){: .align-center}

우선, 위 사진이 구현하려는 아키텍처이다.

정말 다양한 테스트를 해보았지만 ingress-nginx를 통해서 생성된 nlb는 80 포트로 들어오는 경우에는 자동으로 redirect 처리를 하지 못한다.

그건 정말 당연한 이야기인 것이 nlb (network load balancer)는 `layer 4` 네트워크 장비이므로 http -> https redirect가 일어나지 않으며 http -> https redirect는 `layer 7` 영역에서 일어나야 하는 구조이다.

그렇기 때문에 나는 80으로 들어오는 것은 무조건적으로 차단하고 443으로 들어오는 트래픽만 허용하도록 설정했다.

그럼 이제 실습 해보자.

## 1-1. ingress nginx 설치

아래는 ingress-nginx의 helm `values.yaml` 파일 중 핵심적인 부분만 발췌한 것이다.

```yaml
controller:
  service:
    enabled: true
    external:
      enabled: true
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
      service.beta.kubernetes.io/aws-load-balancer-security-groups: "보안그룹 ID"
      service.beta.kubernetes.io/aws-load-balancer-name: "NLB 이름"
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
      service.beta.kubernetes.io/aws-load-balancer-subnets: "NLB가 위치할 서브넷 이름, NLB가 위치할 서브넷 이름"
      service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "ACM ARN"
      service.beta.kubernetes.io/aws-load-balancer-ssl-ports: https
      service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
      service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "60"
      service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    targetPorts:
      http: tohttps
      https: http
```

`service.beta.kubernetes.io/aws-load-balancer-*`라는 속성을 사용하려면 반드시 `aws laod balancer controller` 추가 기능이 설치되어 있어야 한다.

`controller.service.targetPorts`에서 확인할 수 있듯이 http는 https로 보내고 https는 http 통신을 하라고 설정한다.

하지만 정확하게는 http가 https로는 보내지 않는다.

![2](https://github.com/may-30/may-30.github.io/assets/155306250/262c91c8-6e1b-41c9-9e3c-863d131340a1){: .align-center}

생성된 모습이며 `resource map`을 통해서 target group과 연결되어 있는 모습을 확인할 수 있다.

## 1-2. ingress 생성

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.may30.xyz
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
```

![3](https://github.com/may-30/may-30.github.io/assets/155306250/cc7822f5-c6d7-4036-bf41-501ec5eed003){: .align-center}

`pod`와 `service`는 특별한 것이 없기에 `ingress`만 소개하며 `ingress`는 위와 같다.

`spec.rules[0].host`에 원하는 도메인을 입력하면 도메인 기반 호스팅이 이루어진다.

## 1-3. curl 테스트

![4](https://github.com/may-30/may-30.github.io/assets/155306250/7d29a256-f3ee-4361-97ea-13d991f767f4){: .align-center}

```bash
curl -v http://nginx.may30.xyz
```

http는 당연하게 접속 자체가 이루어지지 않는다.

![5](https://github.com/may-30/may-30.github.io/assets/155306250/3c61cd06-f0ca-4e4d-83b9-fb47591da96a){: .align-center}

```bash
curl -I -L https://nginx.may30.xyz
```

https는 정상적으로 접근이 되는 것을 확인할 수 있다.

# 2. 사설 인증서를 tls secret로 등록해서 사용하는 ingress nginx

그러면 ingress nginx를 사용해서 redirect는 절대 못하는 것일까?

그것은 아니다.

왜냐하면 ingress nginx는 설치 시에 ingress nginx controller가 설치되는데 ingress nginx controller가 결국엔 nginx이기 때문이다.

![6](https://github.com/may-30/may-30.github.io/assets/155306250/b675f17a-f9bc-4f4b-ad7d-6d6d5504ded8){: .align-center}

ingress nginx controller pod 에 접속하면 흔히 알고 있는 nginx의 conf까지 들어있으며 전형적인 nginx의 형태를 띄고 있기 때문이다.

그렇기 때문에 기존 nginx에서 proxy pass, ssl redirect 설정들은 그대로 해줄 것이 분명하다.

그렇다면 어떻게 해주는 것일까?

바로 ingress에서 ssl redirect를 해주도록 설정하는 것이다.

![7](https://github.com/may-30/may-30.github.io/assets/155306250/e61dcf5a-b8b2-4e40-ae69-9f6286e92bb6){: .align-center}

최종 아키텍처는 위 사진과 같다.

## 2-1. 사설 tls 인증서 생성

해당 방법은 aws acm을 사용하지 못한다.

그 이유는 acm은 public key, private key, chain key를 제공해주지 않고 자체적인 arn만을 제공하기 때문에 **반드시 사설 tls 인증서가 필요하다**.

사설 tls 인증서를 만들기 위해서는 여러 가지 방법이 있겠지만 실제로는 아래 명령어가 잘 작동했다.

```bash
dnf install -y openssl mod_ssl
dnf install -y python3 augeas-libs pip
python3 -m venv /opt/certbot/
/opt/certbot/bin/pip install --upgrade pip
/opt/certbot/bin/pip install certbot
ln -s /opt/certbot/bin/certbot /usr/bin/certbot
certbot certonly --manual -d *.may30.xyz --preferred-challenges dns --key-type rsa
```

![8](https://github.com/may-30/may-30.github.io/assets/155306250/5f6667da-146b-41b6-b530-41e5fbc9ef29){: .align-center}

이러면 이제 `may30.xyz` 도메인에 대한 wildcard 인증서가 발급 받아진 것이다.

## 2-2. secret tls 생성

```bash
kubectl create secret tls nginx-ssl --key private.pem --cert cert.pem
```

위 명령어를 통해서 secret tls를 생성한다.

## 2-3. ingress nginx 수정

```yaml
controller:
  service:
    enabled: true
    external:
      enabled: true
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
      service.beta.kubernetes.io/aws-load-balancer-security-groups: "보안그룹 ID"
      service.beta.kubernetes.io/aws-load-balancer-name: "NLB 이름"
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
      service.beta.kubernetes.io/aws-load-balancer-subnets: "NLB가 위치할 서브넷 이름, NLB가 위치할 서브넷 이름"
      service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
      service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "60"
      service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    targetPorts:
      http: http
      https: https
```

`acm`과 acm으로 인해서 사용되었던 `ssl port` 또한 제거한다.

http는 http로, https는 https으로 보낸다.

![9](https://github.com/may-30/may-30.github.io/assets/155306250/33ca20c2-be6a-44a7-a03f-a3e541766a72){: .align-center}

## 2-4. ingress 생성

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.may30.xyz
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
  tls:
  - hosts:
    - nginx.may30.shop
    secretName: nginx-ssl
```

![10](https://github.com/may-30/may-30.github.io/assets/155306250/7386c0a8-47c1-4901-88bb-70fa7d5e24b6){: .align-center}

기존 ingress와는 다르게 80 뿐만 아니라 443이 추가된 모습을 확인할 수 있다.

## 2-5. curl 테스트

이번에는 http로 접근해도 https로 redirect가 될 것이기 때문에 기존대로 명령어를 날려보았다.

![11](https://github.com/may-30/may-30.github.io/assets/155306250/f1a44242-4eaf-4707-a40f-3c55962b302a){: .align-center}

```bash
curl -I -L http://nginx.may30.xyz
```

정상적으로 동작하는 것을 확인할 수 있다.

# 3. 마치며

사설 인증서를 활용해서 ingress 단에서 ssl 처리를 하게 되면 결국엔 k8s 레벨에 올라가 있는 실제 애플리케이션들의 성능이 안 좋을 수 있다.

그래서 alb를 통해 ssl offloading을 하도록 설정하거나 nlb를 생성하더라도 다이렉트로 80 포트로는 접근할 수 없도록 하는 것이 제일 좋은 방법으로 보인다.