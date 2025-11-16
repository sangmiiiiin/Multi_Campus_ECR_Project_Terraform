# ECR JobPosting Terraform Infrastructure

AWS에서 채용 공고 시스템을 위한 완전한 클라우드 인프라를 Terraform으로 관리하는 프로젝트입니다.

## 프로젝트 개요

이 프로젝트는 ECS Fargate 기반의 컨테이너 환경에서 Frontend와 Backend(Applicant, JobPosting) 서비스를 배포하고, 모니터링 시스템(Zabbix, Grafana)과 MySQL RDS를 포함한 완전한 3-tier 아키텍처를 구현합니다.

## 아키텍처 구성

```
┌─────────────────────────────────────────────────────────┐
│                    Internet Gateway                      │
└───────────────────────┬─────────────────────────────────┘
                        │
        ┌───────────────┴──────────────┐
        │                              │
┌───────▼────────┐            ┌────────▼────────┐
│   Web ALB      │            │    WAS ALB      │
│  (HTTPS/443)   │            │  (HTTPS/443)    │
└───────┬────────┘            └────────┬────────┘
        │                              │
┌───────┴──────────────────────────────┴────────┐
│              Private Subnets                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │ Frontend │  │Applicant │  │JobPosting│    │
│  │   ECS    │  │   ECS    │  │   ECS    │    │
│  └──────────┘  └──────────┘  └──────────┘    │
│                                                │
│  ┌──────────────────────────────┐            │
│  │    Monitoring EC2            │            │
│  │  (Zabbix + Grafana)          │            │
│  └──────────────────────────────┘            │
└────────────────────────────────────────────────┘
                        │
        ┌───────────────┴──────────────┐
        │                              │
┌───────▼────────┐            ┌────────▼────────┐
│  DB Subnet 1   │            │  DB Subnet 2    │
│  (MySQL RDS    │            │   Multi-AZ)     │
└────────────────┘            └─────────────────┘
```

## 주요 기능

### 1. VPC 네트워크 구성
- **VPC CIDR**: 10.0.0.0/16
- **가용 영역**: ap-northeast-2a, ap-northeast-2c
- **서브넷 구성**:
  - Public Subnet (10.0.0.0/24, 10.0.1.0/24)
  - Private Subnet (10.0.2.0/24, 10.0.3.0/24)
  - DB Subnet (10.0.4.0/24, 10.0.5.0/24)
- **네트워크 게이트웨이**:
  - Internet Gateway (퍼블릭 통신)
  - NAT Gateway (프라이빗 서브넷 아웃바운드)

### 2. Compute 리소스
- **Bastion Host**: Amazon Linux 3 (t3.medium)
  - 퍼블릭 서브넷 배치
  - SSH 접근 관리용
- **Monitoring Server**: Ubuntu 20.04 (t3.large)
  - Zabbix 모니터링
  - Grafana 대시보드
  - 프라이빗 서브넷 배치

### 3. RDS 데이터베이스
- **엔진**: MySQL 8.0
- **인스턴스**: db.t3.medium
- **스토리지**: 20GB GP3
- **고가용성**: Multi-AZ 배포
- **보안**: 프라이빗 서브넷, VPC 내부 접근만 허용

### 4. Application Load Balancer
#### Web ALB
- Frontend 트래픽 (HTTPS:443)
- Zabbix 모니터링 (zabbix.laurar.store)
- Grafana 대시보드 (grafana.laurar.store)
- HTTP to HTTPS 리다이렉트

#### WAS ALB
- Applicant 서비스 (applicant.laurar.store:8080)
- JobPosting 서비스 (jobposting.laurar.store:8888)
- Host Header 기반 라우팅

### 5. ECS Fargate 컨테이너
#### ECS Cluster
- Container Insights 활성화
- Auto Scaling 구성
  - CPU 사용률 50% 목표
  - 메모리 사용률 75% 목표
  - 최소 1개, 최대 5개 태스크

#### 서비스 구성
1. **Frontend Service**
   - 포트: 80
   - Desired Count: 2
   - CloudWatch Logs 통합

2. **Applicant Backend Service**
   - 포트: 8080
   - Desired Count: 2
   - Health Check Grace Period: 30초

3. **JobPosting Backend Service**
   - 포트: 8888
   - Desired Count: 2
   - Health Check Grace Period: 30초

### 6. ECR (Elastic Container Registry)
- Frontend 이미지 저장소
- Backend 이미지 저장소
- 태그 전략:
  - Frontend: `web`
  - Backend: `applicant`, `jobposting`

### 7. Backend State 관리
- **S3 Bucket**: ecr-jobposting-terraform-repo-1
  - Terraform State 파일 저장
  - 암호화 활성화
- **DynamoDB Table**: ecr-jobposting-terraform-lock
  - State 락 관리
  - 동시 실행 방지

## 디렉토리 구조

```
Multi_Campus_ECR_Project_Terraform/
├── main.tf                    # 메인 Terraform 설정
├── provider.tf                # AWS 프로바이더 및 백엔드 설정
├── .gitignore
├── .terraform.lock.hcl
└── module/
    ├── vpc/                   # VPC, Subnet, IGW, NAT, Route Table
    │   ├── vpc.tf
    │   ├── sg.tf              # Security Group 정의
    │   ├── output.tf
    │   └── variable.tf
    ├── compute/               # EC2, ALB, RDS
    │   ├── ec2.tf             # Bastion, Monitoring 서버
    │   ├── lb.tf              # Web/WAS ALB 설정
    │   ├── rds.tf             # MySQL RDS
    │   ├── output.tf
    │   └── variable.tf
    ├── container/             # ECS, ECR
    │   ├── ecs.tf             # ECS Cluster, Service, Task Definition
    │   ├── ecr.tf             # ECR Repository
    │   ├── output.tf
    │   └── variable.tf
    ├── backend/               # S3 Bucket (Terraform State)
    │   ├── backend.tf
    │   └── variables.tf
    └── lock/                  # DynamoDB Lock Table
        ├── lock.tf
        └── variables.tf
```

## 사전 요구사항

1. **Terraform 설치**
   ```bash
   terraform --version
   # Terraform v1.x.x 이상
   ```

2. **AWS CLI 설치 및 자격증명 설정**
   ```bash
   aws configure
   # AWS Access Key ID
   # AWS Secret Access Key
   # Default region: ap-northeast-2
   ```

3. **AWS 리소스 사전 준비**
   - ACM 인증서 발급 (laurar.store 도메인)
   - EC2 Key Pair 생성 (multi-key)
   - IAM Role: ecsTaskExecutionRole

## 설치 및 배포

### 1. 저장소 클론
```bash
git clone <repository-url>
cd Multi_Campus_ECR_Project_Terraform
```

### 2. 설정 값 수정
`main.tf` 파일에서 다음 값들을 환경에 맞게 수정하세요:

```hcl
locals {
  tag_name = "ecr-jobposting"        # 프로젝트 태그명
  region   = "ap-northeast-2"         # AWS 리전
  account  = "662947429974"           # AWS 계정 ID
}

module "vpc" {
  sg_office_ip = "0.0.0.0/0"         # 오피스 IP (보안을 위해 변경 필요)
}

module "compute" {
  key_name    = "multi"               # EC2 Key Pair 이름
  host_header = "laurar.store"        # 도메인 이름
  acm_arn     = "arn:aws:acm:..."    # ACM 인증서 ARN
  db_username = "master"              # RDS 마스터 사용자명
  db_password = "master-password"     # RDS 비밀번호 (보안을 위해 변경 필요)
}
```

### 3. Terraform 초기화
```bash
terraform init
```

### 4. 실행 계획 확인
```bash
terraform plan
```

### 5. 인프라 배포
```bash
terraform apply
```

입력 프롬프트에서 `yes`를 입력하여 배포를 시작합니다.

### 6. ECR에 이미지 푸시

#### ECR 로그인
```bash
aws ecr get-login-password --region ap-northeast-2 | \
docker login --username AWS --password-stdin \
662947429974.dkr.ecr.ap-northeast-2.amazonaws.com
```

#### Frontend 이미지 빌드 및 푸시
```bash
docker build -t ecr-jobposting-container-frontend:web ./frontend
docker tag ecr-jobposting-container-frontend:web \
  662947429974.dkr.ecr.ap-northeast-2.amazonaws.com/ecr-jobposting-container-frontend:web
docker push 662947429974.dkr.ecr.ap-northeast-2.amazonaws.com/ecr-jobposting-container-frontend:web
```

#### Backend 이미지 빌드 및 푸시
```bash
# Applicant 서비스
docker build -t ecr-jobposting-container-backend:applicant ./backend/applicant
docker tag ecr-jobposting-container-backend:applicant \
  662947429974.dkr.ecr.ap-northeast-2.amazonaws.com/ecr-jobposting-container-backend:applicant
docker push 662947429974.dkr.ecr.ap-northeast-2.amazonaws.com/ecr-jobposting-container-backend:applicant

# JobPosting 서비스
docker build -t ecr-jobposting-container-backend:jobposting ./backend/jobposting
docker tag ecr-jobposting-container-backend:jobposting \
  662947429974.dkr.ecr.ap-northeast-2.amazonaws.com/ecr-jobposting-container-backend:jobposting
docker push 662947429974.dkr.ecr.ap-northeast-2.amazonaws.com/ecr-jobposting-container-backend:jobposting
```

## 주요 엔드포인트

배포 완료 후 다음 URL로 접근 가능합니다:

| 서비스 | URL | 설명 |
|--------|-----|------|
| Frontend | https://laurar.store | 메인 웹 애플리케이션 |
| Applicant API | https://applicant.laurar.store | 지원자 관리 API |
| JobPosting API | https://jobposting.laurar.store | 채용공고 관리 API |
| Zabbix | https://zabbix.laurar.store | 시스템 모니터링 |
| Grafana | https://grafana.laurar.store | 메트릭 대시보드 |

## 보안 고려사항

### 현재 설정 (개발 환경)
- `sg_office_ip = "0.0.0.0/0"` - 모든 IP 허용
- RDS 비밀번호 하드코딩

### 프로덕션 권장 사항
1. **Security Group IP 제한**
   ```hcl
   sg_office_ip = "YOUR_OFFICE_IP/32"
   ```

2. **RDS 비밀번호 관리**
   - AWS Secrets Manager 사용
   - 또는 환경변수로 주입
   ```hcl
   db_password = var.db_password
   ```

3. **Bastion Host 접근 제한**
   - SSH Key 관리 강화
   - Multi-Factor Authentication

4. **ACM 인증서 갱신**
   - 자동 갱신 설정 확인

5. **CloudWatch 로그 보존 기간 설정**
   - 비용 최적화를 위한 로그 라이프사이클 정책

## 모니터링

### CloudWatch Logs
모든 ECS 서비스는 CloudWatch Logs에 로그를 전송합니다:
- `/ecs/ecr-jobposting-front-task-1`
- `/ecs/ecr-jobposting-back-job-task-1`
- `/ecs/ecr-jobposting-back-app-task-1`

### Container Insights
ECS Cluster에서 Container Insights가 활성화되어 있습니다:
- CPU/메모리 사용률
- 네트워크 트래픽
- 태스크 상태

### Zabbix & Grafana
Monitoring EC2 인스턴스에서 실행:
- 시스템 메트릭 수집
- 커스텀 대시보드
- 알림 설정

## Auto Scaling

### ECS 서비스 Auto Scaling
- **CPU 기반**:
  - Target: 50%
  - Scale-in/out Cooldown: 300초
- **메모리 기반**:
  - Target: 75%
  - Scale-in/out Cooldown: 300초
- **범위**: 최소 1개 ~ 최대 5개 태스크

## 비용 최적화

1. **인스턴스 타입**
   - Bastion: t3.medium (필요시 t3.micro로 축소)
   - Monitoring: t3.large (워크로드에 따라 조정)
   - RDS: db.t3.medium

2. **Fargate**
   - CPU: 512 (0.5 vCPU)
   - Memory: 1024MB (1GB)

3. **NAT Gateway**
   - 단일 NAT Gateway 사용 (비용 절감)
   - 고가용성이 필요한 경우 AZ별 NAT Gateway 추가

4. **S3 & DynamoDB**
   - Terraform State 관리 비용은 미미함

## 문제 해결

### ECS 태스크가 시작되지 않는 경우
1. CloudWatch Logs 확인
2. Security Group 규칙 확인
3. ECR 이미지 존재 여부 확인
4. Task Definition의 리소스 설정 확인

### RDS 연결 실패
1. Security Group 규칙 확인 (module/vpc/sg.tf:10)
2. DB Subnet Group 확인
3. RDS 인스턴스 상태 확인

### ALB Health Check 실패
1. Target Group Health Check 설정 확인 (module/compute/lb.tf:137-142)
2. 컨테이너의 헬스체크 엔드포인트 확인
3. Security Group 규칙 확인

## 리소스 삭제

인프라를 완전히 삭제하려면:

```bash
terraform destroy
```

주의: 이 명령은 모든 리소스를 삭제하며 복구할 수 없습니다.

### 삭제 전 확인사항
- RDS 스냅샷 백업
- S3 버킷 데이터 백업
- 중요 로그 다운로드

## 기술 스택

- **IaC**: Terraform 5.45.0
- **Cloud Provider**: AWS
- **Container Runtime**: ECS Fargate
- **Database**: MySQL 8.0 (RDS)
- **Load Balancer**: Application Load Balancer
- **Monitoring**: Zabbix, Grafana, CloudWatch
- **State Management**: S3 + DynamoDB

## 기여

프로젝트 개선을 위한 기여는 언제나 환영합니다:

1. Fork the repository
2. Create your feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request

## 라이선스

이 프로젝트는 교육 목적으로 제작되었습니다.

## 연락처

프로젝트 관련 문의사항이 있으시면 이슈를 등록해주세요.
