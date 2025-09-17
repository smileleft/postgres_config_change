# postgres_config_change
docker postgresql config(postgresql.conf) change script

# reports, chunks 적재 중 docker postgres 설정값 조절 절차

---

# 0) 컨테이너 / DB 접속 전제

컨테이너 이름은 `pg`로 가정

```bash
# psql 접속 테스트
docker exec -it pg psql -U postgres -d postgres -c "SELECT version();"

```

---

# 1) WAL 생성률/복제 지연 모니터링 (실시간 체크용)

```bash
# (A) 초당 WAL 생성량 대략 보기 (pg_stat_wal, PG13+)
docker exec -it pg psql -U postgres -c "SELECT * FROM pg_stat_wal;"

# (B) 현재 LSN 확인 → 1~2초 간격으로 두 번 찍어 차이/초 계산
docker exec -it pg psql -U postgres -c "SELECT pg_current_wal_lsn();"

# (C) 복제 지연(물리복제) 바이트 단위
docker exec -it pg psql -U postgres -c \
\"SELECT application_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS byte_lag FROM pg_stat_replication;\"

```

> 적재 중엔 (A)와 (C)를 주기적으로 체크하면서 확인
> 

---

# 2) 설정 변경(재시작 없음): `ALTER SYSTEM` + `pg_reload_conf()`

`ALTER SYSTEM`은 `postgresql.auto.conf`를 갱신하고, `pg_reload_conf()`로 **리로드만**  적용.

(WAL 관련 파라미터 대부분은 리로드로 적용 가능)

### 2.1 `max_wal_size` 등 변경

```bash
# 예: max_wal_size를 128GB로
docker exec -it pg psql -U postgres -c "ALTER SYSTEM SET max_wal_size = '128GB';"
docker exec -it pg psql -U postgres -c "ALTER SYSTEM SET min_wal_size = '32GB';"

# 복제 안전 여유(필요 시)
docker exec -it pg psql -U postgres -c "ALTER SYSTEM SET wal_keep_size = '16GB';"

# (옵션) 논리/물리 복제 슬롯 WAL 상한 (PG14+)
docker exec -it pg psql -U postgres -c "ALTER SYSTEM SET max_slot_wal_keep_size = '40GB';"

# (옵션) 체크포인트 조정
docker exec -it pg psql -U postgres -c "ALTER SYSTEM SET checkpoint_timeout = '30min';"
docker exec -it pg psql -U postgres -c "ALTER SYSTEM SET checkpoint_completion_target = '0.95';"

# (옵션) WAL 압축
docker exec -it pg psql -U postgres -c "ALTER SYSTEM SET wal_compression = 'on';"

```

### 2.2 리로드(재시작 아님)

```bash
# SQL로 리로드
docker exec -it pg psql -U postgres -c "SELECT pg_reload_conf();"

# 또는 쉘에서 pg_ctl로
docker exec -u postgres pg bash -lc "pg_ctl -D \$PGDATA reload"

```

### 2.3 적용 확인

```bash
docker exec -it pg psql -U postgres -c "SHOW max_wal_size;"
docker exec -it pg psql -U postgres -c "SHOW min_wal_size;"
docker exec -it pg psql -U postgres -c "SHOW wal_keep_size;"
docker exec -it pg psql -U postgres -c "SHOW max_slot_wal_keep_size;"
docker exec -it pg psql -U postgres -c "SHOW checkpoint_timeout;"
docker exec -it pg psql -U postgres -c "SHOW checkpoint_completion_target;"
docker exec -it pg psql -U postgres -c "SHOW wal_compression;"

```

> 체크포인트 관련 값은 다음 체크포인트 주기부터 반영. 즉시 반영 확인하고 싶으면 아래처럼 수동 체크포인트를 유도.
> 

```bash
docker exec -it pg psql -U postgres -c "CHECKPOINT;"

```

---

# 3) 필요 시 `postgresql.conf` 직접 수정 + 리로드

`ALTER SYSTEM` 대신 conf 파일을 직접 다루고 싶다면:

```bash
# (예) sed로 conf에 주입
docker exec -u postgres pg bash -lc "echo \"max_wal_size = '128GB'\" >> \$PGDATA/postgresql.conf"
docker exec -u postgres pg bash -lc "pg_ctl -D \$PGDATA reload"

```

> Compose에서 command: ["-c", "max_wal_size=..."]로 넘기는 방식은 컨테이너 재시작이 필요.
> 
> 
> 운영 중 즉시 반영하려면 `ALTER SYSTEM` + `pg_reload_conf()`가 가장 편리함.
> 

---

# 4) 재시작이 **필요한** 파라미터는 별도 관리

아래는 리로드로 안 되는 것들(재시작 필요) → 계획된 점검 때만 변경:

- `wal_level`, `max_wal_senders`, `max_replication_slots`, `shared_buffers`, `max_connections`, `huge_pages`, `effective_io_concurrency` 등

```bash
# 재시작이 필요한 값을 바꿨다면
docker restart pg

```

---

# 5) 권장 운용 패턴(이번 워크로드 기준)

- 적재(백필) 시작 → (1) 모니터링 하며 (2) `max_wal_size`, `wal_keep_size`를 위 방식으로 즉시 조정
- 복제 지연이 커지면:
    - 일시적으로 `wal_keep_size`↑ 또는 `max_slot_wal_keep_size`↑
    - replica I/O/네트워크 확인
- checkpoint가 너무 잦아 I/O 스파이크면 `max_wal_size`↑
- 적재 끝 → 인덱스 생성 → 운영값으로 되돌림

---

## 위 절차를 실행해 주는 bash 스크립트

```bash
#!/usr/bin/env bash
set -euo pipefail

# Default values
CTR="pg"                 # Docker container name
DBUSER="postgres"
DBNAME="postgres"

usage() {
  cat <<'EOF'
Usage:
  pg_wal_tune.sh [options]

Options (common):
  -c, --container NAME            Docker container name (default: pg)
  -U, --user USER                 DB user (default: postgres)
  -d, --dbname DB                 DB name (default: postgres)

WAL/Checkpoint tuning (applies via ALTER SYSTEM + reload):
      --max SIZE                  Set max_wal_size (e.g., 128GB)
      --min SIZE                  Set min_wal_size (e.g., 32GB)
      --keep SIZE                 Set wal_keep_size (e.g., 16GB)
      --slot-keep SIZE            Set max_slot_wal_keep_size (e.g., 40GB) [PG14+]
      --ckpt-timeout DURATION     Set checkpoint_timeout (e.g., 30min)
      --ckpt-target FLOAT         Set checkpoint_completion_target (e.g., 0.95)
      --wal-compress on|off       Set wal_compression (on/off)

Helpers:
      --show                      Show current key settings
      --apply                     Reload config (SELECT pg_reload_conf())
      --checkpoint                Force a CHECKPOINT (careful in prod)
      --reset PARAM               ALTER SYSTEM RESET <param>
      --help                      Show this help

Examples:
  # Increase WAL headroom & keep more WAL for replicas; apply immediately
  ./pg_wal_tune.sh --max 128GB --min 32GB --keep 16GB --apply

  # Set slot WAL cap & longer checkpoint; then show effective values
  ./pg_wal_tune.sh --slot-keep 40GB --ckpt-timeout 30min --ckpt-target 0.95 --apply --show

  # Turn WAL compression on (no restart), then checkpoint now
  ./pg_wal_tune.sh --wal-compress on --apply --checkpoint

  # Reset a parameter to default (e.g., max_wal_size), then reload
  ./pg_wal_tune.sh --reset max_wal_size --apply
EOF
}

die() { echo "ERROR: $*" >&2; exit 1; }

# Parse args
ARGS=()
while [[ $# -gt 0 ]]; do
  case "$1" in
    -c|--container) CTR="${2:-}"; shift 2;;
    -U|--user) DBUSER="${2:-}"; shift 2;;
    -d|--dbname) DBNAME="${2:-}"; shift 2;;
    --max) MAX="${2:-}"; shift 2;;
    --min) MIN="${2:-}"; shift 2;;
    --keep) KEEP="${2:-}"; shift 2;;
    --slot-keep) SLOTKEEP="${2:-}"; shift 2;;
    --ckpt-timeout) CKPT_TIMEOUT="${2:-}"; shift 2;;
    --ckpt-target) CKPT_TARGET="${2:-}"; shift 2;;
    --wal-compress) WALCOMP="${2:-}"; shift 2;;
    --show) SHOW=1; shift;;
    --apply) APPLY=1; shift;;
    --checkpoint) DOCKPT=1; shift;;
    --reset) RESETPARAM="${2:-}"; shift 2;;
    --help|-h) usage; exit 0;;
    *) ARGS+=("$1"); shift;;
  esac
done
set -- "${ARGS[@]:-}"

# Basic checks
command -v docker >/dev/null || die "docker not found"
docker ps --format '{{.Names}}' | grep -Fxq "$CTR" || die "container '$CTR' not running"

psql_exec() {
  docker exec "$CTR" psql -X -v ON_ERROR_STOP=1 -U "$DBUSER" -d "$DBNAME" -tAc "$1"
}

set_param() {
  local PARAM="$1" VAL="$2"
  echo "ALTER SYSTEM SET $PARAM = '$VAL';"
  psql_exec "ALTER SYSTEM SET $PARAM = '$VAL';"
}

reset_param() {
  local PARAM="$1"
  echo "ALTER SYSTEM RESET $PARAM;"
  psql_exec "ALTER SYSTEM RESET $PARAM;"
}

reload_conf() {
  echo "SELECT pg_reload_conf();"
  psql_exec "SELECT pg_reload_conf();"
}

force_checkpoint() {
  echo "CHECKPOINT;"
  psql_exec "CHECKPOINT;"
}

show_all() {
  echo "----- Current settings -----"
  psql_exec "SHOW max_wal_size;"
  psql_exec "SHOW min_wal_size;"
  psql_exec "SHOW wal_keep_size;"
  psql_exec "SHOW max_slot_wal_keep_size;" || true
  psql_exec "SHOW checkpoint_timeout;"
  psql_exec "SHOW checkpoint_completion_target;"
  psql_exec "SHOW wal_compression;"
  echo "----- WAL stats (pg_stat_wal) -----"
  psql_exec "SELECT * FROM pg_stat_wal;" || true
  echo "----- Replication lag (if any) -----"
  psql_exec "SELECT application_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS byte_lag FROM pg_stat_replication;" || true
}

# Apply requested changes
CHANGED=0

[[ -n "${MAX:-}" ]]           && { set_param max_wal_size "$MAX"; CHANGED=1; }
[[ -n "${MIN:-}" ]]           && { set_param min_wal_size "$MIN"; CHANGED=1; }
[[ -n "${KEEP:-}" ]]          && { set_param wal_keep_size "$KEEP"; CHANGED=1; }
[[ -n "${SLOTKEEP:-}" ]]      && { set_param max_slot_wal_keep_size "$SLOTKEEP"; CHANGED=1; }
[[ -n "${CKPT_TIMEOUT:-}" ]]  && { set_param checkpoint_timeout "$CKPT_TIMEOUT"; CHANGED=1; }
[[ -n "${CKPT_TARGET:-}" ]]   && { set_param checkpoint_completion_target "$CKPT_TARGET"; CHANGED=1; }

if [[ -n "${WALCOMP:-}" ]]; then
  case "$WALCOMP" in
    on|off) set_param wal_compression "$WALCOMP"; CHANGED=1;;
    *) die "--wal-compress must be 'on' or 'off'";;
  esac
fi

if [[ -n "${RESETPARAM:-}" ]]; then
  reset_param "$RESETPARAM"
  CHANGED=1
fi

# Reload if requested or if changes were made and --apply provided
if [[ -n "${APPLY:-}" ]]; then
  reload_conf
fi

# Optional checkpoint
[[ -n "${DOCKPT:-}" ]] && force_checkpoint

# Show settings if requested
[[ -n "${SHOW:-}" ]] && show_all

# If nothing was done and no flags, print help
if [[ $CHANGED -eq 0 && -z "${APPLY:-}" && -z "${SHOW:-}" && -z "${DOCKPT:-}" ]]; then
  usage
fi

```

### 스크립트 실행 예시

```bash
chmod +x pg_wal_tune.sh

# 1) WAL 여유 늘리고 즉시 적용
./pg_wal_tune.sh --max 128GB --min 32GB --keep 16GB --apply

# 2) 논리/물리 슬롯 WAL 상한, 체크포인트 조정, 적용 후 확인
./pg_wal_tune.sh --slot-keep 40GB --ckpt-timeout 30min --ckpt-target 0.95 --apply --show

# 3) WAL 압축 켜고(재시작 불필요) 바로 체크포인트
./pg_wal_tune.sh --wal-compress on --apply --checkpoint

# 4) 컨테이너 이름이 다를 때
./pg_wal_tune.sh -c my-postgres --max 256GB --apply

```
