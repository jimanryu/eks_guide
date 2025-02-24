# Chapter01. 사전 준비

AWS의 IAM 사용자 설정 및 AWS CLI, kubectl, eksctl 설치

# EKS 환경에서 도구 간 관계
- AWS CLI : AWS 리소스 전반 관리 및 인증
- kubectl : 클러스터 내부 Kubernetes 리소스 관리 및 모니터링
- eksctl : 클러스터 라이프사이클 관리 및 구성

# AWS CLI란?
AWS CLI(Command Line Interface)는 AWS 서비스를 쉽게 관리할 수 있는 명령줄 도구입니다. AWS CLI를 사용하면 AWS Management Console을 통하지 않고도 AWS 서비스를 관리할 수 있습니다.
# AWS CLI 장점
1. 자동화
AWS CLI를 사용하면 반복적인 작업을 자동화할 수 있습니다. 예를 들어, EC2 인스턴스를 생성하거나 S3 버킷을 생성하는 등의 작업을 자동화할 수 있습니다.

2. 편리성
AWS CLI는 명령줄 도구이기 때문에 GUI(Graphical User Interface)보다 빠르고 효율적입니다. 또한, 작업을 수행하는 데 필요한 단계를 줄일 수 있습니다.

3. 다양한 환경에서 사용 가능
Windows, macOS, Linux 등 다양한 운영체제에서 사용할 수 있습니다. 또한, 다른 명령줄 도구와 함께 사용할 수 있습니다.

4. 스크립트 작성 가능
AWS CLI를 사용하여 Bash, PowerShell 등의 스크립트를 작성할 수 있습니다. 이를 통해 자동화된 작업을 수행하거나, AWS 서비스를 통합한 자신만의 도구를 만들 수 있습니다.

5. 보안
AWS CLI는 AWS IAM(Identity and Access Management)을 사용하여 보안성을 보장합니다. IAM을 사용하여 권한을 관리하고, 액세스 키를 사용하여 AWS 계정에 대한 보안을 강화할 수 있습니다.

AWS CLI는 AWS 관리자, 개발자 및 운영자 모두에게 유용한 도구입니다. AWS CLI를 사용하면 AWS 서비스를 더욱 쉽게 관리하고, 효율적으로 자동화할 수 있습니다.

# AWS CLI 설치
1. Linux
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```
2. macOS
```bash
# Homebrew를 사용한 설치 (권장)
brew install awscli

# 또는 공식 패키지 설치
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /

# 설치 확인
aws --version
```

# kubectl 이란?
kubectl은 Kubernetes 클러스터를 제어하기 위한 명령줄 도구입니다. AWS EKS를 포함한 모든 Kubernetes 클러스터와 통신하는 공식 클라이언트입니다.
kubectl은 Kubernetes 클러스터 관리자, 개발자 및 운영자 모두에게 필수적인 도구입니다. kubectl을 사용하면 Kubernetes 환경에서 애플리케이션과 인프라를 효과적으로 관리하고 모니터링할 수 있습니다.

# kubectl 설치
1. linux
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```
2. macOS
```bash
# Homebrew를 사용한 설치 (권장)
brew install kubectl

# 또는 공식 바이너리 설치
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl

# M1/M2 맥북인 경우 arm64 버전 사용
# curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"

# 설치 확인
kubectl version --client
```

# eksctl 이란?
eksctl은 AWS EKS 클러스터를 생성하고 관리하기 위해 특별히 설계된 명령줄 도구입니다. AWS에서 공식 지원하는 오픈소스 도구 입니다.
eksctl은 AWS EKS를 사용하는 클라우드 엔지니어, 개발자 및 DevOps 전문가에게 매우 유용한 도구입니다. eksctl을 사용하면 EKS 클러스터 구축 시간을 단축하고, 반복 가능한 방식으로 일관된 환경을 구성할 수 있습니다.

# eksctl 설치
1. linux
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```
2. macOS
```bash
# Homebrew를 사용한 설치 (권장)
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# 또는 공식 바이너리 설치
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# 설치 확인
eksctl version
```

# AWS IAM 계정의 Access Key ID, Secret Access Key 발급
![보안 자격 증명](/images/ch01-01.png)
- AWS의 IAM 계정으로 로그인 후 우측 상단에 아이디를 클릭 후 [보안 자격 증명]을 클릭합니다.

![액세스 키 만들기 1](/images/ch01-02.png)
- '액세스 키 만들기'를 클릭합니다.


![액세스 키 만들기 2](/images/ch01-03.png)
- AWS CLI를 사용하기 때문에 'Command Line Interface(CLI)'를 선택하고 하단에 '확인'도 체크한뒤 '다음'을 클릭합니다.

![액세스 키 만들기 3](/images/ch01-04.png)
- 생성하는 액세스 키에 대한 태그 설정입니다.(선택사항)

![액세스 키 만들기 4](/images/ch01-05.png)
- Access Key와 Secret Access Key가 생성되었습니다. Secret Access Key는 생성 시 최초 한번만 확인 가능합니다.(표시 버튼을 클릭하면 확인 할 수 있습니다.) .csv 파일 다운로드를 하여 저장하거나 안전한 곳에 복사하는것을 권장합니다.

# AWS 자격 증명 구성
```bash
aws configure
# AWS Access Key ID, Secret Access Key 입력
# 리전, 출력 형식 설정
AWS Access Key ID [None]: # AWS Access Key ID 입력 
AWS Secret Access Key [None]: # AWS Secret Access Key 입력
Default region name [None]: # 리전 입력
Default output format [None]: # 출력 형식 설정 (json/text/table)
```
모두 입력 후 다시 'aws configure'를 실행하면 설정된 값을 확인 할 수 있다.
```bash
AWS Access Key ID [****************IDO7]: # Access Key 
AWS Secret Access Key [****************P0ub]: # Secret Access Key
Default region name [ap-northeast-2]: # 리전 (서울)
Default output format [json]: # 출력 형식 (json)
```