---
title: "Shell scripting — bash + best practice"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:37:00+09:00
tags: [devops, linux, shell, bash]
---

# Shell scripting — bash + best practice

**[[linux|↑ linux]]**

---

## 1. 표준 header

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

- `-e` — error 시 즉시 종료
- `-u` — 정의 안 된 변수 사용 시 error
- `-o pipefail` — pipe 중 실패도 감지
- `IFS=$'\n\t'` — 공백 split 비활성

---

## 2. function

```bash
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*"
}

error() {
    log "ERROR: $*" >&2
    exit 1
}

retry() {
    local n=$1; shift
    local i=0
    while [ $i -lt $n ]; do
        if "$@"; then return 0; fi
        i=$((i+1))
        sleep $((2**i))   # exponential backoff
    done
    return 1
}

# 사용
log "starting"
retry 3 curl -fsSL http://api/health || error "API down"
```

---

## 3. argument parsing

```bash
# 간단한 positional
NAME="$1"
PORT="${2:-8080}"

# getopts (short option)
while getopts ":n:p:vh" opt; do
    case $opt in
        n) NAME="$OPTARG" ;;
        p) PORT="$OPTARG" ;;
        v) VERBOSE=1 ;;
        h) usage; exit 0 ;;
        \?) error "Invalid option -$OPTARG" ;;
    esac
done
shift $((OPTIND-1))

# long option — manual parse
while [[ $# -gt 0 ]]; do
    case $1 in
        --name) NAME="$2"; shift 2 ;;
        --port) PORT="$2"; shift 2 ;;
        --verbose) VERBOSE=1; shift ;;
        -h|--help) usage; exit 0 ;;
        *) error "unknown: $1" ;;
    esac
done
```

---

## 4. trap (cleanup)

```bash
TMPDIR=$(mktemp -d)
trap 'rm -rf "$TMPDIR"' EXIT     # script 종료 시 항상 cleanup
trap 'echo interrupted; exit 130' INT TERM
```

→ Ctrl+C / 에러 / 정상 종료 어떤 경우든 cleanup.

---

## 5. array / map

```bash
# array
FILES=(a.log b.log c.log)
echo "${FILES[0]}"            # a.log
echo "${FILES[@]}"            # 전체
echo "${#FILES[@]}"           # 개수

for f in "${FILES[@]}"; do
    echo "$f"
done

# associative array (bash 4+)
declare -A USERS
USERS[alice]=admin
USERS[bob]=user

for u in "${!USERS[@]}"; do
    echo "$u → ${USERS[$u]}"
done
```

---

## 6. 외부 도구

```bash
# JSON — jq
curl -s api/users | jq -r '.[].email'
echo '{"a":1,"b":2}' | jq '.a'

# YAML — yq
yq '.spec.replicas' deployment.yaml

# date
date -u +"%Y-%m-%dT%H:%M:%SZ"     # ISO 8601 UTC
date -d "1 hour ago" +%s          # epoch

# UUID / random
uuidgen
openssl rand -hex 16

# checksum
sha256sum file.txt
md5sum file.txt
```

---

## 7. 디버깅

```bash
bash -x script.sh                # trace 모드 실행
bash -n script.sh                # syntax check (실행 X)

# script 안에서
set -x                            # 여기서부터 trace 켜기
set +x                            # 끄기
echo "DEBUG: var=$var" >&2
```

→ **shellcheck** 도구 필수 사용:
```bash
brew install shellcheck
shellcheck script.sh
```

---

## 8. CI / 배포 스크립트 예

```bash
#!/usr/bin/env bash
set -euo pipefail

readonly APP="${APP:?APP required}"
readonly TAG="${TAG:?TAG required}"
readonly ENV="${ENV:-dev}"

log() { echo "[$(date +%T)] $*"; }
die() { echo "ERROR: $*" >&2; exit 1; }

build() {
    log "Building $APP:$TAG"
    docker build -t "$APP:$TAG" .
}

push() {
    log "Pushing $APP:$TAG"
    docker push "registry.example.com/$APP:$TAG"
}

deploy() {
    log "Deploying $APP:$TAG to $ENV"
    kubectl set image "deployment/$APP" \
        "$APP=registry.example.com/$APP:$TAG" \
        -n "$ENV" \
        --record
    kubectl rollout status "deployment/$APP" -n "$ENV" --timeout=5m \
        || die "rollout failed"
}

main() {
    build
    push
    deploy
    log "✓ Done"
}

main "$@"
```

---

## 9. 함정

1. **`set -e` 없음** — error 무시.
2. **`set -u` 없음** — typo 변수 빈 문자열 → `rm -rf /`.
3. **quote 안 함** — 공백 / glob 깨짐.
4. **path hard-code** — `/home/alice/...` → 다른 환경 X.
5. **idempotent 아님** — 두 번 실행 시 다른 결과.
6. **error message stdout 으로** — pipe 시 깨짐. `>&2` 사용.
7. **trap 없음** — temp file 누적 / lock 안 풀림.
8. **shebang 없음** — `bash` 가 아닌 `sh` 로 실행 시 syntax error.

---

## 10. 큰 script ≠ bash

- 200 줄 넘으면 Python / Go 검토.
- 복잡 로직, 테스트, 에러 핸들링 = bash 한계.

---

## 11. 관련

- [[linux|↑ linux]]
- [[shell-basics]]
- [[process-management]]
