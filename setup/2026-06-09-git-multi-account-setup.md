# Git 다중 계정 설정 가이드

> 하나의 컴퓨터에서 GitHub 계정 두 개를 폴더 기준으로 자동 분리하는 설정

---

## 📌 목표

| 폴더 | 적용 계정 |
|------|-----------|
| `D:/workspace/` | `****95@gmail.com` (기본 계정) |
| `D:/workspace/antigravity/` | `****36510@gmail.com` (antigravity 계정) |

---

## 1단계. SSH 키 생성

SSH 키는 GitHub에 **"나 진짜 이 사람 맞아"** 를 증명하는 열쇠예요.  
계정이 2개이므로 열쇠도 2개 만들어야 해요.

```bash
# 기본 계정용
ssh-keygen -t ed25519 -C "****95@gmail.com" -f ~/.ssh/id_ed25519_vscode

# antigravity 계정용
ssh-keygen -t ed25519 -C "****36510@gmail.com" -f ~/.ssh/id_ed25519_antigravity
```

> 각 명령어 실행 후 엔터 3번 (passphrase 없이)

---

## 2단계. SSH config 파일 설정

이 파일은 **"github.com 접속할 때 어떤 열쇠를 쓸지"** 정해주는 파일이에요.

```bash
code ~/.ssh/config
```

아래 내용 붙여넣고 저장 (`Ctrl+S`):

```
Host github-vscode
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_vscode

Host github-antigravity
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_antigravity
```

---

## 3단계. GitHub에 공개키 등록

각 계정 GitHub → **Settings → SSH and GPG keys → New SSH key** 에 등록

```bash
# 기본 계정용 공개키 확인
cat ~/.ssh/id_ed25519_vscode.pub

# antigravity 계정용 공개키 확인
cat ~/.ssh/id_ed25519_antigravity.pub
```

- 기본 계정 GitHub에 `id_ed25519_vscode.pub` 내용 등록
- antigravity 계정 GitHub에 `id_ed25519_antigravity.pub` 내용 등록

---

## 4단계. Git 계정 설정 파일 분리

폴더에 따라 자동으로 계정이 바뀌게 해주는 설정이에요.

```bash
# 기본 계정용 설정 파일
code ~/.gitconfig-vscode
```

```ini
[user]
  name = ********   (기본 계정 GitHub username)
  email = ****95@gmail.com
```

```bash
# antigravity 계정용 설정 파일
code ~/.gitconfig-antigravity
```

```ini
[user]
  name = ********   (antigravity 계정 GitHub username)
  email = ****36510@gmail.com
```

---

## 5단계. `.gitconfig` 에 폴더별 계정 연결

```bash
code ~/.gitconfig
```

파일 맨 아래에 추가:

```ini
[includeIf "gitdir:D:/workspace/"]
  path = ~/.gitconfig-vscode

[includeIf "gitdir:D:/workspace/antigravity/"]
  path = ~/.gitconfig-antigravity
```

> `gitdir:` 경로 끝에 `/` 필수!

---

## 6단계. SSH 에이전트에 키 등록

터미널을 새로 열 때마다 아래 명령어로 키를 등록해줘야 해요.

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_vscode
ssh-add ~/.ssh/id_ed25519_antigravity
```

---

## 7단계. 연결 테스트

```bash
ssh -T git@github-vscode      # Hi [기본계정 username]! 나오면 성공
ssh -T git@github-antigravity # Hi [antigravity계정 username]! 나오면 성공
```

---

## ⚠️ 주의사항

### clone / remote 설정 시 Host 이름 변경 필수

GitHub에서 복사한 SSH 주소의 `github.com` 부분을 Host 이름으로 바꿔야 해요.

```bash
# GitHub에서 복사한 주소 (그대로 사용 X)
git@github.com:username/repo.git

# 기본 계정 레포
git clone git@github-vscode:username/repo.git

# antigravity 계정 레포
git clone git@github-antigravity:username/repo.git
```

### 기존 레포 remote URL 변경

```bash
# HTTPS → SSH로 변경
git remote set-url origin git@github-vscode:username/repo.git
```

### includeIf는 git 저장소 안에서만 작동

새 프로젝트 시작 시 반드시 `git init` 또는 `git clone` 후에 계정 확인:

```bash
git config user.email  # 올바른 계정인지 확인
```

---

## 📁 폴더 구조 정리

```
D:/workspace/
  TIL/                  ← 기본 계정 (****95) 자동 적용
  새프로젝트/            ← 기본 계정 (****95) 자동 적용
  antigravity/
    프로젝트A/           ← antigravity 계정 (****36510) 자동 적용
    프로젝트B/           ← antigravity 계정 (****36510) 자동 적용
```

---

## 💡 이 설정은 현재 컴퓨터에만 적용됩니다

SSH 키, `~/.gitconfig`, `~/.ssh/config` 모두 로컬 파일이므로  
다른 컴퓨터에는 영향을 주지 않아요.
