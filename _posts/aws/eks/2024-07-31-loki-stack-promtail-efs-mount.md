---
title: '[EKS] Loki Stack Promtail EFS Mount'
categories:
  - eks
tags:
  - [monitoring, loki stack]
toc: true
toc_sticky: true
date: 2024-07-31
last_modified_at: 2024-07-31
---

![aws-eks](https://github.com/user-attachments/assets/8f4254fa-6720-400b-a77e-f4269cb100e1){: .align-center}

# 0. 작성 취지

Promtail은 로그를 전부 수집하여 Loki에게 전송하는 역할을 하고 있다.

만약, Java(Spring Boot)로 이루어진 Application Pod에서 실시간으로 로그를 확인하고 싶다면 `kubectl logs -f` 명령어를 입력하면 된다.

하지만 저 명령어도 전제조건이 있는데 Spring Boot에서는 로그를 무슨 형태로 출력시킬 것인가에 대해서 여러가지 옵션이 있고 그 중에서 대표적으로 `logback.xml` 이라는 파일이 존재한다.

`logback.xml` 설정에 대한 간단한 실습 내용은 [링크](https://may-30.github.io/eks/loki-stack-logback/)에서 확인할 수 있다.

그런데 **"실시간으로 확인할 필요 없이 Application Pod는 EFS에 마운트해서 로그를 파일 형태로 떨구고 있다면 Promtail은 Application Pod를 어떻게 가져와야 할까?"** 하는 의문이 생겼다.

답은 Promtail도 Application Pod가 쌓고 있는 EFS에 동일하게 마운트를 해주어야 한다는 것이다.

EFS로 로그를 쌓는 이유는 eks에서는 Application pod가 여러 개, 그리고 서로 다른 가용 영역에 흩어져서 동작하도록 구성하는 것이 권장 사항이기 때문에 하나의 인스턴스에 묶여있는 EBS 구조보단 여러 개의 인스턴스에서 동시에 접근할 수 있는 EFS로 구성하는 것이 맞다고 판단했다.

문제는 Loki Stack을 helm 형태로 만들어두었으니 이걸 변경해주어야 한다는 것이다.

아래 순서와 같이 진행하려고 한다.

1. Logback 파일 수정
2. Promtail 파일 수정
3. 테스트

# 1. Logback 파일 수정

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- File appender with size and time-based rolling -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/%d{yyyy-MM-dd-HH}.%i.log</fileNamePattern>

            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <encoder>
            <charset>UTF-8</charset>
            <pattern>
                %d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n
            </pattern>
        </encoder>
    </appender>

    <!-- Root logger -->
    <root level="DEBUG">
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

`logback.xml` 파일에서 `file appender`를 추가해서 로그를 파일로 출력할 수 있도록 설정한다.

나의 경우에는 `logs` 라는 디렉터리에 생성할 수 있도록 구성했다.

`logback.xml` 설정값을 새롭게 설정했으니 java build, docker build, ecr push를 해주어야 할 것이다.

다만 이번 포스팅에서는 다루지 않을 것이다.

# 2. Promtail 파일 수정

## 2-1. storage 생성

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: loki-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: loki-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  storageClassName: efs
  csi:
    driver: efs.csi.aws.com
    volumeHandle: # EFS ID::EFS ACCESS POINT ID
```

먼저 위의 yaml 파일을 참고해서 promtail에서도 Application에서 사용 중인 log 경로를 참고할 수 있도록 storage를 구성한다.

## 2-2. promtail values 수정

```yaml
promtail:
  enabled: true
  extraArgs:
    - -config.expand-env=true
  config:
    position:
      filename: /tmp/positions.yaml
    snippets:
      extraScrapeConfigs: |
        - job_name: app-logs
          static_configs:
          - targets:
            - localhost
            labels:
              job: app/logs
              __path__: /logs/*.log
  extraVolumes:
    - name: loki-efs
      persistentVolumeClaim:
        claimName: loki-pvc
  extraVolumeMounts:
    - name: loki-efs
      mountPath: /logs
```

`Loki Stack`의 helm values를 위와 같이 수정한다.

여러 가지 방법으로 테스트를 진행했는데 실제 config를 안에 기입해주는 방식인 위 방식으로 작성해주어야 promtail에서 설정값을 읽어드리는 것을 확인했다.

추가로 직전에 생성해둔 storage를 promtail에서도 볼 수 있도록 동일하게 구성한다.

## 2-3. helm upgrade

```bash
helm -n monitoring upgrade loki grafana/loki-stack -f loki-stack.yaml
```

위 명령어를 통해서 promtail의 변경사항을 적용할 수 있도록 한다.

# 3. 테스트

## 3-1. Application EFS Mount 확인

![1](https://github.com/user-attachments/assets/e096e164-222c-46bf-9205-3454d4d93f43){: .align-center}

빨간색 네모 박스를 확인하면 EFS에 정상적으로 마운트가 된 모습을 확인할 수 있다.

## 3-2. Application Logs 확인

![2](https://github.com/user-attachments/assets/450f3848-0706-44d5-9fe9-493200065c6b){: .align-center}

위와 같이 마운트 된 EFS에 정상적으로 log 파일이 쌓이는 모습을 확인할 수 있다.

## 3-3. Promtail Logs 확인

![3](https://github.com/user-attachments/assets/4c21b073-6bf9-4022-b11a-eb4096312087){: .align-center}

promtail에서도 동일하게 log 파일을 확인할 수 있다.

## 3-4. Grafana 확인

![4](https://github.com/user-attachments/assets/834c26ba-25fb-43bc-9fd4-1990cc77e78d){: .align-center}

grafana에서는 filename으로 key를 잡고 시간대에 해당하는 로그를 선택해서 로깅을 분석할 수 있다.

# 4. 마치며

안 그래도 Application에서는 많은 일을 처리하고 있는데 실시간으로 로그를 뿜어내고 파일 형태로 로그를 찍어서 만들어내면 부하가 쉽게 발생하지 않을까 걱정이 되었는데 이번 테스트로 인해서 해당 부분에 대한 부담이 완화되었다.
