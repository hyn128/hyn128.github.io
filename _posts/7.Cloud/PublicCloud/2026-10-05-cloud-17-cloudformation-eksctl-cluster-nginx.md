---
title: CloudFormation Network와 eksctl로 EKS Cluster 구축
description: CloudFormation으로 EKS 실습용 VPC와 Public Subnet을 만들고 eksctl로 Managed Node Group Cluster를 생성한 뒤 Nginx Pod, Port Forwarding과 LoadBalancer Service를 검증합니다
date: 2026-10-05
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> CloudFormation으로 VPC, 세 개의 Public Subnet, Internet Gateway와 Route Table을 먼저 생성합니다. Operator PC에서는 임시 AWS Credential을 사용하는 Profile로 `eksctl`을 실행해 EKS Control Plane과 Managed Node Group을 만들고, `kubectl`로 Nginx Pod를 배포합니다. 마지막에는 Load Balancer를 먼저 제거한 뒤 Cluster를 삭제해 외부 Resource와 비용이 남지 않게 합니다.

## 1. 실습 구조와 실행 위치

---

이 실습은 다음 위치를 구분합니다.

| 표기 | 실행 위치 | 작업 |
| --- | --- | --- |
| `[OPERATOR]` | Local PC 또는 관리용 VM | AWS CLI, `eksctl`, `kubectl` 실행 |
| `[AWS-CONSOLE]` | AWS Management Console | CloudFormation Event와 생성 Resource 확인 |
| `[EKS-CONTROL-PLANE]` | AWS가 관리하는 EKS Control Plane | API 요청, Scheduling과 Cluster 상태 관리 |
| `[WORKER]` | EKS Managed Node Group의 EC2 | kubelet이 Pod Container 실행 |

```text
[OPERATOR] eksctl create cluster
  ↓ AWS API
[EKS-CONTROL-PLANE] Cluster 생성
  ↓ Managed Node Group 연결
[WORKER] EC2 Node 등록
  ↓ kubectl apply
Pod 실행
```

Control Plane 명령을 Worker Node에 접속해 실행하지 않습니다. Operator가 Kubernetes API Server에 요청하고 Control Plane이 Node의 kubelet에 원하는 상태를 전달합니다.

## 2. 인증 방식 결정

---

IAM User에 `IAMFullAccess`, `AmazonVPCFullAccess` 같은 광범위한 정책을 연결하고 장기 Access Key를 발급하는 방식은 사용하지 않습니다. 관리자는 IAM Identity Center 또는 AssumeRole로 수명이 짧은 Credential을 사용하고, Cluster 생성에 필요한 IAM Role을 별도로 설계합니다.

IAM Identity Center를 사용하는 Profile을 구성합니다.

```bash
# [OPERATOR]
aws configure sso --profile eks-lab
```

인증 후 현재 Principal과 Region을 확인합니다. Account ID는 게시물이나 실행 화면에 노출하지 않습니다.

```bash
# [OPERATOR]
aws sts get-caller-identity \
  --profile eks-lab

aws configure get region \
  --profile eks-lab
```

사람의 Cluster 접근은 EKS Access Entry로 관리합니다. 신규 Cluster에서 Legacy `aws-auth` ConfigMap을 직접 편집해 사용자 권한을 추가하지 않습니다.

## 3. CLI 도구 준비

---

Operator PC에 다음 도구를 설치합니다.

| 도구 | 역할 | 확인 명령 |
| --- | --- | --- |
| AWS CLI v2 | AWS API 호출과 Profile 인증 | `aws --version` |
| `eksctl` | EKS Cluster와 Node Group 구성 | `eksctl version` |
| `kubectl` | Kubernetes API 조작 | `kubectl version --client` |

운영체제별 설치 파일은 공식 문서에서 확인합니다.

- [AWS CLI v2 설치](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

- [eksctl 설치](https://eksctl.io/installation/)

- [kubectl 설치](https://kubernetes.io/docs/tasks/tools/)

`kubectl`은 EKS Cluster Version과 호환되는 Version을 사용합니다. 설치 후 명령 경로까지 확인합니다.

```bash
# [OPERATOR]
command -v aws
command -v eksctl
command -v kubectl
```

## 4. 실습용 VPC Template 작성

---

다음 Template을 `eks-base-network.yaml`로 저장합니다. 이 구성은 흐름 확인을 위해 Worker Subnet을 Public으로 만듭니다.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Public network for an EKS learning environment

Parameters:
  ClusterName:
    Type: String
    Default: eks-work-cluster

  VpcBlock:
    Type: String
    Default: 192.168.0.0/16

  WorkerSubnet1Block:
    Type: String
    Default: 192.168.0.0/24

  WorkerSubnet2Block:
    Type: String
    Default: 192.168.1.0/24

  WorkerSubnet3Block:
    Type: String
    Default: 192.168.2.0/24

Resources:
  EksWorkVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcBlock
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub "${ClusterName}-vpc"

  WorkerSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref EksWorkVPC
      AvailabilityZone: ap-northeast-2a
      CidrBlock: !Ref WorkerSubnet1Block
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub "${ClusterName}-public-a"
        - Key: kubernetes.io/role/elb
          Value: "1"
        - Key: !Sub "kubernetes.io/cluster/${ClusterName}"
          Value: shared

  WorkerSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref EksWorkVPC
      AvailabilityZone: ap-northeast-2b
      CidrBlock: !Ref WorkerSubnet2Block
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub "${ClusterName}-public-b"
        - Key: kubernetes.io/role/elb
          Value: "1"
        - Key: !Sub "kubernetes.io/cluster/${ClusterName}"
          Value: shared

  WorkerSubnet3:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref EksWorkVPC
      AvailabilityZone: ap-northeast-2c
      CidrBlock: !Ref WorkerSubnet3Block
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub "${ClusterName}-public-c"
        - Key: kubernetes.io/role/elb
          Value: "1"
        - Key: !Sub "kubernetes.io/cluster/${ClusterName}"
          Value: shared

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref EksWorkVPC
      InternetGatewayId: !Ref InternetGateway

  WorkerSubnetRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref EksWorkVPC
      Tags:
        - Key: Name
          Value: !Sub "${ClusterName}-public-route-table"

  WorkerSubnetRoute:
    Type: AWS::EC2::Route
    DependsOn: VPCGatewayAttachment
    Properties:
      RouteTableId: !Ref WorkerSubnetRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  WorkerSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref WorkerSubnet1
      RouteTableId: !Ref WorkerSubnetRouteTable

  WorkerSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref WorkerSubnet2
      RouteTableId: !Ref WorkerSubnetRouteTable

  WorkerSubnet3RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref WorkerSubnet3
      RouteTableId: !Ref WorkerSubnetRouteTable

Outputs:
  VPC:
    Value: !Ref EksWorkVPC

  WorkerSubnets:
    Value: !Join
      - ","
      - - !Ref WorkerSubnet1
        - !Ref WorkerSubnet2
        - !Ref WorkerSubnet3

  RouteTable:
    Value: !Ref WorkerSubnetRouteTable
```

Public Worker Node는 Public IP를 가지므로 운영 기본 구조로 사용하지 않습니다. 운영 환경에서는 Worker Node를 Private Subnet에 배치하고 NAT Gateway 또는 VPC Endpoint로 필요한 AWS Service에 접근하게 합니다.

## 5. CloudFormation Stack 생성

---

Template 문법을 확인합니다.

```bash
# [OPERATOR]
aws cloudformation validate-template \
  --template-body file://eks-base-network.yaml \
  --profile eks-lab \
  --region ap-northeast-2
```

Stack을 생성하고 완료될 때까지 기다립니다.

```bash
# [OPERATOR]
aws cloudformation create-stack \
  --stack-name eks-work-base \
  --template-body file://eks-base-network.yaml \
  --profile eks-lab \
  --region ap-northeast-2

aws cloudformation wait stack-create-complete \
  --stack-name eks-work-base \
  --profile eks-lab \
  --region ap-northeast-2
```

실패하면 Stack Event부터 확인합니다.

```bash
# [OPERATOR]
aws cloudformation describe-stack-events \
  --stack-name eks-work-base \
  --profile eks-lab \
  --region ap-northeast-2 \
  --query 'StackEvents[0:10].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]' \
  --output table
```

생성한 Subnet ID를 Shell 변수로 받습니다. 화면이나 문서에는 실제 ID를 복사하지 않습니다.

```bash
# [OPERATOR]
worker_subnets=$(aws cloudformation describe-stacks \
  --stack-name eks-work-base \
  --profile eks-lab \
  --region ap-northeast-2 \
  --query 'Stacks[0].Outputs[?OutputKey==`WorkerSubnets`].OutputValue' \
  --output text)

test -n "$worker_subnets" && echo 'Worker subnet output is set'
```

## 6. EKS Cluster와 Managed Node Group 생성

---

현재 EKS에서 지원하는 Kubernetes `1.36`과 Managed Node Group을 사용합니다. Version은 실행 시점의 EKS 지원 목록을 다시 확인합니다.

```bash
# [OPERATOR]
eksctl create cluster \
  --name eks-work-cluster \
  --region ap-northeast-2 \
  --version 1.36 \
  --vpc-public-subnets "$worker_subnets" \
  --nodegroup-name eks-work-nodegroup \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed \
  --profile eks-lab
```

| Option | 역할 |
| --- | --- |
| `--vpc-public-subnets` | 기존 VPC에서 사용할 Subnet을 전달합니다. |
| `--version` | EKS Control Plane의 Kubernetes Version을 지정합니다. |
| `--managed` | EKS Managed Node Group을 생성합니다. |
| `--nodes` | 최초 Desired Node 수입니다. |
| `--nodes-min`, `--nodes-max` | Node Group의 최소·최대 Capacity입니다. |

`t2.small`은 현재 Kubernetes System Pod를 함께 실행하는 실습에 CPU·Memory 여유가 적습니다. 예제는 `t3.medium`으로 변경했으며, 실제 Region의 지원 여부와 비용을 확인합니다.

`eksctl`은 Control Plane과 Node Group을 위한 CloudFormation Stack을 추가하고 kubeconfig Context를 갱신합니다. Node 수의 최소·최대값만 지정했다고 Pod 부하에 따라 자동으로 확장되는 것은 아닙니다. Cluster Autoscaler나 EKS Auto Mode 같은 Scaling 구성 요소가 별도로 필요합니다.

## 7. kubeconfig와 Node 확인

---

현재 Context를 확인합니다.

kubeconfig의 주요 영역은 다음과 같습니다.

| 영역 | 역할 |
| --- | --- |
| `clusters` | Kubernetes API Server URL과 CA 정보 |
| `users` | AWS CLI를 이용해 EKS 인증 Token을 얻는 실행 설정 |
| `contexts` | Cluster와 User, 기본 Namespace의 조합 |
| `current-context` | `kubectl`이 현재 사용할 Context |

```bash
# [OPERATOR]
kubectl config get-contexts
kubectl config current-context
```

필요한 경우 Context를 명시적으로 선택합니다.

```bash
# [OPERATOR]
kubectl config use-context <EKS_CONTEXT_NAME>
```

Node 상태와 상세 정보를 확인합니다.

```bash
# [OPERATOR]
kubectl get nodes -o wide
kubectl describe node <NODE_NAME>
```

Node가 `Ready`가 되면 kubelet이 Control Plane에 등록됐고 Pod를 받을 수 있는 상태입니다. `NotReady`이면 VPC CNI, kube-proxy, Node IAM Role과 Node Log를 점검합니다.

## 8. Nginx Pod 배포

---

`nginx-pod.yaml`을 작성합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx-app
spec:
  containers:
    - name: nginx-container
      image: nginx:stable-alpine
      ports:
        - containerPort: 80
```

Pod를 생성하고 Scheduling 결과를 확인합니다.

```bash
# [OPERATOR]
kubectl apply -f nginx-pod.yaml
kubectl get pod nginx-pod -o wide
kubectl describe pod nginx-pod
```

Control Plane의 Scheduler가 Worker Node를 선택하면 해당 Node의 kubelet이 Nginx Image를 받고 Container를 실행합니다.

## 9. Port Forwarding 검증

---

Local Port 8080을 Pod의 Port 80에 연결합니다.

```bash
# [OPERATOR]
kubectl port-forward pod/nginx-pod 8080:80
```

이 Terminal은 연결을 유지합니다. 다른 Terminal에서 Nginx 응답을 확인합니다.

```bash
# [OPERATOR]
curl -I http://127.0.0.1:8080/
```

`HTTP/1.1 200 OK`를 확인한 뒤 Port Forwarding Terminal에서 `Ctrl+C`로 종료합니다. Port Forwarding은 Local 점검용이며 외부 Service를 대신하지 않습니다.

## 10. LoadBalancer Service

---

`nginx-service.yaml`을 작성합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx-app
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
```

Service를 생성하고 외부 주소가 할당될 때까지 확인합니다.

```bash
# [OPERATOR]
kubectl apply -f nginx-service.yaml
kubectl get service nginx-service --watch
```

외부 주소가 할당되면 HTTP 응답을 확인합니다.

```bash
# [OPERATOR]
curl -I http://<LOAD_BALANCER_DNS_NAME>/
```

기본 Service Controller가 처리하는 Cluster에서는 Legacy CLB가 만들어질 수 있습니다. 이 Controller는 Critical Bug Fix 중심으로 유지됩니다. 새 환경에서는 AWS Load Balancer Controller가 만드는 NLB 또는 EKS Auto Mode의 Load Balancing 기능을 사용하고, Controller가 요구하는 `loadBalancerClass`와 Annotation을 별도로 적용합니다.

## 11. Resource 삭제

---

EKS Control Plane은 Workload가 없어도 비용이 발생합니다. Load Balancer Resource부터 삭제합니다.

```bash
# [OPERATOR]
kubectl get service --all-namespaces
kubectl get ingress --all-namespaces
kubectl delete service nginx-service
kubectl delete pod nginx-pod
```

Service의 `EXTERNAL-IP`와 AWS Load Balancer가 제거됐는지 확인한 뒤 Cluster를 삭제합니다.

```bash
# [OPERATOR]
eksctl delete cluster \
  --name eks-work-cluster \
  --region ap-northeast-2 \
  --profile eks-lab
```

`eksctl`이 만든 Stack이 모두 삭제된 뒤 기반 Network Stack을 삭제합니다.

```bash
# [OPERATOR]
aws cloudformation delete-stack \
  --stack-name eks-work-base \
  --profile eks-lab \
  --region ap-northeast-2

aws cloudformation wait stack-delete-complete \
  --stack-name eks-work-base \
  --profile eks-lab \
  --region ap-northeast-2
```

Cluster를 먼저 삭제하면 Load Balancer와 Network Interface가 VPC에 남아 Network Stack 삭제가 실패할 수 있습니다.

## 12. RDS 연결 전 확인

---

EKS Application이 RDS를 사용할 때 Database를 Worker와 같은 Public Subnet에 둘 필요는 없습니다. RDS는 Private Subnet Group에 배치하고 Database Security Group은 Application에서 오는 Database Port만 허용합니다.

RDS는 Managed Service이므로 Database Host OS에 로그인하지 않습니다. 관리 작업이 필요하면 Public Bastion Host를 유일한 진입점으로 두기보다 AWS Systems Manager Session Manager를 우선 검토합니다. Database Credential은 Secrets Manager에 저장하고 Pod에는 EKS Pod Identity나 IRSA를 통해 필요한 조회 권한만 부여합니다.

이 절에서는 Network와 접근 원칙만 정리하며 RDS Instance, Database User와 Spring Application 구성은 후속 범위로 남깁니다.

> **최종 정리**
> - Operator는 임시 AWS Credential로 CloudFormation, `eksctl`과 `kubectl`을 실행합니다.
>
> - CloudFormation Stack의 Output을 받아 실제 Subnet ID를 문서에 노출하지 않고 Cluster 생성에 전달합니다.
>
> - `kubectl apply` 요청은 Control Plane을 거쳐 Worker Node의 kubelet과 Container 실행으로 이어집니다.
>
> - Port Forwarding은 Local 검증용이고 `LoadBalancer` Service는 AWS Load Balancer Resource를 생성합니다.
>
> - 삭제할 때는 외부 Service와 Ingress, EKS Cluster, 기반 Network Stack 순서를 지킵니다.

## 참고 자료

---

- [Amazon EKS Cluster 생성](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html)

- [eksctl Cluster 생성과 관리](https://docs.aws.amazon.com/eks/latest/eksctl/creating-and-managing-clusters.html)

- [Amazon EKS Kubernetes Version 수명주기](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)

- [Amazon EKS Cluster 삭제](https://docs.aws.amazon.com/eks/latest/userguide/delete-cluster.html)

- [Amazon EKS Load Balancing](https://docs.aws.amazon.com/eks/latest/best-practices/load-balancing.html)

다음 글에서는 [CloudWatch Metric·Logs·Alarm과 SNS 알림]({% post_url 2026-10-06-cloud-18-cloudwatch-metrics-logs-alarm-sns %})을 구성합니다.
