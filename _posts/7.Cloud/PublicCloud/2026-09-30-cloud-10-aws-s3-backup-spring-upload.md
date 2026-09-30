---
title: Amazon S3 Backup과 Spring Boot File Upload
description: AWS Storage Gateway와 S3 Lifecycle을 이용한 Backup 구조를 비교하고 Private S3 Bucket에 Spring Boot Application이 File을 안전하게 업로드하는 방법을 정리합니다
date: 2026-09-30
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> Amazon S3는 Key로 Data를 저장하는 Object Storage입니다. Backup File, Image와 정적 Asset을 Bucket에 Object로 저장하고 Storage Class와 Lifecycle로 보관 비용을 조정할 수 있습니다. 이 글에서는 On-premises와 AWS Workload의 Backup 방식을 비교하고, Public Access를 차단한 S3 Bucket에 Spring Boot Application이 IAM Role로 File을 업로드하는 구성을 정리합니다.

## 1. Backup 대상과 복구 목표

---

Backup에는 Data 복사와 Restore 검증이 모두 포함됩니다. 먼저 다음 값을 결정합니다.

| 항목 | 질문 |
| --- | --- |
| RPO | 장애가 발생했을 때 얼마만큼의 Data 손실을 허용할 수 있는가 |
| RTO | 장애 이후 Service를 얼마 안에 복구해야 하는가 |
| 보존 기간 | 일·월·연 단위로 Backup을 얼마나 오래 보관하는가 |
| 일관성 | File 복사만으로 충분한가, Database Online Backup이 필요한가 |
| 복구 위치 | 동일 Region, 다른 Region 또는 On-premises 중 어디에서 복구하는가 |

Database Data File을 실행 중인 상태에서 그대로 복사하면 Transaction이 완전히 반영되지 않은 Backup이 될 수 있습니다. Engine이 제공하는 Dump, Snapshot 또는 Application-consistent Backup 절차를 사용합니다.

## 2. AWS를 이용한 Backup 방식

---

### 2.1 AWS CLI와 S3

File 단위 Backup은 AWS CLI 또는 SDK로 S3에 직접 Upload할 수 있습니다.

```text
On-premises Server
  ↓ HTTPS, AWS CLI·SDK
S3 Bucket
  ↓ Lifecycle
S3 Glacier Storage Class
```

구성이 단순하지만 Backup Scheduling, 실패 재시도, 암호화, 전송량과 복구 절차를 직접 관리해야 합니다.

### 2.2 EC2와 EBS Backup Server

EC2와 EBS로 Backup Server를 구성하면 Database Utility와 기존 Script를 유연하게 실행할 수 있습니다. 반면 Instance, OS, EBS Capacity, Patch와 Backup Software를 직접 운영하며 EBS Snapshot과 Data Transfer 비용도 고려해야 합니다.

### 2.3 AWS Storage Gateway

S3 File Gateway는 On-premises에 Virtual Appliance를 배치하고 NFS 또는 SMB File Share를 제공합니다. Client가 Share에 기록한 File은 Local Cache를 거쳐 S3 Object로 비동기 Upload됩니다.

```text
On-premises Client
  ↓ NFS 또는 SMB
S3 File Gateway VM
  ├── Local Cache
  └── HTTPS :443
        ↓
      S3 Bucket
```

File Gateway는 File Interface를 제공하지만 일반 NAS와 완전히 같은 File System은 아닙니다. Rename이나 Metadata 변경이 S3 Object 재생성으로 이어질 수 있으며 Versioning과 Lifecycle 설정에 따라 예상하지 못한 저장·조기 삭제 비용이 발생할 수 있습니다.

## 3. Amazon S3 기본 구조

---

S3(Simple Storage Service)는 Bucket 안에 Object를 저장하는 Regional Object Storage Service입니다.

| 개념 | 설명 |
| --- | --- |
| Bucket | Object를 담는 최상위 Container이며 생성 Region을 변경할 수 없음 |
| Object | Data, Key와 Metadata로 구성된 저장 단위 |
| Key | Bucket 안에서 Object를 식별하는 전체 이름 |
| Prefix | `/` 구분자를 이용해 Folder처럼 보이게 하는 Key의 앞부분 |
| Version ID | Versioning 사용 시 같은 Key의 Object Version 식별자 |

S3 Console은 Key Prefix를 File System Directory처럼 표시합니다. 실제 Directory가 별도로 생성되는 구조는 아닙니다.

```text
Bucket: example-private-upload

Object Key
uploads/profile/550e8400-e29b-41d4-a716-446655440000.png
└──────── Prefix ────────┘└──────── Object Name ────────┘
```

General Purpose Bucket Name은 AWS Partition의 모든 Account에서 고유해야 합니다. Account당 기본 Bucket Quota는 10,000개이며 필요하면 Service Quotas에서 증가를 요청할 수 있습니다. 하나의 Object는 최대 50 TB이고, 5 TB를 초과하는 Object는 Multipart Upload와 분할 Download가 필요합니다.

## 4. Storage Class와 Lifecycle

---

S3 Standard를 포함한 다수의 Storage Class는 99.999999999%의 Data Durability를 목표로 설계되었습니다. Availability, 최소 보관 기간과 Retrieval 비용은 Storage Class마다 다릅니다.

| Storage Class | 대표 용도 | 주의 사항 |
| --- | --- | --- |
| S3 Standard | 자주 접근하는 일반 Data | 기본 저장 비용 |
| S3 Intelligent-Tiering | 접근 Pattern이 바뀌거나 예측하기 어려운 Data | Monitoring·Automation 비용 확인 |
| S3 Standard-IA | 자주 접근하지 않지만 즉시 복구할 Data | 최소 보관 기간과 Retrieval 비용 |
| S3 One Zone-IA | 재생성할 수 있는 비중요 Data | 단일 Availability Zone 저장 |
| S3 Glacier Instant Retrieval | Archive이지만 Millisecond 접근 필요 | 최소 보관 기간과 Retrieval 비용 |
| S3 Glacier Flexible Retrieval | 수 분~수 시간 복구를 허용하는 Archive | Restore 절차 필요 |
| S3 Glacier Deep Archive | 장기 보관 | 긴 Restore 시간과 최소 보관 기간 |
| Reduced Redundancy Storage | Legacy Class | 현재 권장되지 않음 |

Lifecycle Rule은 Object의 생성 이후 경과 일수에 따라 Storage Class를 전환하거나 삭제합니다.

```text
생성
  ↓ S3 Standard
30일
  ↓ S3 Standard-IA
90일
  ↓ S3 Glacier Flexible Retrieval
365일
  ↓ Expiration
```

위 기간은 예시입니다. 최소 보관 기간, Retrieval 시간, Compliance와 실제 복구 빈도를 기준으로 결정합니다. Versioning을 사용한다면 Current Version뿐 아니라 Noncurrent Version의 Transition과 Expiration도 함께 설정합니다.

## 5. Private Bucket 생성 원칙

---

Application Upload용 Bucket은 기본적으로 비공개로 유지합니다.

1. Application과 가까운 AWS Region에 General Purpose Bucket을 생성합니다.

2. Object Ownership은 기본값인 `Bucket owner enforced`를 유지해 ACL을 비활성화합니다.

3. `Block all public access`를 활성화합니다.

4. Versioning, Default Encryption과 Lifecycle을 Data 중요도에 맞게 설정합니다.

5. Application에는 IAM Role을 연결하고 필요한 Bucket Prefix에만 권한을 부여합니다.

`Block Public Access`를 해제해야 외부 Upload가 가능하다는 설명은 잘못되었습니다. Backend는 IAM 인증으로 Private Bucket에 Upload할 수 있고, Browser 직접 Upload는 제한된 Presigned URL을 사용할 수 있습니다.

## 6. IAM 최소 권한

---

EC2에서 실행하는 Spring Boot Application에는 Instance Profile을 연결합니다. ECS는 Task Role, EKS는 Pod Identity 또는 IRSA처럼 Workload용 임시 Credential을 사용합니다.

다음 Identity Policy는 특정 Bucket의 `uploads/` Prefix에 Object를 쓰고 읽는 예제입니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListUploadPrefix",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::example-private-upload",
      "Condition": {
        "StringLike": {
          "s3:prefix": "uploads/*"
        }
      }
    },
    {
      "Sid": "ManageUploadObjects",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::example-private-upload/uploads/*"
    }
  ]
}
```

`AmazonS3FullAccess`를 Application User에 부여하거나 `Principal: "*"`로 `PutObject`, `DeleteObject`를 허용하지 않습니다. 삭제 기능이 필요한 Application에만 대상 Prefix의 `s3:DeleteObject`를 추가합니다.

## 7. CORS의 역할

---

CORS는 Browser가 다른 Origin의 S3 Endpoint를 호출할 수 있는지 결정합니다. IAM이나 Bucket Policy를 대신하지 않으므로 CORS에서 Method를 허용해도 S3 권한이 자동으로 생기지 않습니다.

Browser가 Backend로 File을 보내고 Backend가 S3에 Upload한다면 Browser는 S3를 직접 호출하지 않으므로 S3 CORS가 필요하지 않습니다.

Presigned URL로 Browser가 S3에 직접 Upload할 때만 실제 Frontend Origin과 Method를 제한해 설정합니다.

```json
[
  {
    "AllowedHeaders": [
      "content-type"
    ],
    "AllowedMethods": [
      "PUT"
    ],
    "AllowedOrigins": [
      "https://app.example.com"
    ],
    "ExposeHeaders": [
      "ETag"
    ],
    "MaxAgeSeconds": 300
  }
]
```

Write Method와 `AllowedOrigins: ["*"]`를 함께 사용하는 설정은 피합니다.

## 8. Spring Boot Project 의존성

---

기존 AWS SDK for Java 1.x의 `AmazonS3Client`보다 현재 유지되는 AWS SDK for Java 2.x의 `S3Client`를 사용합니다. 다음은 Gradle 예제입니다.

```groovy
dependencies {
    implementation platform('software.amazon.awssdk:bom:2.55.7')
    implementation 'software.amazon.awssdk:s3'

    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

SDK Version은 작성 시점의 예제 값입니다. Project의 Java·Spring Boot Version과 호환되는 현재 2.x Release를 확인한 뒤 동일 BOM 안에서 Module Version을 맞춥니다.

`application.yml`에는 Bucket과 Region만 기록합니다.

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 20MB
      max-request-size: 20MB

app:
  s3:
    bucket: example-private-upload
    region: ap-northeast-2
```

Access Key와 Secret Access Key는 설정 File이나 Git Repository에 저장하지 않습니다. Credential Provider를 별도로 지정하지 않으면 SDK는 Default Credentials Provider Chain을 사용하며 EC2 Instance Profile 같은 임시 Credential을 찾습니다.

## 9. S3 Client 구성

---

Configuration Property를 정의합니다.

```java
import org.springframework.boot.context.properties.ConfigurationProperties;

import software.amazon.awssdk.regions.Region;

@ConfigurationProperties(prefix = "app.s3")
public record S3Properties(String bucket, Region region) {
}
```

`S3Client`는 Application에서 재사용하도록 Bean으로 등록합니다.

```java
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import software.amazon.awssdk.services.s3.S3Client;

@Configuration
@EnableConfigurationProperties(S3Properties.class)
public class S3Configuration {

    @Bean
    S3Client s3Client(S3Properties properties) {
        return S3Client.builder()
                .region(properties.region())
                .build();
    }
}
```

Credential Provider를 코드에서 지정하지 않았으므로 SDK가 Default Credentials Provider Chain을 사용합니다.

## 10. File 이름과 Object Key 생성

---

Client가 보낸 원본 File Name을 그대로 Key로 사용하면 이름 충돌과 경로 조작 문제가 생길 수 있습니다. 원본 이름에서 확장자만 안전하게 추출하고 UUID로 새 이름을 만듭니다.

```java
import java.util.Locale;
import java.util.Set;
import java.util.UUID;

import org.springframework.util.StringUtils;

public final class ObjectKeyFactory {

    private static final Set<String> ALLOWED_EXTENSIONS =
            Set.of("jpg", "jpeg", "png", "gif", "pdf");

    private ObjectKeyFactory() {
    }

    public static String create(String category, String originalFilename) {
        String safeCategory = category.replaceAll("[^a-zA-Z0-9_-]", "");
        if (safeCategory.isBlank()) {
            throw new IllegalArgumentException("유효하지 않은 category입니다.");
        }

        String extension = StringUtils.getFilenameExtension(originalFilename);
        if (extension == null
                || !ALLOWED_EXTENSIONS.contains(extension.toLowerCase(Locale.ROOT))) {
            throw new IllegalArgumentException("허용하지 않는 파일 형식입니다.");
        }

        return "uploads/%s/%s.%s".formatted(
                safeCategory,
                UUID.randomUUID(),
                extension.toLowerCase(Locale.ROOT)
        );
    }
}
```

File 확장자만 신뢰하지 않고 실제 Service 요구 사항에 따라 MIME Type, File Signature, Virus Scan과 Image Re-encoding을 추가합니다.

## 11. Upload Service

---

`MultipartFile`의 InputStream을 S3에 전달하고 Object Key를 반환합니다. Private Bucket이므로 Public Read ACL을 추가하지 않습니다.

```java
import java.io.IOException;

import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;

@Service
public class S3FileService {

    private final S3Client s3Client;
    private final S3Properties properties;

    public S3FileService(S3Client s3Client, S3Properties properties) {
        this.s3Client = s3Client;
        this.properties = properties;
    }

    public String upload(String category, MultipartFile file) {
        if (file.isEmpty()) {
            throw new IllegalArgumentException("빈 파일은 업로드할 수 없습니다.");
        }

        String key = ObjectKeyFactory.create(category, file.getOriginalFilename());
        PutObjectRequest request = PutObjectRequest.builder()
                .bucket(properties.bucket())
                .key(key)
                .contentType(file.getContentType())
                .build();

        try (var inputStream = file.getInputStream()) {
            s3Client.putObject(
                    request,
                    RequestBody.fromInputStream(inputStream, file.getSize())
            );
            return key;
        } catch (IOException exception) {
            throw new FileUploadException("업로드할 파일을 읽지 못했습니다.", exception);
        }
    }
}
```

Application Database에는 영구 Public URL 대신 Bucket과 Object Key를 저장합니다. Client가 Private Object를 내려받아야 할 때는 권한 확인 후 짧은 만료 시간을 가진 Presigned GET URL을 생성하거나 Backend가 Data를 중계합니다.

## 12. Controller와 예외 처리

---

```java
import java.util.Map;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RequestPart;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.multipart.MaxUploadSizeExceededException;
import org.springframework.web.multipart.MultipartFile;

@RestController
public class S3FileController {

    private final S3FileService fileService;

    public S3FileController(S3FileService fileService) {
        this.fileService = fileService;
    }

    @PostMapping("/uploads")
    public Map<String, String> upload(
            @RequestParam String category,
            @RequestPart MultipartFile file
    ) {
        return Map.of("key", fileService.upload(category, file));
    }
}

@RestControllerAdvice
class FileUploadExceptionHandler {

    @ExceptionHandler(MaxUploadSizeExceededException.class)
    ResponseEntity<Map<String, String>> handleMaxUploadSizeExceeded() {
        return ResponseEntity.status(HttpStatus.PAYLOAD_TOO_LARGE)
                .body(Map.of("message", "허용된 업로드 용량을 초과했습니다."));
    }
}
```

`FileUploadException`은 Runtime Exception으로 정의합니다.

```java
public class FileUploadException extends RuntimeException {

    public FileUploadException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

## 13. Upload 검증

---

Application Host의 IAM Role이 적용되었는지 먼저 확인합니다. 장기 Access Key를 출력하거나 Log에 남기지 않습니다.

```bash
aws sts get-caller-identity
```

Application을 실행한 뒤 Test File을 Upload합니다.

```bash
curl -X POST http://localhost:8080/uploads \
  -F 'category=profile' \
  -F 'file=@sample.png'
```

반환된 Key가 Bucket에 생성되었는지 확인합니다.

```bash
aws s3api head-object \
  --bucket example-private-upload \
  --key '<RETURNED_OBJECT_KEY>'
```

| 오류 | 확인 항목 |
| --- | --- |
| `AccessDenied` | IAM Role 연결, Resource ARN과 `s3:PutObject` 권한 |
| `NoSuchBucket` | Bucket Name과 Region |
| `PermanentRedirect` | Client Region과 Bucket Region 불일치 |
| `413 Payload Too Large` | Spring Multipart와 앞단 Proxy·ALB 제한 |
| Browser CORS 오류 | Browser 직접 S3 호출 여부, Origin·Method·Header |
| Upload 후 URL 접근 거부 | Private Object의 정상 동작이며 Presigned URL 또는 Backend Download 필요 |

## 14. 정적 Website Hosting 경계

---

S3는 정적 Website Hosting Endpoint를 제공하지만 Website Endpoint 자체는 HTTPS를 지원하지 않습니다. 운영 Domain에서 HTTPS로 정적 Site를 제공할 때는 Private S3 Origin 앞에 CloudFront와 Origin Access Control을 두고 ACM Certificate를 연결하는 구성을 사용합니다.

```text
Client
  ↓ HTTPS
CloudFront + ACM
  ↓ Origin Access Control
Private S3 Bucket
```

정적 Website CI/CD와 CloudFront 배포는 별도 글에서 다룹니다.

## 15. 실습 후 정리

---

- 실습 Object와 이전 Version을 삭제하고 Multipart Upload가 남아 있는지 확인합니다.

- Bucket을 삭제하지 않을 경우 Lifecycle과 Versioning 비용을 확인합니다.

- Application IAM Role의 임시 Policy와 불필요한 `s3:*` 권한을 제거합니다.

- Bucket Policy, ACL과 Block Public Access 상태를 IAM Access Analyzer for S3로 점검합니다.

- Source Code, Shell History와 Log에 Access Key나 Secret Key가 남아 있지 않은지 검사합니다.

> **최종 정리**
> - S3는 Key로 Object를 식별하며 Console의 Folder는 Key Prefix를 표현합니다.
>
> - Backup은 Storage Class 전환, RPO·RTO, Application Consistency와 Restore 검증을 포함합니다.
>
> - Application은 IAM Role과 Default Credentials Provider Chain을 사용하고 Bucket을 Private으로 유지합니다.
>
> - Browser 직접 전송에는 제한된 CORS와 Presigned URL을 사용하며 CORS를 권한 설정으로 해석하지 않습니다.

## 참고 자료

---

- [S3 File Gateway 동작](https://docs.aws.amazon.com/filegateway/latest/files3/file-gateway-concepts.html)

- [S3 Bucket 생성과 기본 Quota](https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html)

- [S3 Storage Class](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)

- [S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)

- [S3 Object Ownership과 ACL 비활성화](https://docs.aws.amazon.com/AmazonS3/latest/userguide/about-object-ownership.html)

- [AWS SDK for Java 2.x Default Credentials Provider Chain](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/credentials-chain.html)

- [S3 Presigned URL](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)

- [CloudFront와 Private S3를 이용한 정적 Website](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/getting-started-secure-static-website-cloudformation-template.html)
