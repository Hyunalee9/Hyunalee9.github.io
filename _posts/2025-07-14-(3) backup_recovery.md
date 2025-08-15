---
title: "[56일차] Backup & Recovery"
excerpt: "아이티윌 0714_(3)"
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-07-14T17:13
---

# 백업 & 리커버리

```sql
-- 유저, 테이블스페이스 삭제
DROP USER sawon01 CASCADE;  
DROP USER sawon02 CASCADE;
DROP TABLESPACE oltp_tbs INCLUDING CONTENTS AND DATAFILES;
DROP TABLESPACE oltp_temp INCLUDING CONTENTS AND DATAFILES;
DROP TABLESPACE big_tbs INCLUDING CONTENTS AND DATAFILES;
DROP TABLESPACE insa_tbs INCLUDING CONTENTS AND DATAFILES;
DROP TABLESPACE insa_temp INCLUDING CONTENTS AND DATAFILES;

-- 밑에 처럼 맞추자
SELECT * FROM dba_data_files;
```

![image.png](/assets/20250714/9.png)

```sql
SELECT * FROM dba_tablespaces;
```

![image.png](/assets/20250714/10.png)

```sql
-- controlfile 단일화.
SELECT * FROM v$controlfile;

-- 만약 이중화해놓았다면 control01 만 사용하도록 상태 변경 -> DB 껐다 켜기
ALTER SYSTEM SET control_files = '/u01/app/oracle/oradata/ORA19C/control01.ctl' scope = spfile;
```

![image.png](/assets/20250714/11.png)

```sql
-- 그룹 - 멤버 1:1 
SELECT * FROM v$log;
```

![image.png](/assets/20250714/12.png)

```sql
SELECT * FROM v$logfile;
```

![image.png](/assets/20250714/13.png)

```sql
-- 📍 pfile 도 맞추기
SYS@ora19c> create pfile from spfile;
File created.

-- 📍 datafile 확인
SYS@ora19c> select name from v$datafile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/system01.dbf
/u01/app/oracle/oradata/ORA19C/sysaux01.dbf
/u01/app/oracle/oradata/ORA19C/undotbs01.dbf
/u01/app/oracle/oradata/ORA19C/users01.dbf

-- 📍 tempfile 확인
SYS@ora19c> select name from v$tempfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/temp01.dbf

-- 📍 Controlfile 확인 
SYS@ora19c> select name from v$controlfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/control01.ctl

-- 📍 log file 확인
SYS@ora19c> select member from v$logfile;

MEMBER
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/redo03.log
/u01/app/oracle/oradata/ORA19C/redo02.log
/u01/app/oracle/oradata/ORA19C/redo01.log

-- ▶️ DB 정상적 종료 (NO-ARCAIVE MODE)

SYS@ora19c> shutdown immediate
SYS@ora19c> startup

-- 다시 확인
-- 📍 datafile 확인
SYS@ora19c> select name from v$datafile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/system01.dbf
/u01/app/oracle/oradata/ORA19C/sysaux01.dbf
/u01/app/oracle/oradata/ORA19C/undotbs01.dbf
/u01/app/oracle/oradata/ORA19C/users01.dbf

-- 📍 tempfile 확인
SYS@ora19c> select name from v$tempfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/temp01.dbf

-- 📍 Controlfile 확인 
SYS@ora19c> select name from v$controlfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/control01.ctl

-- 📍 log file 확인
SYS@ora19c> select member from v$logfile;

-- 확인하고 DB 종료
SYS@ora19c> shutdown immediate

-- os로 나가기
!

-- 위치 확인 
[oracle@oracle ~]$ pwd
/home/oracle
[oracle@oracle ~]$ mkdir dontouch   -- 백업 목적 디렉토리 생성

[oracle@oracle ~]$ cd dontouch
[oracle@oracle dontouch]$ pwd
/home/oracle/dontouch

-- 파일 복사
ls
cp -v /u01/app/oracle/oradata/ORA19C/*.* .   -- 반드시 백업되었는지 확인 
cp -v $ORACLE_HOME/dbs/initora19c.ora .

[oracle@oracle ~]$ cd /u01/app/oracle/oradata/ORA19C/
[oracle@oracle ORA19C]$ ls
control01.ctl  redo02.log  sysaux01.dbf  temp01.dbf     users01.dbf
redo01.log     redo03.log  system01.dbf  undotbs01.dbf

-- 원본 삭제
[oracle@oracle ORA19C]$ rm -f *
[oracle@oracle ORA19C]$ exit
exit

SYS@ora19c> startup
ORACLE instance started.

Total System Global Area  713027608 bytes
Fixed Size                  8900632 bytes
Variable Size             507510784 bytes
Database Buffers          188743680 bytes
Redo Buffers                7872512 bytes

-- 원본 삭제되었으므로 에러남.
ORA-00205: error in identifying control file, check alert log for more info

SYS@ora19c> show parameter control_files

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
control_files                        string      /u01/app/oracle/oradata/ORA19C
                                                 /control01.ctl
                                                 
                                                 
                                                 SYS@ora19c> shutdown abort
ORACLE instance shut down.
SYS@ora19c> exit
Disconnected from Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0
[oracle@oracle admin]$ cd dontouch/
bash: cd: dontouch/: No such file or directory
[oracle@oracle admin]$ cd
[oracle@oracle ~]$ cd dontouch
[oracle@oracle dontouch]$ ls
control01.ctl   redo01.log  redo03.log    system01.dbf  undotbs01.dbf
initora19c.ora  redo02.log  sysaux01.dbf  temp01.dbf    users01.dbf

-- 복구 작
[oracle@oracle dontouch]$ cp -v *.{dbf,ctl,log} /u01/app/oracle/oradata/ORA19C/
‘sysaux01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/sysaux01.dbf’
‘system01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/system01.dbf’
‘temp01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/temp01.dbf’
‘undotbs01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/undotbs01.dbf’
‘users01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/users01.dbf’
‘control01.ctl’ -> ‘/u01/app/oracle/oradata/ORA19C/control01.ctl’
‘redo01.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo01.log’
‘redo02.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo02.log’
‘redo03.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo03.log’
[oracle@oracle dontouch]$ ls ^C
[oracle@oracle dontouch]$ ls  /u01/app/oracle/oradata/ORA19C/
control01.ctl  redo02.log  sysaux01.dbf  temp01.dbf     users01.dbf
redo01.log     redo03.log  system01.dbf  undotbs01.dbf

[oracle@oracle dontouch]$ sqlplus / as sysdba   -- 접속하기

SQL*Plus: Release 19.0.0.0.0 - Production on Mon Jul 14 16:38:33 2025
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.

Connected to an idle instance.

SYS@ora19c> startup   -- 다시 시작
ORACLE instance started.   -- 정상적으로 시작됨

Total System Global Area  713027608 bytes
Fixed Size                  8900632 bytes
Variable Size             507510784 bytes
Database Buffers          188743680 bytes
Redo Buffers                7872512 bytes
Database mounted.
Database opened.

```

📍 **DBA(Database Administrator) 역할**

🌳 **오라클 관리**

- 스토리지
- 메모리
- 객체
- 유저
- lock
- 패스워드, 리소스
- 백업, 복구
    - 항상 데이터베이스를 사용할 수 있도록 유지
    - 시스템 장애를 최소화하기 위해 예방 조치를 취할 수 있어야 한다.
    - 장애 발생 시 가능한 빨리 데이터베이스 작동하도록 만들어 데이터 손실 최소화해야 한다.
    - 모든 유형의 장애로부터 데이터를 보호하기 위해 DBA는 정기적으로 데이터베이스를 백업해야 한다.
    - 백업은 여러 유형의 장애로부터 데이터를 복구하는데 필요하다.
    

📍 **Backup 용어**

1️⃣ Whole database backup

- 모든 data file, control file, redo log file 백업 받는다.
- 데이터베이스 open 상태이거나,  shutdown 되어 있을 때 백업
- **noarchive log mode, archive log mode** 모두 가능하다.

2️⃣ Partial database backup

- 특정한 테이블 스페이스에 속한 데이터 파일 백업
- archive log mode에서만 가능하다.

3️⃣ 일관성 있는 백업 (Consistent backup)

- closed backup, cold backup, offline backup
- 데이터베이스를 정상적인 종료 후에 백업을 수행 (shudown normal | transactional | immediate)
- 모든 데이터 파일, 컨트롤 파일, 리두로그파일의 checkpoint 정보가 동일하다.
- noarchive log mode, archive log mode 모두 가능하다
- noarchive log mode에서는 일관성 있는 백업만 수행해야 한다.

4️⃣ 일관성 없는 백업 (Inconsistent backup)

- open backup, hot backup, online backup
- 데이터베이스 운영 중에 백업 수행
- archive log mode에서만 가능하다.
- 데이터베이스 레벨, 테이블스페이스 레벨

5️⃣ Full back

- 선택한 데이터 파일에 속한 모든 데이터 블록 백업

6️⃣ Incremental backup

- 이전 full backup 이후에 변경된 블록에 대해서만 백업 ( scn 을 비교해서 변경되었는지 알 수 있음)
- RMAN 백업을 수행해야 한다.

📍 모드 조회 

```sql
SYS@ora19c> archive log list
Database log mode              **No Archive Mode**
Automatic archival             Disabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     9
Current log sequence           11

```