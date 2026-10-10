---
title: AWS Lambda와 S3 Event로 JSON·Thumbnail 처리
description: Amazon S3 Object 생성 Event로 Lambda를 호출해 JSON 온도 데이터를 기록하고 Pillow Layer를 이용해 Thumbnail을 생성하는 과정을 구성합니다
date: 2026-10-07
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> Amazon S3에 Object가 생성되면 Event Notification이 AWS Lambda를 호출하도록 구성합니다. 첫 번째 함수는 JSON Object의 `temperature`를 읽어 CloudWatch Logs에 기록하고, 두 번째 함수는 원본 이미지를 내려받아 Thumbnail을 별도 S3 Bucket에 저장합니다. S3 Bucket은 공개하지 않고 Lambda Execution Role에 필요한 Object 권한만 부여합니다.

## 1. Serverless와 Lambda

---

Serverless는 Server가 존재하지 않는다는 뜻이 아닙니다. 사용자가 Server의 Provisioning, OS 관리와 Capacity 조정을 직접 수행하지 않고 Function이나 Managed Backend 단위로 Code를 실행하는 운영 모델입니다.

| 구분 | 역할 | AWS 예시 |
| --- | --- | --- |
| FaaS | Event에 반응해 Function Code 실행 | AWS Lambda |
| Managed Backend | Storage, Message와 인증 기능 제공 | Amazon S3, Amazon SNS |
| Observability | Function 실행 Log와 Metric 수집 | Amazon CloudWatch |

Lambda는 Trigger가 전달한 Event를 Handler에 넘깁니다. Handler가 다른 AWS Resource를 사용하려면 Lambda Execution Role에 해당 작업 권한이 있어야 합니다. 반대로 S3가 Lambda를 호출하려면 Lambda의 Resource-based Policy에 `lambda:InvokeFunction` 권한이 필요합니다. Console에서 S3 Trigger를 추가하면 이 호출 권한과 Bucket Event Notification을 함께 구성합니다.

## 2. 실습 구조와 실행 위치

---

이 실습은 다음 위치를 구분합니다.

| 표기 | 실행 위치 | 작업 |
| --- | --- | --- |
| `[AWS-CONSOLE]` | AWS Management Console | S3 Bucket, IAM Role, Lambda Function과 Trigger 구성 |
| `[LOCAL]` | Local Terminal | Pillow Layer Package 생성과 Test File 준비 |
| `[LAMBDA]` | Lambda Execution Environment | JSON 해석, Image Resize와 S3 Object 저장 |

전체 흐름은 다음과 같습니다.

```text
JSON 또는 Image Upload
  ↓
Amazon S3 ObjectCreated Event
  ↓
AWS Lambda
  ├── JSON: temperature 확인 → CloudWatch Logs
  └── Image: Pillow로 Resize → Thumbnail Bucket
```

S3 Bucket과 Lambda Function은 같은 Region에 생성합니다. Bucket 이름은 전 세계에서 고유해야 하므로 아래 이름은 실제 환경에 맞게 바꿉니다.

| Resource | 예제 이름 | 용도 |
| --- | --- | --- |
| JSON Source Bucket | `<JSON_SOURCE_BUCKET>` | `.json` Object 저장 |
| Image Source Bucket | `<IMAGE_SOURCE_BUCKET>` | 원본 Image 저장 |
| Thumbnail Bucket | `<THUMBNAIL_BUCKET>` | Resize 결과 저장 |
| JSON Function | `temperature-event-handler` | JSON 온도 데이터 처리 |
| Image Function | `thumbnail-event-handler` | Thumbnail 생성 |

## 3. 공개 Bucket Policy 대신 Execution Role 사용

---

S3 Event를 처리하기 위해 Bucket을 공개할 필요는 없습니다. S3 Block Public Access를 유지하고 Lambda Execution Role에 필요한 권한만 부여합니다.

Lambda가 Log를 기록할 수 있도록 AWS Managed Policy `AWSLambdaBasicExecutionRole`을 Role에 연결합니다. S3 Object 접근은 다음과 같은 Inline Policy로 제한합니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadJsonAndSourceImages",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": [
        "arn:aws:s3:::<JSON_SOURCE_BUCKET>/*",
        "arn:aws:s3:::<IMAGE_SOURCE_BUCKET>/*"
      ]
    },
    {
      "Sid": "WriteThumbnails",
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::<THUMBNAIL_BUCKET>/*"
    }
  ]
}
```

`s3:ListBucket`, `s3:DeleteObject`와 공개 `Principal`은 이 실습에 필요하지 않습니다. Function마다 역할을 분리하면 JSON Function에는 Source JSON `GetObject`만, Thumbnail Function에는 Source Image `GetObject`와 Destination `PutObject`만 부여할 수 있습니다.

## 4. JSON 온도 데이터 처리

---

### 4.1 Lambda Function 생성

`[AWS-CONSOLE]`에서 Python Runtime Lambda Function을 생성하고 앞에서 만든 Execution Role을 지정합니다. Function Code는 S3 Event의 모든 Record를 순회하도록 작성합니다.

```python
import json
import logging
from urllib.parse import unquote_plus

import boto3


logger = logging.getLogger()
logger.setLevel(logging.INFO)
s3_client = boto3.client("s3")


def lambda_handler(event, context):
    results = []

    for record in event.get("Records", []):
        bucket = record["s3"]["bucket"]["name"]
        key = unquote_plus(record["s3"]["object"]["key"])
        event_time = record.get("eventTime", "unknown")

        response = s3_client.get_object(Bucket=bucket, Key=key)
        payload = json.loads(response["Body"].read().decode("utf-8"))
        temperature = payload["temperature"]

        if temperature > 40:
            message = "주의하세요. 매우 덥습니다."
        else:
            message = "나쁘지 않은 날씨입니다."

        logger.info(
            "temperature=%s event_time=%s bucket=%s key=%s message=%s",
            temperature,
            event_time,
            bucket,
            key,
            message,
        )
        results.append({"key": key, "temperature": temperature, "message": message})

    return {"processed": len(results), "results": results}
```

| 처리 | 이유 |
| --- | --- |
| `unquote_plus()` | S3 Event의 URL Encoding된 Object Key 복원 |
| `event.get("Records", [])` | 한 Event에 여러 Record가 포함되는 경우 처리 |
| `record["eventTime"]` | Object 생성 Event가 기록한 UTC 시각 사용 |
| `logger.info()` | CloudWatch Logs에서 Field를 구분해 조회 |

Code를 저장한 뒤 `Deploy`를 눌러 현재 Version에 반영합니다.

### 4.2 S3 Event Notification 생성

`[AWS-CONSOLE]`에서 JSON Source Bucket의 `Properties → Event notifications`로 이동해 다음 값을 설정합니다.

| 항목 | 값 |
| --- | --- |
| 이름 | `temperature-json-created` |
| Event Type | `All object create events` |
| Suffix | `.json` |
| Destination | `temperature-event-handler` Lambda Function |

이 Function은 생성된 Object를 읽으므로 Object 삭제 Event를 연결하지 않습니다.

### 4.3 Test와 Log 확인

다음 Test File을 JSON Source Bucket에 Upload합니다.

```json
{
  "temperature": 45
}
```

CloudWatch에서 `/aws/lambda/temperature-event-handler` Log Group의 최신 Log Stream을 확인합니다. `temperature=45`, Object Key와 Event Time이 기록되면 S3 Event 전달과 Function 실행이 완료된 것입니다.

잘못된 JSON, `temperature` Field 누락 또는 숫자가 아닌 값은 Function Error가 됩니다. 운영 Code에서는 Schema 검증과 실패 Event를 보관할 Destination 또는 Dead-letter 처리를 추가로 설계합니다.

## 5. Image Thumbnail 생성

---

### 5.1 Source와 Destination 분리

Image Source Bucket과 Thumbnail Bucket을 별도로 사용합니다.

```text
<IMAGE_SOURCE_BUCKET>/photo.jpg
  ↓ ObjectCreated Event
thumbnail-event-handler
  ↓ Resize
<THUMBNAIL_BUCKET>/photo.jpg
```

Trigger Bucket에 결과를 다시 쓰면 새 Object가 Function을 반복 호출하는 Recursive Invocation이 발생할 수 있습니다. 별도 Destination Bucket을 사용하는 이유입니다.

Thumbnail Function의 Environment Variable에 다음 값을 설정합니다.

| Key | Value |
| --- | --- |
| `THUMBNAIL_BUCKET` | `<THUMBNAIL_BUCKET>` |

### 5.2 Function Code 작성

Pillow는 Lambda Python Runtime에 기본 포함되지 않으므로 다음 절에서 Layer로 추가합니다. Function Code는 Source Image를 `/tmp`에 내려받고 원본 크기의 절반을 넘지 않도록 축소합니다.

```python
import logging
import os
import tempfile
from pathlib import Path
from urllib.parse import unquote_plus

import boto3
from PIL import Image, ImageOps


logger = logging.getLogger()
logger.setLevel(logging.INFO)
s3_client = boto3.client("s3")
thumbnail_bucket = os.environ["THUMBNAIL_BUCKET"]


def resize_image(source_path, destination_path):
    with Image.open(source_path) as source_image:
        image_format = source_image.format
        image = ImageOps.exif_transpose(source_image)
        target_size = (
            max(1, image.width // 2),
            max(1, image.height // 2),
        )
        image.thumbnail(target_size)
        image.save(destination_path, format=image_format)


def lambda_handler(event, context):
    processed = []

    for record in event.get("Records", []):
        source_bucket = record["s3"]["bucket"]["name"]
        source_key = unquote_plus(record["s3"]["object"]["key"])
        suffix = Path(source_key).suffix

        with tempfile.TemporaryDirectory(dir="/tmp") as work_dir:
            source_path = os.path.join(work_dir, f"source{suffix}")
            output_path = os.path.join(work_dir, f"thumbnail{suffix}")

            s3_client.download_file(source_bucket, source_key, source_path)
            resize_image(source_path, output_path)
            s3_client.upload_file(output_path, thumbnail_bucket, source_key)

        logger.info(
            "source=s3://%s/%s destination=s3://%s/%s",
            source_bucket,
            source_key,
            thumbnail_bucket,
            source_key,
        )
        processed.append(source_key)

    return {"processed": len(processed), "keys": processed}
```

`/tmp`는 Lambda 실행 환경에서 Function이 쓸 수 있는 임시 Storage입니다. 실행 환경 재사용 여부에 의존하지 않도록 호출마다 임시 Directory를 만들고 처리 후 제거합니다.

### 5.3 Pillow Layer 생성

Pillow는 Native Library를 포함하므로 Lambda와 같은 Linux 환경, 같은 Python Version과 같은 CPU Architecture를 기준으로 Package를 만들어야 합니다. 다음 예제는 Python 3.14, `x86_64` Function 기준입니다.

```bash
# [LOCAL]
mkdir -p lambda-layer/python

docker run --rm \
  --platform linux/amd64 \
  --volume "$PWD/lambda-layer:/var/task" \
  public.ecr.aws/sam/build-python3.14 \
  /bin/bash -c 'pip install --no-cache-dir pillow boto3 -t /var/task/python'

cd lambda-layer
zip -r ../pillow-layer.zip python
cd ..
```

| 항목 | 조건 |
| --- | --- |
| Runtime | Function과 Layer의 Python Version 일치 |
| Architecture | `x86_64`이면 `linux/amd64`, `arm64`이면 `linux/arm64` 사용 |
| ZIP Root | 최상위에 `python/` Directory 존재 |
| Build OS | Lambda에서 실행할 수 있는 Linux 호환 Package 생성 |

`[AWS-CONSOLE]`에서 Lambda Layer를 만들고 `pillow-layer.zip`을 Upload한 뒤 Thumbnail Function에 연결합니다. Layer의 Compatible Runtime과 Function Runtime이 일치하는지 확인합니다.

### 5.4 Trigger와 결과 검증

Image Source Bucket에 `All object create events` Notification을 추가하고 Thumbnail Function을 Destination으로 선택합니다. 필요하면 `.jpg`, `.jpeg` 또는 `.png` Suffix별 Notification을 구분합니다.

검증 순서는 다음과 같습니다.

1. Image Source Bucket에 Test Image를 Upload합니다.

2. Thumbnail Bucket에 같은 Key의 Object가 생성됐는지 확인합니다.

3. 두 Object의 Image 가로·세로 크기를 비교합니다.

4. `/aws/lambda/thumbnail-event-handler` Log Group에서 Source와 Destination을 확인합니다.

`No module named PIL`이면 Layer 연결과 ZIP의 `python/` Directory를 확인합니다. `cannot load shared object`와 같은 Error가 발생하면 Build OS 또는 Architecture가 Function과 다른지 확인합니다.

## 6. Resource 정리

---

Test가 끝나면 다음 순서로 정리합니다.

1. Source Bucket의 Event Notification을 제거합니다.

2. Source와 Thumbnail Bucket의 Object를 비웁니다.

3. 세 S3 Bucket을 삭제합니다.

4. 두 Lambda Function과 Pillow Layer Version을 삭제합니다.

5. 실습용 IAM Policy와 Execution Role을 삭제합니다.

6. CloudWatch Log Group을 더 보관할 필요가 없으면 삭제합니다.

> **최종 정리**
> - S3 `ObjectCreated` Event는 Lambda Function을 자동으로 호출할 수 있습니다.
>
> - S3 Bucket을 공개하지 않고 Lambda Execution Role에 필요한 Object 권한만 부여합니다.
>
> - Event의 Object Key는 URL Decoding한 뒤 사용합니다.
>
> - Trigger Source와 결과 Bucket을 분리해 Recursive Invocation을 방지합니다.
>
> - Pillow Layer는 Lambda와 같은 Python Version, Linux 환경과 CPU Architecture에 맞춰 Package로 만듭니다.

## 참고 자료

---

- [Amazon S3 Trigger로 Lambda 호출](https://docs.aws.amazon.com/lambda/latest/dg/with-s3-example.html)

- [S3 Event Notification Type과 Destination](https://docs.aws.amazon.com/AmazonS3/latest/userguide/notification-how-to-event-types-and-destinations.html)

- [Python Lambda Layer 생성](https://docs.aws.amazon.com/lambda/latest/dg/python-layers.html)

- [Lambda Layer Package 구조](https://docs.aws.amazon.com/lambda/latest/dg/packaging-layers.html)
