---
title: "[51일차] undo, bigfile, 유저 관리"
excerpt: "아이티윌 0704 "
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-07-04T23:00
---


undo segment 는 트랜잭션 별로 할당된다. 

기본적으로 10개씩 할당되는데, 모두 사용하면서 11번째로 새로운 DML 시작한다면 또 10개를 할당받는다. 여유 공간을 만들어두는 것이 좋다. 

undo는 shrink 안된다. 그러므로 새로운 undo tablespace를 만들어서 이관작업

▶️ **모든 데이터 파일 이관 작업**

📍 **준비**

```sql
-- 시스템 테이블스페이스 내 불필요 테이블스페이스 삭제 

SQL> select name from v$datafile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/system01.dbf
/u01/app/oracle/oradata/ORA19C/sysaux01.dbf
/u01/app/oracle/oradata/ORA19C/undotbs01.dbf
/u01/app/oracle/oradata/ORA19C/insa_tab01.dbf
/u01/app/oracle/oradata/ORA19C/users01.dbf

SQL> select name from v$tempfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/temp01.dbf

SQL> select tablespace_name from dba_tablespaces;

TABLESPACE_NAME
------------------------------
SYSTEM
SYSAUX
UNDOTBS1
TEMP
USERS
INSA_TAB

6 rows selected.

SQL> show parameter undo

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
temp_undo_enabled                    boolean     FALSE
undo_management                      string      AUTO
undo_retention                       integer     1800
undo_tablespace                      string      UNDOTBS1

SQL> DROP TABLESPACE insa_tab INCLUDING CONTENTS AND DATAFILES;

Tablespace dropped.

SQL> SELECT tablespace_name FROM dba_tablespaces;

TABLESPACE_NAME
------------------------------
SYSTEM
SYSAUX
UNDOTBS1
TEMP
USERS

SQL> select name from v$datafile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/system01.dbf
SP2-0734: /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
/u01/app/oracle/oradata/ORA19C/undotbs01.dbf
/u01/app/oracle/oradata/ORA19C/insa_tab01.dbf
/u01/app/oracle/oradata/ORA19C/users01.dbf

SQL> select name from v$tempfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/temp01.dbf

SQL> select tablespace_name from dba_tablespaces;

TABLESPACE_NAME
------------------------------
SYSTEM
SYSAUX
UNDOTBS1
TEMP
USERS
INSA_TAB

6 rows selected.

SQL> show parameter undo

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
temp_undo_enabled                    boolean     FALSE
undo_management                      string      AUTO
undo_retention                       integer     1800
undo_tablespace                      string      UNDOTBS1

SQL> DROP TABLESPACE insa_tab INCLUDING CONTENTS AND DATAFILES;

Tablespace dropped.

SQL> SELECT tablespace_name FROM dba_tablespaces;

TABLESPACE_NAME
------------------------------
SYSTEM
SYSAUX
UNDOTBS1
TEMP
USERS

-- 물리적으로 다른 위치에다가 이동
-- 디렉토리 만들어 작업.

SQL> !
[oracle@oracle ~]$ pwd
/home/oracle
[oracle@oracle ~]$ mkdir userdata
[oracle@oracle ~]$ cd userdata/

-- 미리 만드는 작업.
select 'mv -v '|| name || '/home/oracle/userdata/' from v$datafile 
union all
select 'mv -v '|| name || '/home/oracle/userdata/' from v$tempfile; 

'MV-V'||NAME||'/HOME/ORACLE/USERDATA/'
--------------------------------------------------------------------------------
mv -v /u01/app/oracle/oradata/ORA19C/system01.dbf/home/oracle/userdata/
mv -v /u01/app/oracle/oradata/ORA19C/sysaux01.dbf/home/oracle/userdata/
mv -v /u01/app/oracle/oradata/ORA19C/undotbs01.dbf/home/oracle/userdata/
mv -v /u01/app/oracle/oradata/ORA19C/users01.dbf/home/oracle/userdata/
mv -v /u01/app/oracle/oradata/ORA19C/temp01.dbf/home/oracle/userdata/

```

▶️ 대략적인 흐름

1️⃣ 오라클 데이터베이스를 정상적인 종료

```sql
shutdown immediate
!
```

2️⃣ 모든 데이터 파일을 새로운 디스크 위치로 이동

```sql
mv -v /u01/app/oracle/oradata/ORA19C/system01.dbf /home/oracle/userdata/
mv -v /u01/app/oracle/oradata/ORA19C/sysaux01.dbf /home/oracle/userdata/
mv -v /u01/app/oracle/oradata/ORA19C/undotbs01.dbf /home/oracle/userdata/
mv -v /u01/app/oracle/oradata/ORA19C/users01.dbf /home/oracle/userdata/
mv -v /u01/app/oracle/oradata/ORA19C/temp01.dbf /home/oracle/userdata/
```

3️⃣ 오라클 데이터베이스를 mount까지만열기

```sql
startup mount
select status from v$instance;
```

4️⃣ 기존 데이터 파일을 새로운 데이터 파일로 수정

```sql
alter database rename file '이전파일' to '새로운파일';
alter database rename file '/u01/app/oracle/oradata/ORA19C/system01.dbf' to '/home/oracle/userdata/system01.dbf';
alter database rename file '/u01/app/oracle/oradata/ORA19C/sysaux01.dbf' to '/home/oracle/userdata/sysaux01.dbf';
alter database rename file '/u01/app/oracle/oradata/ORA19C/undotbs01.dbf' to '/home/oracle/userdata/undotbs01.dbf';
alter database rename file '/u01/app/oracle/oradata/ORA19C/users01.dbf' to '/home/oracle/userdata/users01.dbf';
alter database rename file '/u01/app/oracle/oradata/ORA19C/temp01.dbf' to '/home/oracle/userdata/temp01.dbf';

select name from v$datafile;
select name from v$tempfile;
```

![image.png](/assets/20250704/1.png)

5️⃣ 오라클 데이터베이스 open

```sql
alter database open;   -- 컨트롤 파일 수정 

select status from v$instance;

select name from v$datafile;
select name from v$tempfile;
```

![image.png](/assets/20250704/1.png)

![image.png](/assets/20250704/2.png)

```sql
-- 딕셔너리 체크
select file_name from dba_data_files;

select count(*) from hr.employees;
```

![image.png](/assets/20250704/1.png)

▶️ 이렇게 하고 다시 원래 위치로 되돌리는 작업함

```sql
shutdown immediate
!

mv -v /home/oracle/userdata/system01.dbf /u01/app/oracle/oradata/ORA19C/
mv -v /home/oracle/userdata/sysaux01.dbf /u01/app/oracle/oradata/ORA19C/
mv -v /home/oracle/userdata/undotbs01.dbf /u01/app/oracle/oradata/ORA19C/
mv -v /home/oracle/userdata/users01.dbf /u01/app/oracle/oradata/ORA19C/
mv -v /home/oracle/userdata/temp01.dbf /u01/app/oracle/oradata/ORA19C/

startup mount
select status from v$instance;

alter database rename file '/home/oracle/userdata/system01.dbf' to '/u01/app/oracle/oradata/ORA19C/system01.dbf';
alter database rename file '/home/oracle/userdata/sysaux01.dbf' to '/u01/app/oracle/oradata/ORA19C/sysaux01.dbf';
alter database rename file '/home/oracle/userdata/undotbs01.dbf' to '/u01/app/oracle/oradata/ORA19C/undotbs01.dbf';
alter database rename file '/home/oracle/userdata/users01.dbf' to '/u01/app/oracle/oradata/ORA19C/users01.dbf';
alter database rename file '/home/oracle/userdata/temp01.dbf' to '/u01/app/oracle/oradata/ORA19C/temp01.dbf';

select name from v$datafile;
select name from v$tempfile;

alter database open;   -- 컨트롤 파일 수정 

select status from v$instance;

select name from v$datafile;
select name from v$tempfile;
```

▶️ control file 이중화 → 단일화

```sql
show parameter control_files
select name from v$controlfile;
```

1️⃣ 초기 파라미터 파일 체크, 수정

```sql
show parameter spfile
```

📍 control file 은 static parameter 이다.

```sql
alter system set control_files = '/u01/app/oracle/oradata/ORA19C/control01.ctl' scope=spfile;
```

2️⃣ 데이터베이스 정상적인 종료

```sql
shutdown immediate
```

3️⃣ 데이터베이스 open

```sql
startup
!
```

4️⃣ 의미없는 컨트롤 파일 삭제

```sql
rm /u01/app/oracle/fast_recovery_area/ORA19C/control02.ctl

show parameter control_files
```

![image.png](/assets/20250704/3.png)

📍 parameter 

▶️ 오라클 블록

- 2K, 4K, 8K, 16K, 32K
- 기본 블록 크기는 8K
- 기본 블록 크기는 데이터베이스 생성시 결정한다.

```sql
show parameter db_block_size

NAME          TYPE    VALUE 
------------- ------- ----- 
db_block_size integer 8192  
NAME          TYPE    VALUE 
------------- ------- ----- 
db_block_size integer 8192  

select * from dba_tablespaces;
```

![image.png](/assets/20250704/4.png)

▶️ 4K 블록을 생성하려면

1️⃣ 메모리 확보

📍 non standard block

```sql
db_2k_cache_size
db_4k_cache_size
db_8k_cache_size
db_16k_cache_size
db_32k_cache_size

show parameter db_4K_cache_size

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_4k_cache_size                     big integer 0

select * from v$sgastat;
```

![image.png](/assets/20250704/5.png)

📍 메모리 체크

```sql
-- sga 메모리 free 확인
SELECT sum(bytes)/1024/1024 FROM v$sgastat WHERE name = 'free memory';  
```

📍메모리 확보

```sql
ALTER SYSTEM SET db_4k_cache_size = 12m;
show parameter db_4K_cache_size
```

📍 Granule Size : 메모리에 대한 단편화를 막기 위함 (불필요하게 남는 메모리 방지)

```sql
CREATE TABLESPACE oltp_tbs
DATAFILE '/u01/app/oracle/oradata/ORA19C/oltp_tbs01.dbf' SIZE 5M AUTOEXTEND ON
BLOCKSIZE 4K
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M
SEGMENT SPACE MANAGEMENT AUTO;
```

```sql
CREATE TABLE hr.oltp_emp
TABLESPACE oltp_tbs
AS SELECT * FROM hr.employees;
```

```sql
SELECT * FROM dba_segments WHERE segment_name = 'OLTP_EMP';
SELECT * FROM dba_extents WHERE segment_name = 'OLTP_EMP';

DROP TABLESPACE oltp_tbs INCLUDING CONTENTS AND DATAFILES;
```

📍 BIGFILE 테이블스페이스

- 단일 파일만 포함한다. (기존 테이블스페이스는 small file)

```sql
select * from dba_tablespaces;
```

![image.png](/assets/20250704/6.png)

- 최대 40억개의 블록을 포함할 수 있다.
- 최대 파일의 크기는 32TB(8k block), 128TB(32k block)

```sql
CREATE BIGFILE TABLESPACE big_tbs
DATAFILE '/u01/app/oracle/oradata/ORA19C/big_tbs.dbf' SIZE 1G AUTOEXTEND ON NEXT 2M MAXSIZE 2G
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M
SEGMENT SPACE MANAGEMENT AUTO;
```

오라클의 엑사 DB

```sql
select * from dba_tablespaces;
select * from dba_data_files;

DROP TABLESPACE big_tbs INCLUDING CONTENTS AND DATAFILES;
```

📍 user 관리

```sql
CREATE USER ora1 identified by oracle;
```

📍 데이터베이스 레벨에 기본값으로 설정된 정보 확인

```sql
SELECT * FROM database_properties;
```

![image.png](/assets/20250704/7.png)

📍temp 테이블스페이스 생성

```sql
CREATE TABLESPACE user_tbs
DATAFILE '/u01/app/oracle/oradata/ORA19C/user_tbs01.dbf' SIZE 5M AUTOEXTEND ON
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M
SEGMENT SPACE MANAGEMENT AUTO;

CREATE TEMPORARY TABLESPACE user_temp   -- 오류 난다.
DATAFILE '/u01/app/oracle/oradata/ORA19C/user_temp01.dbf' SIZE 5M AUTOEXTEND ON
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M
SEGMENT SPACE MANAGEMENT AUTO;

CREATE TEMPORARY TABLESPACE user_temp   -- 오류 난다.
TEMPFILE '/u01/app/oracle/oradata/ORA19C/user_temp01.dbf' SIZE 5M AUTOEXTEND ON
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M
SEGMENT SPACE MANAGEMENT AUTO;   -> TEMP 파일의 세그먼트 공간 관리는 MANUAL로  

CREATE TEMPORARY TABLESPACE user_temp -- 정상 작동   
TEMPFILE '/u01/app/oracle/oradata/ORA19C/user_temp01.dbf' SIZE 5M AUTOEXTEND ON
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M
SEGMENT SPACE MANAGEMENT MANUAL;

-- 확인
select * from dba_tablespaces;
select * from dba_data_files;
select * from dba_temp_files;
```

📍 default tablespace 수정

```sql
ALTER DATABASE default tablespace user_tbs;
```

📍 default temporary tablespace 수정

```sql
ALTER DATABASE default temporary tablespace user_temp;
```

📍 유저 생성

```sql
CREATE user ora2
identified by oracle
default tablespace users
temporary tablespace temp;

CREATE user ora3
identified by oracle;

-- 비교해보기
select * from dba_users where username in ('ORA1' , 'ORA2', 'ORA3');
```

![image.png](/assets/20250704/8.png)

📍 default tablespace로 지정된 테이블 스페이스는 삭제할 수 없다.

```sql
DROP tablespace user_tbs including contents and datafiles; -- 오류남
DROP tablespace user_temp including contents and datafiles; -- 오류남
```

📍 default tablespace 수정

```sql
ALTER database default tablespace users;
```

📍 default temporary tablespace 수정

```sql
ALTER database default temporary tablespace users;
```

📍 다시 삭제 시도

```sql
DROP tablespace user_tbs including contents and datafiles;
DROP tablespace user_temp including contents and datafiles;
```

▶️ default tablespace 지정하는 이유 : 지정하지 않으면 system tablespace가 그 자리를 차지하는데

user$ 정보도 들어감. 딕셔너리 정보만 들어가야되는데. 

```sql
-- 확인
select * from dba_users where username in ('ORA1' , 'ORA2', 'ORA3');
```

![image.png](/assets/20250704/9.png)

▶️ 새로 생성한 유저에게 권한 부여

```sql
GRANT CREATE SESSION, CREATE TABLE TO ora1, ora2, ora3;
```

```sql
alter user ora1 quota 1m on users;
alter user ora2 quota 1m on users;
alter user ora3 quota 1m on users;

SELECT * FROM dba_ts_quotas WHERE username IN ('ORA1', 'ORA2' , 'ORA3');
```

📍 putty 

```sql

[oracle@oracle ~]$ sqlplus ora1/oracle

SQL*Plus: Release 19.0.0.0.0 - Production on Thu Jul 3 11:56:52 2025
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.

Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> select * from session_privs;

PRIVILEGE
----------------------------------------
CREATE TABLE
CREATE SESSION

SQL> select * from user_ts_quotas;

TABLESPACE_NAME                     BYTES  MAX_BYTES     BLOCKS MAX_BLOCKS DRO
------------------------------ ---------- ---------- ---------- ---------- ---
USERS                                   0    1048576          0        128 NO

SQL> select default_tablespace from use_users;
select default_tablespace from use_users
                               *
ERROR at line 1:
ORA-00942: table or view does not exist

SQL> select default_tablespace from user_users;

DEFAULT_TABLESPACE
------------------------------
USERS

SQL> create table test(id number, name varchar2(30));

Table created.

SQL> create table new(id number) tablespace users;

```