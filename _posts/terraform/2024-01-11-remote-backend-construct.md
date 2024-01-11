---
title: '[error] remote backend 구성을 같은 디렉터리에 넣은 경우'
categories:
  - terraform
tags:
  - [iac, terraform, error]
toc: true
toc_sticky: true
date: 2024-01-11
last_modified_at: 2024-01-11
---

## 1. 원인

### 상황

terraform `.tfstate`파일을 remote로 관리하기 위해 s3로 업로드를 하려고 한다.

같은 디렉터리 내부에 s3, dynamodb를 생성하는 `resource block`이 존재하고 `terraform block`에서 s3를 참조하는 상황이다.

### 문제

![293834732-31998b7b-9dcd-4b47-b125-7f1e40cfc4f1](https://github.com/may-30/may-30.github.io/assets/155306250/bbe51d58-7685-457b-8779-1f3c7f9d1f8a)

참조하려는 s3 버킷은 이전에 생성되어있어야 한다는 에러 메세지를 반환한다.

### 참고 소스 코드

```hcl
# versions.tf
terraform {
	required_version = "~> 1.6.0"

	required_providers {
		aws = {
			source = "hashicorp/aws"
			version = "~> 5.24.0"
		}

		tls = {
			source = "hashicorp/tls"
			version = "~> 4.0.5"
		}

		helm = {
			source = "hashicorp/helm"
			version = "~> 2.12.1"
		}
	}

	backend "s3" {
		region = "ap-northeast-2"
		bucket = "S3_BUCKET_NAME"
		key = "ENV.tfstate"
		dynamodb_table = "DYNAMODB_NAME"
	}
}
```

## 2. 해결

![123142edasdfasvdsva](https://github.com/may-30/may-30.github.io/assets/155306250/08db2467-48cd-46d6-a279-6971365baee0)

backend를 담당하는 terraform 과 backend를 참조하는 terraform으로 나누는 방법이 현재로썬 최선의 방법인 것으로 보인다.

---
