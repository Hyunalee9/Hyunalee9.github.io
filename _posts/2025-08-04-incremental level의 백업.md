---
title: "[70일차] incremental level의 백업 "
excerpt: "아이티윌 0804"
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-08-04T21:08
---

# incremental level의 백업

백업에 대한 스토리지를 절약

- Whole database backup : 필수 백업 대상인 모든 데이터 파일, 컨트롤 파일 백업 ( 선택적으로 리두 로그 파일, 초기 파일 백업)
- Partial database backup :  특정한 테이블 스페이스 레벨 백업, 특정 데이터 파일 레벨 백업
- full backup : incremental 백업의 용어 (변경된 블록들에 대해서만 full 백업 ?) - RMAN
- 테이프 장비에 대해서도 백업 (RMAN)

📍 Incremental Backup

- 이전 incremental backup 이후에 변경된 블록만 포함하는 백업을 의미한다.
- full backup (0 = full ) 은 모든 데이터 파일 블록을 포함한다.
- level 0 incremental backup은 full backup과 동일하다.
- cumulative level 1 incremental backup 은 마지막 incremental backup 이후 수정한 블록만 포함한다.
- cumulative level 1 incremental backup 을 마지막 level 0 incremental backup 이후 수정한 블록만 포함한다.

📍 level 0 에서 incremental backup 수행하는 방법 (full backup)

```sql
backup incremental level 0 database;
```

📍 differential level backup ( 2~ 4 중 아무거나) 

```sql
backup incremental level 1 database;
-- 변경된 것만 받는다. 
```

📍 cumulative level backup (2 ~ 4 중 아무거나) 

```sql
backup incremental level 1 cumulative database;
-- 0 번 이후 1번까지 전부 다 백업.
```

**따로 정리**

⭐ **증분 백업 (Incremental backup)**

- 이전에 백업 받았던 백업 파일과 비교해서 **변경된 부분만 골라서 백업을 수행**하는 것

1️⃣ 차등 증분 백업

![image.png](/assets/20250804/1.png)

- 백업 받을 때 설정했던 숫자가 자기보다 작거나 같으면 그 시점부터 지금까지 모든 데이터를 백업을 받는 것을 말한다.

2️⃣ 누적 증분 백업

![image.png](/assets/20250804/2.png)

## Block Change Tracking

- 변경 사항 추적 파일에 변경된 블록의 주소를 기록한다. (백그라운드 프로세서 : CTWR)
- 활성화된 경우 RMAN에 의해 자동으로 사용한다.
- 백업 중에 데이터 파일을 전체 스캔하지 않도록 incremental 백업 최적화
- 변경 사항 추적 파일의 최소 크기는 10mb 이며 10mb 새 공간이 증분 할당된다.

```sql
select filename, bytes, status from v$block_change_tracking;

FILENAME
--------------------------------------------------------------------------------
     BYTES STATUS
---------- ----------

           DISABLED
```

📍 활성화

```sql
SYS@ora19c> !
[oracle@oracle ~]$ pwd
/home/oracle
[oracle@oracle ~]$ mkdir -p /home/oracle/backup/rman
[oracle@oracle ~]$ ls
arch1   data_pump  Downloads       insa_tbs.dmp                  Music     Templates
arch2   Desktop    glogin.sql      LINUX.X64_193000_db_home.zip  oradata   userdata
backup  Documents  hr_emp.dmp      login.sql                     Pictures  Videos
clone   dontouch   import_sys.sql  minha.dmp                     Public
[oracle@oracle ~]$ cd backup/rman
[oracle@oracle rman]$ cd
[oracle@oracle ~]$ exit

-- 활성화
-- 바이너리 
alter database enable block change tracking using file '/home/oracle/backup/rman/block_tracking.txt';
select filename, bytes, status from v$block_change_tracking;

FILENAME
--------------------------------------------------------------------------------
     BYTES STATUS
---------- ----------
/home/oracle/backup/rman/block_tracking.txt
  11599872 ENABLED

SYS@ora19c> ! ls -lh /home/oracle/backup/rman/block_tracking.txt
-rw-r-----. 1 oracle oinstall 12M Aug  4 10:39 /home/oracle/backup/rman/block_tracking.txt

SYS@ora19c> ! cat /home/oracle/backup/rman/block_tracking.txt
--> 비활성화
```

📍 비활성화

```sql
alter database disable block change tracking;
```

📍 incremental level 0, full backup

```sql
RMAN> backup incremental level 0 database;
```

```sql
RMAN> delete backupset 43,44; -- full backup 제외

using channel ORA_DISK_1

List of Backup Pieces
BP Key  BS Key  Pc# Cp# Status      Device Type Piece Name
------- ------- --- --- ----------- ----------- ----------
43      43      1   1   AVAILABLE   DISK        /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/o1_mf_nnndf_TAG20250804T105849_n904yt4v_.bkp
44      44      1   1   AVAILABLE   DISK        /u01/app/oracle/fast_recovery_area/ORA19C/autobackup/2025_08_04/o1_mf_s_1208257133_n904yx7y_.bkp

Do you really want to delete the above objects (enter YES or NO)? y
deleted backup piece
backup piece handle=/u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/o1_mf_nnndf_TAG20250804T105849_n904yt4v_.bkp RECID=43 STAMP=1208257130
deleted backup piece
backup piece handle=/u01/app/oracle/fast_recovery_area/ORA19C/autobackup/2025_08_04/o1_mf_s_1208257133_n904yx7y_.bkp RECID=44 STAMP=1208257133
Deleted 2 objects

RMAN> list backup;

List of Backup Sets
===================

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
45      Incr 0  1.32G      DISK        00:00:03     04-AUG-25
        BP Key: 45   Status: AVAILABLE  Compressed: NO  Tag: TAG20250804T105906
        Piece Name: /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/o1_mf_nnnd0_TAG20250804T105906_n904zbyo_.bkp
  List of Datafiles in backup set 45
  File LV Type Ckp SCN    Ckp Time  Abs Fuz SCN Sparse Name
  ---- -- ---- ---------- --------- ----------- ------ ----
  1    0  Incr 4928327    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/system01.dbf
  3    0  Incr 4928327    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
  4    0  Incr 4928327    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
  7    0  Incr 4928327    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/users01.dbf

```

📍 테이블 생성

```sql
create table hr.inc_emp tablespace users as select * from hr.employees;
insert into hr.inc_emp select * from hr.inc_emp;

/
/
/
/
commit;
```

📍 incremental 0  이후 변경된 블록에 대해서만 백업

```sql
backup incremental level 1 database; -- differencia

-- sys
SYS@ora19c> alter system archive log current;

System altered.

SYS@ora19c> alter system switch logfile;

System altered.

-- 백업 정보
-- 4932110

-- 리두

/*
1	13	/u01/app/oracle/oradata/ORA19C/redo01.log	INACTIVE	4912408	25/08/04	4932088	25/08/04
2	14	/u01/app/oracle/oradata/ORA19C/redo02.log	INACTIVE	4932088	25/08/04	4932110	25/08/04
3	15	/u01/app/oracle/oradata/ORA19C/redo03.log	CURRENT	4932110	25/08/04	18446744073709551615	
*/

-- 아카이브
/*
/home/oracle/arch1/arch_1_13_1207934646_.arc	4912408	4932088	25/08/04
/home/oracle/arch2/arch_1_13_1207934646_.arc	4912408	4932088	25/08/04
/home/oracle/arch1/arch_1_14_1207934646_.arc	4932088	4932110	25/08/04  --> 여기에 current 
/home/oracle/arch2/arch_1_14_1207934646_.arc	4932088	4932110	25/08/04

*/
```

▶️ 장애 발생

```sql
! rm /u01/app/oracle/oradata/ORA19C/users01.dbf
! ls /u01/app/oracle/oradata/ORA19C/users01.dbf
```

▶️ 복구 작업

```sql
alter database datafile '/u01/app/oracle/oradata/ORA19C/users01.dbf' offline;

> rman
list failure;  -- 장애 감지 못하면 한번 나갔다 들어오기
report schema;

-- incremental 0 백업 파일 restore
-- 1번 레벨 파일 restore

RMAN> restore datafile 7;

Starting restore at 04-AUG-25
using channel ORA_DISK_1

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00007 to /u01/app/oracle/oradata/ORA19C/users01.dbf
channel ORA_DISK_1: reading from backup piece /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/o1_mf_nnnd0_TAG20250804T105906_n904zbyo_.bkp
channel ORA_DISK_1: piece handle=/u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/o1_mf_nnnd0_TAG20250804T105906_n904zbyo_.bkp tag=TAG20250804T105906
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
Finished restore at 04-AUG-25

--> 확인해보면 0번 레벨 백업 피스 restore 하고 있다. 

RMAN> recover datafile 7;

Starting recover at 04-AUG-25
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00007: /u01/app/oracle/oradata/ORA19C/users01.dbf
channel ORA_DISK_1: reading from backup piece /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/o1_mf_nnnd1_TAG20250804T110920_n905ljwp_.bkp
channel ORA_DISK_1: piece handle=/u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/o1_mf_nnnd1_TAG20250804T110920_n905ljwp_.bkp tag=TAG20250804T110920
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01

starting media recovery
media recovery complete, elapsed time: 00:00:01

Finished recover at 04-AUG-25

--> recover 시 1번 레벨 백업 피스 보인다.

alter database datafile 7 online;

ex) 목요일에 파일이 손상 -> 일요일 (0번 레벨) 은 restore시, 월, 화, 수 (1번 레벨) 은 recover 시 restore 작업

-- 확인
SYS@ora19c> select count(*) from hr.inc_emp;

  COUNT(*)
----------
       856

```

▶️ 데이터 +백업

```sql
SYS@ora19c> insert into hr.inc_emp select * from hr.inc_emp;

856 rows created.

SYS@ora19c> /

1712 rows created.

SYS@ora19c> /

3424 rows created.

SYS@ora19c> commit;

Commit complete.

backup incremental level 1 cumulative database; -- (bs : 49~52)

RMAN> list backup;

List of Backup Sets
===================

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
45      Incr 0  1.32G      DISK        00:00:03     04-AUG-25
        BP Key: 45   Status: AVAILABLE  Compressed: NO  Tag: TAG20250804T105906
        Piece Name: /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/o1_mf_nnnd0_TAG20250804T105906_n904zbyo_.bkp
  List of Datafiles in backup set 45
  File LV Type Ckp SCN    Ckp Time  Abs Fuz SCN Sparse Name
  ---- -- ---- ---------- --------- ----------- ------ ----
  1    0  Incr 4928327    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/system01.dbf
  3    0  Incr 4928327    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
  4    0  Incr 4928327    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
  7    0  Incr 4928327    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/users01.dbf

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
49      Incr 1  11.34M     DISK        00:00:00     04-AUG-25
        BP Key: 49   Status: AVAILABLE  Compressed: NO  Tag: TAG20250804T110920
        Piece Name: /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/o1_mf_nnnd1_TAG20250804T110920_n905ljwp_.bkp
  List of Datafiles in backup set 49
  File LV Type Ckp SCN    Ckp Time  Abs Fuz SCN Sparse Name
  ---- -- ---- ---------- --------- ----------- ------ ----
  1    1  Incr 4932013    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/system01.dbf
  3    1  Incr 4932013    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
  4    1  Incr 4932013    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
  7    1  Incr 4932013    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/users01.dbf

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
50      Full    10.39M     DISK        00:00:00     04-AUG-25
        BP Key: 50   Status: AVAILABLE  Compressed: NO  Tag: TAG20250804T110921
        Piece Name: /u01/app/oracle/fast_recovery_area/ORA19C/autobackup/2025_08_04/o1_mf_s_1208257762_n905ll1h_.bkp
  SPFILE Included: Modification time: 04-AUG-25
  SPFILE db_unique_name: ORA19C
  Control File Included: Ckp SCN: 4932023      Ckp time: 04-AUG-25

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
51      Incr 1  12.12M     DISK        00:00:00     04-AUG-25
        BP Key: 51   Status: AVAILABLE  Compressed: NO  Tag: TAG20250804T112505
        Piece Name: /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/o1_mf_nnnd1_TAG20250804T112505_n906j23t_.bkp
  List of Datafiles in backup set 51
  File LV Type Ckp SCN    Ckp Time  Abs Fuz SCN Sparse Name
  ---- -- ---- ---------- --------- ----------- ------ ----
  1    1  Incr 4934337    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/system01.dbf
  3    1  Incr 4934337    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
  4    1  Incr 4934337    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
  7    1  Incr 4934337    04-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/users01.dbf

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
52      Full    10.39M     DISK        00:00:00     04-AUG-25
        BP Key: 52   Status: AVAILABLE  Compressed: NO  Tag: TAG20250804T112507
        Piece Name: /u01/app/oracle/fast_recovery_area/ORA19C/autobackup/2025_08_04/o1_mf_s_1208258707_n906j35w_.bkp
  SPFILE Included: Modification time: 04-AUG-25
  SPFILE db_unique_name: ORA19C
  Control File Included: Ckp SCN: 4934347      Ckp time: 04-AUG-25

```

▶️ 장애 발생

```sql
! rm /u01/app/oracle/oradata/ORA19C/users01.dbf
! ls /u01/app/oracle/oradata/ORA19C/users01.dbf
```

▶️ 복구 작업

```sql
alter database datafile '/u01/app/oracle/oradata/ORA19C/users01.dbf' offline;
RMAN> restore datafile 7;  -- level 0

Starting restore at 04-AUG-25
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=272 device type=DISK

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00007 to /u01/app/oracle/oradata/ORA19C/users01.dbf
channel ORA_DISK_1: reading from backup piece /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/o1_mf_nnnd0_TAG20250804T105906_n904zbyo_.bkp
channel ORA_DISK_1: piece handle=/u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/o1_mf_nnnd0_TAG20250804T105906_n904zbyo_.bkp tag=TAG20250804T105906
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
Finished restore at 04-AUG-25

RMAN> recover datafile 7;

Starting recover at 04-AUG-25
using channel ORA_DISK_1
channel ORA_DISK_1: starting incremental datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
destination for restore of datafile 00007: /u01/app/oracle/oradata/ORA19C/users01.dbf
channel ORA_DISK_1: reading from backup piece /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/o1_mf_nnnd1_TAG20250804T112505_n906j23t_.bkp
channel ORA_DISK_1: piece handle=/u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/o1_mf_nnnd1_TAG20250804T112505_n906j23t_.bkp tag=TAG20250804T112505
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:02

starting media recovery
media recovery complete, elapsed time: 00:00:00

Finished recover at 04-AUG-25

RMAN> alter database datafile '/u01/app/oracle/oradata/ORA19C/users01.dbf' online;

```

▶️ 3

```sql
-- file# : 0 (컨트롤 파일)
select file#, incremental_level, to_char(completion_time, 'yyyy-mm-dd hh24:mi:ss'), completion_time, blocks, datafile_blocks, used_change_tracking from v$backup_datafile
```

![image.png](/assets/20250804/3.png)

▶️ 4

```sql
> rman 
delete backupset;
report need backup;

-- sys
drop table hr.inc_emp purge; 

>rman
backup incremental level 0 database tag='incremental_0'; -- incremental 0번 백업
```

![image.png](/assets/20250804/4.png)

▶️ 테이블 생성

```sql
create  table hr.inc_emp tablespace users as select * from hr.employees;
insert into hr.inc_emp select * from hr.inc_emp;
/
/
/
commit;
```

▶️ differential level 1

```sql
backup incremental level 1 database tag='incremental_1_1';
select file#, incremental_level, to_char(completion_time, 'yyyy-mm-dd hh24:mi:ss'), completion_time, blocks, datafile_blocks, used_change_tracking from v$backup_datafile;
```

![image.png](/assets/20250804/5.png)

```sql
insert into hr.inc_emp select * from hr.inc_emp;
/
/
/
commit;

backup incremental level 1 database tag='incremental_1_2';
```

![image.png](/assets/20250804/6.png)

```sql
SYS@ora19c> ! ls -lh /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/
total 1.4G
-rw-r-----. 1 oracle oinstall 1.4G Aug  4 11:42 o1_mf_nnnd0_**INCREMENTAL_0**_n907k8nj_.bkp
-rw-r-----. 1 oracle oinstall 824K Aug  4 11:47 o1_mf_nnnd1_**INCREMENTAL_1_1**_n907szlt_.bkp
-rw-r-----. 1 oracle oinstall 2.8M Aug  4 11:49 o1_mf_nnnd1_**INCREMENTAL_1_2**_n907xj7k_.bkp

```

```sql
insert into hr.inc_emp select * from hr.inc_emp;
commit;
```

📍 cumulative level 1

```sql
backup incremental level 1 cumulative database tag='cumulative_1';
```

![image.png](/assets/20250804/7.png)

```sql
select file#, incremental_level, to_char(completion_time, 'yyyy-mm-dd hh24:mi:ss'), completion_time, blocks, datafile_blocks, used_change_tracking from v$backup_datafile;
```

![image.png](/assets/20250804/8.png)

```sql
SYS@ora19c> ! ls -lh /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_04/
total 1.4G
-rw-r-----. 1 oracle oinstall 1.4G Aug  4 11:42 o1_mf_nnnd0_INCREMENTAL_0_n907k8nj_.bkp
-rw-r-----. 1 oracle oinstall 5.9M Aug  4 11:53 o1_mf_nnnd1_CUMULATIVE_1_n9085xbj_.bkp
-rw-r-----. 1 oracle oinstall 824K Aug  4 11:47 o1_mf_nnnd1_INCREMENTAL_1_1_n907szlt_.bkp
-rw-r-----. 1 oracle oinstall 2.8M Aug  4 11:49 o1_mf_nnnd1_INCREMENTAL_1_2_n907xj7k_.bkp
```

▶️ 5

```sql
delete backup;
list backup;
delete copy;
list copy;

!
cd backup/rman/
mkdir img
cd img
pwd
-- /home/oracle/backup/rman/img
```

📍 image copy 를 이용해서 incremental backup

```sql
run {
	allocate channel ch1 device type disk format "/home/oracle/backup/rman/img/%U";
	backup as copy tag 'base01' incremental level 0 database;
	backup as copy current controlfile;
}	

-- sys

SYS@ora19c> ! ls /home/oracle/backup/rman/img/
cf_D-ORA19C_id-1258537005_2b409cpe                   data_D-ORA19C_I-1258537005_TS-SYSTEM_FNO-1_27409cp6    data_D-ORA19C_I-1258537005_TS-USERS_FNO-7_2a409cpd
data_D-ORA19C_I-1258537005_TS-SYSAUX_FNO-3_28409cp9  data_D-ORA19C_I-1258537005_TS-UNDOTBS1_FNO-4_29409cpc
```

▶️ 테이블 생성

```sql
create table hr.emp_temp tablespace users as select * from hr.employees;
insert into hr.emp_temp select * from hr.emp_temp;
/
/
/
commit;
```

▶️ 작업형 돌리기 

```sql
run {
	allocate channel ch1 device type disk format "/home/oracle/backup/rman/img/%U";
	backup incremental level 0 database tag 'incremental_update';
	backup as copy current controlfile;
}	

list backup;
```

![image.png](/assets/20250804/9.png)

```sql
SYS@ora19c> ! ls -lh /home/oracle/backup/rman/img/
total 3.3G
-rw-r-----. 1 oracle oinstall 1.4G Aug  4 14:00 2d409d7m_1_1
-rw-r-----. 1 oracle oinstall  11M Aug  4 13:52 cf_D-ORA19C_id-1258537005_2b409cpe
-rw-r-----. 1 oracle oinstall  11M Aug  4 14:00 cf_D-ORA19C_id-1258537005_2e409d7p
-rw-r-----. 1 oracle oinstall 721M Aug  4 13:52 data_D-ORA19C_I-1258537005_TS-SYSAUX_FNO-3_28409cp9
-rw-r-----. 1 oracle oinstall 921M Aug  4 13:52 data_D-ORA19C_I-1258537005_TS-SYSTEM_FNO-1_27409cp6
-rw-r-----. 1 oracle oinstall 341M Aug  4 13:52 data_D-ORA19C_I-1258537005_TS-UNDOTBS1_FNO-4_29409cpc
-rw-r-----. 1 oracle oinstall  11M Aug  4 13:52 data_D-ORA19C_I-1258537005_TS-USERS_FNO-7_2a409cpd

```

▶️ 리두 로그 확인

```sql
1	13	/u01/app/oracle/oradata/ORA19C/redo01.log	INACTIVE	4912408	25/08/04	4932088	25/08/04
2	14	/u01/app/oracle/oradata/ORA19C/redo02.log	INACTIVE	4932088	25/08/04	4932110	25/08/04
3	15	/u01/app/oracle/oradata/ORA19C/redo03.log	CURRENT	4932110	25/08/04	18446744073709551615	
```

▶️ 아카이브 저장, 로그 스위치 유발

```sql
SYS@ora19c> alter system archive log current;

System altered.

SYS@ora19c> alter system switch logfile;

System altered.

SYS@ora19c> /

System altered.

SYS@ora19c> /

System altered.

SYS@ora19c> /

System altered.

```

▶️ 아카이브 로그 확인

```sql
/home/oracle/arch1/arch_1_16_1207934646_.arc	4958317	4958336	2025-08-04 14:05:44
/home/oracle/arch2/arch_1_16_1207934646_.arc	4958317	4958336	2025-08-04 14:05:44
/home/oracle/arch1/arch_1_17_1207934646_.arc	4958336	4958352	2025-08-04 14:05:53
/home/oracle/arch2/arch_1_17_1207934646_.arc	4958336	4958352	2025-08-04 14:05:53
/home/oracle/arch1/arch_1_18_1207934646_.arc	4958352	4958358	2025-08-04 14:05:59
/home/oracle/arch2/arch_1_18_1207934646_.arc	4958352	4958358	2025-08-04 14:05:59
/home/oracle/arch1/arch_1_19_1207934646_.arc	4958358	4958364	2025-08-04 14:06:00
/home/oracle/arch2/arch_1_19_1207934646_.arc	4958358	4958364	2025-08-04 14:06:00
```

▶️ 장애 발생

```sql
! rm /u01/app/oracle/oradata/ORA19C/users01.dbf
```

▶️ 복구

```sql
alter database datafile '/u01/app/oracle/oradata/ORA19C/users01.dbf' offline;
restore datafile 7;
recover datafile 7;
alter database datafile '/u01/app/oracle/oradata/ORA19C/users01.dbf' online;
```

📍 update

```sql
SYS@ora19c> ! ls -lh /home/oracle/backup/rman/img/
total 3.3G
-rw-r-----. 1 oracle oinstall 1.4G Aug  4 14:00 2d409d7m_1_1
-rw-r-----. 1 oracle oinstall  11M Aug  4 13:52 cf_D-ORA19C_id-1258537005_2b409cpe
-rw-r-----. 1 oracle oinstall  11M Aug  4 14:00 cf_D-ORA19C_id-1258537005_2e409d7p
-rw-r-----. 1 oracle oinstall 721M Aug  4 13:52 data_D-ORA19C_I-1258537005_TS-SYSAUX_FNO-3_28409cp9
-rw-r-----. 1 oracle oinstall 921M Aug  4 13:52 data_D-ORA19C_I-1258537005_TS-SYSTEM_FNO-1_27409cp6
-rw-r-----. 1 oracle oinstall 341M Aug  4 13:52 data_D-ORA19C_I-1258537005_TS-UNDOTBS1_FNO-4_29409cpc
-rw-r-----. 1 oracle oinstall  11M Aug  4 13:52 data_D-ORA19C_I-1258537005_TS-USERS_FNO-7_2a409cpd

-- 0번 레벨 백업보다는 2d409d7m_1_1 백업 피스를 업데이트해서 해보자.
```

▶️ update incremental backup

📍 0  level 백업에 1 level 백업을 적용한다.

```sql
RMAN> list copy;

List of Datafile Copies
=======================

Key     File S Completion Time Ckp SCN    Ckp Time        Sparse
------- ---- - --------------- ---------- --------------- ------
14      1    A 04-AUG-25       4955890    04-AUG-25       NO
        Name: /home/oracle/backup/rman/img/data_D-ORA19C_I-1258537005_TS-SYSTEM_FNO-1_27409cp6
        Tag: BASE01

15      3    A 04-AUG-25       4955898    04-AUG-25       NO
        Name: /home/oracle/backup/rman/img/data_D-ORA19C_I-1258537005_TS-SYSAUX_FNO-3_28409cp9
        Tag: BASE01

16      4    A 04-AUG-25       4955906    04-AUG-25       NO
        Name: /home/oracle/backup/rman/img/data_D-ORA19C_I-1258537005_TS-UNDOTBS1_FNO-4_29409cpc
        Tag: BASE01

17      7    A 04-AUG-25       4955910    04-AUG-25       NO
        Name: /home/oracle/backup/rman/img/data_D-ORA19C_I-1258537005_TS-USERS_FNO-7_2a409cpd
        Tag: BASE01

-- 업데이트 작업을 하고 있다. 
-- incremental 백업본을 가지고 업데이트하는 순간 체크포인트는 일치가 되었다. 
run {
	allocate channel ch1 device type disk format "/home/oracle/backup/rman/img/%U";
	recover copy of database with tag 'Base01';	
}	

list copy;

```

▶️ 장애 발생

```sql
! rm /u01/app/oracle/oradata/ORA19C/users01.dbf

alter database datafile '/u01/app/oracle/oradata/ORA19C/users01.dbf' offline;
restore datafile 7;
recover datafile 7;
alter database datafile '/u01/app/oracle/oradata/ORA19C/users01.dbf' online;
```

📍 압축 : 백업 압축, 테이블 압축, 아카이브 로그 압 축 

## 백업 압축

- 특정 유형의 백업을 수행할 때 RMAN 은 일부 블록을 건너뛸 수 있다. 즉 HWM (High-Water Mark) 위에 있는 할당되지 않은 블록을 건너뛸 수 있다.
- RMAN은 생성되는 모든 백업셋에 대해 이진 압축을 수행할 수 있다.
- 압축된 백업본을 이용해서 복원하기 위해 추가 작업을 수행할 필요 없다.
- basic을 제외한 모든 알고리즘은 Oracle Advanced Compression Database 옵션을 사용해야 한다.
- 장점 : 저장 공간 절약, 전송 속도 향상, 보관 비용, 아카이브 로그 저장 비용 절감
- 단점 : CPU 부하가 늘어날 수 있다.

```sql
-- 압축 백업
backup as compressed backupset database;
list backup;
```

▶️ show all;

```sql
CONFIGURE COMPRESSION ALGORITHM 'BASIC' AS OF RELEASE 'DEFAULT' OPTIMIZE FOR LOAD TRUE ; # default
select * from v$rman_compression_algorithm;

/*

1 BASIC
10.0.0.0.0
good compression ratio
9.2.0.0.0          YES NO  YES          0

무료

*/

-- 오라클 좋은 압축률 : 유료
CONFIGURE COMPRESSION ALGORITHM 'HIGH';
-- 기본값
CONFIGURE COMPRESSION ALGORITHM 'CLEAR';

backup as compressed backupset incremental level 0 database tag='incremental_0';
```

## 암호화

```sql
select file_id, tablespace_name, file_name, bytes/1024/1024 mb from dba_data_files;

/*
1	SYSTEM	/u01/app/oracle/oradata/ORA19C/system01.dbf	920
3	SYSAUX	/u01/app/oracle/oradata/ORA19C/sysaux01.dbf	720
7	USERS	/u01/app/oracle/oradata/ORA19C/users01.dbf	10
4	UNDOTBS1	/u01/app/oracle/oradata/ORA19C/undotbs01.dbf	340
*/

create tablespace insa_

-- strings : 파일의 ASCII 문자를 찾아 출력
! strings /u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf >> insa_data.txt
! cat insa_data.txt
```

📍오라클 암호화

- TDE(Transparent Data Encryption)를  사용하여 테이블 및 테이블스페이스에 저장된 데이터를 암호화 한다.
- 데이터베이스 내의 백업 데이터를 암호화한다.
- 암호화 키는 keystore에 저장되며 데이터베이스에서 관리된다.

📍 keystore 저장 디렉토리 생성

```sql
mkdir -p /home/oracle/wallet
```

📍 wallet_root 파라미터 설정, 암호화 마스터 키를 저장하는데 사용되는 keystore 파일의 위치

```sql
show parameter wallet_root 

/*
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
wallet_root                          string

*/

alter system set wallet_root='/home/oracle/wallet' scope=spfile;
shutdown immediate;
startup

SYS@ora19c> show parameter wallet_root

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
wallet_root                          string      /home/oracle/wallet

-- 키스토어가 저장되야될 위치 지정
SYS@ora19c> select * from v$encryption_wallet;

WRL_TYPE
--------------------
WRL_PARAMETER
--------------------------------------------------------------------------------
STATUS                         WALLET_TYPE          WALLET_OR KEYSTORE FULLY_BAC
------------------------------ -------------------- --------- -------- ---------
    CON_ID
----------
FILE

NOT_AVAILABLE                  UNKNOWN              SINGLE    NONE     UNDEFINED
         0
         
         
! ls /home/oracle/wallet         
```

📍tde_configuration 설정

```sql
show parameter tde_configuration
alter system set tde_configuration="keystore_configuration=file" scope=both;
show parameter tde_configuration
select * from v$encryption_wallet;
```

📍 keystore 생성

```sql
-- 인증서 비밀번호 
administer key management create keystore identified by "oracle1234" ;

! ls /home/oracle/wallet/tde/

/*
SYS@ora19c> ! ls /home/oracle/wallet/tde/
ewallet.p12
*/

select * from v$encryption_wallet;
```

📍 keystore open

```sql
administer key management set keystore open identified by "oracle1234";
select * from v$encryption_wallet;
```

📍 master key 생성

```sql
administer key management set key force keystore identified by "oracle1234" with backup;
! ls /home/oracle/wallet/tde/
```

📍 keystore close

```sql
administer key management set keystore close identified by "oracle1234";
select * from v$encrytion_wallet;
```

📍 keystore open

```sql
administer key management set keystore open identified by "oracle1234";
select * from v$encryption_wallet;
```

📍

```sql
select * from v$encryption_keys;
select masterkey_activated from v$database_key_info;
```

▶️ 테이블스페이스 암호화

```sql
create tablespace insa_enc datafile '/u01/app/oracle/oradata/ORA19C/insa_enc01.dbf' size 5m encrytion using 'AES256' encrypt;
```

📍

```sql
select
from v$encrypted_tablespaces e, v$tablespace t
where t.ts# = e.ts#(+);

select t.name, e.encryptionalg, e.encryptedts, e.status
from v$encrypted_tablespaces e, v$tablespace t
where t.ts# = e.ts#(+);

NAME
------------------------------------------------------------
ENCRYPT      ENC STATUS
-------      --- --------
INSA_ENC     AES256      YES NORMAL
SYSAUX
UNDOTBS1
USERS
TEMP
INSA_TBS
SYSTEM

-- 암호화되었는지 안되었는지.
select tablespace_name, encrypted from dba_tablespaces;

create table hr.insa_enc tablespace insa_enc as select * from hr.employees;

SYS@ora19c> ! strings /u01/app/oracle/oradata/ORA19C/insa_enc01.dbf >> insa_enc.txt
SYS@ora19c> ! cat insa_enc.txt
SYS@ora19c> ! cat insa_data.txt

select * from v$encryption_wallet;

-- keystore open
administer key management set keystore open identified by "oracle1234";
-- keystore close
administer key management set keystore close identified by "oracle1234";

select * from hr.insa_enc;

administer key management set keystore open identified by "oracle1234";
select * from v$encrytion_wallet;
select * from hr.insa_new;
```

▶️ 컬럼 암호

```sql
alter table hr.insa_new modify salary encrypt;

select table_name, column_name, encryption_alg
from dba_encrypted_columns
where table_name = 'INSA_NEW'
and owner = 'HR';
```

```sql
select * from hr.insa_new;    -- keystore 자체가 open 되어있음
select employee_id, salary from hr.insa_new; -- 오류가 난다. 
```

📍 암호화되어 있는 컬럼을 해지

```sql
alter table hr.insa_new modify salary decrypt;
```

📍 open

```sql
administer key management set keystore open identified by "oracle1234";
```

📍암호화 되어 있는 컬럼을 해지

```sql
alter table hr.insa_new modify salary decrypt;
```

📍 암호화 되어 있는 테이블스페이스 해지

```sql
alter tablespace insa_enc encrytion online decrypt;

select t.name, e.encryptionalg, e.encryptedts, e.status
from v$encrypted_tablespaces e, v$tablespace t
where t.ts# = e.ts#(+);
```

▶️ keystore close

```sql
administer key management set keystore close identified by "oracle1234";
select * from v$encrytion_wallet;
```

▶️ 암호화 테이블스페이스 설정

```sql
alter tablespace insa_enc encryption online using 'AES256' encrypt; -- 오류 (open되어있지 않기 때문에)
```

▶️  master key 변경

```sql
administer key management alter keystore password force keystore identified by "oracle1234" set "itwill1234" with backup;
```

```sql
# keystore close
SYS@ora19c> administer key management set keystore close identified by "itwill1234";

# keystore open
SYS@ora19c> administer key management set keystore open identified by "itwill1234";
```

▶️ open시에만 조회가능

```sql
select * from hr.insa_enc;
```

▶️

```sql
-- close 상태
! ls /home/oracle/wallet/tde/
! mv /home/oracle/wallet/tde/* /home/oracle/wallet
! ls .home/oracle/wallet.tde  -- 키스토어가 사라졌다. 누가 오더라도 못살린다.
-- 다시 원복
SYS@ora19c> administer key management set keystore open identified by "itwill1234";
ERROR at line 1:
ORA-28367: wallet does not exist

SYS@ora19c> ! mv /home/oracle/wallet/ewallet.p12 /home/oracle/wallet/tde/
SYS@ora19c> ! ls /home/oracle/wallet/tde/
ewallet.p12
```

▶️ 암호화가 되어있더라도 테이블스페이스 삭제 가능 

```sql
drop tablespace insa_enc including contents and datafiles;
```

▶️ 암호 설정 해지

```sql
alter system reset wallet_root scope=spfile;
alter system reset tde_configuration scope=both;
```

📍 image copy 백업은 암호화 할 수 없다.

▶️ RMAN에서 사용할 수 있는 암호 알고리즘

```sql
select * from v$rman_encryption_algorithms;
CONFIGURE ENCRYPTION ALGORITHM 'AES128'; #default
CONFIGURE ENCRYPTION ALGORITHM 'AES128';
-- rman 세션에서만
SET ENCRYPTION ALGORITHM 'AES256';
CONFIGURE ENCRYPTION ALGORITHM CLEAR;
```