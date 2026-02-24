# [우리FISA 6기] MySQL GTID 기반 복제 & Failover 실습 정리
# **GTID 기반 복제 환경 구성**

**📌 목표**

- File/Position 기반이 아닌
- **GTID(Global Transaction Identifier)** 기반 복제 구성
- AUTO_POSITION=1을 이용한 자동 위치 동기화

---

## **1. Source 설정 (GTID 활성화)**

**1-1) 컨테이너 진입**

```bash
docker exec -it mysql-source bash
```

**1-2) 편집기 설치**

```bash
microdnf install -y vim
```

**1-3) /etc/my.cnf 수정**

```bash
vim /etc/my.cnf
```

- 아래 내용 추가
    
    ```
    [mysqld]
    server-id=1
    log-bin=mysql-bin
    binlog_format=ROW
    
    gtid_mode=ON
    enforce_gtid_consistency=ON
    ```
    

**1-4) 컨테이너 재시작**

```bash
docker restart mysql-source
```

---

## **2. Replica 설정**

**2-1) /etc/my.cnf 수정**

```bash
[mysqld]
server-id=2
log-bin=mysql-bin
relay-log=relay-bin
binlog_format=ROW

gtid_mode=ON
enforce_gtid_consistency=ON

read_only=ON
```

**2-2) 컨테이너 재시작**

```bash
docker restart mysql-replica
```

---

## 3. GTID 모드 확인

각 컨테이너에서:

```sql
SHOW VARIABLES LIKE 'gtid_mode';
```

---

## 4. GTID 기반 복제 연결

File/Position 없이
GTID 기반 자동 위치 계산

```bash
docker exec -it mysql-replica mysql -uroot -p1234 -e "
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql-source',
  SOURCE_USER='jaypark',
  SOURCE_PASSWORD='1234',
  SOURCE_PORT=3306,
  SOURCE_AUTO_POSITION=1;
START REPLICA;
"
```

---

## 5. 복제 상태 확인

```sql
SHOW REPLICA STATUS\G
```

- 아래와 같이 나와야 한다.
Replica_IO_Running: Yes
Replica_SQL_Running: Yes

---

# 🔥 Failover 실습

> 
> 
> 
> Failover 실습은 **Primary(Source) 서버 장애 발생 상황을 가정**하여
> Replica를 승격(Promote)하고, 이후 기존 Source를 다시 복구하여
> 새로운 Replica로 재합류시키는 과정을 단계별로 확인하는 실습입니다.
> 

## 1. Source 장애 발생

```bash
docker stop mysql-source
```

---

## **2. Replica 승격 (Promote)**

```bash
docker exec -it mysql-replica mysql -uroot -p1234 -e "
STOP REPLICA;
RESET REPLICA ALL;
SET GLOBAL read_only=OFF;
"
```

- 이제 mysql-replica는 새로운 Source 컨테이너가 된다.

---

## 3. 승격된 서버에서 write 테스트

```sql
USE testdb;
INSERT INTO products VALUES ('after_failover');
```

이 시점부터 GTID 히스토리는 아래와 같다:

```
A:1-10  (기존)
B:1-3   (승격 후 생성)
```

---

## 🔃 Failback(기존 Source 재합류)

## 4. 기존 Source 재시작

```bash
docker start mysql-source
```

이 시점에서는

- 기존 mysql-source는 과거 상태(장애 발생 시점 전)
- 새로운 트랜잭션(B)을 모르는 상태

---

## 5. 기존 Source를 Replica로 전환

### 5-1. 복제 설정 초기화 + read_only

```bash
docker exec -it mysql-source mysql -uroot -p1234 -e "
STOP REPLICA;
RESET REPLICA ALL;
SET GLOBAL read_only=ON;
"
```

### **5-2. 새 Source(mysql-replica) 바라보도록 설정**

```bash
docker exec -it mysql-source mysql -uroot -p1234 -e "
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='mysql-replica',
  SOURCE_USER='jaypark',
  SOURCE_PASSWORD='1234',
  SOURCE_PORT=3306,
  SOURCE_AUTO_POSITION=1;
START REPLICA;
"
```

이후 mysql-replica에서 write 발생 시
mysql-source에도 자동 반영이 된다.

---

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
