---
layout: post
title:  "Cloudflare Pages + GitHub Actions Self-hosted Runner로 프론트/백엔드 자동 배포 파이프라인 구축하기"
slug: "cloudflare-pages-github-actions-self-hosted-runner-deploy-pipeline"
date:   2026-03-26 23:00:00 +0900
categories: blog
---
<style>a, li, code { word-break: break-all; }</style>

<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-121955159-1"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'UA-121955159-1');
</script>
<script async src="//pagead2.googlesyndication.com/pagead/js/adsbygoogle.js"></script>
<!-- fureweb-github -->
<ins class="adsbygoogle"
     style="display:block"
     data-ad-client="ca-pub-6234418861743010"
     data-ad-slot="8427857156"
     data-ad-format="auto"></ins>
<script>
(adsbygoogle = window.adsbygoogle || []).push({});
</script>

<div class="fb-like" data-href="https://fureweb-com.github.io{{page.url}}" data-layout="button_count" data-action="like" data-size="small" data-show-faces="true" data-share="true"></div>

---

모노레포에서 프론트엔드와 백엔드를 각각 다른 인프라에 배포해야 하는 경우가 많습니다. 프론트엔드는 Cloudflare Pages 같은 정적 호스팅에, 백엔드는 직접 관리하는 VM에 배포하는 구성이 대표적입니다.

이 글에서는 하나의 GitHub 저장소에서 `main` 브랜치에 push할 때 **프론트엔드는 Cloudflare Pages로**, **백엔드는 Self-hosted Runner를 통해 VM으로** 자동 배포하는 파이프라인을 구축하는 방법을 다룹니다.

## 전체 구조

```
repo/
├── frontend/          # Cloudflare Pages로 배포
├── backend/           # VM에 배포 (Self-hosted Runner)
└── .github/workflows/
    ├── deploy-fe.yml  # 프론트엔드 배포
    └── deploy-be.yml  # 백엔드 배포
```

두 워크플로우 모두 `main` push에 반응하되, `paths` 필터로 각자 담당하는 디렉토리가 변경된 경우에만 실행됩니다.

## 1. 프론트엔드: Cloudflare Pages 배포

Cloudflare Pages는 GitHub Actions에서 `wrangler`를 통해 배포할 수 있습니다. GitHub의 ubuntu 러너에서 빌드 후 결과물을 Cloudflare에 업로드하는 방식입니다.

### 워크플로우 (deploy-fe.yml)

```yaml
name: Deploy Frontend

on:
  push:
    branches: [main]
    paths:
      - 'frontend/**'
      - '.github/workflows/deploy-fe.yml'

jobs:
  deploy:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Deploy to Cloudflare Pages
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: pages deploy dist/ --project-name=my-project --commit-dirty=true
          workingDirectory: frontend
```

### 필요한 Secrets

| Secret | 설명 |
|--------|------|
| `CLOUDFLARE_API_TOKEN` | Cloudflare API 토큰 (Edit Cloudflare Pages 권한) |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare 계정 ID |

Cloudflare 대시보드에서 API 토큰을 생성할 때 **Cloudflare Pages > Edit** 권한을 부여해야 합니다.

## 2. 백엔드: Self-hosted Runner 배포

백엔드 서버가 외부에서 SSH 접근이 어려운 환경(사설 네트워크, VPN 등)에 있다면 **Self-hosted Runner**가 좋은 선택입니다. 서버 VM에 runner를 설치하면 GitHub Actions가 해당 서버에서 직접 명령을 실행할 수 있습니다.

### 2-1. Runner 설치 (Ubuntu)

서버 VM에 SSH 접속 후 실행합니다.

```bash
# runner 디렉토리 생성 및 다운로드
mkdir -p ~/actions-runner && cd ~/actions-runner
curl -o actions-runner-linux-x64-2.322.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.322.0/actions-runner-linux-x64-2.322.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.322.0.tar.gz
```

GitHub에서 등록 토큰을 발급받습니다:

> Repository > Settings > Actions > Runners > New self-hosted runner

```bash
# runner 등록
./config.sh --url https://github.com/<owner>/<repo> --token <YOUR_TOKEN>
```

등록 시 물어보는 항목:
- **Runner name**: 원하는 이름 (예: `my-server`)
- **Labels**: 워크플로우에서 식별할 라벨 (예: `backend`)
- 나머지는 기본값(엔터)

```bash
# 시스템 서비스로 등록 (재부팅 시 자동 시작)
sudo ./svc.sh install
sudo ./svc.sh start
```

### 2-2. 워크플로우 (deploy-be.yml)

```yaml
name: Deploy Backend

on:
  push:
    branches: [main]
    paths:
      - 'backend/**'
      - '.github/workflows/deploy-be.yml'

env:
  DEPLOY_DIR: /home/ubuntu/my-project

jobs:
  deploy:
    runs-on: [self-hosted, backend]

    steps:
      - name: Pull latest code
        run: >
          cd $DEPLOY_DIR &&
          git pull https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/<owner>/<repo>.git main

      - name: Restart backend
        shell: bash -l {0}
        run: |
          cd $DEPLOY_DIR/backend

          STATUS=$(npx pm2 jlist 2>/dev/null | node -e "
            const d = JSON.parse(require('fs').readFileSync('/dev/stdin','utf8') || '[]');
            const p = d.find(x => x.name === 'server');
            console.log(p ? p.pm2_env.status : 'not_found');
          " 2>/dev/null || echo "no_daemon")

          echo "PM2 server status: $STATUS"

          case "$STATUS" in
            online)
              npx pm2 reload server --update-env
              ;;
            stopped|errored)
              npx pm2 delete server
              NODE_ENV=production npx pm2 start server.js --name server
              ;;
            *)
              npx pm2 delete server 2>/dev/null
              NODE_ENV=production npx pm2 start server.js --name server
              ;;
          esac

          npx pm2 save

      - name: Health check
        run: |
          for i in 1 2 3 4 5; do
            if curl -sf http://localhost:4000/ > /dev/null 2>&1; then
              echo "Health check passed"
              exit 0
            fi
            sleep 2
          done
          echo "Health check failed after 10s"
          cd $DEPLOY_DIR/backend && npx pm2 logs server --lines 20 --nostream
          exit 1
```

### 워크플로우 핵심 포인트

**`shell: bash -l {0}`** — 로그인 셸로 실행합니다. Self-hosted runner는 서비스로 동작하기 때문에 사용자의 `.bashrc`, `.profile`이 자동으로 로드되지 않습니다. nvm이나 특정 PATH가 필요한 경우 이 설정이 필수입니다.

**`GITHUB_TOKEN`으로 HTTPS pull** — 서버의 git remote가 SSH로 설정되어 있어도, runner 환경에서는 GitHub의 host key 문제가 발생할 수 있습니다. `GITHUB_TOKEN`을 활용한 HTTPS pull이 가장 안정적입니다.

**절대 경로 사용** — `~`(틸드)는 YAML의 `working-directory`에서 shell 확장이 되지 않습니다. 반드시 `/home/ubuntu/...` 같은 절대 경로를 사용해야 합니다.

**PM2 상태별 분기** — `pm2 kill`(데몬 전체 종료) 대신 프로세스 상태를 확인하고 분기 처리합니다:

| 상태 | 동작 |
|------|------|
| `online` | `pm2 reload` (graceful restart, 무중단) |
| `stopped` / `errored` | `pm2 delete` 후 새로 `pm2 start` |
| 프로세스 없음 / 데몬 없음 | 새로 `pm2 start` |

서버 재부팅이나 비정상 종료 등 어떤 상황에서도 배포가 정상 동작합니다.

**npx 사용** — PM2를 글로벌 설치하지 않고 프로젝트의 `node_modules`에 있는 PM2를 `npx`로 실행합니다. 글로벌 패키지 의존성을 줄일 수 있습니다.

**헬스체크** — 배포 후 서버가 실제로 응답하는지 확인합니다. 2초 간격으로 5회 재시도하고, 실패 시 PM2 로그를 출력하여 원인 파악이 가능합니다.

## 3. 삽질 회고: Self-hosted Runner 도입 시 주의할 점

Self-hosted runner를 처음 도입하면 예상치 못한 문제를 만나게 됩니다. 실제로 겪었던 문제들을 정리합니다.

### SSH host key 문제

서버의 git remote가 SSH(`git@github.com:...`)로 설정되어 있으면, runner에서 `git pull origin main` 실행 시 `Host key verification failed` 에러가 발생합니다. runner 서비스 환경에는 `known_hosts`가 설정되어 있지 않기 때문입니다.

**해결**: `GITHUB_TOKEN`을 활용한 HTTPS URL로 pull합니다.

```yaml
run: git pull https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/owner/repo.git main
```

### PATH / 환경변수 누락

Runner가 시스템 서비스로 동작하면 사용자의 shell profile이 로드되지 않습니다. `node`, `npm`, `pm2` 등의 명령어를 찾지 못하는 `command not found` 에러가 발생합니다.

**해결**: `shell: bash -l {0}`으로 로그인 셸을 명시합니다.

### 틸드(`~`) 경로 미확장

GitHub Actions의 `working-directory` 설정에서 `~/my-project`처럼 틸드를 사용하면 문자 그대로 `~/my-project`라는 디렉토리를 찾으려 합니다.

**해결**: `env`로 절대 경로를 정의하고 `run` 블록 내에서 `cd`로 이동합니다.

### PM2 데몬 전체 종료 문제

`pm2 kill`은 PM2 데몬 자체를 종료합니다. Runner 환경에서 데몬을 죽인 뒤 다시 시작하면 환경 차이로 프로세스가 제대로 뜨지 않을 수 있습니다.

**해결**: `pm2 reload`나 `pm2 restart`로 프로세스만 재시작하고, 프로세스가 없는 경우에만 `pm2 start`로 새로 등록합니다. `pm2 save`로 상태를 저장해두면 서버 재부팅 후에도 PM2가 자동 복구합니다.

## 4. Runner 관리 명령어

설치 이후 운영에 필요한 명령어를 정리합니다.

```bash
# 서비스 상태 확인
sudo ./svc.sh status

# 서비스 중지 / 시작
sudo ./svc.sh stop
sudo ./svc.sh start

# 서비스 제거 (runner 해제 시)
sudo ./svc.sh uninstall

# runner 등록 해제
./config.sh remove --token <TOKEN>

# 로그 확인
journalctl -u actions.runner.<서비스명> -f
```

## 마무리

이 구성의 장점은 **네트워크 제약 없이** 배포를 자동화할 수 있다는 것입니다. 프론트엔드는 GitHub의 클라우드 러너에서 빌드하여 Cloudflare에 배포하고, 백엔드는 서버에 설치된 Self-hosted runner가 직접 pull하고 재시작합니다.

SSH 포트를 외부에 열거나, 별도의 CI/CD 서버를 구축하지 않아도 됩니다. `main` 브랜치에 push하면 변경된 디렉토리에 따라 각각의 워크플로우가 독립적으로 실행됩니다.

제가 운영 중인 서비스의 경우, 처음에는 프론트엔드를 Vercel에서 서비스하다가 Cloudflare Pages로 이전했고, 백엔드는 집에서만 접속 가능한 NAS를 경유해 VM에 수동으로 접근한 뒤 git pull과 PM2 재시작을 직접 수행하는 방식이었습니다. 이번에 위 구조로 정리하고 나니, `main` 브랜치에 push만 하면 프론트와 백엔드가 각각 자동으로 배포되어 훨씬 간편해졌습니다.

다만 백엔드 배포 시 PM2가 프로세스를 재시작하는 동안 짧은 다운타임이 발생한다는 점은 아직 과제로 남아 있습니다. 다음 글에서는 이 부분을 어떻게 개선할 수 있을지 고민한 결과를 공유해 보겠습니다.
