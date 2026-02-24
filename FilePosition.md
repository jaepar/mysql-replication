# 🐬 MySQL Replication 실습

## 📌 개요

Docker 기반으로 MySQL Source 와 Replica 를 구성하고  
Binary Log 기반 복제를 이해

- `mysql-source` → 쓰기 발생
- `mysql-replica` → Source 데이터를 동기화

---

## ⚙️ 동작 원리

1. Source에서 데이터 변경
2. 변경 내용을 **Binary Log**에 기록
3. Replica IO Thread → Binlog 읽어 **Relay Log**에 저장
4. Replica SQL Thread → Relay Log 실행

=> Source 작업이 Replica에 동일하게 반영됨

---

## 🚀 실습 절차

### [1️] Source 실행
**1.1 source 실행**

```bash
docker run -p 3306:3306 \
  --name mysql-source \
  -e MYSQL_ROOT_PASSWORD=1234 \
  -d mysql
```
**1.2 설정 (/etc/my.cnf)**

```ini
[mysqld]
log-bin=mysql-bin
server-id=1
```


### [2] 복제 사용자 생성 및 테이블 생성
**2.1 복제 사용자 생성**
```sql
CREATE USER 'name'@'%' IDENTIFIED BY '1234';
GRANT REPLICATION SLAVE ON *.* TO 'name'@'%';
FLUSH PRIVILEGES;
```
**2.2 테이블 생성**

### [3] source Dump 생성

```bash
mysqldump -u root -p --all-databases --master-data > dump.sql
```

## [4] Replica 실행
**4.1 replica 실행**
```bash
docker run -p 3307:3306 \
  --name mysql-replica \
  --link mysql-source \
  -e MYSQL_ROOT_PASSWORD=1234 \
  -d mysql
```

**4.2 설정 (/etc/my.cnf)**

```ini
[mysqld]
log-bin=mysql-bin
server-id=2
```

**4.3 Dump 적용**

```bash
docker cp dump.sql mysql-replica:/dump.sql
docker exec -it mysql-replica mysql -u root -p < /dump.sql
```


### [5] Replica 연결 설정

**5.1 Source에서**

```sql
SHOW BINARY LOG STATUS\G;
```

예:
```
File: {파일명}
Position: {위치}
```

**5.2 Replica에서**

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST={},
  SOURCE_USER={},
  SOURCE_PASSWORD={},
  SOURCE_LOG_FILE={파일명}',
  SOURCE_LOG_POS={위치};
```


### [6️] 복제 시작

```sql
START REPLICA;
SHOW REPLICA STATUS\G;
```

정상 상태:
- Replica_IO_Running: Yes
- Replica_SQL_Running: Yes


### [7] 테스트

source에서 작업한 내용이 replica에 그대로 반영되는지 확인

---

## 📚 알아보기

### 1. REPLICATION SLAVE 권한 의미

```
GRANT REPLICATION SLAVE ON *.* TO 'name'@'%';
```
=> Replica가 Source의 **Binary Log를 읽을 수 있는 권한**

권한이 없으면 IO Thread 접속 실패


### 2️. CHANGE REPLICATION SOURCE TO 의미

```
SOURCE_LOG_FILE='mysql-bin.000001'
SOURCE_LOG_POS=1548
```

> mysql-bin.000001 파일의  
> 1548 바이트 위치부터 읽기 시작

- Binary Log = 파일
- Position = 바이트 오프셋
- Commit 될 때마다 Position 증가

=> 이 Position을 기억하기 때문에 재시작 후에도 이어서 동기화 가능
