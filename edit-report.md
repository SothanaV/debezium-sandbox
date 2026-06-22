# Edit Report: Fix debezium_server_oracle

## Problem

Container `debezium_server_oracle` crash loop ตลอด เชื่อมต่อ Oracle ไม่ได้

---

## Root Causes & Fixes

### 1. Oracle image version incompatible กับ Debezium 2.7

**File:** `docker-compose.yaml`

`gvenzl/oracle-free:23-full` (floating tag) ตอนนี้ pull มาได้ Oracle 26ai (version `23.26.2.0.0`) ซึ่ง version banner เปลี่ยนรูปแบบเป็น:

> `Oracle AI Database 26ai Free Release 23.26.2.0.0`

Debezium 2.7 parse version string นี้ไม่ได้ เลย error `Failed to resolve Oracle database version`

**Fix:** Pin image เป็น `gvenzl/oracle-free:23.8-full` ซึ่ง version banner ยังเป็นรูปแบบเดิมที่ Debezium รองรับ

```diff
- image: gvenzl/oracle-free:23-full
+ image: gvenzl/oracle-free:23.8-full
```

---

### 2. Dockerfile download JDBC driver ซ้ำซ้อน

**File:** `debezium-oracle/Dockerfile`

Debezium Server 2.7 base image มี `ojdbc8-23.3.0.23.09.jar` อยู่แล้ว ไม่ต้อง download เพิ่ม

**Fix:** ลด Dockerfile เหลือแค่ base image

```diff
- FROM quay.io/debezium/server:2.7
- USER root
- RUN cd /debezium/lib && \
-     curl -O https://repo1.maven.org/maven2/com/oracle/database/jdbc/ojdbc11/23.2.0.0/ojdbc11-23.2.0.0.jar
- USER 1001
+ FROM quay.io/debezium/server:2.7
```

---

### 3. Oracle ไม่ได้เปิด ARCHIVELOG mode

LogMiner ต้องการ ARCHIVELOG mode ถึงจะทำ CDC ได้ แต่ `gvenzl/oracle-free` ไม่ได้เปิดมาให้ default

**Fix:** สร้างไฟล์ใหม่ `oracle-init/01-enable-archivelog.sh` ที่ทำงานก่อน SQL setup

```bash
#!/bin/bash
sqlplus -S / as sysdba <<EOF
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
EXIT;
EOF
```

และย้าย `setup-logminer.sql` เป็น `02-setup-logminer.sql` เพื่อให้ run ตามลำดับ

---

### 4. c##dbzuser ขาด privileges

**File:** `oracle-init/02-setup-logminer.sql`

Debezium ต้องการ privileges เพิ่มเติมที่ script เดิมไม่ได้ grant:

| Privilege | ใช้ทำอะไร |
|---|---|
| `LOCK ANY TABLE` | Lock table ตอน snapshot |
| `ALTER ANY TABLE` | จัดการ supplemental logging |
| `CREATE TABLE` | สร้าง flush table สำหรับ LogMiner |
| `UNLIMITED TABLESPACE` | quota สำหรับ flush table |

**Fix:** เพิ่ม grants

```sql
GRANT LOCK ANY TABLE TO c##dbzuser CONTAINER=ALL;
GRANT ALTER ANY TABLE TO c##dbzuser CONTAINER=ALL;
GRANT CREATE TABLE TO c##dbzuser CONTAINER=ALL;
GRANT UNLIMITED TABLESPACE TO c##dbzuser CONTAINER=ALL;
```

---

### 5. Debezium config ใช้ SYSTEM user

**File:** `docker-compose.yaml`

Config เดิมใช้ `SYSTEM` user ซึ่งไม่มี LOGMINING privilege และไม่ใช่ best practice

**Fix:** เปลี่ยนเป็น `c##dbzuser` ที่สร้างมาเฉพาะสำหรับ Debezium + เปลี่ยน topic prefix เป็น `cdc-oracle` ไม่ให้ชนกับ PostgreSQL connector

```diff
- DEBEZIUM_SOURCE_DATABASE_USER=SYSTEM
- DEBEZIUM_SOURCE_DATABASE_PASSWORD=password
- DEBEZIUM_SOURCE_TOPIC_PREFIX=cdc
+ DEBEZIUM_SOURCE_DATABASE_USER=c##dbzuser
+ DEBEZIUM_SOURCE_DATABASE_PASSWORD=dbzpassword
+ DEBEZIUM_SOURCE_TOPIC_PREFIX=cdc-oracle
```

---

## Result

Debezium Oracle CDC ทำงานได้สมบูรณ์:

- Snapshot completed
- LogMiner streaming active
- Test INSERT `('Test User', 'test@example.com')` ถูก capture ไปยัง Redis stream `cdc-oracle.C__APPUSER.CUSTOMERS`
