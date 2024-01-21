---
title: '[error] remote backend 구성 후 backend 참조할 때 default profile이 없는 경우'
categories:
  - terraform error
tags:
  - [iac, terraform, error]
toc: true
toc_sticky: true
date: 2024-01-13
last_modified_at: 2024-01-13
---

## 1. 원인

### 상황

![123142edasdfasvdsva](https://github.com/may-30/may-30.github.io/assets/155306250/08db2467-48cd-46d6-a279-6971365baee0)

terraform `.tfstate`파일을 remote로 관리하기 위해 backend를 담당하는 terraform 폴더를 별도로 두어 s3 및 dynamodb를 생성하였다.

그 후 teddy라는 terraform 폴더에서 생성되는 `.tfstate`파일을 remote backend로 참조하려는 상황이다.

### 문제

![293837643-7cfe6fe4-b173-49b1-b796-b47f5550bfb6](https://github.com/may-30/may-30.github.io/assets/155306250/40412c51-1352-4744-9219-e6919fe7517b)

분명 profile을 지정하고 실행시켰는데 느닷없는 권한 부족이 발생한다.

설정한 profile은 Admin 권한을 가지고 있는 profile 인데 권한 부족이 발생한다는 것은 이상한 일이다.

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

여러 곳을 구글링해본 결과 그나마 비슷한 증상을 해결할 수 있을 것으로 보이는 [링크](https://github.com/hashicorp/terraform-provider-aws/issues/26074)를 발견했다.

바로 aws cli의 default profile을 설정하지 않아서 발생하는 문제라고 한다.

나는 평소 보안이 조금 걱정되어서 aws cli의 default profile을 설정하지 않고 있었다.

![293839772-ea567f99-e0fb-4e0e-83e6-cd5af60d056b](https://github.com/may-30/may-30.github.io/assets/155306250/09ae185d-c8ef-431e-8688-76115d4107bb)

aws cli의 default profile을 설정하고 나니 정상적으로 backend s3를 참조할 수 있다고 나오며 에러가 발생하지 않는다.

---
