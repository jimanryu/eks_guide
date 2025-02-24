---

## 1. 목표 이해
- **설명**: 이 과제의 목표는 AWS EKS(Elastic Kubernetes Service) 클러스터를 구축하고, 모니터링 도구인 Grafana와 Prometheus를 배포하여 외부에서 HTTPS로 접근 가능하도록 설정하는 것입니다.
- **필요 조건**: 
  - AWS 계정 (IAM 사용자 인증 정보 필요).
  - 기본 네트워크 개념 이해 (VPC, 서브넷, 라우팅 테이블 등).
  - 로컬 환경에 설치된 도구: AWS CLI, kubectl, Helm.
- **결과물**: 브라우저에서 `https://grafana.example.com`으로 Grafana 로그인 페이지에 접근 가능.
- **중요 고려사항**: EKS는 VPC 내에서 실행되며, ALB(Application Load Balancer)를 통해 외부 트래픽을 받아야 하므로 퍼블릭 서브넷과 인터넷 게이트웨이가 필수입니다.

---

## 2. AWS 환경 준비
- **설명**: EKS 클러스터와 ALB를 관리하기 위한 AWS 환경을 설정합니다. IAM 권한과 CLI 설정이 선행되어야 VPC와 EKS를 구성할 수 있습니다.
- **세부 단계**:
  1. **IAM 역할 생성**  
     - **왜 필요?**: EKS 클러스터와 ALB를 생성/관리하려면 AWS 리소스에 대한 권한이 필요합니다. 예를 들어, `AmazonEKSClusterPolicy`는 EKS 클러스터를 만들고, `AWSLoadBalancerControllerIAMPolicy`는 ALB를 설정하는 데 필수입니다.
     - **방법**:
       - AWS 콘솔: IAM > 역할 > 역할 생성.
         - 신뢰 관계: EKS 서비스 (`eks.amazonaws.com`).
         - 정책: `AmazonEKSClusterPolicy`, `AmazonEC2FullAccess`, `IAMFullAccess`.
         - 이름: `eks-admin-role`.
       - CLI:
         ```bash
         aws iam create-role --role-name eks-admin-role --assume-role-policy-document file://trust-policy.json
         aws iam attach-role-policy --role-name eks-admin-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
         aws iam attach-role-policy --role-name eks-admin-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
         ```
       - `trust-policy.json`:
         ```json
         {
           "Version": "2012-10-17",
           "Statement": [
             {"Effect": "Allow", "Principal": {"Service": "eks.amazonaws.com"}, "Action": "sts:AssumeRole"}
           ]
         }
         ```
     - **확인**: `aws iam list-attached-role-policies --role-name eks-admin-role`.

  2. **AWS CLI 설치 및 구성**  
     - **왜 필요?**: VPC, EKS, ALB 등을 명령어로 설정하려면 CLI가 필수입니다.
     - **방법**:
       - 설치 (Linux 예시):
         ```bash
         curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
         unzip awscliv2.zip
         sudo ./aws/install
         ```
       - 구성:
         ```bash
         aws configure
         ```
         - Access Key ID, Secret Access Key: IAM 사용자 인증 정보.
         - Region: `ap-northeast-2` (서울 리전).
         - Output format: `json`.
     - **확인**: `aws sts get-caller-identity`.

---

## 3. VPC 구성
- **설명**: EKS 클러스터와 ALB를 배포할 네트워크 환경을 설정합니다. 외부 접근을 위해 퍼블릭 서브넷과 인터넷 게이트웨이(IGW)를 포함하며, 프라이빗 서브넷은 EKS 노드와 Pod를 보호합니다.
- **세부 단계**:
  1. **VPC 생성**  
     - **왜 필요?**: 모든 서브넷, IGW, NAT 등이 포함될 네트워크 공간입니다.
     - **ID**: `vpc-example-001`.
     - **CIDR**: `10.0.0.0/16` (65,536개 IP 주소).
     - **CLI**:
       ```bash
       aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region ap-northeast-2
       ```
       - 결과에서 `VpcId` 기록.
     - **태그**: 
       ```bash
       aws ec2 create-tags --resources vpc-example-001 --tags Key=Name,Value=example-vpc
       ```

  2. **인터넷 게이트웨이(IGW) 생성 및 연결**  
     - **왜 필요?**: 퍼블릭 서브넷이 외부 인터넷과 통신하려면 IGW가 필수입니다. ALB가 외부 트래픽을 받기 위해 필요합니다.
     - **ID**: `igw-example-001`.
     - **CLI**:
       ```bash
       aws ec2 create-internet-gateway --region ap-northeast-2
       aws ec2 attach-internet-gateway --internet-gateway-id igw-example-001 --vpc-id vpc-example-001
       ```
     - **확인**: `aws ec2 describe-internet-gateways --internet-gateway-ids igw-example-001`.

  3. **서브넷 생성**  
     - **왜 필요?**: EKS 노드는 프라이빗 서브넷에, ALB는 퍼블릭 서브넷에 배포됩니다.
     - **퍼블릭 서브넷**:
       - `subnet-public-a-001` (10.0.2.0/24, ap-northeast-2a).
       - `subnet-public-c-002` (10.0.3.0/24, ap-northeast-2c).
       - CLI:
         ```bash
         aws ec2 create-subnet --vpc-id vpc-example-001 --cidr-block 10.0.2.0/24 --availability-zone ap-northeast-2a
         aws ec2 create-subnet --vpc-id vpc-example-001 --cidr-block 10.0.3.0/24 --availability-zone ap-northeast-2c
         ```
       - 태그:
         ```bash
         aws ec2 create-tags --resources subnet-public-a-001 --tags Key=Name,Value=example-subnet-public1-ap-northeast-2a
         aws ec2 create-tags --resources subnet-public-c-002 --tags Key=Name,Value=example-subnet-public2-ap-northeast-2c
         ```
     - **프라이빗 서브넷**:
       - `subnet-private-a-001` (10.0.0.0/24, ap-northeast-2a).
       - `subnet-private-c-002` (10.0.1.0/24, ap-northeast-2c).
       - CLI:
         ```bash
         aws ec2 create-subnet --vpc-id vpc-example-001 --cidr-block 10.0.0.0/24 --availability-zone ap-northeast-2a
         aws ec2 create-subnet --vpc-id vpc-example-001 --cidr-block 10.0.1.0/24 --availability-zone ap-northeast-2c
         ```
       - 태그:
         ```bash
         aws ec2 create-tags --resources subnet-private-a-001 --tags Key=Name,Value=example-subnet-private1-ap-northeast-2a
         aws ec2 create-tags --resources subnet-private-c-002 --tags Key=Name,Value=example-subnet-private2-ap-northeast-2c
         ```

  4. **라우팅 테이블 설정**  
     - **왜 필요?**: 트래픽 흐름을 제어합니다. 퍼블릭 서브넷은 IGW로, 프라이빗 서브넷은 NAT로 연결됩니다.
     - **퍼블릭 라우팅 테이블**:
       - **ID**: `rtb-public-001`.
       - **규칙**: `0.0.0.0/0` → `igw-example-001`.
       - CLI:
         ```bash
         aws ec2 create-route-table --vpc-id vpc-example-001 --region ap-northeast-2
         aws ec2 create-route --route-table-id rtb-public-001 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-example-001
         aws ec2 associate-route-table --route-table-id rtb-public-001 --subnet-id subnet-public-a-001
         aws ec2 associate-route-table --route-table-id rtb-public-001 --subnet-id subnet-public-c-002
         ```
       - 태그:
         ```bash
         aws ec2 create-tags --resources rtb-public-001 --tags Key=Name,Value=example-rtb-public
         ```
     - **프라이빗 라우팅 테이블** (NAT 연결은 다음 단계):
       - **ID**: `rtb-private-a-001`, `rtb-private-c-002`.
       - CLI:
         ```bash
         aws ec2 create-route-table --vpc-id vpc-example-001 --region ap-northeast-2
         aws ec2 associate-route-table --route-table-id rtb-private-a-001 --subnet-id subnet-private-a-001
         aws ec2 create-route-table --vpc-id vpc-example-001 --region ap-northeast-2
         aws ec2 associate-route-table --route-table-id rtb-private-c-002 --subnet-id subnet-private-c-002
         ```
       - 태그:
         ```bash
         aws ec2 create-tags --resources rtb-private-a-001 --tags Key=Name,Value=example-rtb-private1-ap-northeast-2a
         aws ec2 create-tags --resources rtb-private-c-002 --tags Key=Name,Value=example-rtb-private2-ap-northeast-2c
         ```

---

## 4. NAT 게이트웨이 설정
- **설명**: 프라이빗 서브넷에서 외부로의 통신(예: 소프트웨어 업데이트, 외부 API 호출)을 위해 NAT 게이트웨이를 설정합니다. NAT는 퍼블릭 서브넷에 배포되어야 합니다.
- **세부 단계**:
  1. **Elastic IP 할당**  
     - **왜 필요?**: NAT 게이트웨이는 고정 IP를 통해 외부와 통신합니다.
     - **ID**: `eip-example-a`, `eip-example-c`.
     - **CLI**:
       ```bash
       aws ec2 allocate-address --domain vpc --region ap-northeast-2
       ```
       - 결과에서 `AllocationId` 기록 (예: `eipalloc-001`, `eipalloc-002`).

  2. **NAT 게이트웨이 생성**  
     - **왜 필요?**: 프라이빗 서브넷이 인터넷으로 나가려면 NAT가 필요합니다.
     - **ID**: `nat-example-a` (퍼블릭 `subnet-public-a-001`), `nat-example-c` (퍼블릭 `subnet-public-c-002`).
     - **CLI**:
       ```bash
       aws ec2 create-nat-gateway --subnet-id subnet-public-a-001 --allocation-id eipalloc-001 --region ap-northeast-2
       aws ec2 create-nat-gateway --subnet-id subnet-public-c-002 --allocation-id eipalloc-002 --region ap-northeast-2
       ```
     - **확인**: `aws ec2 describe-nat-gateways --nat-gateway-ids nat-example-a nat-example-c` (`State: available` 대기).

  3. **프라이빗 라우팅 테이블 업데이트**  
     - **왜 필요?**: 프라이빗 서브넷이 NAT를 통해 외부로 나가도록 설정합니다.
     - **CLI**:
       ```bash
       aws ec2 create-route --route-table-id rtb-private-a-001 --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-example-a
       aws ec2 create-route --route-table-id rtb-private-c-002 --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-example-c
       ```

---

## 5. EKS 클러스터 생성
- **설명**: EKS 클러스터를 프라이빗 서브넷에 배포하여 Kubernetes 환경을 설정합니다.
- **세부 단계**:
  1. **EKS 클러스터 생성**  
     - **왜 필요?**: EKS는 Grafana와 Prometheus를 실행할 Kubernetes 클러스터를 제공합니다.
     - **이름**: `example-eks-cluster`.
     - **CLI**:
       ```bash
       aws eks create-cluster --name example-eks-cluster --role-arn arn:aws:iam::123456789012:role/eks-admin-role --resources-vpc-config subnetIds=subnet-private-a-001,subnet-private-c-002,securityGroupIds=sg-example-001 --region ap-northeast-2
       ```
       - `sg-example-001`: VPC 기본 보안 그룹 사용 가정 (필요 시 생성).
     - **대기**: 10~15분 소요.
     - **확인**: 
       ```bash
       aws eks describe-cluster --name example-eks-cluster --region ap-northeast-2
       ```
       - `status: ACTIVE` 확인.

  2. **kubectl 설정**  
     - **왜 필요?**: 로컬에서 EKS 클러스터를 제어하려면 kubeconfig 업데이트가 필요합니다.
     - **CLI**:
       ```bash
       aws eks update-kubeconfig --name example-eks-cluster --region ap-northeast-2
       ```
     - **확인**: `kubectl get nodes`.

---

## 6. 워커 노드 그룹 생성
- **설명**: EKS에서 Pod를 실행할 워커 노드를 프라이빗 서브넷에 배포합니다.
- **세부 단계**:
  1. **노드 역할 생성**  
     - **왜 필요?**: 워커 노드가 EKS와 통신하려면 권한이 필요합니다.
     - **이름**: `eks-node-role`.
     - **CLI**:
       ```bash
       aws iam create-role --role-name eks-node-role --assume-role-policy-document file://node-trust-policy.json
       aws iam attach-role-policy --role-name eks-node-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
       aws iam attach-role-policy --role-name eks-node-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
       ```
       - `node-trust-policy.json`:
         ```json
         {
           "Version": "2012-10-17",
           "Statement": [
             {"Effect": "Allow", "Principal": {"Service": "ec2.amazonaws.com"}, "Action": "sts:AssumeRole"}
           ]
         }
         ```
  2. **노드 그룹 생성**  
     - **왜 필요?**: Pod를 실행할 실제 컴퓨팅 리소스입니다.
     - **이름**: `example-node-group`.
     - **CLI**:
       ```bash
       aws eks create-nodegroup --cluster-name example-eks-cluster --nodegroup-name example-node-group --subnets subnet-private-a-001 subnet-private-c-002 --instance-types t3.medium --scaling-config minSize=1,maxSize=3,desiredSize=2 --node-role arn:aws:iam::123456789012:role/eks-node-role --region ap-northeast-2
       ```
     - **확인**: `kubectl get nodes` (노드 추가 확인).

---

## 7. ALB 컨트롤러 설치
- **설명**: Ingress를 통해 외부 트래픽을 ALB로 라우팅하려면 ALB 컨트롤러가 필요합니다.
- **세부 단계**:
  1. **IAM 정책 생성**  
     - **왜 필요?**: ALB 컨트롤러가 ALB를 생성/관리하려면 권한이 필요합니다.
     - **CLI**:
       ```bash
       curl -o alb-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
       aws iam create-policy --policy-name alb-controller-policy --policy-document file://alb-policy.json
       aws iam attach-role-policy --role-name eks-admin-role --policy-arn arn:aws:iam::123456789012:policy/alb-controller-policy
       ```
  2. **Helm으로 설치**  
     - **왜 필요?**: Helm은 패키지 관리 도구로, ALB 컨트롤러 배포를 간소화합니다.
     - **CLI**:
       ```bash
       helm repo add eks https://aws.github.io/eks-charts
       helm repo update
       helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=example-eks-cluster --set serviceAccount.create=true --set serviceAccount.name=aws-load-balancer-controller --set region=ap-northeast-2
       ```
     - **확인**: `kubectl get pods -n kube-system | grep aws-load-balancer`.

---

## 8. Grafana와 Prometheus 배포
- **설명**: 모니터링을 위해 Grafana와 Prometheus를 배포합니다.
- **세부 단계**:
  1. **네임스페이스 생성**  
     - **왜 필요?**: 모니터링 리소스를 별도 네임스페이스에서 관리.
     - **CLI**:
       ```bash
       kubectl create namespace monitoring
       ```
  2. **Prometheus 설치**  
     - **왜 필요?**: 클러스터 메트릭 수집.
     - **CLI**:
       ```bash
       helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
       helm repo update
       helm install prometheus prometheus-community/prometheus -n monitoring
       ```
     - **확인**: `kubectl get pods -n monitoring | grep prometheus`.
  3. **Grafana 설치**  
     - **왜 필요?**: 메트릭 시각화 및 대시보드.
     - **CLI**:
       ```bash
       helm repo add grafana https://grafana.github.io/helm-charts
       helm repo update
       helm install grafana grafana/grafana -n monitoring
       ```
     - **확인**: `kubectl get pods -n monitoring | grep grafana`.

---

## 9. Ingress 설정
- **설명**: ALB를 통해 Grafana에 외부 접근을 제공합니다.
- **세부 단계**:
  1. **ACM 인증서 생성**  
     - **왜 필요?**: HTTPS로 안전한 통신을 위해 인증서가 필요합니다.
     - **ARN**: `arn:aws:acm:ap-northeast-2:123456789012:certificate/example-cert-001`.
     - **콘솔**: ACM > 인증서 요청 > `*.example.com` 추가.
     - **CLI** (수동 대체): 콘솔 사용 권장.
  2. **Ingress YAML 작성**  
     - **왜 필요?**: ALB를 구성하고 Grafana 서비스로 라우팅.
     - `grafana-ingress.yaml`:
       ```yaml
       apiVersion: networking.k8s.io/v1
       kind: Ingress
       metadata:
         name: grafana-ingress
         namespace: monitoring
         annotations:
           alb.ingress.kubernetes.io/scheme: internet-facing
           alb.ingress.kubernetes.io/target-type: ip
           alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
           alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-2:123456789012:certificate/example-cert-001
       spec:
         ingressClassName: alb
         rules:
         - host: grafana.example.com
           http:
             paths:
             - path: /
               pathType: Prefix
               backend:
                 service:
                   name: grafana
                   port:
                     number: 80
       ```
     - **적용**:
       ```bash
       kubectl apply -f grafana-ingress.yaml
       ```
     - **확인**: `kubectl get ingress -n monitoring` (ALB DNS 확인).

---

## 10. Route 53 설정
- **설명**: 도메인(`grafana.example.com`)으로 ALB에 연결합니다.
- **세부 단계**:
  1. **ALB DNS 확인**  
     - **왜 필요?**: Route 53에 ALB를 매핑하려면 DNS가 필요합니다.
     - **CLI**:
       ```bash
       kubectl get ingress grafana-ingress -n monitoring
       ```
       - 예: `example-web-alb-123456.ap-northeast-2.elb.amazonaws.com`.
  2. **Route 53 레코드 추가**  
     - **왜 필요?**: 외부에서 도메인으로 접근 가능하도록 설정.
     - **CLI**:
       ```bash
       aws route53 change-resource-record-sets --hosted-zone-id ZEXAMPLE123 --change-batch '{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{"Name":"grafana.example.com","Type":"A","AliasTarget":{"HostedZoneId":"Z14GRHDC4JWTQ8","DNSName":"example-web-alb-123456.ap-northeast-2.elb.amazonaws.com","EvaluateTargetHealth":false}}}]}}'
       ```
     - **확인**: `dig +short grafana.example.com`.

---

## 11. 테스트 및 검증
- **설명**: 모든 설정이 올바르게 작동하는지 확인합니다.
- **세부 단계**:
  1. **DNS 해석 확인**  
     - **왜 필요?**: 도메인이 ALB IP로 해석되는지 확인.
     - **CLI**:
       ```bash
       dig +short grafana.example.com
       ```
  2. **접근 테스트**  
     - **왜 필요?**: HTTPS 연결과 Grafana 응답 확인.
     - **CLI**:
       ```bash
       curl -v https://grafana.example.com
       ```
       - 예상: `HTTP/2 302` (로그인 페이지로 리다이렉션).
  3. **브라우저 확인**  
     - **왜 필요?**: 최종 사용자 경험 검증.
     - 브라우저에서 `https://grafana.example.com` → Grafana 로그인 페이지 확인.
