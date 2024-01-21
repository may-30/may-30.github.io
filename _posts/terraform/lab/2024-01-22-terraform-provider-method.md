---
title: '[terraform lab] aws provider 선언 방식 3가지'
categories:
  - terraform lab
tags:
  - [iac, terraform, variables]
toc: true
toc_sticky: true
date: 2024-01-22
last_modified_at: 2024-01-22
---

## 1. 어떤 정보인가?

aws로 terraform을 사용할 때 provider를 선언하는 방식이 대표적으로 3가지가 있다.

1. access key, secret key를 직접 사용하는 방식.
2. profile을 등록해서 사용하는 방식.
3. assume role을 사용하는 방식.

최소한의 보안을 위하여 개인 노트북에서 사용할 때는 1번보다는 2번의 방식을 사용하고 있다.

가장 보안적인 방법은 3번이다.

이유는 terraform을 aws 내부에서만 사용할 수 있도록 강제할 수 있는 방법이기도 하고 role에 붙어있는 policy를 통해 최소 권한 원칙으로 권한을 부여할 수 있기 때문이다.

## 2. 정보 설명

![스크린샷 2024-01-22 03 52 12](https://github.com/may-30/may-30.github.io/assets/155306250/22106033-a6eb-4713-8a75-7f04c54836bb){: .align-center}

이번 테스트는 provider 구성 테스트이기 때문에 terraform을 위 사진처럼 아주 심플하게 구성할 것이다.

**구성을 반드시 사진처럼 구현할 필요는 없다.**

```hcl
# version.tf
# - 글 작성일 (24.01.22) 기준 가장 최신 버전이다.
terraform {
    required_providers {
        aws = {
            source = "hashicorp/aws"
            version = "~> 5.33.0"
        }
    }
    
    required_version = "~> 1.7.0"
}
```

```hcl
# main.tf
resource "aws_vpc" "test" {
    cidr_block = "10.0.0.0/16"
    enable_dns_hostnames = true
    enable_dns_support = true

    tags = {
        Name = "test"
    }
}
```

### 2-1. access key, secret key 직접 사용

```hcl
# provider.tf
provider "aws" {
    region = "ap-northeast-2"
    access_key = "YOUR_ACCESS_KEY"
    secret_key = "YOUR_SECRET_KEY"
}
```

### 2-2. profile 등록 후 사용

```hcl
# provider.tf
provider "aws" {
    region = local.region
    profile = local.profile
}
```

```hcl
# variables.tf
locals {
    region = "ap-northeast-2"
    profile = "YOUR_PROFILE"
}
```

### 2-3. assume role 사용

(assume role을 테스트 하기 위해서 codebuild를 사용했고 codebuild의 자세한 구성 방법은 추후에 글을 올리도록 하겠다.)

```hcl
# provider.tf
provider "aws" {
    region = local.region

    assume_role {
        role_arn = local.role_arn
        session_name = local.session_name
    }
}
```

```hcl
# variables.tf
locals {
    region = "ap-northeast-2"
    role_arn = "ROLE_ARN"
    session_name = "tf-test"
}
```

![스크린샷 2024-01-22 05 09 59](https://github.com/may-30/may-30.github.io/assets/155306250/041dca6a-4898-4a50-8e46-5b879eeccf5e){: .align-center}

![스크린샷 2024-01-22 05 10 56](https://github.com/may-30/may-30.github.io/assets/155306250/81370298-0cf8-4f0d-a0f8-d7e6426b4e23){: .align-center}

성공적으로 구현한 것을 확인할 수 있다.

VPC ID는 비교를 위하여 일부러 가리지 않았다.