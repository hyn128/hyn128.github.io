---
title: CloudWatch Metric·Logs·Alarm과 SNS 알림
description: CloudWatch의 Metric, Logs, Dashboard와 Alarm 상태를 구분하고 EC2 Detailed Monitoring, SNS Email 알림과 CPU 부하 검증 절차를 구성합니다
date: 2026-10-06
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> Amazon CloudWatch는 AWS Resource와 Application의 Metric, Log와 Alarm을 수집·조회하는 Monitoring Service입니다. EC2 CPU Metric을 기준으로 Alarm을 만들고 Amazon SNS Email 구독으로 상태 변경을 알립니다. Metric, Log와 Event의 차이를 먼저 구분한 뒤 Alarm 생성, CPU 부하 발생, 상태 확인과 Resource 정리 순서로 실습합니다.

## 1. CloudWatch에서 다루는 데이터

---

CloudWatch의 Metric과 Log는 다른 데이터입니다.

| 구분 | 내용 | 예시 |
| --- | --- | --- |
| Metric | 시간에 따른 수치 데이터 | EC2 `CPUUtilization`, EBS `VolumeReadOps` |
| Logs | Application과 System이 남긴 Text Event | Web Access Log, Container Standard Output |
| Alarm | Metric이나 Metric Math 결과를 임계값과 비교한 상태 | 평균 CPU가 70%를 초과하면 `ALARM` |
| Dashboard | Metric과 Log Widget을 한 화면에 배치 | EC2·EBS 운영 현황 |
| Event | AWS Resource 상태 변화나 일정 | Instance 상태 변경, 매일 정해진 시간 실행 |

CloudWatch는 Log를 자동으로 Metric Graph로 바꾸지 않습니다. AWS Service가 발행한 Metric을 Graph로 표시하거나, Log에서 Metric Filter를 만들어 필요한 수치를 별도로 발행합니다.

CloudWatch Events 기능은 Amazon EventBridge로 확장됐습니다. 기존 Events API는 호환되지만 새로운 Event Pattern과 Schedule Rule은 EventBridge Console에서 관리합니다.

## 2. CloudWatch Metrics

---

Metric은 다음 요소로 식별됩니다.

| 요소 | 역할 | EC2 예제 |
| --- | --- | --- |
| Namespace | Metric을 묶는 범위 | `AWS/EC2` |
| Metric Name | 측정 항목 | `CPUUtilization` |
| Dimension | Resource 식별 조건 | `InstanceId=<EC2_INSTANCE_ID>` |
| Statistic | 기간 내 집계 방식 | `Average`, `Maximum`, `Sum` |
| Period | 한 Data Point가 집계하는 시간 | 60초, 300초 |
| Unit | 값의 단위 | `Percent` |

`Average CPUUtilization`과 `Maximum CPUUtilization`은 같은 원본 Metric을 서로 다르게 해석합니다. Alarm 목적에 맞는 Statistic과 Period를 선택합니다.

## 3. EC2 기본 Monitoring과 상세 Monitoring

---

EC2 Monitoring 주기는 다음과 같습니다.

| 구분 | EC2 Metric 주기 | 비용 | 용도 |
| --- | --- | --- | --- |
| Basic Monitoring | 5분 | 추가 비용 없음 | 장기 추세와 완만한 변화 확인 |
| Detailed Monitoring | 1분 | 추가 비용 발생 | 짧은 Spike와 빠른 Alarm 판단 |

Monitoring 주기는 AWS Service마다 다릅니다. EC2의 5분·1분 구분을 모든 Service에 동일하게 적용하지 않습니다.

EC2 기본 Metric에는 CPU와 Network Traffic이 포함되지만 File System 사용률이나 Memory 사용률은 포함되지 않습니다. OS 내부 값은 CloudWatch Agent 같은 수집기를 설치해 Custom Metric으로 보내야 합니다. EBS Volume의 I/O는 `AWS/EBS` Namespace에서 확인합니다.

현재 EC2 Instance에 Detailed Monitoring이 적용됐는지 확인합니다.

```bash
# [OPERATOR]
aws ec2 describe-instances \
  --instance-ids <EC2_INSTANCE_ID> \
  --query 'Reservations[0].Instances[0].Monitoring.State' \
  --output text
```

1분 주기 CPU Alarm을 검증할 때 Detailed Monitoring을 활성화합니다.

```bash
# [OPERATOR]
aws ec2 monitor-instances \
  --instance-ids <EC2_INSTANCE_ID>
```

실습 후 필요하지 않으면 추가 비용을 막기 위해 해제합니다.

```bash
# [OPERATOR]
aws ec2 unmonitor-instances \
  --instance-ids <EC2_INSTANCE_ID>
```

## 4. CloudWatch Logs

---

CloudWatch Logs는 다음 계층으로 데이터를 관리합니다.

```text
Log Group
└── Log Stream
    └── Log Event
```

| 계층 | 역할 |
| --- | --- |
| Log Group | 같은 Retention과 접근 정책을 적용할 Log 묶음 |
| Log Stream | 하나의 Instance, Container 또는 Process가 보내는 Log 흐름 |
| Log Event | Timestamp와 Message로 구성된 개별 기록 |

Log Group에는 Retention을 설정합니다. 기간을 지정하지 않으면 Log가 계속 보관돼 Storage 비용이 증가할 수 있습니다.

```bash
# [OPERATOR]
aws logs put-retention-policy \
  --log-group-name /example/application \
  --retention-in-days 30
```

CloudWatch Logs Insights에서는 필요한 Field를 선택하고 조건을 적용해 Log를 조회합니다.

```text
fields @timestamp, @message, @logStream
| filter @message like /ERROR/
| sort @timestamp desc
| limit 50
```

Log Group의 일정 기간을 S3로 Export할 수 있습니다. 지속적인 Archive가 필요하면 반복 Export Task보다 Subscription을 이용해 전달하는 구성을 검토합니다.

## 5. Dashboard

---

Dashboard는 여러 Metric과 Log Query를 한 화면에 배치합니다.

```text
Operations Dashboard
├── EC2 CPUUtilization
├── EC2 NetworkIn·NetworkOut
├── EBS Read·Write Operations
├── Alarm Status
└── Application ERROR Log Query
```

Widget마다 Region, Namespace, Dimension과 Period를 확인합니다. 서로 다른 Period와 Statistic을 한 Graph에 섞으면 값의 의미를 잘못 해석할 수 있습니다.

자동 Dashboard는 Service Resource를 빠르게 확인할 때 사용합니다. 운영 Dashboard에는 Service Level과 장애 판단에 필요한 Metric만 배치합니다.

## 6. Alarm 상태

---

CloudWatch Alarm은 세 상태 중 하나입니다.

| 상태 | 의미 |
| --- | --- |
| `OK` | 평가한 Data Point가 Alarm 조건을 충족하지 않습니다. |
| `ALARM` | 설정한 수의 Data Point가 임계 조건을 충족합니다. |
| `INSUFFICIENT_DATA` | 생성 직후이거나 평가할 Data Point가 부족합니다. |

`INSUFFICIENT_DATA`는 항상 장애를 뜻하지 않습니다. Metric 발행 주기, Dimension, Period와 Missing Data 처리 방식을 확인해야 합니다.

Alarm은 상태가 바뀔 때 SNS 알림, EC2 Action이나 Auto Scaling Action을 실행할 수 있습니다. Alarm 자체가 원인을 분석하거나 Instance 성능을 임의로 낮추지는 않습니다.

## 7. 실습 Resource 준비

---

다음 Resource를 준비합니다.

| Resource | 예제 | 목적 |
| --- | --- | --- |
| EC2 Instance | Ubuntu, `<EC2_INSTANCE_ID>` | CPU 부하 발생 대상 |
| SNS Topic | `ec2-cpu-alarm-topic` | Alarm 상태 변경 전달 |
| Email Endpoint | `<ALERT_EMAIL>` | 알림 수신과 구독 확인 |
| CloudWatch Alarm | `ec2-high-cpu` | CPU 임계값 평가 |

EC2 접속은 Session Manager를 우선 사용합니다. SSH가 필요한 환경에서는 Security Group의 Port 22를 관리 Client IP로 제한하고 Private Key를 Repository에 저장하지 않습니다.

## 8. SNS Topic과 Email 구독

---

SNS Topic을 생성합니다.

```bash
# [OPERATOR]
aws sns create-topic \
  --name ec2-cpu-alarm-topic
```

출력된 Topic ARN은 다음 형식입니다.

```text
arn:aws:sns:<AWS_REGION>:<AWS_ACCOUNT_ID>:ec2-cpu-alarm-topic
```

Email 구독을 만듭니다.

```bash
# [OPERATOR]
aws sns subscribe \
  --topic-arn <SNS_TOPIC_ARN> \
  --protocol email \
  --notification-endpoint '<ALERT_EMAIL>'
```

AWS가 보낸 Subscription Confirmation Email에서 Confirm Link를 눌러야 알림을 받을 수 있습니다. 상태를 확인합니다.

```bash
# [OPERATOR]
aws sns list-subscriptions-by-topic \
  --topic-arn <SNS_TOPIC_ARN>
```

`SubscriptionArn`이 `PendingConfirmation`이면 아직 확인되지 않은 상태입니다.

## 9. EC2 CPU Alarm 생성

---

평균 CPU 사용률이 2분 연속 70%를 초과하면 `ALARM`이 되도록 설정합니다. 이 예제는 Detailed Monitoring이 활성화된 EC2 Instance를 전제로 합니다.

```bash
# [OPERATOR]
aws cloudwatch put-metric-alarm \
  --alarm-name ec2-high-cpu \
  --alarm-description 'Average CPU exceeds 70 percent for two minutes' \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=<EC2_INSTANCE_ID> \
  --statistic Average \
  --period 60 \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data missing \
  --alarm-actions <SNS_TOPIC_ARN> \
  --ok-actions <SNS_TOPIC_ARN>
```

| Option | 역할 |
| --- | --- |
| `--period 60` | 60초 단위로 Metric을 집계합니다. |
| `--evaluation-periods 2` | 최근 두 구간을 평가합니다. |
| `--datapoints-to-alarm 2` | 두 Data Point가 모두 조건을 충족해야 Alarm이 됩니다. |
| `--treat-missing-data missing` | 누락 값을 정상이나 장애로 임의 변환하지 않습니다. |
| `--alarm-actions` | `ALARM` 전환 시 SNS Topic으로 알립니다. |
| `--ok-actions` | 정상 복구 시에도 SNS Topic으로 알립니다. |

생성한 설정과 현재 상태를 확인합니다.

```bash
# [OPERATOR]
aws cloudwatch describe-alarms \
  --alarm-names ec2-high-cpu \
  --query 'MetricAlarms[0].{state:StateValue,reason:StateReason,period:Period,evaluationPeriods:EvaluationPeriods,threshold:Threshold}'
```

## 10. CPU 부하 발생과 검증

---

Ubuntu EC2에 접속한 뒤 CPU 수를 확인합니다.

```bash
# [EC2]
nproc
```

`stress` Package를 설치합니다.

```bash
# [EC2]
sudo apt-get update
sudo apt-get install -y stress
```

두 CPU Worker에 5분 동안 부하를 발생시킵니다.

```bash
# [EC2]
stress --cpu 2 --timeout 300
```

다음 순서로 결과를 확인합니다.

1. CloudWatch의 EC2 `CPUUtilization` Graph가 상승하는지 확인합니다.

2. Alarm 상태가 `INSUFFICIENT_DATA` 또는 `OK`에서 `ALARM`으로 바뀌는지 확인합니다.

3. SNS Email이 도착했는지 확인합니다.

4. 부하 종료 후 Alarm이 `OK`로 돌아오고 복구 Email이 오는지 확인합니다.

Metric 수집, Alarm 평가와 SNS 전달에는 시간이 걸립니다. `stress` 실행 직후 상태가 바뀌지 않았다고 설정 실패로 단정하지 않습니다.

## 11. Network와 EBS Metric 확인

---

File 전송으로 Network와 EBS I/O 변화를 확인할 수 있습니다. SSH가 허용된 실습 환경에서 다음 Placeholder를 실제 경로와 Host로 교체합니다.

```bash
# [OPERATOR]
scp \
  -i <PRIVATE_KEY_PATH> \
  <LOCAL_LARGE_FILE> \
  <EC2_USER>@<EC2_PUBLIC_DNS>:/tmp/
```

전송 전후에 다음 Metric을 확인합니다.

| Namespace | Metric | 의미 |
| --- | --- | --- |
| `AWS/EC2` | `NetworkIn`, `NetworkOut` | Instance Network Byte |
| `AWS/EBS` | `VolumeReadOps`, `VolumeWriteOps` | EBS I/O Operation 수 |
| `AWS/EBS` | `VolumeReadBytes`, `VolumeWriteBytes` | EBS 전송 Byte |

File System의 남은 용량은 기본 EC2 Metric이 아닙니다. EC2 내부에서 `df -h`로 확인하거나 CloudWatch Agent로 `disk_used_percent` Custom Metric을 발행합니다.

## 12. EventBridge와 Schedule

---

AWS Resource Event나 일정에 반응하는 Rule은 EventBridge에서 생성합니다.

```text
AWS Service Event 또는 Schedule
  ↓ EventBridge Rule
Target
  ├── Lambda
  ├── SNS
  ├── SQS
  └── Step Functions
```

Event Pattern은 Source, Detail Type과 세부 조건으로 필요한 Event만 선택합니다. Schedule은 Cron 또는 Rate 표현식으로 실행 시점을 정의합니다. CloudWatch Alarm의 Metric 평가와 EventBridge Rule의 Event Routing은 목적이 다릅니다.

## 13. Resource 정리

---

실습이 끝나면 Alarm, Subscription과 Topic을 정리합니다.

```bash
# [OPERATOR]
aws cloudwatch delete-alarms \
  --alarm-names ec2-high-cpu

aws sns unsubscribe \
  --subscription-arn <SNS_SUBSCRIPTION_ARN>

aws sns delete-topic \
  --topic-arn <SNS_TOPIC_ARN>
```

Detailed Monitoring을 유지하지 않을 경우 해제하고, 실습용 EC2 Instance도 중지 또는 종료합니다. Dashboard, Custom Metric, Log Storage, Alarm과 SNS 전송에는 항목별 비용이 발생할 수 있습니다.

> **최종 정리**
> - Metric은 수치 시계열이고 Logs는 개별 Text Event입니다.
>
> - EC2 Basic Monitoring은 5분, Detailed Monitoring은 1분 주기로 Metric을 발행합니다.
>
> - Memory와 File System 사용률은 기본 EC2 Metric이 아니므로 CloudWatch Agent가 필요합니다.
>
> - Alarm은 `OK`, `ALARM`, `INSUFFICIENT_DATA` 상태를 가지며 SNS로 상태 변경을 전달할 수 있습니다.
>
> - CloudWatch Events의 Event Routing 기능은 현재 Amazon EventBridge에서 관리합니다.

## 참고 자료

---

- [CloudWatch Basic Monitoring과 Detailed Monitoring](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-metrics-basic-detailed.html)

- [CloudWatch Alarm 상태](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)

- [CloudWatch Logs Insights Query](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)

- [CloudWatch Logs를 S3로 Export](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/S3Export.html)

- [Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html)
