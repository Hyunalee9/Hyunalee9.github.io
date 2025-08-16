---
title: "[72일차] Flashback Query "
excerpt: "아이티윌 0806_(2)"
categories:
      - ORACLE19c
tags:
      - ORACLE19c
      - TIL
last-modified-at: 2025-08-06T21:08
---

# Flashback Query

📍 데이터 잘못 삭제했다. 복구 방법은? 

▶️ logminer

▶️ time base recovery

▶️ flashback query (10g)

- commit된 데이터를 과거의 특정 시점에 존재했던 상태로 볼 수 있다. point-in-time의 모든 데이터를 query할 수 있다.
- select 명령을 as of 절과 함께 사용하면 시간 또는 scn을 통해 과거 시점을 참조할 수 있다.
- 단 undo_retention까지만 보장한다.

```sql
SYS@ora19c> show parameter undo_retention

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
undo_retention                       integer     1800 (초단위, 기본값)

```

- undo_retention을 보장하려면
    - undo free 공간이 있어야 한다.
    - 무조건 보장해야 한다면
    
    ```sql
    -- 언두 공간이 없을 경우 undo retention을 보장하기 위해서 새로운 트랜잭션이 실패될 수 있다.
    alter tablespace undotbs1 retention guarantee;
    ```
    

▶️ 해지

```sql
alter tablespace undotbs1 retention noguarntee;
```

▶️ DML

```sql
select salary from hr.emp where employee_id = 200;
update hr.emp set salary = salary * 1.2 where employee_id = 200;
```

▶️ flashback

```sql
-- undo_retention
SYS@ora19c> select salary from hr.emp as of timestamp to_timestamp('2025-08-06 12:13:50','yyyy-mm-dd hh24:mi:ss')
 where employee_id = 200;  
 
 -- undo에 가지고 있는 값 

    SALARY
----------
      4400

select salary
from hr.emp as of scn SCN
where employee_id = 200; 
```

📍 flashback version query

- version 절을 사용하여 두 point-in-time 또는 두 scn사이에 존재하는 행의 모든 값을 검색할 수 있다.

```sql
select current_scn, checkpoint_change#, systimestamp from v$database;

/*
CURRENT_SCN CHECKPOINT_CHANGE#
----------- ------------------
SYSTIMESTAMP
---------------------------------------------------------------------------
    5312929            5279202
06-AUG-25 01.46.30.802641 PM +09:00
*/

SYS@ora19c> select employee_id, last_name, salary from hr.emp where employee_id in(110,111);

/*
EMPLOYEE_ID LAST_NAME                     SALARY
----------- ------------------------- ----------
        110 Chen                            8200
        111 Sciarra                         7700
*/

update hr.emp set salary = 1000 where employee_id = 110;
delete from hr.emp where employee_id = 111;
commit;

select versions_xid, employee_id, last_name, salary
from hr.emp versions between timestamp to_timestamp('2025-08-06 13:45:50','yyyy-mm-dd hh24:mi:ss') and to_timestamp('2025-08-06 13:50:00', 'yyyy-mm-dd hh24:mi:ss')
where employee_id in (110,111);

/*

VERSIONS_XID     EMPLOYEE_ID LAST_NAME                     SALARY
---------------- ----------- ------------------------- ----------
09001100F9050000         111 Sciarra                         7700
                         110 Chen                            8200
                         111 Sciarra                         7700
*/

select versions_xid, employee_id, last_name, salary
from hr.emp versions between scn minvalue and maxvalue
where employee_id in (110,111);

/*
VERSIONS_XID     EMPLOYEE_ID LAST_NAME                     SALARY
---------------- ----------- ------------------------- ----------
09001100F9050000         111 Sciarra                         7700
                         110 Chen                            8200
                         111 Sciarra                         7700
*/

select current_scn, checkpoint_change#, systimestamp from v$database;

/*
CURRENT_SCN CHECKPOINT_CHANGE#
----------- ------------------
SYSTIMESTAMP
---------------------------------------------------------------------------
    5314583            5279202
06-AUG-25 01.59.20.247753 PM +09:00
*/

select versions_xid, employee_id, last_name, salary
from hr.emp versions between scn 트랜잭션 시작 전scn and 시작 후 scn
where employee_id in (110,111);

```

📍 DML 너무 여러 건 잘못 했다. → flashback query 는 조금 비효율적

📍 Flashback Table

- 백업으로 복원하지 않고 테이블을 특정 시점으로 recovery할 수 있다.
- 데이터베이스 온라인 상태를 유지한다.
- flashback table 작업을 수행하기 위해 언두 테이블스페이스에서 언두 데이터를 사용한다.
- flashback table 에 대한 행 이동이 활성화 되어야 한다.
- flashback table 고려사항
    - 통계 정보는 flashback 되지 않습니다.
    - 현재 인덱스와 종속 객체 유지된다.
    - 딕셔너리 테이블은 수행할 수 없다.
    - 테이블 구조를 변경했거나 테이블 축소(shrink)작업 이전으로 되돌릴 수 없다.
    - flashback table 수행하는 동안 undo, redo 데이터 발생한다.

▶️  권한 부여

```sql
grant flashback on hr.emp to insa; -- 객체 권한
grant flashback any table to insa; -- 시스템 권한
```

▶️ SCN 조회

```sql
select current_scn, checkpoint_change#, systimestamp from v$database;

/*
CURRENT_SCN CHECKPOINT_CHANGE#
----------- ------------------
SYSTIMESTAMP
---------------------------------------------------------------------------
    5316935            5279202
06-AUG-25 02.14.14.991377 PM +09:00

*/
```

▶️ 삭제

```sql
delete from hr.emp;
commit;
select * from hr.emp;
select * from hr.emp as of timestamp to_timestamp('2025-08-06 14:11:22', 'yyyy-mm-dd hh24:mi:ss');
```

▶️ 복구

```sql
insert into hr.emp
select * from hr.emp as of timestamp to_timestamp('2025-08-06 14:11:22', 'yyyy-mm-dd hh24:mi:ss');

select * from hr.emp;
rollback;
```

▶️ 행 이동 활성화 여부 체크

```sql
select row_movement from dba_tables where owner='HR' and table_name='EMP';

/*
ROW_MOVE
--------
DISABLED
*/
```

▶️ 행 이동 활성화

```sql
alter table hr.emp enable row movement;
select row_movement from dba_tables where owner='HR' and table_name='EMP';

/*

ROW_MOVE
--------
ENABLED

*/
```

▶️ flashback table 수행

```sql
flashback table hr.emp to timestamp to_timestamp('2025-08-06 14:11:22', 'yyyy-mm-dd hh24:mi:ss');
select * from hr.emp;
```

▶️ 비활성화

```sql
alter table hr.emp disable row movement;
```

📍 Flashback Data Archive

- **기록 데이터 저장소**
- fbda 백그라운드 프로세스를 사용하여 flashback data archive 에 대한 활성화되어 있는 테이블의 데이터를 자동으로 추적하고 아카이브한다.
- **flashback archive administer 시스템 권한이 필요**하다.
- **flashback archive 객체 권한이 필요**하다.

```sql
select a.file#, b.name tbs_name, a.name file_name, a.status, a.checkpoint_change#
from v$datafile a, v$tablespace b
where a.ts# = b.ts#;
```

📍 Flashback Data Archive를 저장할 테이블스페이스 생성

```sql
SYS@ora19c> create tablespace fda_tbs datafile '/u01/app/oracle/oradata/ORA19C/fda_tbs01.dbf' size 10m autoextend on next 1m;

Tablespace created.

SYS@ora19c> select a.file#, b.name tbs_name, a.name file_name, a.status, a.checkpoint_change#
  from v$datafile a, v$tablespace b
  where a.ts# = b.ts#;

    FILE#  TBS_NAME   FILE_NAME                                               STATUS   CHECKPOINT_CHANGE#
---------- ---------- ------------------------------------------------------ -------- -----------------
         1 SYSTEM     /u01/app/oracle/oradata/ORA19C/system01.dbf             SYSTEM   5937124
         3 SYSAUX     /u01/app/oracle/oradata/ORA19C/sysaux01.dbf             ONLINE   5937124
         4 UNDOTBS1   /u01/app/oracle/oradata/ORA19C/undotbs01.dbf            ONLINE   5937124
         7 USERS      /u01/app/oracle/oradata/ORA19C/users01.dbf              ONLINE   5938726
         5 FDA_TBS    /u01/app/oracle/oradata/ORA19C/fda_tbs01.dbf            ONLINE   5968024
```

📍 Flashback Data Archive 생성

```sql
create flashback archive fda1 tablespace fda_tbs quota 10m retention 1 year;
select * from dba_flashback_archive;
```

📍 Flashback Data Archive 테이블 활성화

```sql
alter table hr.emp flashback archive fda1;
select * from dba_flashback_archive_tables;
```

📍 

```sql
select employee_id, salary from hr.emp where department_id = 20;
update hr.emp set salary = 3000 where department_id = 20;
commit;
select employee_id, salary from hr.emp where department_id = 20;

select employee_id, salary
from hr.emp as of timestamp(systimestamp - interval '1' minute)
where department_id = 20;

EMPLOYEE_ID     SALARY
----------- ----------
        201       3000
        202       3000

SYS@ora19c> ! ps -ef | grep fbda
oracle    8765     1  0 15:00 ?        00:00:00 ora_fbda_ora19c
oracle   13260 25382  0 15:11 pts/0    00:00:00 /bin/bash -c  ps -ef | grep fbda
oracle   13262 13260  0 15:11 pts/0    00:00:00 grep fbda

delete from hr.emp;
commit;
select * from hr.emp as of timestamp(systimestamp - interval '2' minute);
insert into hr.emp select * from hr.emp as of timestamp(systimestamp - interval '2' minute);
commit;
```

▶️ Flashback Data Archive 대상 테이블 비활성화

```sql
alter table hr.emp no flashback archive;
select * from dba_flashback_archive_tables;
```

▶️ 

```sql
select * from hr.emp as of timestamp(systimestamp - interval '10' minute);
select * from dba_flashback_archive;
```

▶️ Flashback Data Archive 보존 시간 변경

```sql
alter flashback archive fda1 modify retention 2 year;
select * from dba_flashback_archive;

```

▶️

```sql
select * from dba_flashback_archive_ts;
```

▶️ flashback data archive 저장 사이즈 변경

```sql
alter flashback archive fda1 modify tablespace fda_tbs quota 20m;
```

▶️ 확인

```sql
select * from dba_flashback_archive_ts;

/*
FLASHBACK_ARCHIVE_NAME
--------------------------------------------------------------------------------
FLASHBACK_ARCHIVE# TABLESPACE_NAME
------------------ ------------------------------
QUOTA_IN_MB
----------------------------------------
FDA1
                 1 FDA_TBS
*/
```

📍 Flashback Data Archive 데이터 지우기

```sql
alter flashback archive fda1 purge before timestamp(systimestamp - interval '1' day);
```

📍 Flashback Data Archive 삭제

```sql
drop flashback archive fda1;
drop tablespace fda_tbs including contents and datafiles;
```

## Flashback Database

▶️ truncate 잘못 했다.

▶️ time base recovery

- 데이터베이스에 대해 되감기 버튼처럼 작동한다.
- 이전 데이터로 되감기 하기 위해서 리두, 아카이브 정보를 이용한다.
- 아카이브 모드에서 수행된다.
- rvwr(recovery writer) 백그라운드 프로세스에서 수행된다.

📍 **flashback database 를 사용할 수 없는 경우**

- control file이 복원되었거나 재생성된 경우
- 테이블스페이스 삭제된 경우
- 데이터 파일 크기가 감소된 경우

```sql
select log_mode from v$database;
```

📍 flashback database를 사용할 수 있는 시간을 분단위로 설정

```sql
show parameter db_flashback_retention_target
```

📍 

```sql
alter system set db_flashback_retention_target = 2880 scope=both;
show parameter db_flashback_retention_target

/*

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_flashback_retention_target        integer     2880

*/
```

📍 Flashback Database 활성화

```sql
select flashback_on from v$database;

/*
FLASHBACK_ON
------------------
NO
*/

alter database flashback on;
select flashback_on from v$database;

-- flashback buffer가 shared pool 메모리에 생성
select * from v$sgastat where name like '%flashback%';

/*
POOL           NAME                            BYTES     CON_ID
-------------- -------------------------- ---------- ----------
shared pool    flashback_marker_cache_si        9200          0
shared pool    flashback generation buff     3981120          0
*/

select * from v$flash_recovery_area_usage;

SYS@ora19c> select systimestamp from dual;

SYSTIMESTAMP
---------------------------------------------------------------------------
06-AUG-25 04.19.51.680921 PM +09:00

--restore point 찍기
SYS@ora19c> select * from v$restore_point;

/*
       SCN DATABASE_INCARNATION# GUA STORAGE_SIZE
---------- --------------------- --- ------------
TIME
---------------------------------------------------------------------------
RESTORE_POINT_TIME                                                          PRE
--------------------------------------------------------------------------- ---
NAME
--------------------------------------------------------------------------------
PDB CLE PDB_INCARNATION# REP     CON_ID
--- --- ---------------- --- ----------
   5337706                     6 NO             0
06-AUG-25 04.20.28.000000000 PM
                                                                            NO

       SCN DATABASE_INCARNATION# GUA STORAGE_SIZE
---------- --------------------- --- ------------
TIME
---------------------------------------------------------------------------
RESTORE_POINT_TIME                                                          PRE
--------------------------------------------------------------------------- ---
NAME
--------------------------------------------------------------------------------
PDB CLE PDB_INCARNATION# REP     CON_ID
--- --- ---------------- --- ----------
BEFORE_HR_TRUNC
NO  NO                 0 NO           0

*/
```

▶️ 

```sql
truncate table hr.emp;
select * from hr.emp;
shutdown immediate;
startup
flashback database to restore point before_hr_trunc;
alter database open read only;
select count(*) from hr.emp;
shutdown immediate;
startup mount;
alter database open resetlogs;
```

▶️ restore point 삭제

```sql
drop restore point before_hr_trunc;
select * from v$restore_point;
```

▶️ 

```sql
flashback database to time = "to_date('2025-08-06 16.20.29','yyyy-mm-dd hh24:mi:ss')";
flashback database to timestamp(sysdate-1/24);
flashback database to scn = 5981527;
select * from v$restore_point;
select * from v$flashback_database_log;
select flashback on from v$database;
alter database flashback off;
```