# AWS 관리형 서비스 연동 실습 정리

- **날짜**: 2026-09-16
- **과정**: AIBE7 인프라 트레이닝
- **범위**: RDS · S3 · ElastiCache · ALB
- **환경**: EC2 (Ubuntu 26.04 arm64, t4g.small)

> 상태는 관리형 서비스(RDS/ElastiCache/S3)에 두고, EC2에는 Nginx와 애플리케이션만 남긴다 — 이 분리가 동일 구성 노드를 복제해 로드밸런서 뒤에서 수평 확장하는 기반이다.

## 1. 오늘 강의 목표

- 데이터 계층을 **RDS·ElastiCache·S3**로 분리해 EC2를 **무상태(stateless) 컴퓨팅 노드**로 전환
- 보안 그룹 체이닝과 비공개 엔드포인트로 데이터 계층의 인터넷 직접 접근 차단
- Presigned URL로 비공개 S3 객체 접근을 임시 위임
- Compose에서 로컬 MySQL을 제거하고 Spring Boot의 JDBC 연결을 RDS 기준으로 재구성
- 커스텀 AMI로 노드를 복제하고 대상 그룹에 등록해 ALB 라운드로빈 분산을 검증
- 실습 종료 시 과금이 계속되는 관리형 자원을 전면 삭제

## 2. 아키텍처 한눈에 보기

| 구성 요소 | 역할 | 접근 모델 |
|---|---|---|
| RDS MySQL | 영속 트랜잭션 데이터, 패치·백업 관리 | 공인 IP 없음 · EC2 SG만 `3306` 허용 |
| ElastiCache Redis | 인메모리 캐시 · 세션 저장소 | 비공개 엔드포인트 · EC2 SG만 `6379` 허용 |
| S3 | 정적 파일 · 대용량 객체 저장 | 버킷 비공개 유지, Presigned URL로 임시 위임 |
| ALB | 단일 DNS 진입점, 헬스체크, 요청 분산 | 퍼블릭 서브넷 수신 → 정상 EC2의 `80`으로 전달 |
| EC2 (Nginx+App) | Nginx 리버스 프록시 + Spring Boot 실행 | ALB SG에서 들어오는 요청만 허용하는 무상태 노드 |

**요청 경로**

```
외부 클라이언트
      │ HTTP
      ▼
     ALB
      │
  ┌───┴────┐
  ▼        ▼
Node1    Node2
(Nginx → Spring Boot)
  │        │
  ├── JDBC 3306 ──► RDS MySQL
  ├── Redis 6379 ─► ElastiCache Redis
  └── HTTPS ───────► S3 (Presigned URL)
```

데이터 포트(3306/6379)는 CIDR가 아니라 **EC2 보안 그룹 ID**를 소스로 허용한다. 인스턴스가 늘거나 IP가 바뀌어도 규칙을 다시 쓸 필요가 없다.

## 3. 실습 전체 흐름 (강의자료 기준)

| 단계 | 내용 |
|---|---|
| STEP0-1 | 환경변수·SSO 로그인, 키페어·보안그룹·EC2 생성 ✅ *오늘 디버깅* |
| STEP2 | EC2에 Docker 설치, nginx·앱 이미지 pull |
| STEP3 | RDS/ElastiCache용 보안그룹·서브넷 그룹 생성 |
| STEP4 | RDS(MySQL)·ElastiCache(Redis) 인스턴스 생성 |
| STEP5 | S3 버킷 생성, 객체 업로드, Presigned URL 발급 |
| STEP6 | RDS·ElastiCache 프로비저닝 완료 대기, 엔드포인트 조회 |
| STEP7 | compose.yml에서 로컬 DB 제거 → RDS 연동으로 전환 |
| STEP8 | EC2에서 redis-cli로 ElastiCache 연결 검증 |
| STEP9-10 | IntelliJ에서 SSH 터널로 RDS·Redis 조회 |
| STEP11 | 커스텀 AMI로 노드 복제 (2번째 인스턴스) |
| STEP12 | 대상 그룹·ALB 생성, 라운드로빈 분산 검증 ✅ *오늘 디버깅* |

오늘 대화에서는 **STEP1(EC2 생성)**과 **STEP12(ALB 대상 그룹·라운드로빈)** 구간에서 실제로 에러가 나서 같이 디버깅했다. 나머지 단계는 강의자료 기준 흐름이니 진행하면서 체크리스트로 써도 좋다.

## 4. 오늘 겪은 오류와 해결

### 4.1 Git Bash 붙여넣기 시 줄바꿈이 사라져 명령어가 뒤섞임 (2회 발생)

```
bash: export: `instance-running': not a valid identifier
aws: [ERROR] Unknown options: ,  --image-id
bash: syntax error near unexpected token `do'
```

**원인**: 여러 줄짜리 스크립트를 붙여넣을 때, `\`로 이어진 줄만 유지되고 나머지 줄바꿈은 공백으로 뭉개졌다. 그 결과 전체 스크립트가 맨 앞의 `export` 또는 `for` 한 줄짜리 명령으로 합쳐져버렸다.

**해결**: 여러 줄 스크립트는 파일로 저장해 `bash script.sh`로 실행하거나, 각 명령을 세미콜론(`;`)으로 이어 한 줄로 붙여넣는다.

### 4.2 SSM 퍼블릭 파라미터 조회 실패

```
aws: [ERROR] An error occurred (ParameterNotFound) when calling
the GetParameter operation
```

**원인**: Canonical의 Ubuntu AMI 파라미터뿐 아니라 Amazon 소유 퍼블릭 파라미터까지 전부 실패했다. 경로 문제가 아니라 **교육용 SSO 역할(PowerUser)에 SSM 퍼블릭 파라미터 조회 권한 자체가 막혀 있는 것**이었다. (AWS는 권한 없음을 AccessDenied 대신 ParameterNotFound로 돌려주는 경우가 있다.)

**해결**: SSM 대신 EC2 API로 우회한다.

```bash
aws ec2 describe-images --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-*-26.04-arm64-server-*" \
  "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" --output text
```

### 4.3 EC2 내부 SSH 세션에서 aws 명령 실행 시도

```
Command 'aws' not found, but can be installed with:
sudo apt install awscli
```

**원인**: `ubuntu@ip-172-...` 프롬프트, 즉 SSH로 접속한 EC2 인스턴스 안에는 AWS CLI가 없다. 타겟 그룹 생성·타겟 등록 같은 명령은 AWS 인프라를 조작하는 것이라 EC2 안에서 실행할 이유가 없다.

**해결**: `exit`로 로컬(Windows) 터미널로 돌아가서 실행한다. AWS CLI와 자격증명(student10 프로필)은 이미 로컬에 설정되어 있었다.

### 4.4 Git Bash의 자동 경로 변환으로 `/` 인자가 깨짐

```
Health check path 'C:/Program Files/Git/' must begin with a '/'
character and can only contain printable ASCII characters...
```

**원인**: MINGW64(Git Bash)는 인자에 `/`로 시작하는 문자열이 보이면 자동으로 Windows 경로로 바꿔버린다. `--health-check-path /`의 `/`가 Git 설치 경로로 둔갑했다.

**해결**: 세션 시작 시 한 번 설정한다.

```bash
export MSYS_NO_PATHCONV=1
```

## 5. 앞으로 꼭 기억할 것

- **줄바꿈 있는 스크립트는 파일로.** 터미널에 직접 붙여넣지 말고 `.sh`로 저장 후 실행하거나, 세미콜론으로 한 줄에 이어 쓴다.
- **Git Bash 세션마다** `export MSYS_NO_PATHCONV=1`을 먼저 설정해두면 `/` 인자가 Windows 경로로 바뀌는 사고를 막을 수 있다.
- **aws 명령은 항상 로컬(Windows) 터미널에서.** EC2에 SSH로 들어간 뒤에는 프롬프트가 `ubuntu@ip-...`로 바뀌는지 꼭 확인한다.
- **export 직후엔 echo로 값 확인.** 특히 `$INSTANCE_ID`, `$MY_TG_ARN`처럼 뒤 단계에서 그대로 쓰는 변수는 비어있으면 연쇄로 에러가 난다.
- **SSM 퍼블릭 파라미터가 막히면** `aws ec2 describe-images --owners 099720109477`로 캐노니컬 공식 AMI를 직접 조회한다.

## 6. 참고 자료

- [Amazon RDS의 VPC 구성](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_VPC.WorkingWithRDSInstanceinaVPC.html)
- [ElastiCache 접근 패턴](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/accessing-elasticache.html)
- [S3 Presigned URL](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [ALB 대상 그룹 헬스체크](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html)

---
*출처: 강의자료 「07-3 AWS 관리형 서비스 연동 — RDS·S3·ElastiCache·ALB」(56p) + 오늘 실습 대화 로그 기준으로 정리*
