---
title: Django Image를 ECR와 ECS에 배포하는 GitHub Actions Pipeline
description: Django REST API를 Gunicorn Container Image로 만들고 ECR에 저장한 뒤 GitHub OIDC와 최소 IAM 권한으로 ECS Service에 자동 배포합니다
date: 2026-10-02
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> Django REST API를 Production용 Container Image로 만들고 Amazon ECR에 저장합니다. GitHub Actions는 장기 Access Key 대신 OIDC로 AWS IAM Role을 맡아 Test, Image Build·Push, Task Definition Rendering과 ECS Service 배포를 수행합니다. Application 실행 권한과 배포 Pipeline 권한을 분리하고 각 단계의 검증 방법을 함께 구성합니다.

## 1. 전체 배포 흐름

---

이 글은 [Django REST API 생성과 Container 배포 준비]({% post_url 2026-10-01-cloud-13-django-rest-api-container-preparation %})에서 만든 Application과 [Nginx로 이해하는 ECS Task·Service와 ALB]({% post_url 2026-10-02-cloud-14-aws-ecs-fargate-nginx-service %})에서 구성한 ECS Service 구조를 연결합니다.

```text
Developer
  ↓ git push
GitHub Actions
  ├── Django Check와 Test
  ├── OIDC로 AWS IAM Role Assume
  ├── Container Image Build
  └── ECR에 Commit SHA Tag로 Push
        ↓
Task Definition 새 Revision 등록
        ↓
ECS Service Rolling Deployment
        ↓
ALB Health Check와 Service 안정화 확인
```

Pipeline이 사용하는 Role과 실행 중인 Application이 사용하는 Role은 분리합니다.

| IAM Role | 사용 주체 | 역할 |
| --- | --- | --- |
| GitHub Deployment Role | GitHub Actions | ECR Push, Task Definition 등록, ECS Service 갱신 |
| Task Execution Role | ECS Agent | ECR Image Pull, CloudWatch Logs 전송, 시작 시 Secret 조회 |
| Task Role | Django Container | 실행 중 AWS API를 호출할 때 사용하는 Application 권한 |

Task Execution Role이나 Task Role에 ECS Full Access를 부여하지 않습니다. 배포 Role에도 필요한 ECR Repository, ECS Service와 PassRole 대상만 허용합니다.

## 2. Production 실행 Package 추가

---

Django의 `runserver`는 개발용입니다. 가상환경을 활성화한 Project Root에서 Gunicorn을 설치하고 의존성을 다시 고정합니다.

```bash
python -m pip install gunicorn
python -m pip freeze > requirements.txt
```

설치 여부를 확인합니다.

```bash
python -m gunicorn --version
```

`apiserver.wsgi:application`은 `apiserver/wsgi.py`의 WSGI Application Object를 가리킵니다.

## 3. Dockerfile 작성

---

Project Root에 `Dockerfile`을 작성합니다.

```dockerfile
FROM python:3.14-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt ./
RUN python -m pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd --create-home --uid 10001 appuser \
    && chown -R appuser:appuser /app

USER appuser

EXPOSE 8000

CMD ["gunicorn", "apiserver.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "2", "--access-logfile", "-", "--error-logfile", "-"]
```

| 설정 | 목적 |
| --- | --- |
| `python:3.14-slim` | Django 5.2가 지원하는 Python Runtime을 작은 Debian 기반 Image로 사용합니다. |
| `PYTHONDONTWRITEBYTECODE` | Container 내부에 `.pyc` File을 생성하지 않습니다. |
| `PYTHONUNBUFFERED` | Log를 Buffering하지 않고 Standard Output으로 즉시 전달합니다. |
| `--no-cache-dir` | Package Download Cache를 Image Layer에 남기지 않습니다. |
| `USER appuser` | Application을 Root가 아닌 사용자로 실행합니다. |
| `--access-logfile -` | Access Log를 Standard Output으로 보내 CloudWatch Logs가 수집하게 합니다. |

Worker 수는 CPU, 응답 시간과 Memory 사용량을 부하 Test로 확인해 조정합니다. 예제의 `2`는 모든 환경에 적용할 고정값이 아닙니다.

## 4. Build Context 제외

---

`.dockerignore`를 작성해 Secret, Local 가상환경과 불필요한 File이 Build Context와 Image Layer에 들어가지 않게 합니다.

```dockerignore
.git
.github
.venv
__pycache__/
*.py[cod]
*.sqlite3
.env
.env.*
!.env.example
.pytest_cache/
.mypy_cache/
Dockerfile*
README.md
```

`.dockerignore`는 이미 Commit된 Secret을 제거하지 않습니다. 실제 Credential이 Git History에 들어갔다면 값을 폐기·교체한 뒤 History 처리 범위를 별도로 결정해야 합니다.

## 5. Local Image 검증

---

Image를 Build합니다.

```bash
docker build \
  --tag apiserver:local \
  .
```

운영 Secret과 다른 Local 전용 값을 환경 변수로 주입해 Container를 실행합니다.

```bash
docker run --rm \
  --name apiserver-local \
  --publish 8000:8000 \
  --env DJANGO_SECRET_KEY='<LOCAL_DEVELOPMENT_SECRET>' \
  --env DJANGO_ALLOWED_HOSTS='127.0.0.1,localhost' \
  apiserver:local
```

다른 Terminal에서 API를 검증합니다.

```bash
curl -i http://127.0.0.1:8000/api/
```

응답이 `200 OK`인지 확인하고 실행 Terminal에서 `Ctrl+C`로 종료합니다. 실제 Secret을 Dockerfile의 `ENV`나 Build Argument로 전달하면 Image Metadata나 Layer에 남을 수 있으므로 Runtime에 주입합니다.

## 6. ECR Repository 생성

---

ECR에 Private Repository를 만듭니다. Commit SHA처럼 한 번 발행한 Tag를 덮어쓰지 않도록 Immutable Tag를 사용하고 Push 시 기본 Image Scan을 활성화합니다.

```bash
aws ecr create-repository \
  --repository-name apiserver \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --region ap-northeast-2
```

Repository URI를 확인합니다.

```bash
aws ecr describe-repositories \
  --repository-names apiserver \
  --region ap-northeast-2 \
  --query 'repositories[0].repositoryUri' \
  --output text
```

Image Registry는 다른 제품으로 대체할 수 있습니다. AWS ECR를 사용하면 IAM, ECS Image Pull과 Private Network 구성을 같은 AWS 권한 체계로 관리할 수 있습니다.

## 7. GitHub OIDC Provider와 Trust Policy

---

장기 Access Key를 GitHub Secret에 저장하지 않고 GitHub Actions의 OIDC Token으로 단기 AWS Credential을 발급받습니다.

Access Key ID와 Secret Access Key를 GitHub Secret에 저장하는 방식도 현재 지원되지만 장기 Credential의 유출·회수·교체 부담이 생깁니다. GitHub Actions와 AWS가 OIDC를 지원하는 환경에서는 이를 기본 인증 방식으로 사용하지 않습니다.

AWS IAM에 GitHub OIDC Provider가 없다면 다음 값을 사용해 등록합니다.

| 항목 | 값 |
| --- | --- |
| Provider URL | `https://token.actions.githubusercontent.com` |
| Audience | `sts.amazonaws.com` |

GitHub Deployment Role의 Trust Policy는 Repository와 Production Environment를 제한합니다. Placeholder를 실제 값으로 교체하되 개인·조직 식별자는 공개 문서에 Commit하지 않습니다.

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
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<GITHUB_OWNER>/<GITHUB_REPOSITORY>:environment:production"
        }
      }
    }
  ]
}
```

이 Workflow는 `production` Environment를 사용하므로 Trust Policy의 `sub`도 Environment 형식으로 맞춥니다. GitHub Environment에 Required Reviewer와 허용 Branch를 설정하면 승인과 Branch 조건을 배포 전에 적용할 수 있습니다. Environment를 사용하지 않을 때는 `sub`를 `repo:<OWNER>/<REPOSITORY>:ref:refs/heads/main` 형식으로 제한합니다.

## 8. Deployment Role 최소 권한

---

Deployment Role에는 ECR Push, Task Definition 등록, ECS Service 갱신과 제한된 `iam:PassRole` 권한이 필요합니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GetEcrAuthorizationToken",
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Sid": "PushImageToApplicationRepository",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:UploadLayerPart"
      ],
      "Resource": "arn:aws:ecr:ap-northeast-2:<AWS_ACCOUNT_ID>:repository/apiserver"
    },
    {
      "Sid": "RegisterTaskDefinition",
      "Effect": "Allow",
      "Action": "ecs:RegisterTaskDefinition",
      "Resource": "*"
    },
    {
      "Sid": "DeployApplicationService",
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeServices",
        "ecs:UpdateService"
      ],
      "Resource": "arn:aws:ecs:ap-northeast-2:<AWS_ACCOUNT_ID>:service/<ECS_CLUSTER>/<ECS_SERVICE>"
    },
    {
      "Sid": "PassOnlyApplicationTaskRoles",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::<AWS_ACCOUNT_ID>:role/<TASK_EXECUTION_ROLE_NAME>",
        "arn:aws:iam::<AWS_ACCOUNT_ID>:role/<TASK_ROLE_NAME>"
      ],
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": "ecs-tasks.amazonaws.com"
        }
      }
    }
  ]
}
```

권한 오류를 해결하기 위해 `AmazonECS_FullAccess`를 추가하면 배포 대상 밖의 Resource까지 변경할 수 있습니다. CloudTrail의 거부 Event와 Workflow Log를 확인한 뒤 필요한 Action과 Resource만 추가합니다.

## 9. ECS Task Definition을 Code로 관리

---

Project Root에 `task-definition.json`을 저장합니다. Workflow가 `image` 값을 새 ECR Image URI로 교체하므로 `image` 속성을 삭제하지 않습니다.

```json
{
  "family": "django-api",
  "taskRoleArn": "arn:aws:iam::<AWS_ACCOUNT_ID>:role/<TASK_ROLE_NAME>",
  "executionRoleArn": "arn:aws:iam::<AWS_ACCOUNT_ID>:role/<TASK_EXECUTION_ROLE_NAME>",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "enableFaultInjection": false,
  "containerDefinitions": [
    {
      "name": "apiserver",
      "image": "IMAGE_URI_REPLACED_BY_WORKFLOW",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "DJANGO_ALLOWED_HOSTS",
          "value": "<APPLICATION_DOMAIN>,<ALB_DNS_NAME>"
        }
      ],
      "secrets": [
        {
          "name": "DJANGO_SECRET_KEY",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-2:<AWS_ACCOUNT_ID>:secret:<SECRET_NAME>"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/django-api",
          "awslogs-region": "ap-northeast-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

`enableFaultInjection`은 현재 Task Definition의 유효한 속성이며 기본값은 `false`입니다. Fault Injection을 사용하지 않는다는 뜻이므로 Export File에서 반드시 제거할 항목이 아닙니다.

기존 Task Definition을 Export할 때는 다음 명령을 사용할 수 있습니다.

```bash
aws ecs describe-task-definition \
  --task-definition django-api \
  --query taskDefinition \
  > task-definition.json
```

`describe-task-definition` 응답에는 새 Revision 등록 요청에 사용할 수 없는 읽기 전용 속성이 포함될 수 있습니다. `taskDefinitionArn`, `revision`, `status`, `requiresAttributes`, `compatibilities`, `registeredAt`, `registeredBy`를 제거하고 JSON을 Version Control에서 검토합니다. `image`는 Render Action이 찾는 필드이므로 유지합니다.

Secrets Manager 값을 시작 시 읽으려면 Task Execution Role에 해당 Secret의 `secretsmanager:GetSecretValue`를 허용해야 합니다. Application이 실행 중 AWS API를 호출할 때 필요한 권한은 Task Role에 별도로 부여합니다.

## 10. GitHub Repository 설정

---

GitHub Repository의 **Settings → Secrets and variables → Actions → Variables**에 다음 값을 등록합니다.

| Variable | 역할 | 예제 형식 |
| --- | --- | --- |
| `AWS_ROLE_ARN` | OIDC로 맡을 Deployment Role | `arn:aws:iam::<AWS_ACCOUNT_ID>:role/<DEPLOY_ROLE_NAME>` |
| `AWS_REGION` | ECR와 ECS Region | `ap-northeast-2` |
| `ECR_REPOSITORY` | Image를 Push할 Repository | `apiserver` |
| `ECS_CLUSTER` | 배포 대상 Cluster | `<ECS_CLUSTER>` |
| `ECS_SERVICE` | 배포 대상 Service | `<ECS_SERVICE>` |
| `ECS_CONTAINER_NAME` | Task Definition Container 이름 | `apiserver` |

이 값들은 인증 Credential이 아닙니다. Resource 식별자도 외부 공개를 제한해야 한다면 GitHub Environment 또는 Secret에 보관합니다. `AWS_ACCESS_KEY_ID`와 `AWS_SECRET_ACCESS_KEY`는 이 OIDC Workflow에 등록하지 않습니다.

## 11. GitHub Actions Workflow 작성

---

`.github/workflows/deploy-ecs.yml`을 작성합니다. `${{ ... }}`는 GitHub Actions가 Runtime에 해석하므로 Jekyll 문서에서는 Liquid와 충돌하지 않도록 Raw 영역으로 감쌉니다.

{% raw %}
```yaml
name: Test, build, and deploy Django to ECS

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

concurrency:
  group: production-ecs
  cancel-in-progress: false

env:
  ECS_TASK_DEFINITION: task-definition.json

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout source
        uses: actions/checkout@v6

      - name: Set up Python
        uses: actions/setup-python@v7
        with:
          python-version: "3.14"
          cache: pip

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          python -m pip install -r requirements.txt

      - name: Check Django configuration
        env:
          DJANGO_SECRET_KEY: ci-only-placeholder-not-for-production
          DJANGO_ALLOWED_HOSTS: 127.0.0.1,localhost
        run: python manage.py check

      - name: Run tests
        env:
          DJANGO_SECRET_KEY: ci-only-placeholder-not-for-production
          DJANGO_ALLOWED_HOSTS: 127.0.0.1,localhost
        run: python manage.py test

      - name: Configure temporary AWS credentials
        uses: aws-actions/configure-aws-credentials@v6.3.0
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Log in to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: ${{ vars.ECR_REPOSITORY }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          image_uri="$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"
          docker build --tag "$image_uri" .
          docker push "$image_uri"
          echo "image=$image_uri" >> "$GITHUB_OUTPUT"

      - name: Render image into task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: ${{ env.ECS_TASK_DEFINITION }}
          container-name: ${{ vars.ECS_CONTAINER_NAME }}
          image: ${{ steps.build-image.outputs.image }}

      - name: Deploy task definition to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          cluster: ${{ vars.ECS_CLUSTER }}
          service: ${{ vars.ECS_SERVICE }}
          wait-for-service-stability: true
```
{% endraw %}

| 단계 | 생성·변경되는 대상 | 실패 시 확인할 항목 |
| --- | --- | --- |
| Check·Test | 변경 없음 | Django 환경 변수, Migration, Test 결과 |
| OIDC 인증 | 수명이 짧은 AWS Session Credential | Trust Policy의 `aud`, `sub`, Workflow Permission |
| ECR Login | Docker의 임시 Registry 인증 | `ecr:GetAuthorizationToken` |
| Build·Push | SHA Tag가 붙은 ECR Image | Dockerfile, Repository 권한, Tag 중복 |
| Render | Runner의 임시 Task Definition JSON | Container 이름과 `image` 속성 |
| Deploy | 새 Task Definition Revision과 Service Deployment | `RegisterTaskDefinition`, `PassRole`, Service 권한 |

Immutable Repository에서는 같은 Commit SHA Tag를 다시 Push하면 실패합니다. 같은 Commit을 재실행해야 한다면 기존 Image를 재사용하는 단계나 별도의 재시도 Tag 정책을 설계합니다.

Action의 Major Tag는 같은 Major Version 안에서 변경될 수 있습니다. 공급망 변경을 엄격히 통제하는 Repository에서는 검증한 Full Commit SHA로 고정하고 Dependabot으로 Update를 검토합니다.

## 12. 배포 검증

---

GitHub Actions가 성공한 뒤 ECR Image를 확인합니다.

```bash
aws ecr describe-images \
  --repository-name apiserver \
  --region ap-northeast-2 \
  --query 'reverse(sort_by(imageDetails,& imagePushedAt))[0].{tags:imageTags,digest:imageDigest,pushedAt:imagePushedAt}'
```

ECS Service의 Running Count와 Deployment를 확인합니다.

```bash
aws ecs describe-services \
  --cluster <ECS_CLUSTER> \
  --services <ECS_SERVICE> \
  --query 'services[0].{desired:desiredCount,running:runningCount,deployments:deployments[*].{status:status,taskDefinition:taskDefinition,running:runningCount,failed:failedTasks}}'
```

ALB를 통해 API를 호출합니다.

```bash
curl -i http://<ALB_DNS_NAME>/api/
```

Task Log에서 Gunicorn 시작과 요청 처리 결과를 확인합니다.

```bash
aws logs tail /ecs/django-api \
  --since 10m
```

배포가 안정화되지 않으면 다음 순서로 점검합니다.

1. ECS Service Event에서 새 Task가 중지된 이유를 확인합니다.

2. CloudWatch Logs에서 Django 설정, Database와 Gunicorn 오류를 확인합니다.

3. Target Group의 Health Check Path와 Container Port 8000을 확인합니다.

4. Task Security Group이 ALB Security Group의 Traffic을 허용하는지 확인합니다.

5. `DJANGO_ALLOWED_HOSTS`에 실제 Application Domain과 ALB Health Check에 필요한 Host가 포함됐는지 확인합니다.

## 13. Rollback과 Image 추적

---

Commit SHA를 Image Tag로 사용하면 Git Commit, ECR Image와 Task Definition Revision을 연결해 추적할 수 있습니다.

```text
Git Commit SHA
  = ECR Image Tag
  → Task Definition의 Image URI
  → ECS Service Deployment
```

이전의 정상 Task Definition Revision으로 Service를 되돌립니다.

```bash
aws ecs update-service \
  --cluster <ECS_CLUSTER> \
  --service <ECS_SERVICE> \
  --task-definition django-api:<PREVIOUS_REVISION>
```

Rollback 후에도 Service 안정화, Target Health와 API 응답을 다시 확인합니다. 자동 Rollback이 필요하면 ECS Deployment Circuit Breaker와 CloudWatch Alarm을 함께 구성합니다.

## 14. Domain과 HTTPS 연결

---

Application 배포 이후에는 Route 53 Record가 ALB를 가리키게 하고 ACM Certificate를 ALB HTTPS Listener에 연결합니다. HTTP Listener는 HTTPS로 Redirect하도록 구성합니다.

CloudFront를 ALB 앞에 배치하거나 별도 CDN을 사용하는 구조는 Cache, Origin 접근 제한과 TLS 종료 위치를 추가로 설계해야 합니다. Domain과 Certificate 흐름은 [Route 53 Domain과 ACM HTTPS 구성]({% post_url 2026-09-30-cloud-09-aws-route53-acm-https %})에서 확인할 수 있습니다.

> **최종 정리**
> - Django는 `runserver` 대신 Gunicorn으로 실행하고 Secret은 Image가 아닌 ECS Runtime에 주입합니다.
>
> - ECR Image는 Commit SHA Tag로 저장해 Source, Image와 Task Definition Revision을 연결합니다.
>
> - GitHub Actions는 장기 Access Key 대신 OIDC로 제한된 Deployment Role을 맡습니다.
>
> - Deployment Role, Task Execution Role과 Task Role은 사용 주체와 권한을 분리합니다.
>
> - Workflow는 Django Test가 통과한 Image만 Push하고 새 Task Definition Revision으로 ECS Service를 갱신합니다.

## 참고 자료

---

- [GitHub Actions에서 Amazon ECS 배포](https://docs.github.com/en/actions/how-tos/deploy/deploy-to-third-party-platforms/amazon-elastic-container-service)

- [AWS Credentials Action과 OIDC](https://github.com/aws-actions/configure-aws-credentials)

- [Amazon ECR Login Action](https://github.com/aws-actions/amazon-ecr-login)

- [Amazon ECS Render Task Definition Action](https://github.com/aws-actions/amazon-ecs-render-task-definition)

- [Amazon ECS Deploy Task Definition Action](https://github.com/aws-actions/amazon-ecs-deploy-task-definition)

- [Amazon ECS RegisterTaskDefinition API](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_RegisterTaskDefinition.html)

- [Django 5.2 Release Notes](https://docs.djangoproject.com/en/5.2/releases/5.2/)
