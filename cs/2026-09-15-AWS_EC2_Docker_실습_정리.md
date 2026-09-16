# AWS 클라우드 기초 & EC2/Docker 배포 실습 정리

> 강의자료 `06_클라우드 기초 개념과 AWS 환경 세팅`, `07-1_AWS EC2 배포 Docker 설치와 외부 Aiven DB 연동` 내용 정리 + 실습 중 직접 겪었던 오류·헷갈렸던 부분 정리

---

## 1. 클라우드 기초 개념

### 1-1. 서비스 모델 3형제 (IaaS / PaaS / SaaS)

아래 계층이 위 계층의 기반이 되며, 위로 갈수록 내가 관리할 영역은 줄어든다.

| 서비스 모델 | 관리 주체(내가 만지는 영역) | 대표 서비스 |
|---|---|---|
| IaaS | OS·미들웨어·네트워크·런타임까지 전부 직접 | AWS, GCP, Azure |
| PaaS | 소스코드·이미지·DB 스키마만 | Render, Railway, Aiven |
| SaaS | 데이터·사용자 설정만 | Notion, Slack, GitHub |

- 엔터프라이즈가 굳이 IaaS를 쓰는 이유: 세밀한 사설망 격리 + 아키텍처 제어권. 관리 범위는 넓어지지만 보안 경계·비용을 직접 설계할 수 있음.
- 이번 실습(EC2 + Docker)은 **IaaS** 관점 — 런타임(도커 엔진)까지 내가 직접 설치해야 함. PaaS(Render 등)와 달리 "기본 제공되는 런타임"이 없다는 게 핵심 차이.

### 1-2. 리전 / 가용영역(AZ)

- **리전**: 수천 km 격리된 지리적 거점 (실습은 전부 서울 `ap-northeast-2`로 고정)
- **가용영역(AZ)**: 같은 리전 안에서 수십 km 이격된 센터군. AZ 간 전용 광케이블로 1ms 미만 지연 동기화 → 2개 이상 AZ에 분산 배치하면 한 센터 장애가 나도 자동 절체됨
- 기본 VPC에는 AZ마다 서브넷이 하나씩 자동 생성되어 있음

### 1-3. 인증·자격증명 흐름 (SSO)

```
학습자 터미널(AWS CLI v2) → aws sso login (MFA 인증) → IAM Identity Center
                          → 단기 STS 토큰 발급 → 서울 리전 API 호출(HTTPS 443)
```

- 브라우저 SSO + MFA를 거쳐 **유효기간이 짧은 임시 토큰**을 발급받는 구조 (장기 액세스 키를 쓰지 않음)
- 환경변수로 세션을 고정: `AWS_PROFILE`(프로필 자동 적용), `AWS_REGION`(리전 고정), `AWS_PAGER=""`(페이저 비활성화)
- 계정 가드레일: `t4g.nano`/`t4g.micro`/`t4g.small` 외 인스턴스 유형은 IAM 정책/SCP 레벨에서 API 단계부터 즉시 `Deny`됨 (콘솔에서 선택은 가능해 보여도 생성 요청 자체가 거부됨)

### 1-4. 보안 그룹 = 상태 저장(Stateful) 방화벽

| 구분 | 인바운드 | 아웃바운드 |
|---|---|---|
| 기본 정책 | 전체 차단(화이트리스트) | 전체 허용 |
| 특징 | 지정 포트/CIDR만 개방 | 인바운드로 들어온 요청의 **응답 트래픽은 별도 규칙 없이 자동 허용** |

→ 80/8080 포트를 인바운드로 열었다면, 응답이 나가는 임시 포트를 아웃바운드에 따로 열 필요가 없음 (Stateful이기 때문).

### 1-5. IP 주소 체계

| 유형 | 정의 | 특징 |
|---|---|---|
| 사설 IP (`172.31.x.x`) | VPC 내부 전용 | 인터넷에서 직접 접근 불가 |
| 공인 IP | 인터넷 게이트웨이 경유 | **인스턴스를 중지 후 재시작하면 매번 바뀜** |
| 탄력적 IP(EIP) | 계정에 영구 고정 할당 | 재시작해도 불변, 도메인 연결 시 필수 |

탄력적 IP를 연결하지 않았다면, 재시작할 때마다 공인 IP를 다시 조회해서 변수에 다시 담아야 함.

---

## 2. EC2 + Docker + 외부 관리형 DB(Aiven) 배포 흐름

### 2-1. 전체 아키텍처

```
호스트 터미널(curl) → 공인 IP:8080 → [EC2] 보안그룹(8080 허용) → Docker 데몬 → 컨테이너(8080)
                                                                        ↓ TLS·TCP 3306
                                                              Aiven MySQL (외부 관리형 DBaaS, 인터넷 구간)
```
데이터베이스가 VPC 내부가 아니라 인터넷 너머에 있으므로, **아웃바운드 통신 + TLS 암호화**가 전제되어야 함.

### 2-2. 실습 순서 요약

1. **인스턴스 준비**: (강의자료 기준) 기존 t4g.nano 인스턴스를 `stop` → `t4g.micro`로 `modify-instance-attribute` → `start` → `wait instance-status-ok`
2. **SSH 접속 및 자원 점검**: `uname -m`(aarch64 확인), `free -h`(메모리), `df -h /`, `ip addr show`(사설 IP), `curl ifconfig.me`(공인 IP 대조), `ss -tulnp`(리스닝 포트 확인)
3. **Docker 엔진 설치**: `curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh` → `systemctl status docker`로 `active (running)` 확인 → `docker info`에서 `Architecture: aarch64` 확인
4. **GitHub Actions 멀티플랫폼 빌드**: `docker/setup-buildx-action` + `platforms: linux/amd64,linux/arm64`로 빌드해 GHCR에 푸시 → 하나의 `:latest` 태그가 아키텍처별 다이제스트를 가리키는 매니페스트 인덱스가 됨
5. **이미지 pull + 컨테이너 구동**: `docker pull ghcr.io/.../simple-back:latest` (호스트 커널 아키텍처에 맞는 레이어 자동 선택) → `.env` 파일에 DB 접속정보 분리 저장(자격증명이 `~/.bash_history`나 `ps` 출력에 평문 노출되지 않도록) → `docker run -d --restart unless-stopped -p 8080:8080 --env-file aiven.env ...`
6. **검증**: 원격 서버 안이 아니라 **로컬 호스트에서 공인 IP로** `curl -i http://$PUBLIC_IP:8080/` 호출 → 인터넷 구간까지 포함한 전체 경로 검증
7. **정리**: `docker stop` → `aws ec2 stop-instances`(terminate 아님! 재사용 위해 stop) → 컴퓨팅 요금 0원, EBS 스토리지 요금만 소액 유지

### 2-3. 인스턴스 유형 선택 기준 (Appendix)

| 유형 | vCPU·메모리 | 비고 |
|---|---|---|
| t4g.nano | 2vCPU·0.5GiB | JVM 힙 + Docker 데몬 + OS가 512MiB를 다투다가 **OOM으로 강제 종료될 위험 큼** |
| t4g.micro | 2vCPU·1GiB | 단일 컨테이너 구동에 필요한 최소 버퍼 확보 |

### 2-4. 멀티 아키텍처 이미지 원리

동일한 `:latest` 태그 하나가 "매니페스트 리스트(OCI Image Index)" 역할을 하며, 클라이언트 커널 아키텍처에 맞는 다이제스트만 선택적으로 내려받는 구조. x86 러너에서 **amd64 전용**으로만 빌드한 이미지를 ARM 인스턴스에서 실행하면 `exec format error`로 기동이 거부됨 → 해결책은 buildx로 `linux/amd64,linux/arm64`를 **동시에** 빌드해 하나의 매니페스트로 묶어 푸시하는 것.

---

## 3. 실습하면서 실제로 틀렸던 부분 / 헤맸던 부분

아래는 강의자료의 "정석 흐름"과 달리, 실제 실습 중 발생했던 문제와 원인을 정리한 것이다.

### 3-1. AWS CLI v2 설치 단계

| 증상 | 원인 | 교훈 |
|---|---|---|
| `curl -o AWSCLIV2.msi ... msiexec.exe /i ...`가 한 줄로 붙어 `curl: URL rejected` 연쇄 오류 | 두 명령을 구분자(`&&` 또는 줄바꿈) 없이 이어 씀 → curl이 뒤 토큰들을 전부 URL로 해석 | 여러 명령을 이어 쓸 땐 반드시 `&&`나 줄바꿈으로 분리 |
| `msiexec /qn` 설치가 계속 실패 (최종 에러 1603) | **Git Bash를 관리자 권한 없이 실행** → MSI가 `ALLUSERS=1`(전체 사용자용) 설치라 UAC 상승이 필요한데, `/qn`(완전 무음)은 승인 창을 띄울 수 없어 조용히 실패 (진짜 원인은 로그 안쪽의 **Error 1925 — 관리자 권한 부족**) | Windows에서 시스템 전체용 프로그램을 설치할 땐 터미널 자체를 "관리자 권한으로 실행"해야 함. 인텔리제이 내장 터미널도 IDE 프로세스 권한을 그대로 물려받으므로, IDE 자체를 관리자 권한으로 켜야 터미널도 관리자 권한이 됨 |
| 설치 후 새 창에서도 `aws: command not found` | 설치 시 시스템 PATH에 `AWSCLIV2` 경로가 등록되지 않음 (설치 자체는 파일 복사까지는 성공) | 환경변수 편집(GUI)에서 PATH에 수동 추가 후 재부팅 필요 |

### 3-2. `aws configure` / 자격증명 관리

| 증상 | 원인 | 교훈 |
|---|---|---|
| `aws: [ERROR]: Unknown output type: format: json` | output format 입력란에 안내 문구 `format: json`을 그대로 입력함 | 프롬프트에는 값만(`json`) 입력해야 함. 잘못 저장된 `~/.aws/config`의 `output = format: json` 줄을 직접 고쳐도 됨 |
| 여러 번 반복된 `Unable to locate credentials` | ① `--profile` 옵션을 안 붙임 ② **Git Bash 새 창을 열 때마다 `export AWS_PROFILE=...`이 초기화됨**을 계속 잊음 | 세션 변수는 창 단위로 휘발된다는 것을 기억하고, 매 창마다 프로필부터 재설정하는 습관 필요. 자주 쓰면 `~/.bashrc`에 등록 |

### 3-3. 변수 관리(가장 자주 발생한 실수 유형)

Git Bash 창을 새로 열 때마다 `export`로 만든 변수(`VPC_ID`, `MY_SG_ID`, `MY_KEY_NAME`, `AMI_ID`, `INSTANCE_ID`, `PUBLIC_IP`, `AWS_PROFILE` 등)가 전부 사라진다는 사실을 여러 차례 놓쳐서, 빈 변수로 명령을 실행해 `MissingParameter`류 오류가 반복 발생했음. (`Dexport`처럼 복붙 과정의 단순 오타도 한 번 있었음)

→ **교훈**: 새 터미널을 열면 가장 먼저 "지금 세션에 필요한 변수가 다 있는지" `echo`로 확인하는 습관을 들일 것.

### 3-4. Ubuntu AMI 조회 (SSM Parameter Store)

- 강의자료(22p)는 `aws ssm get-parameter --name /aws/service/canonical/ubuntu/server/26.04/.../ami-id`로 최신 AMI를 조회하도록 안내하지만, 실습 계정(`InfraTrainingPowerUser`)에서는 이 방식이 **AWS 자체 공식 파라미터(Amazon Linux 등)조차도** `ParameterNotFound`로 실패했음 → 이는 이 학습 계정의 IAM 정책이 AWS 공용(public) SSM 파라미터 네임스페이스(`/aws/service/...`) 접근을 허용하지 않기 때문으로 확인됨.
- 추가로 `get-parameters-by-path`는 애초에 `/aws`로 시작하는 경로는 조회할 수 없다는 것도 새로 확인함 (버그가 아니라 AWS API 자체 제약).
- **우회 해결**: SSM 대신 EC2 API로 직접 조회 — `aws ec2 describe-images --owners 099720109477(Canonical 공식 계정) --filters "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-*-24.04-arm64-server-*"`

→ **교훈**: 강의자료의 "정석 명령"이 특정 실습 계정의 IAM 제약 때문에 안 먹힐 수 있다는 것, 그리고 그럴 때 대체 조회 경로(EC2 API)를 알아두면 좋다는 것.

### 3-5. 인스턴스 생성 시 변수 오염 → SSH 이중 장애

`run-instances` 실행 시점에 `$MY_SG_ID`, `$MY_KEY_NAME` 변수가 **다른 학생(student09)의 값**으로 잘못 설정되어 있어서, 실제로는 student09의 보안 그룹·키 페어로 인스턴스가 생성됨. 그 결과:

1. SSH 22번 포트 접속이 계속 타임아웃 → 원인: 붙어있는 보안 그룹이 내 IP를 허용하지 않는 다른 사람 것이었음 (`modify-instance-attribute`로 올바른 SG로 교체해 해결)
2. SG를 고쳤는데도 `Permission denied (publickey)` → 원인: 인스턴스가 `student09-key`로 생성되어 내 `.pem` 키와 애초에 안 맞음 (키는 생성 후 변경 불가 → 인스턴스를 통째로 재생성해서 해결)

→ **교훈**: 공유 실습 환경에서는 변수 이름이 겹치거나 예전 세션의 값이 섞여 들어갈 수 있으므로, 인스턴스 생성 직전에 `echo`로 변수 값을 반드시 눈으로 확인할 것.

### 3-6. Docker 이미지 아키텍처 불일치

`docker pull ghcr.io/.../simple-back-ghcr:latest` 시 `no matching manifest for linux/arm64/v8` 오류 발생 — 강의자료 14페이지에서 정확히 경고하는 그 상황("x86 러너에서 만든 AMD64 전용 이미지를 ARM 인스턴스에서 실행하면 `exec format error`")과 같은 원인으로, 해당 이미지가 amd64 전용으로만 빌드되어 있었음.

→ 대응: ① 이미지를 melti-arch(`linux/amd64,linux/arm64`)로 다시 빌드하거나, ② 인스턴스를 x86_64 계열(t3.micro 등)로 바꿔서 사용.

### 3-7. 기타

- 셸 세션 도중 원인 불명으로 `echo`, `aws --version`까지 완전히 무출력이 되는 현상 발생 (아마 복붙 과정에서 출력 리다이렉션이 꼬였던 것으로 추정) → 터미널을 완전히 새로 열어서 해결.
- 실습 중 인스턴스가 안내 없이 자동으로 **강제 종료(terminate)** 되는 일이 있었음. 강의자료는 "실습을 마치면 stop(중지)만 하고 terminate는 하지 말 것"을 원칙으로 안내하는데, 이 계정에는 별도의 자동 정리 정책이 걸려있는 것으로 보임 — 작업 중간에 저장 시점을 자주 만들어두는 습관이 필요.

---

## 4. 한눈에 보는 핵심 명령어 체크리스트

```bash
# 세션 변수 (새 창마다 다시 설정!)
export AWS_PROFILE=student10
export AWS_REGION=ap-northeast-2
export AWS_PAGER=""

# 자격 증명/리전 확인
aws sts get-caller-identity

# 기본 VPC / 보안 그룹
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text)
export MY_SG_ID=$(aws ec2 create-security-group --group-name "내SG이름" --vpc-id "$VPC_ID" --query GroupId --output text)

# 인바운드 규칙 (SSH는 내 IP만, 웹 포트는 전역)
export MY_IP=$(curl -fsS https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress --group-id "$MY_SG_ID" --protocol tcp --port 22 --cidr "$MY_IP/32"

# 키 페어
aws ec2 create-key-pair --key-name "내키" --query "KeyMaterial" --output text > ./내키.pem
chmod 400 ./내키.pem

# 인스턴스 생성 → 대기 → IP 조회
export INSTANCE_ID=$(aws ec2 run-instances --image-id "$AMI_ID" --instance-type t4g.micro \
  --key-name "$MY_KEY_NAME" --security-group-ids "$MY_SG_ID" --query "Instances[0].InstanceId" --output text)
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-status-ok --instance-ids "$INSTANCE_ID"
export PUBLIC_IP=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" --query "Reservations[0].Instances[0].PublicIpAddress" --output text)

# SSH 접속
ssh -i "$MY_KEY_NAME".pem -o StrictHostKeyChecking=accept-new ubuntu@"$PUBLIC_IP"

# 마무리 (재사용 목적이면 stop, 완전 종료면 terminate)
aws ec2 stop-instances --instance-ids "$INSTANCE_ID"
```
