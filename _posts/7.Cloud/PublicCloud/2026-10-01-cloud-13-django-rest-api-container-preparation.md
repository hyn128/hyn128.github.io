---
title: Django REST API 생성과 Container 배포 준비
description: Python 가상환경에서 Django Project와 REST API Application을 만들고 설정·URL·검증 절차를 Container 배포 전 단계까지 구성합니다
date: 2026-10-01
updated_at: 2026-10-02
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> ECS에 Application을 배포하기 전에 Local에서 재현 가능한 Django Project와 REST Endpoint를 준비합니다. Python 가상환경으로 의존성을 격리하고, Django Project와 Application의 역할을 분리하며, 환경 변수로 Host와 Secret을 주입할 수 있게 구성합니다. 이 글은 Container Image, ECR와 ECS 배포 전 단계인 Application 생성과 Local 검증까지 다룹니다.

## 1. Django Project와 Application

---

Django에서 Project와 Application은 역할이 다릅니다.

| 구분 | 역할 | 이 글의 이름 |
| --- | --- | --- |
| Project | 전체 Site의 설정, Root URL, WSGI·ASGI Entry Point 관리 | `apiserver` |
| Application | 특정 Domain의 View, Model, URL과 Test 관리 | `api` |

하나의 Project는 여러 Application을 포함할 수 있습니다. `apiserver` Package 자체를 업무 기능 Application처럼 `INSTALLED_APPS`에 추가하지 않고 실제 기능을 담당하는 `api` Application을 생성합니다.

## 2. 작업 Directory와 가상환경 생성

---

Python 가상환경은 Project별 Package Version을 분리합니다. Python 3.10 이상을 확인합니다.

```bash
python3 --version
```

Project Directory를 만들고 이동합니다.

```bash
mkdir django-api
cd django-api
```

`.venv` Directory에 가상환경을 생성합니다.

```bash
python3 -m venv .venv
```

운영체제와 Shell에 맞는 명령으로 활성화합니다.

| 환경 | 명령 |
| --- | --- |
| Linux·macOS Bash/Zsh | `source .venv/bin/activate` |
| Windows PowerShell | `.venv\Scripts\Activate.ps1` |
| Windows Command Prompt | `.venv\Scripts\activate.bat` |

활성화된 Python과 `pip`가 `.venv`를 가리키는지 확인합니다.

```bash
python -c 'import sys; print(sys.executable)'
python -m pip --version
```

## 3. Django와 Django REST Framework 설치

---

이 예제는 장기 지원 버전인 Django 5.2 계열을 사용합니다. `pip` 자체를 갱신한 뒤 Django와 Django REST Framework를 설치합니다.

```bash
python -m pip install --upgrade pip
python -m pip install "Django==5.2.*" djangorestframework
```

설치 Version을 확인합니다.

```bash
python -m django --version
python -m pip show djangorestframework
```

`pip` 대신 `python -m pip`를 사용하면 현재 활성화한 Python Interpreter와 Package 설치 위치를 일치시키기 쉽습니다.

## 4. Project와 Application 생성

---

현재 Directory에 `manage.py`와 `apiserver` Package를 생성합니다.

```bash
django-admin startproject apiserver .
```

REST Endpoint를 담당할 `api` Application을 생성합니다.

```bash
python manage.py startapp api
```

생성된 주요 구조는 다음과 같습니다.

```text
django-api/
├── .venv/
├── api/
│   ├── migrations/
│   ├── apps.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── apiserver/
│   ├── settings.py
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
└── manage.py
```

| File | 역할 |
| --- | --- |
| `manage.py` | 개발 Server, Migration, Test와 점검 명령 실행 |
| `apiserver/settings.py` | Application, Database, Time Zone와 보안 설정 |
| `apiserver/urls.py` | Project의 Root URL Routing |
| `apiserver/wsgi.py` | WSGI Server의 Application Entry Point |
| `apiserver/asgi.py` | ASGI Server의 Application Entry Point |
| `api/views.py` | HTTP 요청 처리와 Response 생성 |
| `api/urls.py` | `api` Application 내부 URL Routing |

## 5. Project 설정

---

`apiserver/settings.py`에서 표준 Library의 `os`를 Import합니다.

```python
import os
from pathlib import Path
```

배포 환경에서 Secret Key를 Source Code에 고정하지 않도록 환경 변수에서 읽습니다.

```python
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
```

허용할 Host도 환경 변수로 주입합니다.

```python
ALLOWED_HOSTS = [
    host.strip()
    for host in os.environ.get(
        "DJANGO_ALLOWED_HOSTS",
        "127.0.0.1,localhost",
    ).split(",")
    if host.strip()
]
```

`ALLOWED_HOSTS = ["*"]`는 모든 Host Header를 허용하므로 운영 기본값으로 사용하지 않습니다. ALB Domain, Service Domain과 Health Check 경로에 실제로 필요한 Host만 지정합니다.

`INSTALLED_APPS`에는 Django REST Framework와 생성한 Application을 등록합니다.

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "rest_framework",
    "api",
]
```

Time Zone을 설정합니다.

```python
TIME_ZONE = "Asia/Seoul"
USE_TZ = True
```

`USE_TZ = True`이면 Django는 내부 DateTime을 UTC 기준으로 다루고 표시 단계에서 설정한 Time Zone으로 변환합니다.

## 6. Local 환경 변수 설정

---

Local Shell에서 개발용 값을 설정합니다. `<LOCAL_DEVELOPMENT_SECRET>`에는 실제 운영 Secret과 다른 충분히 긴 임의 값을 사용합니다.

```bash
export DJANGO_SECRET_KEY='<LOCAL_DEVELOPMENT_SECRET>'
export DJANGO_ALLOWED_HOSTS='127.0.0.1,localhost'
```

운영 환경에서는 ECS Task Definition의 평문 Environment Variable에 Secret을 직접 입력하지 않고 AWS Secrets Manager 또는 Systems Manager Parameter Store를 이용해 주입합니다. 실제 값과 `.env` File은 Git에 Commit하지 않습니다.

환경 변수 이름만 확인하고 값은 출력하지 않습니다.

```bash
test -n "$DJANGO_SECRET_KEY" && echo 'DJANGO_SECRET_KEY is set'
```

## 7. REST API View 작성

---

`api/views.py`에 GET 요청을 처리하는 View를 작성합니다.

```python
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response


@api_view(["GET"])
def index(request):
    data = {
        "result": "success",
        "data": [
            {"id": "user-001", "name": "Example User"},
            {"id": "user-002", "name": "Sample User"},
        ],
    }
    return Response(data, status=status.HTTP_200_OK)
```

`@api_view(["GET"])`는 허용할 HTTP Method를 제한하고 Django REST Framework가 Request와 Response를 처리하게 합니다. 다른 Method로 요청하면 `405 Method Not Allowed`가 반환됩니다.

## 8. Application URL 구성

---

`api/urls.py`를 새로 생성합니다.

```python
from django.urls import path

from .views import index


app_name = "api"

urlpatterns = [
    path("", index, name="index"),
]
```

Project의 `apiserver/urls.py`에서 `api` URL을 포함합니다.

```python
from django.contrib import admin
from django.urls import include, path


urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/", include("api.urls")),
]
```

Routing 흐름은 다음과 같습니다.

```text
GET /api/
  ↓ apiserver/urls.py
include("api.urls")
  ↓ api/urls.py
index View
  ↓
JSON Response
```

## 9. Database 초기화와 설정 점검

---

Django 기본 Application이 사용하는 Database Table을 생성합니다.

```bash
python manage.py migrate
```

Project 설정과 URL 구성을 점검합니다.

```bash
python manage.py check
```

Migration 상태를 확인합니다.

```bash
python manage.py showmigrations
```

기본 설정은 Local SQLite File을 사용합니다. Container를 교체하면 Container 내부의 SQLite File도 사라질 수 있으므로 운영 ECS 환경에서는 RDS 같은 외부 Database와 Network·Credential 구성을 별도로 준비합니다.

## 10. API Test 작성

---

`api/tests.py`에 Endpoint의 Status Code와 JSON 구조를 검증하는 Test를 작성합니다.

```python
from django.test import TestCase
from django.urls import reverse


class IndexApiTest(TestCase):
    def test_index_returns_success(self):
        response = self.client.get(reverse("api:index"))

        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json()["result"], "success")
        self.assertEqual(len(response.json()["data"]), 2)
```

Test를 실행합니다.

```bash
python manage.py test
```

Test가 통과해야 Build Pipeline이 다음 단계로 진행하도록 CI에 연결할 수 있습니다.

## 11. 개발 Server 실행과 검증

---

Django 개발 Server를 Local Loopback Address에서 실행합니다.

```bash
python manage.py runserver 127.0.0.1:8000
```

다른 Terminal에서 API를 호출합니다.

```bash
curl -i http://127.0.0.1:8000/api/
```

예상 Body는 다음과 같습니다.

```json
{
  "result": "success",
  "data": [
    {
      "id": "user-001",
      "name": "Example User"
    },
    {
      "id": "user-002",
      "name": "Sample User"
    }
  ]
}
```

지원하지 않는 Method가 거부되는지도 확인합니다.

```bash
curl -i -X POST http://127.0.0.1:8000/api/
```

`runserver`는 개발용이며 Production Traffic을 처리하도록 설계되지 않았습니다. Container 배포에서는 Gunicorn 같은 Production WSGI Server 또는 요구 사항에 맞는 ASGI Server를 사용합니다.

## 12. 의존성 고정

---

현재 가상환경의 Package Version을 `requirements.txt`로 저장합니다.

```bash
python -m pip freeze > requirements.txt
```

생성된 File을 확인합니다.

```bash
sed -n '1,120p' requirements.txt
```

다른 환경에서 동일한 의존성을 설치할 때 사용합니다.

```bash
python -m pip install -r requirements.txt
```

`pip freeze`는 직접 설치하지 않은 하위 의존성도 모두 기록합니다. 운영 Project에서는 Dependabot 같은 도구로 취약점과 Update를 관리하고 변경 후 Test를 수행합니다.

## 13. Container 배포 전 점검

---

Application을 Image로 만들기 전에 다음 항목을 확인합니다.

| 점검 항목 | 확인 방법 |
| --- | --- |
| 설정 오류 | `python manage.py check` |
| Unit Test | `python manage.py test` |
| 의존성 고정 | `requirements.txt` 생성·재설치 확인 |
| Secret 분리 | 실제 Secret이 Source와 Git History에 없는지 검사 |
| Host 제한 | 운영 Domain과 Health Check Host만 허용 |
| Database | Container 외부 Database와 Migration 전략 준비 |
| Static File | `collectstatic`과 S3·CloudFront 사용 여부 결정 |
| Process Server | `runserver` 대신 Production WSGI·ASGI Server 선택 |
| Health Check | Container와 Load Balancer가 호출할 Endpoint 정의 |
| Log | Standard Output·Error로 출력해 중앙 수집 준비 |

현재 단계의 요청 흐름은 다음과 같습니다.

```text
Local Client
  ↓ GET /api/
Django Development Server
  ↓ URL Resolver
DRF Function View
  ↓
JSON Response
```

후속 Container 배포에서는 `Dockerfile → Image Build → ECR Push → ECS Task Definition → ECS Service` 흐름으로 확장합니다. 이 글에는 아직 Dockerfile, ECR Repository와 ECS Resource를 생성하는 절차를 포함하지 않습니다.

> **최종 정리**
> - Django Project는 전체 설정과 Root URL을 관리하고 Application은 실제 기능을 분리합니다.
>
> - `SECRET_KEY`와 허용 Host는 환경 변수로 주입하고 실제 값을 Repository에 저장하지 않습니다.
>
> - REST Endpoint는 Application URL을 Project URL에 포함해 연결합니다.
>
> - `check`, Test와 Local HTTP 호출을 통과한 Application을 다음 단계에서 Container Image로 만듭니다.

## 참고 자료

---

- [Django 5.2 첫 Application 작성](https://docs.djangoproject.com/en/5.2/intro/tutorial01/)

- [Django 배포 점검 목록](https://docs.djangoproject.com/en/5.2/howto/deployment/checklist/)

- [Django REST Framework 설치](https://www.django-rest-framework.org/#installation)

- [Python venv](https://docs.python.org/3/library/venv.html)

다음 글에서는 [Nginx로 이해하는 ECS Task·Service와 ALB]({% post_url 2026-10-02-cloud-14-aws-ecs-fargate-nginx-service %})에서 ECS의 실행 구조와 Service 배포 흐름을 구성합니다.
