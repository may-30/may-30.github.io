---
title: '[aws network] tgw appliance mode 알아보기 + inspection vpc 구축'
categories:
  - aws network
tags:
  - [aws, network, tgw, tgw appliance mode]
toc: true
toc_sticky: true
date: 2024-01-21
last_modified_at: 2024-01-21
---

## 1. 어떤 정보인가?

1. 방화벽 특징

- 방화벽은 들어온 패킷을 검사하고 나가는 패킷을 검사한다는 아주 중요한 특징이 있다.

2. AZ 친화성

- AWS 는 같은 가용영역끼리 통신을 먼저 주고 받으려는 경향이 아주 강하게 구성되어 있다.

3. TGW Appliance Mode

- AWS 상에서 TGW 를 통해 방화벽 장비를 거쳐 서로 다른 AZ 와의 통신이 필요로 할 때 AZ 친화성을 무너뜨려주는 TGW 의 기능 중 하나이다.

## 2. 정보 설명

### 2-1. 아키텍처 구성도

![285097370-ec8b8945-987b-4be1-bba7-c119bf32d564](https://github.com/may-30/may-30.github.io/assets/155306250/0148cbb3-1790-4175-a2f5-b6540d88ebbb){: .align-center}

NAT Gateway는 생성하지 않았고 서버에는 미리 golang으로 웹 서버를 구축하여 테스트를 진행하였다.

### 2-2. tgw appliance mode off

![285105042-f45593ef-77a5-4b87-8bd9-16be77913d28](https://github.com/may-30/may-30.github.io/assets/155306250/8cb99164-5420-484e-8737-e8b5355af427){: .align-center}

우선 tgw appliance mode가 off 되어 있는 경우이다.

![285097444-a4f38038-c045-45e5-89f3-7bf4047700ea](https://github.com/may-30/may-30.github.io/assets/155306250/8124cdcb-e643-4e13-a7bd-65136f2c3fec){: .align-center}

방화벽은 기본적으로 들어온 패킷과 나가는 패킷을 검사하는데 위 사진을 자세히 보면 파란색 선과 빨간색 선이 서로 다른 Firewall Endpoint를 향한다는 것을 볼 수 있다.

이럴 경우 가용영역 c에 위치한 방화벽 입장에서는 **"나한테서 출발시킨 패킷이 없는데 왜 return 패킷을 보내지?"** 라며 혼란에 빠지게 된다.

![285097866-cd581bc8-af26-4ef8-91af-29935af0808c](https://github.com/may-30/may-30.github.io/assets/155306250/001f902d-d358-4346-bde3-e593b0941e21){: .align-center}

그렇게 되면 가용영역 c에 위차한 방화벽은 패킷을 드랍시키게 되며 패킷이 드랍되었으므로 당연히 통신은 되지 않고 위 사진처럼 `Time Out`이 발생하게 된다.

이렇게 구성되는 이유는 AWS의 같은 AZ를 우선적으로 통신하게끔 설정해두는 AZ 친화성이 기본적으로 구축되어 있기 떄문이다.

### 2-3. tgw appliance mode on

그렇다면 AWS의 같은 AZ를 우선적으로 통신하게끔 설정해두는 AZ 친화성을 무너뜨릴 수 있다면 서로 다른 AZ끼리 통신이 가능하다는 소리 아닌가?

그 역할을 해주는 것이 tgw appliance mode를 활성화 시켜주는 것이다.

![285105071-9da72c4a-733e-4866-acd2-7d8b9f2c0bf7](https://github.com/may-30/may-30.github.io/assets/155306250/ee2c6e4d-c1a3-4d80-870f-d181ec887234){: .align-center}

tgw appliance mode를 활성화시켜준다.

![285097973-90b48cc2-8b7e-4d11-aa4a-40b82ce0542b](https://github.com/may-30/may-30.github.io/assets/155306250/191ae613-5ed8-4b7a-b4f0-5e47cea6f417){: .align-center}

그렇게 되면 앞서 `Time Out`이 발생했던 화면이 정상적으로 통신이 되는 것을 확인할 수 있다.

- (바로 반영되지는 않고 체감상 5분 정도는 기다려야 적용된다.)

![285098009-beab6d9a-2ace-48b3-b1a0-b09ffa5fef0a](https://github.com/may-30/may-30.github.io/assets/155306250/3d083269-45b8-4f49-b84f-24361e1f613e){: .align-center}

네트워크의 흐름도는 위 사진과 같은데 `Return 패킷` (빨간색 선)이 가용영역 c에서 출발해도 다시 가용영역 a에 있는 방화벽으로 흘러들어가게 된다.

이는 AWS의 같은 AZ를 우선적으로 통신하는 설정인 AZ 친화성이 꺼진 것으로 봐도 무방하다.

### 2-4. BONUS 만약 방화벽이 없는 상태라면 어떻게 동작하는가?

편하게 생각해보자.

![285098836-3dae0833-f142-4030-90c0-49f986ad1b18](https://github.com/may-30/may-30.github.io/assets/155306250/b200f599-d526-48a3-9940-4a1d65a66774){: .align-center}

![285098066-ecb1c51e-20d3-4128-9b78-a0cfa7f5ba76](https://github.com/may-30/may-30.github.io/assets/155306250/561258cb-2a20-4995-9166-c5967df17651){: .align-center}

방화벽이 없을 때는 네트워크 흐름이 단순하게 tgw를 타고 흐르는 것이므로 정상적으로 동작해야만 한다.

이를 `비대칭 통신`이라고 부르며 AWS에서는 라우팅 테이블에 기본적으로 들어있는 `local`이라는 경로 때문에 가능하다.

AWS는 라우팅 테이블을 생성하면 항상 기본적으로 자신의 `VPC Cidr`를 `local`이라는 이름으로 사전에 등록을 시켜서 생성해준다.

이로 인해 서버의 보안그룹만 알맞게 설정해준다면 동일 VPC의 az 간 통신(az-2a <-> az-2c)의 통신은 가능하다.