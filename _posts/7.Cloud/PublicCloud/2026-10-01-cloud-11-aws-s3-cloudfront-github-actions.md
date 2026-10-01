---
title: S3·CloudFront 정적 Website와 GitHub Actions 배포
description: Private S3 Origin과 CloudFront로 정적 Website를 제공하고 GitHub Actions OIDC로 Build·배포·Cache 무효화를 자동화합니다
date: 2026-10-01
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> S3는 HTML, CSS, JavaScript와 Image 같은 정적 File을 저장하고 CloudFront는 이를 전 세계 Edge Location에 Cache해 전달합니다. 운영 환경에서는 S3 Website Endpoint를 공개하기보다 Private S3 Bucket을 CloudFront Origin으로 연결하고 Origin Access Control로 접근을 제한합니다. 이 글에서는 정적 Application을 Build하고 GitHub Actions가 OIDC로 AWS IAM Role을 맡아 S3 Upload와 CloudFront Cache 무효화를 수행하는 흐름을 정리합니다.

## 1. 정적 Website 배포 구조

---

정적 Frontend Application은 Build 결과만 Web Server에서 제공하면 됩니다. API 요청은 별도의 Backend Endpoint로 전달합니다.

```text
GitHub Repository
  ↓ push
GitHub Actions Runner
  ├── npm ci
  ├── npm run build
  ├── S3 Sync
  └── CloudFront Invalidation
        ↓
Private S3 Bucket
  ↑ Origin Access Control
CloudFront + ACM Certificate
  ↑
Route 53 Alias
  ↑ HTTPS
Client
```

S3 정적 Website Hosting과 Private S3 Origin은 같은 구성이 아닙니다.

| 구성 | Origin | HTTPS | S3 공개 여부 | 용도 |
| --- | --- | --- | --- | --- |
| S3 Website Endpoint 직접 접속 | `s3-website` Endpoint | 지원하지 않음 | 공개 읽기 필요 | 기능 확인용 단순 실습 |
| CloudFront + S3 Website Endpoint | Website Endpoint | CloudFront에서 제공 | Website Endpoint가 공개됨 | S3 Redirect·Website 기능이 필요한 경우 |
| CloudFront + Private S3 Origin | S3 REST Endpoint | CloudFront에서 제공 | 비공개 유지 | 일반적인 운영 구성 |

Origin Access Control(OAC)은 S3 REST Endpoint에 적용합니다. S3 Website Endpoint는 Custom Origin으로 취급되므로 OAC를 사용할 수 없습니다.

## 2. S3 Bucket 준비

---

운영용 Bucket은 다음 원칙으로 생성합니다.

1. CloudFront Origin으로 사용할 General Purpose Bucket을 생성합니다.

2. Object Ownership은 `Bucket owner enforced`를 유지합니다.

3. `Block all public access`를 활성화합니다.

4. Versioning과 Default Encryption을 요구 사항에 맞게 설정합니다.

5. CloudFront OAC를 생성한 뒤 해당 Distribution만 Object를 읽을 수 있도록 Bucket Policy를 연결합니다.

정적 File을 직접 Upload해 동작을 확인할 때도 `--acl public-read`를 사용하지 않습니다. ACL이 비활성화된 Bucket에서는 해당 Option이 실패하며, OAC 구성에서는 Public ACL이 필요하지 않습니다.

## 3. Frontend Application 생성과 Build

---

Create React App은 신규 Application을 위한 도구로 Deprecated되었습니다. 기존 Project의 `npm run build`는 계속 사용할 수 있지만 새 Project는 Vite 같은 Build Tool을 사용할 수 있습니다.

다음 명령은 Vite 기반 React Project를 생성합니다.

```bash
npm create vite@latest static-web -- --template react
cd static-web
npm install
npm run dev
```

개발 Server는 Local 확인용입니다. 배포할 정적 File을 만들려면 Production Build를 수행합니다.

```bash
npm run build
```

Vite의 기본 Output Directory는 `dist/`입니다. 기존 Create React App Project는 `build/`을 사용하므로 Workflow의 경로를 Project에 맞게 지정해야 합니다.

| 도구 | 기본 Build Directory | 확인 항목 |
| --- | --- | --- |
| Vite | `dist/` | `vite.config.*`의 `build.outDir` |
| Create React App | `build/` | `package.json`의 Build Script |

Build 전에 불필요한 Import, Type Error와 Lint Error를 Local에서도 검사합니다. CI 환경은 `CI` 설정이나 Build Tool Version에 따라 Warning을 Error로 처리할 수 있습니다.

## 4. AWS CLI로 수동 배포 확인

---

Local에서 AWS CLI를 사용할 때는 기본 Profile과 분리된 이름을 사용할 수 있습니다.

```bash
aws configure --profile static-site-deploy
```

이 명령은 Access Key를 Source Code에 기록하지 않지만 장기 IAM User Credential을 Local 설정 File에 저장합니다. 실습 이후에는 불필요한 Key를 폐기하고, EC2·CloudShell·CI 환경에서는 IAM Role과 임시 Credential을 우선합니다.

현재 Identity를 확인합니다.

```bash
aws sts get-caller-identity \
  --profile static-site-deploy
```

Build 결과를 Bucket과 동기화합니다.

```bash
aws s3 sync dist/ s3://example-static-site \
  --delete \
  --profile static-site-deploy
```

`--delete`는 Local Output에 없는 Object를 대상 Bucket에서도 삭제합니다. 전용 배포 Bucket이나 Prefix에서만 사용하고 Upload 대상 경로를 먼저 확인합니다.

```bash
aws s3 sync dist/ s3://example-static-site \
  --delete \
  --dryrun \
  --profile static-site-deploy
```

## 5. CloudFront Distribution 구성

---

CloudFront Distribution은 다음 항목을 기준으로 구성합니다.

| 항목 | 설정 방향 |
| --- | --- |
| Origin Domain | S3 Website Endpoint가 아닌 Bucket REST Endpoint |
| Origin Access | Origin Access Control 생성·연결 |
| Signing Behavior | 요청을 항상 서명 |
| Viewer Protocol Policy | HTTP를 HTTPS로 Redirect |
| Default Root Object | `index.html` |
| Cache Policy | 정적 Asset 특성에 맞는 Managed 또는 Custom Policy |
| Alternate Domain | `www.example.com` 같은 실제 Domain |
| Certificate | `us-east-1`에서 발급한 ACM Certificate |

OAC를 Distribution에 연결하면 Console이 제안하는 S3 Bucket Policy를 적용합니다. Policy의 `AWS:SourceArn`은 특정 CloudFront Distribution ARN으로 제한합니다.

Single Page Application은 `/users/1`처럼 실제 S3 Object가 없는 경로로 직접 접근할 수 있습니다. 이 경우 `403` 또는 `404` 응답을 `index.html`로 매핑하는 Custom Error Response가 필요한지 Router 동작과 함께 결정합니다. API 오류까지 `index.html`로 바꾸지 않도록 Behavior와 Origin을 분리합니다.

## 6. GitHub Actions와 AWS OIDC

---

GitHub Actions에 장기 `AWS_ACCESS_KEY_ID`와 `AWS_SECRET_ACCESS_KEY`를 저장하지 않습니다. GitHub의 OIDC Token을 신뢰하는 IAM Role을 만들고 Workflow 실행 시 짧은 수명의 AWS Credential을 발급받습니다.

```text
GitHub Actions Job
  ↓ OIDC Token
AWS IAM Role Trust Policy
  ↓ AssumeRoleWithWebIdentity
Temporary AWS Credential
  ↓
S3 Sync + CloudFront Invalidation
```

IAM Role의 Trust Policy는 Repository와 Branch 또는 GitHub Environment를 제한합니다. 다음 예제는 `example-org/static-web` Repository의 `main` Branch만 허용합니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:example-org/static-web:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

Role의 Permission Policy에는 배포 대상만 허용합니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::example-static-site"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:DeleteObject",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::example-static-site/*"
    },
    {
      "Effect": "Allow",
      "Action": "cloudfront:CreateInvalidation",
      "Resource": "arn:aws:cloudfront::<AWS_ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
    }
  ]
}
```

`s3:DeleteObject`는 `aws s3 sync --delete`에 필요합니다. 삭제 동기화를 사용하지 않으면 제거할 수 있습니다.

## 7. Repository Variable 구성

---

GitHub Repository의 `Settings → Secrets and variables → Actions`에서 다음 Variable을 등록합니다.

| Variable | 역할 | 예제 형식 |
| --- | --- | --- |
| `AWS_ROLE_ARN` | Workflow가 맡을 IAM Role | `arn:aws:iam::<AWS_ACCOUNT_ID>:role/<ROLE_NAME>` |
| `AWS_REGION` | S3 Bucket Region | `ap-northeast-2` |
| `AWS_S3_BUCKET` | 배포 대상 Bucket Name | `example-static-site` |
| `AWS_CLOUDFRONT_DISTRIBUTION_ID` | Cache 무효화 대상 | `<DISTRIBUTION_ID>` |

이 값들은 인증 Secret 자체가 아닙니다. 다만 Repository 구조와 운영 Resource 식별자를 외부에 공개하지 않아야 한다면 GitHub Environment 또는 Secret으로 접근 범위를 제한할 수 있습니다.

## 8. GitHub Actions Workflow

---

`.github/workflows/deploy-static-site.yml`을 작성합니다. `${{ ... }}` 표현은 GitHub Actions가 Runtime에 해석합니다.

{% raw %}
```yaml
name: Deploy static site

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout source
        uses: actions/checkout@v6

      - name: Set up Node.js
        uses: actions/setup-node@v6
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Configure temporary AWS credentials
        uses: aws-actions/configure-aws-credentials@v6.3.0
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Upload build output to S3
        run: aws s3 sync dist/ "s3://${{ vars.AWS_S3_BUCKET }}" --delete

      - name: Invalidate CloudFront cache
        run: >-
          aws cloudfront create-invalidation
          --distribution-id "${{ vars.AWS_CLOUDFRONT_DISTRIBUTION_ID }}"
          --paths "/*"
```
{% endraw %}

| 설정 | 역할 |
| --- | --- |
| `id-token: write` | GitHub OIDC Token 발급 허용 |
| `contents: read` | Repository Checkout 허용 |
| `npm ci` | Lock File과 일치하는 의존성을 재현 가능하게 설치 |
| `aws s3 sync` | 변경된 정적 File Upload와 제거된 File 삭제 |
| `create-invalidation` | Edge Cache의 기존 Object 무효화 요청 |

Production Environment에 Required Reviewer를 설정하면 승인 이후에만 AWS Role을 맡도록 통제할 수 있습니다. 이 경우 IAM Trust Policy의 `sub` 조건도 Branch 대신 Environment 형식으로 맞춰야 합니다.

## 9. Cache 갱신 전략

---

CloudFront Cache를 항상 `/*`로 무효화하면 배포는 단순하지만 Invalidation 비용과 Cache Miss가 늘어납니다.

정적 Asset에는 Content Hash가 포함된 File Name을 사용합니다.

```text
app.8f32c1.js
style.a91b20.css
```

새 Build는 새 File Name을 만들기 때문에 기존 Cache와 충돌하지 않습니다. 짧은 Cache를 적용한 `index.html`과 삭제된 File만 무효화하면 Cache 효율을 높일 수 있습니다.

```bash
aws cloudfront create-invalidation \
  --distribution-id '<DISTRIBUTION_ID>' \
  --paths '/index.html'
```

## 10. Domain과 HTTPS 연결

---

CloudFront에 연결할 Public ACM Certificate는 `us-east-1` Region에서 요청해야 합니다.

1. `us-east-1`의 ACM에서 Domain Certificate를 요청합니다.

2. DNS Validation Record를 Route 53 Hosted Zone에 등록합니다.

3. CloudFront Distribution의 Alternate Domain Name과 Certificate를 연결합니다.

4. Route 53에서 CloudFront Distribution을 대상으로 Alias `A` Record를 생성합니다.

5. IPv6를 사용하면 Alias `AAAA` Record도 생성합니다.

DNS와 TLS 상태를 확인합니다.

```bash
dig +short www.example.com A
curl -I https://www.example.com
```

## 11. 배포 결과 검증

---

GitHub Actions 실행 결과와 AWS Resource를 함께 확인합니다.

```bash
aws s3 ls s3://example-static-site --recursive
```

```bash
aws cloudfront get-distribution \
  --id '<DISTRIBUTION_ID>' \
  --query 'Distribution.Status'
```

```bash
curl -I https://www.example.com/index.html
```

| 증상 | 확인 항목 |
| --- | --- |
| S3 `AccessDenied` | OIDC Role, Permission Policy, Bucket ARN과 Prefix |
| CloudFront `403` | OAC 연결, Bucket Policy, Origin Domain |
| 이전 화면 표시 | Object Cache-Control, File Versioning, Invalidation 상태 |
| 하위 Route 새로고침 실패 | SPA Router와 Custom Error Response |
| Certificate 오류 | `us-east-1` Certificate, Domain 일치, Distribution 배포 완료 |
| Workflow Build 실패 | Node Version, Lock File, Lint·Type Error, Build Directory |

> **최종 정리**
> - S3 Website Endpoint는 HTTP만 지원하며 OAC를 사용할 수 없습니다.
>
> - 운영 정적 Website는 Private S3 REST Origin과 CloudFront OAC를 이용해 S3 직접 접근을 차단할 수 있습니다.
>
> - GitHub Actions는 장기 Access Key 대신 OIDC로 IAM Role을 맡아 임시 Credential을 사용합니다.
>
> - Content Hash와 필요한 범위의 Invalidation을 조합하면 Cache 효율과 배포 반영 속도를 함께 관리할 수 있습니다.

## 참고 자료

---

- [CloudFront에서 S3 Origin 선택](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesOrigin.html)

- [Origin Access Control로 S3 접근 제한](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)

- [CloudFront File 무효화](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html)

- [CloudFront에서 ACM Certificate 사용](https://docs.aws.amazon.com/acm/latest/userguide/acm-services.html)

- [GitHub Actions에서 AWS OIDC 사용](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)

- [AWS Credential Action](https://github.com/aws-actions/configure-aws-credentials)

- [Create React App Deprecated 안내](https://react.dev/blog/2025/02/14/sunsetting-create-react-app)

다음 글인 [AWS Container Service와 ECS·EKS 실행 구조](/cloud-12-aws-container-services-architecture/)에서는 ECS·EKS Control Plane과 EC2·Fargate Data Plane의 조합을 비교합니다.
