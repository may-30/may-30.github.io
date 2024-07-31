---
title: '[EKS] Loki Stack Logback 설정'
categories:
  - eks
tags:
  - [monitoring, loki stack, logback]
toc: true
toc_sticky: true
date: 2024-07-31
last_modified_at: 2024-07-31
---

![eks](https://github.com/user-attachments/assets/d94253f8-3026-42fc-9509-57f47c12c43b){: .align-center}

# 0. 작성 취지

사실 이번 블로그는 eks 카테고리에 넣어야 할지, application 카테고리로 따로 빼야 할지에 대해서 상당히 많은 고민을 했다.

하지만, **"logback 설정을 왜 이렇게 해야 하는가?"**의 관점에서 두고 보니 eks 카테고리로 작성하는 것이 맞겠다 싶었다.

테스트는 아주 간단한 애플리케이션 하나를 동작시키는 것이고 로그를 출력해보는 것을 실습해볼 것이다.

테스트는 아래 순서로 진행한다.

1. logback 없이 애플리케이션 동작
2. logback 설정

# 1. logback 없이 애플리케이션 동작

Spring Boot에는 `logback.xml`이라는 log 출력 관련 파일이 존재한다.

`logback.xml` 파일은 로그를 어떻게 관리할 것인지를 설정할 수 있는 파일이다.

```java
package com.may30.test_app;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

@SpringBootApplication
public class TestAppApplication {

	public static void main(String[] args) {
		SpringApplication.run(TestAppApplication.class, args);
	}

	@Controller
    @RequestMapping("/")
    class RootController {
        @GetMapping
        @ResponseBody
        public String root() {
            return "Hello Root";
        }
    }
}
```

(**❗️ 소스 수정은 더이상 이루어지지 않습니다.**)

정말 간단한 Spring Boot Application을 작성했다. `logback.xml` 파일을 작성하지 않고 그냥 동작시키면 어떻게 될까?

## 1-1. 로컬 테스트

```bash
./mvnw clean package
```

위 명령어를 통해서 java를 실행할 수 있는 실행 파일인 jar 파일을 만드는 과정이다.

이를 보통 빌드라고 칭해서 Application을 Bulid 한다고 한다.

```bash
./mvnw spring-boot:run
```

해당 예제로는 로컬에서 테스트를 먼저 해보겠다.

![1](https://github.com/user-attachments/assets/8d22a686-adb9-461f-aa9d-a687ee60a997){: .align-center}

위 사진처럼 실행된 홈페이지를 몇 번을 갱신하더라도 로그는 그 어느 곳에도 쌓이지 않는 것을 확인할 수 있다.

## 1-2. eks 테스트

```bash
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin {aws account 계정번호 12자리}.dkr.ecr.ap-northeast-2.amazonaws.com
```

위 명령어를 통해서 ecr에 login을 할 수 있도록 한다.

이유는 이미지를 레지스트리에 push하기 위해서는 해당 레지스트리에 접속할 수 있는 권한이 필요하기 때문이다.

```bash
docker buildx build --platform=linux/amd64 -t {aws account 계정번호 12자리}.dkr.ecr.ap-northeast-2.amazonaws.com/{ecr 이름}:{이미지 태그} --push .
```

위 명령어로 docker build와 ecr push를 한번에 할 수 있다.

![2](https://github.com/user-attachments/assets/4b166c1f-41f3-40d7-b72e-797f3dd96c3a){: .align-center}

이 상태로 k8s deployment, service, targetgroupbindings(경우에 따라서 ingress)를 생성하고 로그를 확인해보면 위 이미지에서 확인할 수 있듯이 로컬 환경에서처럼 아무런 로그가 찍히지 않는 것을 확인할 수 있다.

# 2. logback 설정

이제 `logback.xml`을 수정해보도록 하자.

`logback.xml`은 `appender`라는 개념을 사용해서 로그 출력 형식을 지정할 수 있다.

그래서 다양한 `appender`를 통해서 여러 가지의 로그 출력 형식을 동시에 가져갈 수 있는 구조이다.

# 2-1. console

우선, console에 로그가 찍히도록 구현해볼 것이다.

```bash
kubectl logs -f {pod 이름}
```

이것은 eks 레벨에서는 위 명령어와 동일한 수준으로 확인할 수 있는 로그 출력이다.

즉, `logback.xml`에서 console appender를 설정해주어야만 `kubectl logs -f` 명령어를 사용할 수 있다.

```xml
<!-- logback.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <!-- Console appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <charset>UTF-8</charset>
            <pattern>
              %d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n
            </pattern>
        </encoder>
    </appender>

    <!-- Root logger -->
    <root level="DEBUG">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

`logback.xml` 설정값을 변경했으니 java build, docker build, ecr push를 해주어야 할 것이다.

해당 과정은 위에서 소개되어 있으므로 중복해서 적진 않을 것이다.

![3](https://github.com/user-attachments/assets/2feeac6c-2efc-4bdb-8994-dc0eac7607d0){: .align-center}

위 사진과 같이 `kubectl logs -f` 명령어로 현재 로그가 찍히는 것을 확인할 수 있으며 이제 로깅이라는 것을 할 수 있게 되는 것이다.

![4](https://github.com/user-attachments/assets/28b37c5e-7a54-4c37-94b5-3ef3b6b6eb1a){: .align-center}

위 사진과 같이 `Loki Stack`까지 설치해서 구성해놓은 상태라면 `grafana`에서 접근해서 로그를 확인할 수 있다.

이는 `promtail`에서 로그를 수집해서 가능한 결과이다.

# 2-2. loki4j

이번 방식은 `loki4j`라는 `appender`를 사용해서 로깅을 구현할 것이다.

이 방법은 `promtail`이 log를 가져가는 방식이 아닌 Application이 `loki`에게 직접 로그를 보내는 방식이다.

`promtail`의 부하를 줄일 수는 있겠지만 `loki`의 부하와 Application이 많이 바쁘다는 단점이 존재하긴 한다.

```xml
<!-- logback.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <!-- Loki appender -->
  <appender name="LOKI" class="com.github.loki4j.logback.Loki4jAppender">
        <http>
            <url>http://loki.{loki가 설치되어 있는 namespace 이름}.svc:3100/loki/api/v1/push</url>
        </http>
        <format>
            <label>
                <pattern>loki=test-app,host=${HOSTNAME}</pattern>
            </label>
            <message>
                <charset>UTF-8</charset>
                <pattern>
                  %d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n
                </pattern>
            </message>
        </format>
    </appender>

  <!-- Root logger -->
    <root level="DEBUG">
        <appender-ref ref="LOKI"/>
    </root>
</configuration>
```

`logback.xml`을 위와 같이 변경한다.

(**❗️ 보다 정확한 테스트를 위해서 console은 삭제했습니다.**)

```xml
<!-- pom.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<dependencies>
		<dependency>
			<groupId>com.github.loki4j</groupId>
			<artifactId>loki-logback-appender</artifactId>
			<version>1.5.1</version>
		</dependency>
	</dependencies>
</project>
```

loki4j를 사용하기 위해서는 `pom.xml`에 dependencies를 위와 같이 추가해주어야 한다.

(**❗️ pom.xml은 새로 추가된 dependency만 추려냈습니다.**)

`logback.xml` 설정값을 변경했으니 java build, docker build, ecr push를 해주어야 할 것이다.

해당 과정은 위에서 소개되어 있으므로 중복해서 적진 않을 것이다.

![5](https://github.com/user-attachments/assets/efbd2972-2822-4ac2-94c7-32a8723ca179){: .align-center}

`logback.xml`에서 console 부분을 지웠기 때문에 위 사진과 같이 `kubectl logs -f` 명령어로는 로그가 쌓이지 않는 것을 확인할 수 있다.

![6](https://github.com/user-attachments/assets/5e62cc50-63f4-49f0-b539-a13a12176dce){: .align-center}

위 사진의 빨간 네모 박스와 같이 `loki: test-app`으로 설정하면 `grafana`에서 접근해서 로그를 확인할 수 있다.

해당 부분은 `logback.xml`의 `label`에 정의해둔 key, value 값이다.

# 3. 마치며

logback 설정을 직접 구현해봄으로써 eks 환경의 로깅 구성을 직접 경험해보았다.

확실히 eks 영역으로 오게 되면서 개발자와 인프라의 경계가 모호해진다는 말이 무엇인지 슬슬 체감이 되기 시작한다.

이렇게 되면 조금이라도 개발자의 시각으로 볼 수 있어 eks에서 가장 편리한 방식으로 확인할 수 있는지에 대한 고민을 할 수 있게 되는 것 같다.
