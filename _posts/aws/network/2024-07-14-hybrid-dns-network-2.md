---
title: "[Network] Hybrid DNS Network #2 (feat. AWS, IDC)"
categories:
  - network
tags:
  - [hybrid network]
toc: true
toc_sticky: true
date: 2024-07-14
last_modified_at: 2024-07-15
---

![amazon-r53](https://github.com/user-attachments/assets/8fa07248-7181-4e2c-8821-5ad40127d1fa){: .align-center}

# 0. 작성 취지

![1](https://github.com/user-attachments/assets/4d877315-afd0-49c4-81c8-36fcfcff0b1b){: .align-center}

위 사진은 최종 아키텍처이다.

[지난 1편](https://may-30.github.io/network/hybrid-dns-network-1/)에서 사전 준비를 다 끝냈다.

모든 테스트 진행 순서는 아래와 같다.

1. [AWS - IDC 환경] Site to Site VPN 구축
2. [IDC 환경] Openswan 구축, DNS 서버 구축
3. [AWS - IDC 환경] Routing 설정
4. [AWS 환경] Private Hosted Zone Route53 구축
5. IDC -> AWS로 질의하는 경우
   1. [AWS 환경] Inbound Resolver Endpoint 생성
   2. [IDC 환경] DNS Forwarder 설정
6. AWS -> IDC로 질의하는 경우
   1. [AWS 환경] Outbound Resolver Endpoint 생성
   2. [AWS 환경] Rules 생성

이번 포스팅에서는 본격 테스트인 2가지를 다룰 것이다.

5. IDC -> AWS로 질의하는 경우
   1. [AWS 환경] Inbound Resolver Endpoint 생성
   2. [IDC 환경] DNS Forwarder 설정
6. AWS -> IDC로 질의하는 경우
   1. [AWS 환경] Outbound Resolver Endpoint 생성
   2. [AWS 환경] Rules 생성

# 5. IDC -> AWS로 질의하는 경우

## 5-1. [AWS 환경] Inbound Resolver Endpoint 생성

### 5-1-1. Security Group 생성

![1](https://github.com/user-attachments/assets/2ec7b71a-c5cc-4ed5-af1c-0b112a4440a6){: .align-center}

Security Group의 경우 Inbound Resolver Endpoint로 질의를 하는 것은 오직 idc vpc의 DNS 서버이다.

그러므로 idc vpc의 DNS 서버의 IP만 UDP 53 Port로 열어주는 것이 보안적인 측면에서 안전하다.

### 5-1-2. Inbound Resolver Endpoint 생성

![2](https://github.com/user-attachments/assets/3e115945-f899-4377-b79d-ad7719692f6d){: .align-center}

![3](https://github.com/user-attachments/assets/4b7afb2c-04e5-47a1-a8e1-da50b1b211d0){: .align-center}

권장하는 사항으로 Inbound Resolver Endpoint를 생성할 때는 IDC DNS 서버에 등록해주어야 하니 빨간색 네모박스처럼 IP를 고정해서 가져가는 것이 좋다.

## 5-2. [IDC 환경] DNS Forwarder 설정

### 5-2-1. DNS Forwarder 설정

`/etc/named.conf` 파일 수정

```md
zone "cloud.com" {
  type forward;
  forward only;
  forwarders { << cloud vpc Inbound Resolve Endpoint 1; cloud vpc Inbound Resolve Endpoint 2; >> };
}
```

```bash
service named restart
```

`named` service를 재시작한다.

### 5-2-2. DNS 테스트

![4](https://github.com/user-attachments/assets/994dd89d-76c4-47c9-9979-bf0dfacab7ec){: .align-center}

이렇게 되면 idc vpc의 App 서버에서 cloud vpc의 App 서버에 `nslookup`이 안되다가 `nslookup`이 성공적으로 되는 것과 `ping` 또한 성공적으로 되는 것을 확인할 수 있다.

# 6. AWS -> IDC로 질의하는 경우

## 6-1. [AWS 환경] Outbound Resolver Endpoint 생성

### 6-1-1. Security Group 생성

![5](https://github.com/user-attachments/assets/bd111beb-a695-4f17-b659-e2d8a1b67a16){: .align-center}

cloud vpc의 DNS 서버 IP만 입력한다.

(**❗️ AWS 자체에서 제공하는 DNS 서버는 VPC Cidr + 2입니다.**)

### 6-1-2. Outbound Resolver Endpoint 생성

![6](https://github.com/user-attachments/assets/4fc27d64-221d-4efe-8c4a-6a15ff6231a9){: .align-center}

![7](https://github.com/user-attachments/assets/72569c1f-037b-4a7e-bf88-47329325d538){: .align-center}

마찬가지로 Outbound Resolver Endpoint를 생성할 때 빨간색 네모박스처럼 IP를 고정하는 것이 좋다.

## 6-2. [AWS 환경] Rules 생성

### 6-2-1. Rules 생성

idc vpc의 DNS 서버에서 Forwarder 설정을 해주었던 것처럼 AWS에서도 마찬가지로 Forwarder 설정을 해주어야 한다.

![8](https://github.com/user-attachments/assets/84fe2dc4-3039-4405-9acc-81c4be44e59c){: .align-center}

빨간색 네모박스처럼 Domain 이름은 `idc.com`으로 설정한다.

그리고 Target은 idc vpc의 DNS 서버를 작성한다.

### 6-2-2. DNS 테스트

![9](https://github.com/user-attachments/assets/82afba0f-8b6d-48db-938c-d8005f84ae12){: .align-center}

cloud vpc의 App 서버에 접속해서 idc vpc의 App 서버에 `nslookup`이 안되다가 `nslookup`이 성공적으로 되는 것과 `ping` 또한 성공적으로 되는 것을 확인할 수 있다.

# 7. 마치며

한번 구축하고 나니 결국 AWS에서도 IDC와 비슷한 설정, 환경을 구성해야 한다는 것을 알게 되었다.