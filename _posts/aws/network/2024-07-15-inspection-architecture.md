---
title: "[Network] Inspection Architecture"
categories:
  - network
tags:
  - [tgw, inspection architecture]
toc: true
toc_sticky: true
date: 2024-07-15
last_modified_at: 2024-07-15
---

![aws-transit-gateway](https://github.com/user-attachments/assets/30b4f25d-e88e-4c83-9c9f-df2771e6861d){: .align-center}

# 0. 작성 취지

이번 포스팅은 Inspection Architecture에 대해서 간단하게 구성하는 포스팅이지만 사실 **Transit Gateway Routing Table과 Transit Gateway Subnet Routing Table의 차이점** 이 메인이다.

그 이유는 Transit Gateway를 사용해서 네트워크를 구축하던 중에 문득 궁금증이 생겼다.

**Transit Gateway 자체의 Routing Table과 Transit Gateway의 ENI가 존재하는 Routing Table의 역할은 정확히 무엇이 다를까?** 였다.

정확한 차이점을 인지하고 있지 않다 보니 라우팅 작업을 할 때도 헷갈리게 되고 다른 사람들에게 설명을 할 때도 "이게 맞나?" 하는 생각이 들었다.

이번 기회에 정확한 테스트를 하고 나서 완벽하게 인지해보려고 한다.

![1](https://github.com/user-attachments/assets/e1c5f2a9-b8ae-4ab2-8e3a-9ee61785c5f9){: .align-center}

구현하려는 최종 아키텍처이며 보다 정확한 테스트를 위해서 아키텍처에 라우팅 테이블까지 표기했다.

위 아키텍처가 Inspection Architecture이다.

(**❗️ 이번 포스팅에서는 Transit Gateway Routing을 중점적으로 볼 것이기 때문에 Network Firewall에 대해선 테스트 외에는 언급하진 않을 것입니다.**)

테스트는 아래와 같이 진행할 것이다.

1. ins vpc의 tgw subnet routing table 제거

# 1. ins vpc의 tgw subnet routing table 제거


![2](https://github.com/user-attachments/assets/598fb21c-37db-4c8a-8671-c503f38a9e3f){: .align-center}

![3](https://github.com/user-attachments/assets/77e45062-0734-4da8-9ff9-dfbdfdffbb3e){: .align-center}

기존에는 dev vpc의 App 서버와 prd vpc의 App 서버 간에 서로 `ping` 통신이 잘 되는 모습을 확인할 수 있다.

(**❗️ Network Firewall Rule Group에 ICMP를 열어주었습니다.**)

![4](https://github.com/user-attachments/assets/f7e37765-1f9a-4f5a-bcff-567b6b63f1a9){: .align-center}

![5](https://github.com/user-attachments/assets/878cb179-b03a-49a0-a5a1-bdb370aa1bec){: .align-center}

아키텍처 상에 빨간색 네모 박스에 해당되는 부분을 삭제하여 AWS 콘솔 화면처럼 삭제한 모습을 확인할 수 있다.

![6](https://github.com/user-attachments/assets/364877b2-5537-4b18-85b9-c223ed73a79d){: .align-center}

![7](https://github.com/user-attachments/assets/9ce293b5-bfcb-434b-96f9-3c123ef18496){: .align-center}

곧장 `ping` 통신을 보내보면 `packet loss`가 발생한다.

이를 통해서 확실하게 확인할 수 있게 되었다.

<div class="notice--primary" markdown="1">
Transit Gateway Routing Table
- Transit Gateway를 향해 자신의 VPC에서 타겟 VPC로 나가는 트래픽에 대해 라우팅 확인.

Transit Gateway Subnet Routing Table
- Transit Gateway를 통해 자신의 VPC에 들어오는 트래픽에 대해서 라우팅 결정.
</div>

# 2. 마치며

이번 테스트를 통해 간단하지만 Inspection Architecture에 대해서 알아보고 Transit Gateway의 Routing Table과 Transit Gateway Subnet의 Routing Table 간의 차이점에 대해서도 알아보았다.