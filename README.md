# HiveWiki CI

🏆 2026학년도 1학기 한림대학교 SW캡스톤디자인 경진대회 금상 수상 프로젝트

HiveWiki CI는 HiveWiki 프로젝트에서 공통으로 사용하는 GitHub Actions reusable workflow를 관리하는 레포지토리입니다.
각 서비스 레포지토리는 이 레포지토리의 workflow를 호출해 컨테이너 이미지 빌드, 보안 스캔, 이미지 푸시, Slack 알림 같은 반복적인 CI 작업을 같은 방식으로 실행할 수 있습니다.

## 제공 workflow

| Workflow | 파일 | 역할 |
| --- | --- | --- |
| Reusable Build, Scan, and Push Image | `.github/workflows/build-scan-push-image.yml` | Docker 이미지를 빌드하고 Trivy로 검사한 뒤 GHCR과 Amazon ECR에 푸시합니다. |
| Reusable Trivy Scan | `.github/workflows/trivy.yml` | 호출한 레포지토리의 파일 시스템 또는 설정 파일을 Trivy로 검사합니다. |

## 이미지 빌드 workflow

`build-scan-push-image.yml`은 서비스 레포지토리에서 Docker 이미지를 배포할 때 사용합니다.

### 동작 흐름

1. 호출한 레포지토리를 checkout합니다.
2. 브랜치 이름으로 배포 환경을 결정합니다.
3. Docker Buildx와 GitHub Actions cache를 사용해 `linux/arm64` 이미지를 빌드합니다.
4. 빌드된 로컬 이미지를 Trivy로 검사합니다.
5. GHCR과 Amazon ECR에 로그인합니다.
6. commit SHA 기반 태그로 이미지를 푸시합니다.
7. 성공 또는 실패 결과를 Slack으로 알립니다.

### 브랜치 규칙

이 workflow는 아래 브랜치에서만 실행되도록 설계되어 있습니다.

| 브랜치 | 환경 | 이미지 태그 예시 |
| --- | --- | --- |
| `develop` | `dev` | `dev-1a2b3c4` |
| `main` | `prod` | `prod-1a2b3c4` |

다른 브랜치에서 호출하면 workflow가 실패합니다.

### 입력값

| 이름 | 필수 여부 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `image_name` | 선택 | 호출한 레포지토리 이름 | 이미지 이름입니다. 비워두면 호출한 레포지토리 이름을 소문자로 변환해 사용합니다. |
| `dockerfile` | 선택 | `./Dockerfile` | 빌드에 사용할 Dockerfile 경로입니다. |
| `context` | 선택 | `.` | Docker build context 경로입니다. |
| `aws_region` | 선택 | `ap-northeast-2` | Amazon ECR이 위치한 AWS 리전입니다. |

### 필요한 secrets

| 이름 | 설명 |
| --- | --- |
| `AWS_ACCESS_KEY_ID` | Amazon ECR 로그인에 사용할 AWS access key입니다. |
| `AWS_SECRET_ACCESS_KEY` | Amazon ECR 로그인에 사용할 AWS secret key입니다. |
| `AWS_ACCOUNT_ID` | ECR registry 주소를 만들 때 사용하는 AWS account ID입니다. |
| `GHCR_USERNAME` | GHCR 로그인 사용자 이름입니다. |
| `GHCR_TOKEN` | GHCR push 권한이 있는 token입니다. |
| `SLACK_WEBHOOK_URL` | 빌드 결과를 보낼 Slack incoming webhook URL입니다. |

### 호출 예시

```yaml
name: Build and Push Image

on:
  push:
    branches:
      - develop
      - main

jobs:
  image:
    uses: hivewiki/hivewiki-ci/.github/workflows/build-scan-push-image.yml@main
    with:
      image_name: hivewiki-api
      dockerfile: ./Dockerfile
      context: .
      aws_region: ap-northeast-2
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
      GHCR_USERNAME: ${{ secrets.GHCR_USERNAME }}
      GHCR_TOKEN: ${{ secrets.GHCR_TOKEN }}
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Trivy scan workflow

`trivy.yml`은 서비스 레포지토리에서 취약점 또는 secret 검사를 공통 방식으로 실행할 때 사용합니다.

### 입력값

| 이름 | 필수 여부 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `scan_type` | 필수 | 없음 | Trivy scan type입니다. 예: `fs`, `config` |
| `scanners` | 필수 | 없음 | 실행할 scanner 목록입니다. 예: `vuln,secret,misconfig` |
| `skip_dirs` | 선택 | 빈 문자열 | 검사에서 제외할 디렉터리 목록입니다. |
| `skip_files` | 선택 | 빈 문자열 | 검사에서 제외할 파일 목록입니다. |

Trivy는 `HIGH`, `CRITICAL` 심각도의 이슈를 발견하면 workflow를 실패시킵니다.
아직 수정 버전이 없는 취약점은 `ignore-unfixed` 옵션으로 제외합니다.

### 호출 예시

```yaml
name: Security Scan

on:
  pull_request:
  push:
    branches:
      - develop
      - main

jobs:
  trivy:
    uses: hivewiki/hivewiki-ci/.github/workflows/trivy.yml@main
    with:
      scan_type: fs
      scanners: vuln,secret,misconfig
      skip_dirs: node_modules,dist
      skip_files: package-lock.json
```

## 개발 환경

이 레포지토리는 workflow와 문서 품질을 유지하기 위해 아래 도구를 사용합니다.

- GitHub Actions
- pre-commit
- gitleaks
- commitizen

## 시작하기

### 1. 레포지토리 clone

```bash
git clone <repository-url>
cd hivewiki-ci
```

### 2. pre-commit 설치

`pre-commit`은 Python 기반 도구입니다.

```bash
pip install pre-commit
```

### 3. Git hook 설치

```bash
pre-commit install --hook-type pre-commit --hook-type commit-msg
```

이 레포지토리는 pre-commit hook을 사용합니다.
hook이 파일을 자동으로 수정하면 커밋이 중단될 수 있습니다.
이 경우 수정된 파일을 확인한 뒤 다시 `git add`를 실행하고 커밋하면 됩니다.

커밋 메시지는 영어로 작성해야 하며 Conventional Commits 규칙을 따릅니다.
자세한 규칙은 [commitizen 문서](https://commitizen-tools.github.io/commitizen/tutorials/writing_commits/)를 참고하세요.

## 유지보수 기준

- workflow action은 가능하면 commit SHA로 고정합니다.
- 공통 workflow의 입력값이나 secrets를 변경하면 이 README도 함께 업데이트합니다.
- 호출하는 레포지토리에 영향을 줄 수 있는 변경은 적용 전에 예시 workflow와 브랜치 규칙을 다시 확인합니다.
