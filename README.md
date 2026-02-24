# mysql-replication
고가용성을 보장하기 위한 이원화





## 🔧 Troubleshooting

### ❗ Error
![GTID Error](images/error_image.png)

@@GLOBAL.GTID_PURGED cannot be changed:
the added gtid set must not overlap with @@GLOBAL.GTID_EXECUTED


### 📌 에러를 마주한 상황

- Replica 서버에 직접 SQL 실행 → 새로운 GTID 생성
- 이후 복구를 위해 Source에서 dump 생성 후 Replica에 적용
- dump 파일에 `SET @@GLOBAL.GTID_PURGED='xxx';` 구문 포함
- Replica에 기존 `GTID_EXECUTED` 기록이 남아있어 충돌 발생


### 📌 원인

Replica에 이미 실행된 `GTID_EXECUTED`가 존재하는 상태에서  
dump가 `GTID_PURGED`를 새로 설정하려 하면서 GTID 집합이 겹쳐 충돌 발생.

정리하면,
> Replica는 이미 처리한 트랜잭션 기록이 있는데  
> dump가 새로운 GTID 족보를 강제로 세팅하려 해서 에러가 발생한 것이다.


### 🛠 해결 방법 1: Replica 완전 초기화

Replica를 새 서버처럼 초기화 후 복구 진행

```sql
STOP REPLICA;
RESET REPLICA ALL;
RESET MASTER;  -- MySQL 8.0
-- 또는
RESET BINARY LOGS AND GTIDS;  -- MySQL 8.4+
```

이후:
- dump 복구


### 🛠 해결 방법 2: dump 생성 시 GTID 제외

Source에서 dump 생성 시 '--set-gtid-purged=OFF'를 넣어 진행 :

```bash
mysqldump --set-gtid-purged=OFF -u root -p testdb > clean2.sql
```

→ dump 파일에 `SET @@GLOBAL.GTID_PURGED` 구문이 포함되지 않아 Replica의 기존 GTID와 충돌하지 않음.

### 📌 참고: 언제 SET @@GLOBAL.GTID_PURGED 가 포함되는가?
- `mysqldump`의 `--set-gtid-purged` 옵션에 의해 결정됨
- 기본값: `AUTO`
- `--set-gtid-purged=AUTO` + `gtid_mode=ON` 인 경우 dump 파일에 포함됨
- `gtid_mode=OFF` 인 경우 포함되지 않음


### 📸 에러 해결 전 → 해결 후

<p align="center">
  <img src="images/error-before.png" width="45%" />
  <img src="images/error-after.png" width="45%" />
</p>
