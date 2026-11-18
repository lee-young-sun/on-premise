# 설치 중 `needs_recovery` 복구 절차 (서버 1, `replication` 단계)

온프레미스 설치기가 서버 1의 `replication` 단계에서 실패한 뒤 `needs_recovery`로 잠겼을 때의 수동 복구 절차다.

- 설치기에는 이 잠금을 푸는 명령이 없다. 아래는 설치기 코드를 읽어 정리한 절차이며 실제 서버에서 검증하지 않았다.
- 상태 파일을 직접 고치는 작업이 들어 있다. 현장 담당자가 아니라 개발 담당 판단으로 진행한다.
- 경로는 기본 설치 경로(`/data/looma`) 기준이다. 다르게 설치했다면 `<install_root>`로 바꾼다.

## `needs_recovery`란

원인이 아니라 잠금이다. DB를 바꾸는 단계(`migrations`, `bootstrap`, `clone`, `replication`)가 실패하거나 도중에 끊기면 설치 상태가 `needs_recovery`로 남고, 이후 실행은 단계를 시작하기 전에 같은 메시지로 멈춘다. 다시 실행해도 결과는 같다.

실제 원인은 `<install_root>/state/state.json`에 남는다.

| 값 | 의미 |
|---|---|
| `running` | 멈춘 순간 실행 중이던 단계 |
| `error_code` | 그 단계가 실패한 원인 코드 |
| `completed` | 끝난 단계 목록 |
| `database_running` | `true`이면 DB 단계 도중에 멈춘 것 |

## 서버 1의 `replication` 단계에서 날 수 있는 오류

이 단계는 복제 계정 두 개를 만들고 서버 2에서 받아 오기를 걸어 둔다. 서버 2가 없어도 통과하므로 방화벽이나 서버 2 쪽 문제는 원인이 아니다.

| 종류 | 언제 나는가 | `error_code` |
|---|---|---|
| 디스크 부족 | 계정 생성과 복제 설정을 디스크에 쓰지 못함 | `command_failed` 또는 `command_timeout` |
| DB가 복제 옵션 없이 떠 있음 | GTID가 꺼진 DB에서 받아 오기 설정이 거절됨. 이전 설치 시도의 DB 컨테이너가 재사용된 경우 | `command_failed` |
| DB 컨테이너가 도중에 죽음 | 메모리 부족, 수동 정지 등 | `command_failed` |
| SQL은 됐는데 확인이 실패 | 실행 뒤 계정과 받아 오기 대상이 기대와 다름 | `postcondition_failed` |
| 실행이 끊김 | Ctrl-C, SSH 끊김, 재부팅 | 값 없음, `running`이 `replication` |

`disk_insufficient`는 사전 점검이 내는 별개의 결과다. 실제 디스크가 찬 것이 아니라 "새 설치 요구량 + 예비분"에 못 미친다는 뜻이며, 설치 기록이 있으면 재실행 때 검사하지 않으므로 설치를 막지 않는다.

## 복구 절차

### 1. 적용 대상인지 확인

```bash
sudo cat /data/looma/state/state.json
```

- `running`이 `replication`이고 `completed`에 `replication`이 없어야 한다.
- `error_code`가 `needs_recovery`이면 이 절차로 풀리지 않는다. 끝난 DB 단계가 재확인에서 실패한 경우라 잠금을 풀어도 다시 잠긴다.

### 2. 원인 확인과 제거

DB 컨테이너 이름을 확인한다.

```bash
docker ps -a --filter label=com.docker.compose.service=db --format '{{.Names}}\t{{.Status}}'
```

어느 SQL까지 반영됐는지 조회한다. 읽기 전용이다.

```bash
docker exec <DB 컨테이너 이름> mysql --defaults-extra-file=/run/secrets/db_client_config -e "SELECT @@gtid_mode, @@server_id; SELECT user, host FROM mysql.user WHERE user='looma_repl'; SELECT HOST FROM performance_schema.replication_connection_configuration;"
```

| 결과 | 조치 |
|---|---|
| `gtid_mode`가 `OFF` | DB 컨테이너가 복제 옵션 없이 떠 있다. `docker rm -f <DB 컨테이너 이름>`으로 컨테이너만 지운다. 데이터는 호스트의 DB 데이터 폴더에 남고, 재실행 때 올바른 옵션으로 다시 뜬다 |
| 명령이 실패 (컨테이너 중단) | `docker logs --tail 200 <DB 컨테이너 이름>`으로 죽은 이유를 확인해 고친다. 재실행 때 `db` 단계가 다시 띄운다 |
| `gtid_mode`가 `ON`, 계정 2행, `HOST`가 서버 2 주소 | SQL은 이미 반영됐다. 3번으로 간다 |
| `gtid_mode`가 `ON`인데 계정이나 `HOST`가 비어 있음 | 일시적 실패였을 가능성이 크다. 3번으로 가서 재실행한다 |

원인을 고치지 않고 잠금만 풀면 같은 자리에서 다시 잠긴다.

### 3. 백업 후 잠금 해제

```bash
sudo cp -a /data/looma/state /root/looma-state-backup-$(date +%Y%m%d-%H%M%S)
```

```bash
sudo python3 - <<'EOF'
import json, os
p = '/data/looma/state/state.json'
s = json.load(open(p))
assert s.get('running') == 'replication' and 'replication' not in s.get('completed', []), '대상 아님'
s.update(phase='failed', database_running=False)
tmp = p + '.tmp'
with open(tmp, 'w') as f:
    json.dump(s, f, sort_keys=True); f.flush(); os.fsync(f.fileno())
os.chmod(tmp, 0o600)
os.replace(tmp, p)
print('unlocked')
EOF
```

바꾸는 값은 `phase`와 `database_running` 둘뿐이다. 1번 조건에 맞지 않으면 아무것도 바꾸지 않고 멈춘다.

### 4. 같은 명령으로 재실행

처음 설치할 때 쓴 마법사 명령을 입력값 그대로 다시 실행한다. 설치 기록이 있으므로 이어 하기를 묻고 사전 점검은 건너뛴다. 끝난 단계는 실제 상태를 확인하고 넘어가며 `replication`부터 다시 실행한다. 입력값이 달라지면 `resume_mismatch`로 거절된다.

이 단계의 SQL(계정 만들기, 받아 오기 설정)은 다시 실행해도 안전하게 만들어져 있다.

### 5. 확인

설치가 끝나면 `status` 출력의 `replication` 묶음을 본다. 서버 2 설치 전의 서버 1은 `state`가 `degraded`, `to_peer`가 `unreachable`인 것이 정상이다.

## 이 절차로 안 될 때

다시 `replication`에서 잠기거나 DB 상태가 의심스러우면 "전부 내리고 처음부터 다시 설치"로 간다. 서버 1 첫 설치라 운영 데이터가 없을 때만 가능하고, DB 데이터 폴더와 생성된 비밀을 지우므로 되돌릴 수 없다. 실제 `state.json`과 위 조회 결과를 확인한 뒤 진행한다.

## 주의

- DB 컨테이너 로그를 외부로 전달할 때는 복제 적용 오류 줄에 DB 행 값이 찍힐 수 있다.
- 상태 폴더와 DB 데이터 폴더는 지우지 않는다.
