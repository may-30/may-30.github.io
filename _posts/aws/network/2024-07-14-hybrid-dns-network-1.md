---
title: "[Network] Hybrid DNS Network #1 (feat. AWS, IDC)"
categories:
  - network
tags:
  - [hybrid network]
toc: true
toc_sticky: true
date: 2024-07-14
last_modified_at: 2024-07-14
---

![amazon-r53](https://github.com/user-attachments/assets/8fa07248-7181-4e2c-8821-5ad40127d1fa){: .align-center}

# 0. 작성 취지

사설 DNS를 AWS 환경과 IDC 환경에서 모두 사용해야 하는 경우에는 AWS에서와 IDC에서는 어떻게 해야 하는지를 테스트 해보고자 한다.

![1](https://github.com/user-attachments/assets/4d877315-afd0-49c4-81c8-36fcfcff0b1b){: .align-center}

위 사진이 만들고자 하는 최종 아키텍처인데 Direct Connect를 개인 구축하는 것은 비용과 장비 측면에서 어렵기 때문에 Site to Site VPN으로 대체해서 진행한다.

모든 테스트 진행 순서는 아래와 같다.

1. [AWS - IDC 환경] Site to Site VPN 구축
2. [IDC 환경] Openswan 구축, DNS 서버 구축
3. [AWS - IDC 환경] Routing 설정
4. [AWS 환경] Private Hosted Zone Route53 구축
5. AWS -> IDC로 질의하는 경우
   1. [AWS 환경] Outbound Resolver Endpoint 생성
   2. [AWS 환경] Rules 생성
6. IDC -> AWS로 질의하는 경우
   1. [AWS 환경] Inbound Resolver Endpoint 생성
   2. [IDC 환경] DNS Fowarder 설정

이번 포스팅에서는 사전 준비 격인 아래 3가지를 테스트할 것이다.

1. [AWS - IDC 환경] Site to Site VPN 구축
2. [IDC 환경] Openswan 구축, DNS 서버 구축
3. [AWS - IDC 환경] Routing 설정
4. [AWS 환경] Private Hosted Zone Route53 구축

# 1. [AWS - IDC 환경] Site to Site VPN 구축

Site to Site VPN을 구축할 때 필요한 것은 AWS 환경 측의 `Virtual Private Gateway`와 IDC 환경 측의 `Openswan에 연결된 Public IP`이다.

## 1-1. Virtual Private Gateway 생성

Virtual Private Gateway는 AWS와 IDC 네트워크를 연결하는 AWS 쪽의 라우터 역할이다.

Virtual Private Gateway는 1개의 VPC와 연결되지만 여러 개의 Customer Gateway와 연결 가능하다.

![2](https://github.com/user-attachments/assets/ac9279a7-e82f-4cf7-8988-cf283bef1b06){: .align-center}

Virtual Private Gateway를 생성한다.

![3](https://github.com/user-attachments/assets/72622c4e-5769-4218-804c-163a97f9ef8a){: .align-center}

**cloud vpc**와 연결한다.

## 1-2. Customer Gateway 생성

Customer Gateway는 온프레미스 네트워크의 Customer Gateway Device의 설정 값을 Virtual Private Gateway에 제공하기 위한 가상 게이트웨이이다.

이번 포스팅에서 Customer Gateway Device는 Openswan이 된다.

![4](https://github.com/user-attachments/assets/155f8517-1952-4b3f-b0a8-fccb1c48fe39){: .align-center}

Public IP라서 가렸지만 가려진 IP는 Openswan에 할당한 Elastic IP이다.

## 1-3. Site to Site VPN 생성

![5](https://github.com/user-attachments/assets/900f1864-bdd3-47e7-852b-09bd9454485b){: .align-center}

주의할 점으로는 아래와 같다.

- Local IPv4 Network CIDR: IDC Cidr
- Remote IPv4 Network CIDR: Cloud Cidr

# 2. [IDC 환경] Openswan 서버 구축, DNS 서버 구축

## 2-1. Openswan 서버 구축

### 2-1-1. Openswan 패키지 다운로드

```bash
yum install -y openswan
```

(**❗️ OS는 Amazon Linux 2를 이용했습니다.**)

(**❗️ Security Group의 경우 Site to Site VPN이 생성되고 Tunnel 1의 공인 IP를 UDP 4500 Port로 열어주어야 합니다.**)

### 2-1-2. Site to Site VPN 터널 구성

![6](https://github.com/user-attachments/assets/82625a53-91dc-47f1-adc7-0aefc54bf27a){: .align-center}

위 사진처럼 Site to Site VPN의 구성을 Openswan으로 맞추고 다운로드 받는다.

![7](https://github.com/user-attachments/assets/d8d8baaa-a6d1-4e55-8a50-1df125db0a9b){: .align-center}

위 사진처럼 txt 파일로 가이드 문서가 다운로드가 되는데 해당 문서대로 Openswan 서버에서 직접 작업을 진행하면 된다.

![8](https://github.com/user-attachments/assets/0c4da582-dc35-4997-867e-7aae411f1dc9){: .align-center}

위 사진처럼 VPN Tunnel 중 1개만 Up 상태가 된다면 성공이다.

## 2-2. DNS 서버 구축

### 2-2-1. Bind 패키지 다운로드

```bash
yum update -y

yum install -y bind bind-utils
```

(**❗️ Security Group의 경우 aws vpc와 idc vpc를 UDP 53 Port로 열어주어야 합니다.**)

### 2-2-2. DNS 설정

`/var/named/idc.com.zone` 파일을 생성하고 아래 내용을 추가한다.

```txt
$TTL 86400
@ IN SOA ns1.cloud.com. root.cloud.com. (
    2013042201 ;Serial
    3600 ;Refresh
    1800 ;Retry
    604800 ;Expire
    86400 ;Minimum TTL
)

; Specify our two nameservers
    IN NS dnsA.idc.com.
    IN NS dnsB.idc.com.

; Resolve nameserver hostnames to IP, replace with your two droplet IP addresses.
dnsA IN A 1.1.1.1
dnsB IN A 8.8.8.8

; Define hostname -> IP pairs which you wish to resolve
@ IN A << idc vpc App Private IP >>
app IN A << idc vpc App Private IP >>
```

`/etc/named.conf` 파일의 내용을 아래와 같이 변경한다.

```txt
options {
    directory "/var/named";
    dump-file "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    allow-query { any; };
    allow-transfer { localhost; << idc vpc DNS Private IP >>; };
    recursion yes;
    forward first;
    forwarders {
        192.168.0.2;
    };
    dnssec-enable yes;
    dnssec-validation yes;
    dnssec-lookaside auto;
    /* Path to ISC DLV key */
    bindkeys-file "/etc/named.iscdlv.key";
    managed-keys-directory "/var/named/dynamic";
};

zone "idc.com" IN {
    type master;
    file "idc.com.zone";
    allow-update { none; };
};
```

```bash
service named restart
chkconfig named on
```

![9](https://github.com/user-attachments/assets/efcf4fdd-185a-4862-8c5f-bf08c178b174){: .align-center}

```bash
service named status
```

정상적으로 동작하는지 확인할 수 있다.

## 2-3. App 서버 DNS 설정 변경

idc vpc의 App 서버에서 모든 DNS 요청을 idc vpc의 DNS 서버로 할 수 있게 변경해주어야 한다.

```txt
DNS1=<< idc vpc DNS Private IP >>
```

`/etc/sysconfig/network-scripts/ifcfg-eth0` 에서 위 내용을 추가해야 한다.

```bash
reboot
```

변경사항을 적용시키기 위해 재부팅을 한다.

![10](https://github.com/user-attachments/assets/c1f1128a-870c-47f9-b118-0082664a1f7b){: .align-center}

`nslookup` 및 `ping` 도 전부 자기 자신을 가리키고 있다는 것을 확인할 수 있다.

# 3. [AWS - IDC 환경] Routing 설정

## 3-1. AWS 환경 Routing 설정

![11](https://github.com/user-attachments/assets/84c9cdca-c997-40ee-a4e5-dd1b9eaf8a0a){: .align-center}

위 사진처럼 cloud vpc에 존재하는 Routing Table에는 `route propagation` 기능이 있으며 해당 기능을 켜주면 자동으로 Virtual Private Gateway에 연결된 cidr 대역이 추가된다.

cloud vpc에 존재하는 Private Subnet의 Routing Table에 모두 켜준다.

## 3-2. IDC 환경 Routing 설정

### 3-2-1. Openswan EC2 Network 설정 변경

![12](https://github.com/user-attachments/assets/f6221f74-48a8-4189-99e7-8fb7734b0ecc){: .align-center}

위 사진처럼 Openswan의 EC2 인스턴스에 Source / Destination Check 기능을 꺼주어야 한다.

Openswan 입장에서는 자기 자신으로부터 출발하고 도착해야 하는 것이 아닌 다른 서버들(idc vpc의 DNS, App)도 자신을 통해서 트래픽이 나가야 하기 때문이다.

### 3-2-2. IDC 환경 Routing 변경

![13](https://github.com/user-attachments/assets/aa83bef1-2375-45a3-981d-962d6141f472){: .align-center}

idc vpc에 존재하는 모든 Private Subnet의 Routing Table은 cloud vpc에 대해서 Openswan EC2 인스턴스로 가도록 변경해주어야 한다.

# 4. [AWS 환경] Private Hosted Zone Route53 구축

## 4-1. Private Hosted Zone Route53 구축

![14](https://github.com/user-attachments/assets/a05aa83e-0878-4727-9ca9-868fb3867e53){: .align-center}

![15](https://github.com/user-attachments/assets/5aafce93-efa2-4cfb-98db-2d98b7b43554){: .align-center}

cloud.com에 대한 Private Hosted Zone을 생성하고 may30 이라는 Sub Domain도 연결해준다.

# 5. 마치며

AWS와 IDC 간의 Hybrid DNS Network를 직접 구축 및 테스트까지 해보고 싶었다.

이번 포스팅은 테스트를 위한 구축이었고 다음 포스팅은 실제 Rule, Forward 등의 테스트를 해보려고 한다.