---
title: "데이터 이관 (Data pump, DB LINK)"
excerpt: "아이티윌 0729_(1)"
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-07-29T21:08
---

# 데이터 이관 (Data pump, DB LINK)

📍 xe db(11g) 특정 유저 데이터 →  ora19c(19c) 특정 유저에 이관

▶️ XE DB

```sql
-- 데이터베이스 및 버전 확인
SELECT * FROM v$database;
SELECT * FROM v$version;
```

▶️ 유저 생성

```sql
DROP USER insa CASCADE;

CREATE USER insa
IDENTIFIED BY oracle
DEFAULT TABLESPACE USERS
QUOTA UNLIMITED ON USERS;

GRANT CONNECT, RESOURCE TO insa;
```

▶️ 테이블 생성

```sql
CREATE TABLE insa.emp TABLESPACE USERS AS SELECT * FROM hr.employees;
CREATE TABLE insa.dept TABLESPACE USERS AS SELECT * FROM hr.departments;
```

▶️ C드라이브 밑에 물리적 디렉토리 생성 후 논리적 디렉토리 생성

```sql
-- 논리적 디렉토리 생성
CREATE DIRECTORY xe_pump AS 'C:\xe_pump';

SELECT * FROM dba_directories;

-- 권한 
GRANT read, write ON DIRECTORY xe_pump TO hr,insa;
```

▶️ CMD 관리자 권한 실행 후 datapump export

```sql
C:\Windows\System32>expdp system/oracle schemas=insa directory=xe_pump dumpfile=insa_schema.dmp

Export: Release 11.2.0.2.0 - Production on 화 7월 29 10:19:06 2025

Copyright (c) 1982, 2009, Oracle and/or its affiliates.  All rights reserved.

Connected to: Oracle Database 11g Express Edition Release 11.2.0.2.0 - 64bit Production
Starting "SYSTEM"."SYS_EXPORT_SCHEMA_01":  system/******** schemas=insa directory=xe_pump dumpfile=insa_schema.dmp
Estimate in progress using BLOCKS method...
Processing object type SCHEMA_EXPORT/TABLE/TABLE_DATA
Total estimation using BLOCKS method: 128 KB
Processing object type SCHEMA_EXPORT/USER
Processing object type SCHEMA_EXPORT/SYSTEM_GRANT
Processing object type SCHEMA_EXPORT/ROLE_GRANT
Processing object type SCHEMA_EXPORT/DEFAULT_ROLE
Processing object type SCHEMA_EXPORT/TABLESPACE_QUOTA
Processing object type SCHEMA_EXPORT/PRE_SCHEMA/PROCACT_SCHEMA
Processing object type SCHEMA_EXPORT/TABLE/TABLE
. . exported "INSA"."DEPT"                                   7 KB      27 rows
. . exported "INSA"."EMP"                                16.80 KB     107 rows
Master table "SYSTEM"."SYS_EXPORT_SCHEMA_01" successfully loaded/unloaded
******************************************************************************
Dump file set for SYSTEM.SYS_EXPORT_SCHEMA_01 is:
  C:\XE_PUMP\INSA_SCHEMA.DMP
Job "SYSTEM"."SYS_EXPORT_SCHEMA_01" successfully completed at 10:19:20
```

▶️ insa_schema.dmp 를 차세대 DB에 이관하는 과정

![image.png](/assets/20250729/1.png)

▶️ insa_schema.dmp → james 유저 데이터 이관

```sql
-- ora19c db
select * from v$database;
select * from v$version;
select * from dba_directories;

impdp system/oracle directory=pump_dir dumpfile=INSA_SCHEMA.DMP tables=insa.emp,insa.dept remap_schema='insa':'james'

-- 권한 문제로 오류났다면 
ALTER USER james DEFAULT TABLESPACE users;
GRANT UNLIMITED TABLESPACE TO james;
-- EMP, DEPT 미리 지워놓고 다시 시도하자.

impdp system/oracle directory=pump_dir dumpfile=INSA_SCHEMA.DMP tables=insa.emp,insa.dept remap_schema='insa':'james'

Connected to: Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Master table "SYSTEM"."SYS_IMPORT_TABLE_01" successfully loaded/unloaded
Starting "SYSTEM"."SYS_IMPORT_TABLE_01":  system/******** directory=pump_dir dumpfile=INSA_SCHEMA.DMP tables=insa.emp,insa.dept remap_schema=insa:james
Processing object type SCHEMA_EXPORT/TABLE/TABLE
Processing object type SCHEMA_EXPORT/TABLE/TABLE_DATA
. . imported "JAMES"."DEPT"                                  7 KB      27 rows
. . imported "JAMES"."EMP"                               16.80 KB     107 rows
Job "SYSTEM"."SYS_IMPORT_TABLE_01" successfully completed at Mon Jul 28 19:59:21 2025 elapsed 0 00:00:03
```

```sql
DROP TABLE EMP PURGE;
DROP TABLE DEPT PURGE;

-- DB LINK를 이용해 차세대 DB에서 레거시 DB(XE DB)를 특정한 유저의 테이블을
-- access 하고싶다. 차세대 DB에 DB LINK를 만들자. 

-- ora19c의 sys 세션
select * from dba_db_links;

/*만약 이미 존재한다면*/
drop public database link xe_link;

/* DB Link 만들기 */
create public database link xe_link connect to system identified by oracle using 'xe_t';
/* 차세대 DB (ora19c) 에서 레거시 DB 접근 */
select * from insa.emp@xe_link; 

/* ora19c sys 세션 */
expdp system/oracle directory=pump_dir network_link=xe_link schemas=insa dumpfile=xe_insa.dmp

impdp system/oracle directory=pump_dir dumpfile=xe_insa.dmp tables=insa.emp,insa.dept remap_schema='insa':'james'
```