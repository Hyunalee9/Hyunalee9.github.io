---
title: "[63일차] time in recovery 와 clone db"
excerpt: "아이티윌 0729_(3)"
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-07-29T21:08
---

```sql
select supplemental_log_data_min from v$database;

SUPPLEMET
--------
YES

SELECT a.group#, b.sequence#, a.member, b.status, b.first_change#, to_char(b.first_time, 'yyyy-mm-dd hh24:mi:ss') first, b.next_change#, to_char(b.next_time, 'yyyy-mm-dd hh24:mi:ss') next  
FROM v$logfile a, v$log b
WHERE a.group# = b.group#
ORDER BY b.group#;

/*
1	34	/u01/app/oracle/oradata/ORA19C/redo01.log	ACTIVE	4063449	2025-07-29 15:23:47	4064654	2025-07-29 15:32:03
2	35	/u01/app/oracle/oradata/ORA19C/redo02.log	ACTIVE	4064654	2025-07-29 15:32:03	4064678	2025-07-29 15:32:12
3	36	/u01/app/oracle/oradata/ORA19C/redo03.log	CURRENT	4064678	2025-07-29 15:32:12	18446744073709551615	
*/

select sequence#,  
       name,  
       first_change#,  
       to_char(first_time, 'yyyy-mm-dd hh24:mi:ss') first,  
       next_change#,  
       to_char(next_time, 'yyyy-mm-dd hh24:mi:ss') next  
from   v$archived_log;

/*

33	/home/oracle/arch2/arch_1_33_1205348976_.arc	4060949	2025-07-29 15:18:58	4063449	2025-07-29 15:23:47
34	/home/oracle/arch1/arch_1_34_1205348976_.arc	4063449	2025-07-29 15:23:47	4064654	2025-07-29 15:32:03
34	/home/oracle/arch2/arch_1_34_1205348976_.arc	4063449	2025-07-29 15:23:47	4064654	2025-07-29 15:32:03
35	/home/oracle/arch1/arch_1_35_1205348976_.arc	4064654	2025-07-29 15:32:03	4064678	2025-07-29 15:32:12
35	/home/oracle/arch2/arch_1_35_1205348976_.arc	4064654	2025-07-29 15:32:03	4064678	2025-07-29 15:32:12
*/

```

▶️ hr 유저 삭제

```sql
drop user hr cascade; 
```

▶️ 유저 삭제 시간을 찾아야 한다.

```sql
BEGIN
	dbms_logmnr.add_logfile(logfilename=>'/u01/app/oracle/oradata/ORA19C/redo03.log', options=>dbms_logmnr.new);
	dbms_logmnr.add_logfile(logfilename=>'/u01/app/oracle/oradata/ORA19C/redo02.log', options=>dbms_logmnr.addfile);
END;
/

select db_name, filename from v$logmnr_logs;
execute dbms_logmnr.start_logmnr(options=>dbms_logmnr.dict_from_online_catalog);

-- 객체를 삭제하기 전 시간으로 가야함
select to_char(timestamp, 'yyyy-mm-dd hh24:mi:ss') time, operation, sql_redo
  from v$logmnr_contents
  where seg_owner = 'HR';

```

▶️ 목표 시간 2025-07-29 15:35:47 전 

```sql
execute dbms_logmnr.end_logmnr

-- log 확인
  select a.group#, 
       b.sequence#, 
       a.member, 
       b.bytes/1024/1024 mb, 
       b.archived, 
       b.status, 
       b.first_change#, 
       to_char(b.first_time,'yyyy-mm-dd hh24:mi:ss') first, 
       b.next_change#,  
       to_char(b.next_time,'yyyy-mm-dd hh24:mi:ss') next
from v$logfile a, v$log b
where a.group# = b.group#
order by 1;

/*
1	34	/u01/app/oracle/oradata/ORA19C/redo01.log	200	YES	INACTIVE	4063449	2025-07-29 15:23:47	4064654
2	35	/u01/app/oracle/oradata/ORA19C/redo02.log	200	YES	INACTIVE	4064654	2025-07-29 15:32:03	4064678
3	36	/u01/app/oracle/oradata/ORA19C/redo03.log	200	NO	CURRENT	4064678	2025-07-29 15:32:12	18446744073709551615

*/

alter system archive log current;

-- 아카이브 파일 확인
select sequence#,  
       name,  
       first_change#,  
       to_char(first_time, 'yyyy-mm-dd hh24:mi:ss') first,  
       next_change#,  
       to_char(next_time, 'yyyy-mm-dd hh24:mi:ss') next  
from   v$archived_log;

/home/oracle/arch1/arch_1_34_1205348976_.arc
/home/oracle/arch2/arch_1_34_1205348976_.arc
/home/oracle/arch1/arch_1_35_1205348976_.arc
/home/oracle/arch2/arch_1_35_1205348976_.arc
/home/oracle/arch1/arch_1_36_1205348976_.arc
/home/oracle/arch2/arch_1_36_1205348976_.arc

[oracle@oracle arch1]$ cp -v arch_1_{34,35,36}_1205348976_.arc /home/oracle/clone
‘arch_1_34_1205348976_.arc’ -> ‘/home/oracle/clone/arch_1_34_1205348976_.arc’
‘arch_1_35_1205348976_.arc’ -> ‘/home/oracle/clone/arch_1_35_1205348976_.arc’
‘arch_1_36_1205348976_.arc’ -> ‘/home/oracle/clone/arch_1_36_1205348976_.arc’

```

▶️ pfile 생성

```sql
create pfile='/home/oracle/clone/initclone.ora' from spfile;
```

▶️ control file trace 생성

```sql
alter database backup controlfile to trace as '/home/oracle/clone/control.txt';
! ls /home/oracle/clone
```

▶️ pfile 수정

```sql
!
cd clone/
vi initclone.ora

-- 데이터베이스 띄울때 필요한 정보
*.compatible='19.0.0'
*.control_files='/u01/app/oracle/oradata/ORA19C/control01.ctl'
*.db_name='ora19c'
*.log_archive_dest_1='location=/home/oracle/arch1 mandatory'
*.log_archive_format='arch_%t_%s_%r .arc'
*.undo_tablespace='UNDOTBS1'

/*--> clone DB */
ggDG로 다 지우기 

*.compatible='19.0.0'
*.control_files='/home/oracle/clone/control.ctl'
*.db_name='clone'
*.log_archive_dest_1='location=/home/oracle/arch1 mandatory'
*.log_archive_format='arch_%t_%s_%r .arc'
*.undo_tablespace='UNDOTBS1'
```

▶️ 새로운 control file 생성

```sql
CREATE CONTROLFILE SET DATABASE "CLONE" RESETLOGS ARCHIVELOG
    MAXLOGFILES 16
    MAXLOGMEMBERS 3
    MAXDATAFILES 100
    MAXINSTANCES 8
    MAXLOGHISTORY 292
LOGFILE
  GROUP 1 '/home/oracle/clone/redo01.log' SIZE 200M BLOCKSIZE 512, --여기서 BLOCKSIZE는 OS블록임
  GROUP 2 '/home/oracle/clone/redo02.log' SIZE 200M BLOCKSIZE 512,
  GROUP 3 '/home/oracle/clone/redo03.log' SIZE 200M BLOCKSIZE 512
DATAFILE
  '/home/oracle/clone/system01.dbf',
  '/home/oracle/clone/sysaux01.dbf',
  '/home/oracle/clone/undotbs01.dbf',
  '/home/oracle/clone/users01.dbf'
CHARACTER SET AL32UTF8;
```

▶️ pfile 이용해서 nomount 시작해야한다.

```sql
startup pfile=/home/oracle/clone/initclone.ora nomount;
```

▶️ 컨트롤 파일 생성

```sql
CREATE CONTROLFILE SET DATABASE "CLONE" RESETLOGS ARCHIVELOG
    MAXLOGFILES 16
    MAXLOGMEMBERS 3
    MAXDATAFILES 100
    MAXINSTANCES 8
    MAXLOGHISTORY 292
LOGFILE
  GROUP 1 '/home/oracle/clone/redo01.log' SIZE 200M BLOCKSIZE 512, --여기서 BLOCKSIZE는 OS블록임
  GROUP 2 '/home/oracle/clone/redo02.log' SIZE 200M BLOCKSIZE 512,
  GROUP 3 '/home/oracle/clone/redo03.log' SIZE 200M BLOCKSIZE 512
DATAFILE
  '/home/oracle/clone/system01.dbf',
  '/home/oracle/clone/sysaux01.dbf',
  '/home/oracle/clone/undotbs01.dbf',
  '/home/oracle/clone/users01.dbf'
CHARACTER SET AL32UTF8;
```

▶️ 목표시간 2025-07-29 15:35:47 을 기준으로 time base recovery 수행

```sql
alter session set nls_date_format='yyyy-mm-dd hh24:mi:ss';

Session altered.

SYS@clone> recover database using backup controlfile until time '2025-07-29 15:35:47'
ORA-00279: change 4063220 generated at 07/29/2025 15:22:02 needed for thread 1
ORA-00289: suggestion : /home/oracle/arch1/arch_1_33_1205348976_.arc
ORA-00280: change 4063220 for thread 1 is in sequence #33

Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
> auto
ORA-00279: change 4063449 generated at 07/29/2025 15:23:47 needed for thread 1
ORA-00289: suggestion : /home/oracle/arch1/arch_1_34_1205348976_.arc
ORA-00280: change 4063449 for thread 1 is in sequence #34
ORA-00278: log file '/home/oracle/arch1/arch_1_33_1205348976_.arc' no longer
needed for this recovery

ORA-00279: change 4064654 generated at 07/29/2025 15:32:03 needed for thread 1
ORA-00289: suggestion : /home/oracle/arch1/arch_1_35_1205348976_.arc
ORA-00280: change 4064654 for thread 1 is in sequence #35
ORA-00278: log file '/home/oracle/arch1/arch_1_34_1205348976_.arc' no longer
needed for this recovery

ORA-00279: change 4064678 generated at 07/29/2025 15:32:12 needed for thread 1
ORA-00289: suggestion : /home/oracle/arch1/arch_1_36_1205348976_.arc
ORA-00280: change 4064678 for thread 1 is in sequence #36
ORA-00278: log file '/home/oracle/arch1/arch_1_35_1205348976_.arc' no longer
needed for this recovery

Log applied.
Media recovery complete.
SYS@clone> alter database open resetlogs;

Database altered.

-- hr 유저 살아났다.
SYS@clone> select * from dba_users where username = 'HR';

```

▶️ export

```sql
! expdp system/oracle schemas=hr directory=pump_dir dumpfile=hr_schema.dmp

-- Shared pool 오류
Connected to: Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Starting "SYSTEM"."SYS_EXPORT_SCHEMA_01":  system/******** schemas=hr directory=pump_dir dumpfile=hr_schema.dmp
Processing object type SCHEMA_EXPORT/TABLE/TABLE_DATA
Processing object type SCHEMA_EXPORT/TABLE/INDEX/STATISTICS/INDEX_STATISTICS
Processing object type SCHEMA_EXPORT/TABLE/STATISTICS/TABLE_STATISTICS
Processing object type SCHEMA_EXPORT/STATISTICS/MARKER
Processing object type SCHEMA_EXPORT/USER
Processing object type SCHEMA_EXPORT/SYSTEM_GRANT
Processing object type SCHEMA_EXPORT/ROLE_GRANT
Processing object type SCHEMA_EXPORT/DEFAULT_ROLE
Processing object type SCHEMA_EXPORT/TABLESPACE_QUOTA
Processing object type SCHEMA_EXPORT/PRE_SCHEMA/PROCACT_SCHEMA
Processing object type SCHEMA_EXPORT/SEQUENCE/SEQUENCE
ORA-39014: One or more workers have prematurely exited.
ORA-39029: worker 1 with process name "DW00" prematurely terminated
ORA-31671: Worker process DW00 had an unhandled exception.
ORA-04031: unable to allocate 472 bytes of shared memory ("shared pool","select /*+ rule */ bucket_cn...","SQLA^f5f698ce","qertbs:qertbIAllocate")
ORA-06512: at "SYS.KUPW$WORKER", line 2506
ORA-06512: at "SYS.KUPW$WORKER", line 13964
ORA-06512: at "SYS.KUPW$WORKER", line 4000
ORA-06512: at "SYS.KUPW$WORKER", line 15444
ORA-06512: at "SYS.DBMS_METADATA", line 9571
ORA-06512: at "SYS.DBMS_METADATA", line 2976
ORA-06512: at "SYS.DBMS_METADATA", line 3608
ORA-06512: at "SYS.DBMS_METADATA", line 5008
ORA-06512: at "SYS.DBMS_METADATA", line 5327
ORA-06512: at "SYS.DBMS_METADATA", line 9552
ORA-06512: at "SYS.KUPW$WORKER", line 15109
ORA-06512: at "SYS.KUPW$WORKER", line 3907
ORA-06512: at "SYS.KUPW$WORKER", line 13736
ORA-06512: at "SYS.KUPW$WORKER", line 2429
ORA-06512: at line 2

shutdown immediate
free -h
vi /home/oracle/clone/initclone.ora    /*  *.shared_pool_size=300m */
startup pfile=/home/oracle/clone/initclone.ora
select owner_name, job_name, operation, job_mode, state from dba_datapump_jobs;
drop table SYSTEM.SYS_EXPORT_SCHEMA_01 purge;

! rm /u01/app/oracle/admin/ora19c/dpdump/hr.dmp
! expdp system/oracle directory=data_pump_dir dumpfile=hr.dmp schemas=hr

```

▶️

```sql
Export: Release 19.0.0.0.0 - Production on Tue Jul 29 16:49:29 2025
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Connected to: Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Starting "SYSTEM"."SYS_EXPORT_SCHEMA_01":  system/******** directory=data_pump_dir dumpfile=hr.dmp schemas=hr
Processing object type SCHEMA_EXPORT/TABLE/TABLE_DATA
Processing object type SCHEMA_EXPORT/TABLE/INDEX/STATISTICS/INDEX_STATISTICS
Processing object type SCHEMA_EXPORT/TABLE/STATISTICS/TABLE_STATISTICS
Processing object type SCHEMA_EXPORT/STATISTICS/MARKER
Processing object type SCHEMA_EXPORT/USER
Processing object type SCHEMA_EXPORT/SYSTEM_GRANT
Processing object type SCHEMA_EXPORT/ROLE_GRANT
Processing object type SCHEMA_EXPORT/DEFAULT_ROLE
Processing object type SCHEMA_EXPORT/TABLESPACE_QUOTA
Processing object type SCHEMA_EXPORT/PRE_SCHEMA/PROCACT_SCHEMA
Processing object type SCHEMA_EXPORT/SEQUENCE/SEQUENCE
Processing object type SCHEMA_EXPORT/TABLE/TABLE
Processing object type SCHEMA_EXPORT/TABLE/GRANT/OWNER_GRANT/OBJECT_GRANT
Processing object type SCHEMA_EXPORT/TABLE/COMMENT
Processing object type SCHEMA_EXPORT/FUNCTION/FUNCTION
Processing object type SCHEMA_EXPORT/PROCEDURE/PROCEDURE
Processing object type SCHEMA_EXPORT/PROCEDURE/GRANT/OWNER_GRANT/OBJECT_GRANT
Processing object type SCHEMA_EXPORT/FUNCTION/ALTER_FUNCTION
Processing object type SCHEMA_EXPORT/PROCEDURE/ALTER_PROCEDURE
Processing object type SCHEMA_EXPORT/VIEW/VIEW
Processing object type SCHEMA_EXPORT/TABLE/INDEX/INDEX
Processing object type SCHEMA_EXPORT/TABLE/CONSTRAINT/CONSTRAINT
Processing object type SCHEMA_EXPORT/TABLE/CONSTRAINT/REF_CONSTRAINT
Processing object type SCHEMA_EXPORT/TABLE/TRIGGER
. . exported "HR"."EMP"                                  17.08 KB     107 rows
. . exported "HR"."EMPLOYEES"                            17.08 KB     107 rows
. . exported "HR"."LOCATIONS"                            8.437 KB      23 rows
. . exported "HR"."JOBS"                                 7.109 KB      19 rows
. . exported "HR"."DEPARTMENTS"                          7.125 KB      27 rows
. . exported "HR"."COUNTRIES"                            6.367 KB      25 rows
. . exported "HR"."DEPT"                                 6.023 KB      27 rows
. . exported "HR"."REGIONS"                              5.546 KB       4 rows
. . exported "HR"."TIME"                                   5.5 KB       1 rows
. . exported "HR"."ITWILL"                               5.476 KB       1 rows
. . exported "HR"."EMP_50_STMAN"                             0 KB       0 rows
. . exported "HR"."JOB_HISTORY"                              0 KB       0 rows
Master table "SYSTEM"."SYS_EXPORT_SCHEMA_01" successfully loaded/unloaded
******************************************************************************
Dump file set for SYSTEM.SYS_EXPORT_SCHEMA_01 is:
  /u01/app/oracle/admin/ora19c/dpdump/hr.dmp
Job "SYSTEM"."SYS_EXPORT_SCHEMA_01" successfully completed at Tue Jul 29 16:50:25 2025 elapsed 0 00:00:54

```

▶️ 클론 디비 설치 경로 

```sql
[root@oracle ~]# vi /etc/oratab
```

![image.png](/assets/20250729/5.png)