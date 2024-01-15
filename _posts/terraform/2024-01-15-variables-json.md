---
title: '모듈을 생성할 때 별도의 variable을 json 파일로 참조하기'
categories:
  - terraform
tags:
  - [iac, terraform, variables]
toc: true
toc_sticky: true
date: 2024-01-15
last_modified_at: 2024-01-15
---

## 1. 무슨 문제가 있었는가?

Cloud SA로 근무하면서 틈틈이 terraform을 취미로 만지고 있을 때 아래 3가지를 해결하면 좋겠다는 생각이 들었다.

- 용도에 따른 subnet 생성.
- 용도에 따른 subnet의 route table 생성 및 route table assoication 생성.
- **가변적으로 생성 가능한 여러 개의 network.**

## 2. 해결했는가?

성공/실패 여부만 보았을 때는 성공이지만 내 마음에 들지 않는 성공이다.

그 이유는 json 파일을 보면 알 수 있는데 json 파일의 모양이 이쁘지 않기 때문이다.

## 3. 어떻게 해결했는가?

![스크린샷 2024-01-15 오후 11 37 08](https://github.com/may-30/may-30.github.io/assets/155306250/f0a7cc2c-3563-46f0-a73a-936be97d0d4a)

전체적인 그림이다. (불필요한 부분은 삭제함.)

### variables/dev_network.json

- **⚠️ subnet은 테스트용으로 public, private 각 1개씩 구현되어 있다!**
- **⚠️ subnetting은 엉망이다!**

```json
// 1개의 vpc만 생성하는 경우
{
    "vpc_map": {
        "dev": {
            "cidr": "10.10.0.0/16",
            "az": ["a", "c"],
            "subnet": {
                "pub": {
                    "pub-sub-2a": {
                        "az": "a",
                        "cidr": "10.10.1.0/24",
                        "route": "igw"
                    },
                    "pub-sub-2c": {
                        "az": "c",
                        "cidr": "10.10.3.0/24",
                        "route": "igw"
                    }
                },
                "pri": {
                    "pri-sub-2a": {
                        "az": "a",
                        "cidr": "10.10.111.0/24",
                        "route": "local"
                    },
                    "pri-sub-2c": {
                        "az": "c",
                        "cidr": "10.10.113.0/24",
                        "route": "local"
                    }
                }
            }
        }
    }
}
```

```json
// 여러 개의 vpc를 생성하는 경우
{
    "vpc_map": {
        "dev": {
            "cidr": "10.10.0.0/16",
            "az": ["a", "c"],
            "subnet": {
                "pub": {
                    "pub-sub-2a": {
                        "az": "a",
                        "cidr": "10.10.1.0/24",
                        "route": "igw"
                    },
                    "pub-sub-2c": {
                        "az": "c",
                        "cidr": "10.10.3.0/24",
                        "route": "igw"
                    }
                },
                "pri": {
                    "pri-sub-2a": {
                        "az": "a",
                        "cidr": "10.10.111.0/24",
                        "route": "local"
                    },
                    "pri-sub-2c": {
                        "az": "c",
                        "cidr": "10.10.113.0/24",
                        "route": "local"
                    }
                }
            }
        },
        "prd": {
            "cidr": "10.20.0.0/16",
            "az": ["a", "c"],
            "subnet": {
                "pub": {
                    "pub-sub-2a": {
                        "az": "a",
                        "cidr": "10.20.1.0/24",
                        "route": "igw"
                    },
                    "pub-sub-2c": {
                        "az": "c",
                        "cidr": "10.20.3.0/24",
                        "route": "igw"
                    }
                },
                "pri": {
                    "pri-sub-2a": {
                        "az": "a",
                        "cidr": "10.20.111.0/24",
                        "route": "local"
                    },
                    "pri-sub-2c": {
                        "az": "c",
                        "cidr": "10.20.113.0/24",
                        "route": "local"
                    }
                }
            }
        }
    }
}
```

1. json 파일에서 각 정보를 읽어와서 `modules/network`에 넘겨준다.

2. 넘겨받은 `modules/network`에서 각각의 sub modules(vpc, igw, subnet, ...)에 `for_each`로 반복시켜 준다.

## 4. 개선사항은 있는가?

json 파일의 구성을 이쁘게 개선하고 싶다.

하지만, 지금은 내 머리가 저 구성이 정답이라고 확정 지어 놓은 상태여서 다른 모듈을 구성하면서 변경을 시도해야 할 것 같다.

---