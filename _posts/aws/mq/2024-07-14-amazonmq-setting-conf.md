---
title: "[Amazon MQ] 설정 및 구성"
categories:
  - mq
tags:
  - [active mq]
toc: true
toc_sticky: true
date: 2024-07-14
last_modified_at: 2024-07-14
---

![amazon-mq](https://github.com/user-attachments/assets/7af0402a-29ea-4c21-b9b2-72500ba8d1a2){: .align-center}

# 0. 작성 취지

프로젝트에서 Amazon MQ를 구축할 일이 생겨 공부를 하다가 내용을 정리하는 것이 기억에도 오래 남을 것 같아서 작성한다.

Amazon MQ는 AWS의 대표적인 Message Queue 서비스 중 하나이다.

Message Queue를 사용하는 이유는 많은 이유가 있겠지만 그 중 가장 많은 비중을 차지하는 것은 디커플링이다.

![1](https://github.com/user-attachments/assets/d37b764e-4f38-47ad-a1ce-d5c1c0125dc9){: .align-center}

위 사진과 같이 디커플링의 반대인 커플링 아키텍처의 경우이다.

심플하지만 여기서의 문제점은 애플리케이션 1의 로직이 변경된다면 애플리케이션 A, B, C에서 애플리케이션 1를 호출하는 로직 또한 변경해주어야 할 수 있다.

**즉, 하나의 애플리케이션만 변경했는데 이와 연결된 모든 애플리케이션 또한 같이 변경해주어야 한다는 상당한 리스크가 발생하게 된다.**

![2](https://github.com/user-attachments/assets/759b013f-9090-4035-85be-3ddfa48d041f){: .align-center}

그와 반대로 위 사진처럼 애플리케이션 A, B, C와 애플리케이션 1 사이에 전달자 역할을 하는 매개체가 있는 상태에서 애플리케이션 1의 수정사항이 변경된다면 어떻게 될까?

매개체 쪽에서만 애플리케이션 1에 대한 호출 방식을 변경해주면 될 것이고, 애플리케이션 A, B, C에서 별도로 수정할 사항 없이 사용할 수 있다는 장점이 있다.

Amazon MQ에 대해서 아래와 같이 알아볼 것이다.

1. Amazon MQ 구성도
2. Amazon MQ 테스트

# 1. Amazon MQ 구성도

Amazon MQ는 아래 2가지 구성이 존재한다.

1. 단일 인스턴스 구조
2. Active / Stand-By 구조

단일 인스턴스 구조는 또 다시 아래 2가지 구성으로 나뉜다.

![3](https://github.com/user-attachments/assets/c7519e4c-16ae-4a9c-a2ee-aa4f2778831b){: .align-center}

1. EBS 스토리지를 사용하는 단일 인스턴스

EBS 스토리지를 사용하면 짧은 대기 시간과 높은 처리량에 최적화된 블록 스토리지를 제공하도록 설계되었다.

![4](https://github.com/user-attachments/assets/1e74ef22-73ed-4585-96b5-6a6bfd63a1ce){: .align-center}

1. EFS 스토리지를 사용하는 단일 인스턴스

EFS 스토리지를 사용하면 여러 가용 영역에 중복 저장하기 때문에 내구성과 가용성을 제공하도록 설계되었다.

![5](https://github.com/user-attachments/assets/995bb0b8-b026-4119-84de-8535ea7d6a17){: .align-center}

이번 글에서 중점적으로 다룰 Active / Stand-By 구조이다.

Active / Stand-By 구조는 자동 Failover를 지원하고 있기 때문에 Active에서 예기치 못한 장애 상황이 발생했을 때 즉각적으로 Stand-By가 Active로 변환되어 나머지 로직을 수행할 수 있다.

Active / Stand-By 구조에서는 스토리지를 EFS로 밖에 지원하지 않고 있는데 그 이유는 총 2개의 Amazon MQ에서 저장되어 있는 Queue를 처리해야 하기 때문이다.

# 2. Amazon MQ 테스트

Amazon MQ를 생성하고 Queue에 메시지가 쌓이는 것을 확인해보려고 한다.

## 2-1. 보안 그룹 생성

![6](https://github.com/user-attachments/assets/1fe8c3a1-0620-4b49-97d7-a40e8cf63312){: .align-center}

Amazon MQ가 생성되면 웹으로 접근해서 Queue와 Topic을 생성할 수 있는데 이때 필요한 Port가 8162이다.

기존 TCP를 통해 호출하고 있었다면 `OpenWire`를 사용해서 호출해야 하며 이때 필요한 Port가 61617이다.

## 2-2. 구성 생성

우선, configuration이라는 구성을 생성해주어야 하는데 이는 RDS의 Parameter Group과 상당히 유사하다.

1개의 configuration으로 여러 개의 Amazon MQ에 동시 적용할 수 있다.

![7](https://github.com/user-attachments/assets/354a2751-35aa-49c5-aaf2-343049ae64f1){: .align-center}

위 사진처럼만 하면 기본적인 구성이 생성된다.

### 2-2-1. Queue, Topic 자동 삭제 방지

여러 테스트를 해본 결과 기본으로 생성된 구성을 활용하면 Queue, Topic이 일정 시간이 지나면 바로 삭제된다는 점이었다.

이는 활성화되지 않은 Queue, Topic에 대해서 삭제하는 구성이 안에 녹아져있기 때문이다.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<broker schedulePeriodForDestinationPurge="0" xmlns="http://activemq.apache.org/schema/core">
<!-- 중간 생략 -->
  <destinationPolicy>
    <policyMap>
      <policyEntries>
        <policyEntry gcInactiveDestinations="false" inactiveTimoutBeforeGC="600000" topic="&gt;">
        </policyEntry>
        <policyEntry gcInactiveDestinations="false" inactiveTimoutBeforeGC="600000" queue="&gt;"/>
        </policyEntry>
      </policyEntries>
    </policyMap>
  </destinationPolicy>
<!-- 중간 생략 -->
</broker>
```

이를 방지하고자 생성한 configuration을 위처럼 변경해주어야 한다.

- `schedulePeriodForDestinationPurge="0"`으로 변경.
- `gcInactiveDestinations="false"`으로 변경.

## 2-3. Amazon MQ 생성

![8](https://github.com/user-attachments/assets/ce66c365-cb31-4547-8400-54d9a0138417){: .align-center}

![9](https://github.com/user-attachments/assets/5d39931b-6708-4f0c-9c79-698274e0559a){: .align-center}

Active / Stand-By 구조를 선택하면 EFS만 선택할 수 없다는 것을 확인할 수 있다.

![10](https://github.com/user-attachments/assets/043b68da-644a-4abc-9229-063e8d583f90){: .align-center}

Amazon MQ의 인스턴스 사양을 선택하고 Active MQ에 웹으로 접근하려는 최초 계정(관리자)의 아이디와 비밀번호를 입력한다.

비밀번호의 경우에는 패스워드 복잡도가 AWS 자체에서 걸려있기 때문에 신경써서 만들어주어야 한다.

![11](https://github.com/user-attachments/assets/df450d27-3ef8-4a15-85ff-dbaa2696ea79){: .align-center}

마이너 버전 업그레이드를 켜준 이유도 Active / Stand-By 구조이기 때문에 활성화 시켰다.

사전에 생성해두었던 configuration을 선택하고 현재는 테스트 환경이기 때문에 Public access로 선택히였다.

생성해둔 VPC와 Subnet을 선택하고 생성해둔 보안그룹을 선택하면 생성 완료이다.

## 2-4. Failover 시 자동으로 다른 Endpoint 호출 가능하도록 선언

![12](https://github.com/user-attachments/assets/42891392-e806-4210-8817-b8aeb36334d0){: .align-center}

생성 완료된 Amazon MQ의 엔드포인트 상태를 보면 1개의 엔드포인트를 제공하는 것이 아닌 Active / Stand-By의 엔드포인트를 제공하는 모습을 확인할 수 있다.

그렇다면 업데이트 작업이나 예기치 못한 상황이 발생했을 때마다 소스에서 Broker URL을 수정하라는 뜻인건가?, 이러면 Auto Failover가 아니지 않을까? 하는 의문점이 들었다.

그런데 찾아보니 정말 단순하게 해결할 수 있다.

```java
private static String brokerURL = "failover:(OpenWire주소 1, OpenWire주소 2)?randomize=false";
```

위와 같이 선언해주면 Auto Failover가 동작하는 것을 확인할 수 있다.

# 3. 마치며

Message Broker에 대해서 직접 만져볼 경황이 없었는데 좋은 기회가 생겨서 경험해 볼 수 있어서 값진 경험이었다.

역시 AWS는 가용성과 확장성을 내새워서 승부를 보고 자동화에 대한 편리함을 사용자에게 지속적으로 제공해주어서 클라우드에 계속 묶어놓으려고 하는 것이 보인다.