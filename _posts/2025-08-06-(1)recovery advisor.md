---
title: "[72일차] Recovery Advisor "
excerpt: "아이티윌 0806_(1)"
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-08-06T21:08
---

# Recovery Advisor

```sql
delete backup;
backup database;
```

📍 spfile, pfile 손상

```sql
select dbid, name from v$database;  --dbid 는 따로 적어놓는 것이 좋다.

/*

      DBID NAME
---------- ---------
1258537005 ORA19C

*/

shutdown immediate;
exit
cd $ORACLE_HOME/dbs
[oracle@oracle dbs]$ mv initora19c.ora initora19c.bak
[oracle@oracle dbs]$ mv spfileora19c.ora spfileora19c.bak
[oracle@oracle dbs]$ ls

-- pfile, spfile 손상되어 에러
SYS@ora19c> startup
ORA-01078: failure in processing system parameters
LRM-00109: could not open parameter file '/u01/app/oracle/product/19.3.0/dbhome_1/dbs/initora19c.ora'

startup nomount;  -- 복구를 위한 메모리 생성

using target database control file instead of recovery catalog
INSTANCE_NAME    STATUS
---------------- ------------
ora19c           STARTED

RMAN> restore spfile from autobackup;

Starting restore at 06-AUG-25
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=20 device type=DISK

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of restore command at 08/06/2025 09:58:50
RMAN-06495: must explicitly specify DBID with SET DBID command

RMAN> SET DBID 1258537005
restore spfile from autobackup; -- 오류난다. 수동으로 해야한다.
restore spfile from '/u01/app/oracle/fast_recovery_area/ORA19C/autobackup/2025_08_06/o1_mf_s_1208425856_n959qjwb_.bkp';
exit

sqlplus / as sysdba
select status from v$instance;
shutdown abort
startup

show parameter spfile;

```

▶️ 2 Recovery Advisor : Advise 진단 받아볼 것 -rac에서는 안됨. 불완전한 복구 안됨.

```sql
shutdown immediate
! rm /u01/app/oracle/oradata/ORA19C/users01.dbf
! ls /u01/app/oracle/oradata/ORA19C/users01.dbf
startup
exit
rman target / 
select status from v$instance;
list failure 
list failure 82 detail;
advise failure;

/*

Database Role: PRIMARY

List of Database Failures
=========================

Failure ID Priority Status    Time Detected Summary
---------- -------- --------- ------------- -------
82         HIGH     OPEN      31-JUL-25     One or more non-system datafiles are                                    missing
  Impact: See impact for individual child failures
  List of child failures for parent failure ID 82
  Failure ID Priority Status    Time Detected Summary
  ---------- -------- --------- ------------- -------
  1585       HIGH     OPEN      01-AUG-25     Datafile 7: '/u01/app/oracle/orada                                   ta/ORA19C/users01.dbf' is missing
    Impact: Some objects in tablespace USERS might be unavailable

analyzing automatic repair options; this may take some time
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=259 device type=DISK
analyzing automatic repair options complete

Mandatory Manual Actions
========================
no manual actions available

Optional Manual Actions
=======================
1. If file /u01/app/oracle/oradata/ORA19C/users01.dbf was unintentionally rename                                   d or moved, restore it

Automated Repair Options
========================
Option Repair Description
------ ------------------
1      Restore and recover datafile 7
  Strategy: The repair includes complete media recovery with no data loss
  Repair script: /u01/app/oracle/diag/rdbms/ora19c/ora19c/hm/reco_2414118529.hm

*/

repair failure preview;

/*
Strategy: The repair includes complete media recovery with no data loss
Repair script: /u01/app/oracle/diag/rdbms/ora19c/ora19c/hm/reco_2414118529.hm

contents of repair script:
   # restore and recover datafile
   restore ( datafile 7 );
   recover datafile 7;
   sql 'alter database datafile 7 online';

*/

-- 나 대신 해줘 
RMAN> repair failure;

```

- **LIST FAILURE**: ADR에 기록된 장애 목록 표시.
- **ADVISE FAILURE**: 발견된 장애에 대한 수리 방안 제안.
- **REPAIR FAILURE**: 제안된 수리 방안을 실행.
- **CHANGE FAILURE**: 장애의 속성(우선순위, 설명 등) 변경.
- **CLOSE FAILURE**: 복구된 장애를 종료 상태로 표시.

📍 블록 손상(Block corruption)

- 손상된 데이터 블록은 인식할 필요 없는 오라클 형식이거나 해당 내용이 내부적으로 일치하지 않는 블록
- 블록 읽거나 쓸 때마다 일관성 검사 수행
    - 블록 버전
    - 블록 DBA값과 비교되는 캐시의 DBA
    - 블록 checksum
- 손상된 블록
    - media 손상
    - 논리적, 소프트웨어 손상

```sql
drop table hr.emp purge;
create table hr.emp tablespace users as select * from hr.employees;
select file_id, block_id from dba_extents where owner = 'HR' and segment_name = 'EMP';

/*

SYS@ora19c> select file_id, block_id from dba_extents where owner = 'HR' and segment_name = 'EMP';

   FILE_ID   BLOCK_ID
---------- ----------
         7        376

*/

select f.tablespace_name, f.file_name, count(*)
from dba_extents e, dba_data_files f
where e.file_id = f.file_id
  and e.segment_name = 'EMP'
  and e.owner = 'HR'
group by f.tablespace_name, f.file_name;

/*
TABLESPACE_NAME
------------------------------
FILE_NAME
--------------------------------------------------------------------------------
  COUNT(*)
----------
USERS
/u01/app/oracle/oradata/ORA19C/users01.dbf

/*
EMPLOYEE_ID ROWID              DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID)
----------- ------------------ ------------------------------------
        100 AAASTHAAHAAAAF7AAA                                  379

*/

*/
-- bms_rowid.rowid_block_number(rowid): rowid에서 block 번호만 뽑아내는 프로그램 
select employee_id, rowid, dbms_rowid.rowid_block_number(rowid) from hr.emp;

-- block 단위로 
vi a.txt
oracle engineer
```

📍 dd

- 블록 단위로 파일을 복사하거나 파일 변환을 할 수 있는 명령

```sql
[oracle@oracle ~]$ dd if=a.txt of=b.txt
0+1 records in
0+1 records out
16 bytes (16 B) copied, 0.00042582 s, 37.6 kB/s

-- 손상 
--/dev/zero => 계속 0을 출력하도
-- bs = 8192 블록 크기 8192바이트(8KB) 단위로 읽고 씀
dd if=/dev/zero of=/u01/app/oracle/oradata/ORA19C/users01.dbf bs=8192 seek=379 count=2 conv=notrunc

-- data buffer cache flush
alter system flush buffer_cache;
alter system flush buffer_cache;

select count(*) from hr.emp;  -- 에러

-- alert.log => block corrupted 
```

📍 dbverify utility

- 오라클 7버전
- 데이터 블록을 점검하는 유틸리티

📍 데이터 파일에 대해서 검사

```sql
dbv userid=system/oracle file=/u01/app/oracle/oradata/ORA19C/users01.dbf blocksize=8192

/*

[oracle@oracle ~]$ dbv userid=system/oracle file=/u01/app/oracle/oradata/ORA19C/users01.dbf blocksize=8192

DBVERIFY: Release 19.0.0.0.0 - Production on Wed Aug 6 11:06:12 2025

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

DBVERIFY - Verification starting : FILE = /u01/app/oracle/oradata/ORA19C/users01.dbf
Page 379 is marked corrupt
Corrupt block relative dba: 0x01c0017b (file 7, block 379)
Completely zero block found during dbv:

Page 380 is marked corrupt
Corrupt block relative dba: 0x01c0017c (file 7, block 380)
Completely zero block found during dbv:

DBVERIFY - Verification complete

Total Pages Examined         : 1280    : 검사한 총 블록의 수
Total Pages Processed (Data) : 676     : 검사한 총 테이블 블록 수
Total Pages Failing   (Data) : 0       
Total Pages Processed (Index): 16      : 검사한 총 인덱스 블록 수
Total Pages Failing   (Index): 0
Total Pages Processed (Other): 424     : 테이블, 인덱스 이와 다른 블록 수
Total Pages Processed (Seg)  : 39      : 특정 segment에 대해서 검사한 블록 수
Total Pages Failing   (Seg)  : 0
Total Pages Empty            : 123     : 비어 있는 블록 수
Total Pages Marked Corrupt   : 2       : 문제가 있어서 corrupt marked된 블록 수 
Total Pages Influx           : 0       : 다른 사용자가 먼저 데이터를 변경하고 있어서 dbv를 하기 위해 다시 읽은 블록 수 
Total Pages Encrypted        : 0       : 암호화된 블록 수 
Highest block SCN            : 5285764 (0.5285764)  : 검사한 블록들 중 가장 최근 SCN번호 

-- pages : 블록 

*/

```

📍 segment에 대해서 검사

```sql
select t.ts#, s.header_file, s.header_block
from v$tablespace t, dba_segments s
where t.name = s.tablespace_name
and s.owner = 'HR'
and s.segment_name = 'EMP';

/*
       TS# HEADER_FILE HEADER_BLOCK
---------- ----------- ------------
         4           7          378

*/

-- 특정 세그먼트에 대해서 검사 
[oracle@oracle ~]$ dbv userid=system/oracle segment_id =4.7.378

DBVERIFY: Release 19.0.0.0.0 - Production on Wed Aug 6 11:14:39 2025

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

DBVERIFY - Verification starting : SEGMENT_ID = 4.7.378
Page 379 is marked corrupt
Corrupt block relative dba: 0x01c0017b (file 7, block 379)
Completely zero block found during dbv:

Page 380 is marked corrupt
Corrupt block relative dba: 0x01c0017c (file 7, block 380)
Completely zero block found during dbv:

DBVERIFY - Verification complete

Total Pages Examined         : 8
Total Pages Processed (Data) : 3
Total Pages Failing   (Data) : 0
Total Pages Processed (Index): 0
Total Pages Failing   (Index): 0
Total Pages Processed (Other): 2
Total Pages Processed (Seg)  : 1
Total Pages Failing   (Seg)  : 0
Total Pages Empty            : 0
Total Pages Marked Corrupt   : 2
Total Pages Influx           : 0
Total Pages Encrypted        : 0
Highest block SCN            : 5285764 (0.5285764)

```

📍 rman 유틸리티 

```sql
-- ▶️ 백업을 수행하기 전에 검사하는 것을 권

-- 데이터베이스 레벨로 검사
validate database;

-- 특정한 데이터 파일에 대해서 검사
-- trace 파일에 생성됨
validate datafile 7;

list failure;
list failure 2827 detail;
select * from v$database_block_corruption;

/*
     FILE#     BLOCK#     BLOCKS CORRUPTION_CHANGE# CORRUPTIO     CON_ID
---------- ---------- ---------- ------------------ --------- ----------
         7        379          2                  0 ALL ZERO           0

*/

-- advise 기능을 사용해보기. 
advise failure;
repair failure preview;

Strategy: The repair includes complete media recovery with no data loss
Repair script: /u01/app/oracle/diag/rdbms/ora19c/ora19c/hm/reco_2919577703.hm

contents of repair script:
   # block media recovery for multiple blocks
   recover datafile 7 block 379 to 380;

/*
recover datafile 7 block 379;
recover datafile 7 block 380;

or

recover 
				datafile 7 block 379
				datafile 7 block 380
*/

repair failure;

RMAN> select count(*) from hr.emp;
/*

  COUNT(*)
----------
       107
*/
```

📍 데이터베이스 블록 검사를 시작하는 파라미터

📍 메모리 및 데이터 손상이 방지되는 경우

📍 기본값 : FALSE,  권장 : FULL,Medium

```sql
SYS@ora19c> show parameter db_block_checking

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_block_checking                    string      FALSE
```

📍디스크에 작성할 때 모든 데이터 블록의 캐시 헤더에서 checksum의 개선 및 저장을 시작한다.

📍 checksum은 기본 디스크, 저장 영역 시스템, I/O 시스템에 의한 손상을 감지하는데 사용된다.(기본값 :TYPICAL, 권장 : FULL)

```sql
SYS@ora19c> show parameter db_block_checksum

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_block_checksum                    string      TYPICAL
```