---
title: '[aws network] igw edge association 알아보기'
categories:
  - aws network
tags:
  - [aws, network, igw, igw edge association]
toc: true
toc_sticky: true
date: 2024-01-21
last_modified_at: 2024-01-22
---

## 1. 어떤 정보인가?

![278942965-b4392268-0dcb-4558-9830-0eb792b6e609](https://github.com/may-30/may-30.github.io/assets/155306250/3c4a0f86-e12e-4176-b800-41a4ca8e354d){: .align-center}

Edge Association은 VPC로 들어오는 Ingress 트래픽을 Appliance로 라우팅하는데 사용하는 라우팅 테이블이다.

Internet Gateway(IGW) 혹은 Virtual Private Gateway(VPG)와 연결한 라우팅 테이블에 대상을 Appliance의 Eni로 지정하면 된다.

## 2. 정보 설명

### 2-1. IGW 전용 라우팅 테이블 생성

![278201980-2a457f2f-78f7-4934-96b2-80a0d30b5565](https://github.com/may-30/may-30.github.io/assets/155306250/87cbae60-4ea9-41d6-aa62-3da06a8f5423){: .align-center}

일반적인 라우팅 테이블을 생성하는 것과 동일하며 라우팅 테이블의 이름은 편한대로 지정하면 된다.

### 2-2. IGW 전용 라우팅 테이블에 IGW 연결

![278202721-d5be6e18-1b40-43d9-ab2b-c424030f8f73](https://github.com/may-30/may-30.github.io/assets/155306250/72c2b59d-95a1-4286-b764-0d53740d3f9b){: .align-center}

Destination: Ingress 트래픽이 발생하는 VPC Cidr를 적어준다.

- ex) `public nlb subnet cidr` 혹은 `public ec2 subnet cidr` 등

Target: FW의 ENI를 선택한다.

- 사진에서는 ANFW 의 GWLB Endpoint 이기 떄문에 `vpce-X`로 표기되는 중이다.

### 2-3. IGW 전용 라우팅 테이블 라우팅 설정

![278203006-d1c721dc-c1d1-4c10-92c2-73f9e1dd7e2f](https://github.com/may-30/may-30.github.io/assets/155306250/83a654d0-384f-438d-b1bf-8e5eb19585fe){: .align-center}

Route Table을 선택하고 Edge Assocation 탭을 클릭한다.

![278203306-4f91820e-15bd-48da-9367-f90cfef0f237](https://github.com/may-30/may-30.github.io/assets/155306250/9003c18d-7a27-4e99-8def-58cb094a7b86){: .align-center}

IGW가 나오게 되고 연결하고자 하는 IGW를 선택하여 저장해주면 된다.

### 2-4. 정리

3가지 단계를 거치게 되면 ingress 트래픽 중 public nlb cidr로 들어오는 트래픽은 ANFW Firewall Policy를 거치고 최종 목적지로 가게 된다.