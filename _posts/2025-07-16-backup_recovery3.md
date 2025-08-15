---
title: "[58일차] Backup & Recovery3"
excerpt: "아이티윌 0716"
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-07-16T20:20
---

# 백업 & 리커버리3

- **📍 백업 받지 않은 테이블스페이스 손상된 경우 기존 위치가 아닌 새로운 위치로 복구 작업(리두 정보 존재)**
    
    ```sql
    -- 데이터 파일들 잘 있는지 확인
    select tablespace_name, file_name from dba_data_files;
    
    TABLESPACE_NAME                FILE_NAME
    ------------------------------ ----------------------------------------------------------------------
    SYSTEM                         /u01/app/oracle/oradata/ORA19C/system01.dbf
    SYSAUX                         /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
    USERS                          /u01/app/oracle/oradata/ORA19C/users01.dbf
    UNDOTBS1                       /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
    
    -- 새로운 테이블스페이스 생성
    create tablespace hr_tbs datafile '/u01/app/oracle/oradata/ORA19C/hr_tbs01.dbf' size 10m;
    
    -- 다시 확인
    select tablespace_name, file_name from dba_data_files;
    
    TABLESPACE_NAME                FILE_NAME
    ------------------------------ ----------------------------------------------------------------------
    SYSTEM                         /u01/app/oracle/oradata/ORA19C/system01.dbf
    SYSAUX                         /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
    HR_TBS                         /u01/app/oracle/oradata/ORA19C/hr_tbs01.dbf
    USERS                          /u01/app/oracle/oradata/ORA19C/users01.dbf
    UNDOTBS1                       /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
    ---> 하나가 추가되었다. 
    
    select * from v$log;  -- 가장 최근 체크 포인트  #2767692
    
    -- 새롭게 생성한 테이블스페이스 hr_tbs를 이용하는 테이블을 만들어보자.
    SYS@ora19c> create table hr.new(id number) tablespace hr_tbs;
    
    Table created.
    
    -- 데이터 삽입
    SYS@ora19c> insert into hr.new(id) values(1);
    
    1 row created.
    
    SYS@ora19c> commit;
    
    Commit complete.
    
    -- 조회해보기.
    SYS@ora19c> select * from hr.new;
    
            ID
    ----------
             1
             
    -- 장애 유발 (백업 받지 않은 상태에서) 
    SYS@ora19c> ! ls /u01/app/oracle/oradata/ORA19C/hr_tbs01.dbf
    /u01/app/oracle/oradata/ORA19C/hr_tbs01.dbf
    
    --파일이 손상됨 
    SYS@ora19c> ! rm /u01/app/oracle/oradata/ORA19C/hr_tbs01.dbf
    
    SYS@ora19c> ! ls /u01/app/oracle/oradata/ORA19C/hr_tbs01.dbf
    ls: cannot access /u01/app/oracle/oradata/ORA19C/hr_tbs01.dbf: No such file or directory
    
    select name, status from v$datafile;  -- 우리가 만든 테이블스페이스 사라짐
    
    SYS@ora19c> select name, status from v$datafile;   -- 다시 테스트해보니 이렇게 나오기는 하는데..
    
    NAME                                               STATUS
    -------------------------------------------------- -------
    /u01/app/oracle/oradata/ORA19C/system01.dbf        SYSTEM
    /u01/app/oracle/oradata/ORA19C/sysaux01.dbf        ONLINE
    /u01/app/oracle/oradata/ORA19C/undotbs01.dbf       ONLINE
    /u01/app/oracle/oradata/ORA19C/hr_tbs01.dbf        ONLINE
    /u01/app/oracle/oradata/ORA19C/users01.dbf         ONLINE
    
    -- 그래서 일부러 체크포인트해서 에러 유발 
    SYS@ora19c> alter system checkpoint;
    alter system checkpoint
    *
    ERROR at line 1:
    ORA-03113: end-of-file on communication channel
    Process ID: 25542
    Session ID: 261 Serial number: 53684
    
    -- 여기서 shutdown도 안되고, startup도 안되는 상황이 계속 벌어져서
    exit -- 하고
    
    sqlplus / as sysdba -- 재접
    
    select * from v$log; 
    
    -- 복구 작업 수행해야 할  대상 데이터 파일을 offline으로 
    alter database datafile '/u01/app/oracle/oradata/ORA19C/hr_tbs01.dbf' offline drop; -- 물리적인 파일 경로 적어도 된다.
    
    --확인
    select name, status from v$datafile;
    
    --백업본이 없는 손상된 데이터 파일에 대해서 물리적인 파일을 기존 위치가 아닌 새로운 위치로 생성
    alter database create datafile '/u01/app/oracle/oradata/ORA19C/hr_tbs01.dbf' as '/home/oracle/hr_tbs01.dbf';
    
    SYS@ora19c> ! ls /home/oracle/hr_tbs01.dbf
    /home/oracle/hr_tbs01.dbf
    
    select name, status from v$datafile;
    
    NAME
    --------------------------------------------------------------------------------
    STATUS
    -------
    /u01/app/oracle/oradata/ORA19C/system01.dbf
    SYSTEM
    
    /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
    ONLINE
    
    /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
    ONLINE
    
    NAME
    --------------------------------------------------------------------------------
    STATUS
    -------
    /home/oracle/hr_tbs01.dbf
    RECOVER
    
    /u01/app/oracle/oradata/ORA19C/users01.dbf
    ONLINE
    
    -- 테이블스페이스 생성 시점부터 현재까지 리두 정보를 이용해서 복구 작업 수행
    SYS@ora19c> alter database recover datafile '/home/oracle/hr_tbs01.dbf';
    
    Database altered.
    
    -- 또는 
    -- 📍 이렇게도 가능
    recover datafile '/home/oracle/hr_tbs01.dbf';
    
    -- 복구 작업 완료한 데이터 파일에 대해서 online으로 변경
    SYS@ora19c> alter database datafile '/home/oracle/hr_tbs01.dbf' online;
    
    Database altered.
    
    -- 확인 작업
    select name, status from v$datafile;
    
    -- new 테이블로 복구 완료 유무를 알 수 있다. (세그먼트 확인)
    select * from hr.new;
    
    -- 테이블스페이스 삭제
    drop tablespace hr_tbs including contents and datafiles;
    ```
    
- **📍 system 데이터 파일 손상된 경우 복구 (백업 이후 리두 정보가 있는 경우)**
    
    ```sql
    -- 원복 먼저. (불완전 복구)
    
    shutdown abort
    
    [oracle@oracle noarch]$ cp -v *.* /u01/app/oracle/oradata/ORA19C/
    ‘control01.ctl’ -> ‘/u01/app/oracle/oradata/ORA19C/control01.ctl’
    ‘redo01.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo01.log’
    ‘redo02.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo02.log’
    ‘redo03.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo03.log’
    ‘sysaux01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/sysaux01.dbf’
    ‘system01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/system01.dbf’
    ‘temp01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/temp01.dbf’
    ‘undotbs01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/undotbs01.dbf’
    ‘users01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/users01.dbf’
    [oracle@oracle noarch]$ ls
    control01.ctl  redo02.log  sysaux01.dbf  temp01.dbf     users01.dbf
    redo01.log     redo03.log  system01.dbf  undotbs01.dbf
    
    SYS@ora19c> startup
    
    SYS@ora19c> create table hr.new(id number) tablespace users;
    
    Table created.
    
    SYS@ora19c> insert into hr.new(id) values(1);
    
    1 row created.
    
    SYS@ora19c> commit;
    
    -- 장애 유발
    SYS@ora19c> ! ls /u01/app/oracle/oradata/ORA19C/system01.dbf
    /u01/app/oracle/oradata/ORA19C/system01.dbf
    
    --파일이 손상됨 
    SYS@ora19c> ! rm /u01/app/oracle/oradata/ORA19C/system01.dbf
    
    -- 시스템 테이블스페이스 삭제했는데 오류가 나지 않는 이유?
    -- shared pool 의 데이터 딕셔너리 캐쉬, 데이터 버퍼 캐쉬에 아직 남아있어서
    -- alert.log에도 별 말 없다.
    
    select count(*) from sys.user$;
    
      COUNT(*)
    ----------
           138
    
    -- full 체크 포인트 유발
    SYS@ora19c> alter system checkpoint;
    alter system checkpoint
    *
    ERROR at line 1:
    ORA-03113: end-of-file on communication channel
    Process ID: 30456
    Session ID: 56 Serial number: 31015
    
    SYS@ora19c> conn / as sysdba
    Connected to an idle instance.
    
    SYS@ora19c> startup
    ORACLE instance started.
    
    Total System Global Area  713027608 bytes
    Fixed Size                  8900632 bytes
    Variable Size             507510784 bytes
    Database Buffers          188743680 bytes
    Redo Buffers                7872512 bytes
    Database mounted.
    ORA-01157: cannot identify/lock data file 1 - see DBWR trace file
    ORA-01110: data file 1: '/u01/app/oracle/oradata/ORA19C/system01.dbf'
    
    SYS@ora19c> select status from v$instance;
    
    STATUS
    ------------
    MOUNTED
    
    SYS@ora19c> select * from v$recover_file;
    
         FILE# ONLINE  ONLINE_
    ---------- ------- -------
    ERROR                                                                CHANGE#
    ----------------------------------------------------------------- ----------
    TIME          CON_ID
    --------- ----------
             1 ONLINE  ONLINE
    FILE NOT FOUND                                                             0
                       0
    
    SYS@ora19c> select name,status from v$datafile;
    
    NAME
    --------------------------------------------------------------------------------
    STATUS
    -------
    /u01/app/oracle/oradata/ORA19C/system01.dbf
    SYSTEM
    
    /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
    ONLINE
    
    /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
    ONLINE
    
    NAME
    --------------------------------------------------------------------------------
    STATUS
    -------
    /u01/app/oracle/oradata/ORA19C/users01.dbf
    ONLINE
    
    -- system datafile 백업본을 찾아서 restore
    SYS@ora19c> ! cp -v backup/noarch/system01.dbf  /u01/app/oracle/oradata/ORA19C
    ‘backup/noarch/system01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/system01.dbf’
    
    -- 백업 이후에 변경 정보를 이용해서 복구 작업 (리두 적용)
    recover tablespace system
    
    recover datafile 1
    recover datafile '/u01/app/oracle/oradata/ORA19C/system01.dbf'
    
    -- 데이터베이스 open
    alter database open;
    
    --
    SYS@ora19c> shutdown abort
    ORACLE instance shut down.
    
    ```
    
- **📍 system 데이터 파일 손상된 경우 복구 (백업 이후 리두 정보가 없는 경우)**
    
    ```sql
    -- 원복 먼저. (불완전 복구)
    
    shutdown abort
    
    [oracle@oracle noarch]$ cp -v *.* /u01/app/oracle/oradata/ORA19C/
    ‘control01.ctl’ -> ‘/u01/app/oracle/oradata/ORA19C/control01.ctl’
    ‘redo01.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo01.log’
    ‘redo02.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo02.log’
    ‘redo03.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo03.log’
    ‘sysaux01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/sysaux01.dbf’
    ‘system01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/system01.dbf’
    ‘temp01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/temp01.dbf’
    ‘undotbs01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/undotbs01.dbf’
    ‘users01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/users01.dbf’
    [oracle@oracle noarch]$ ls
    control01.ctl  redo02.log  sysaux01.dbf  temp01.dbf     users01.dbf
    redo01.log     redo03.log  system01.dbf  undotbs01.dbf
    
    SYS@ora19c> startup
    
    SYS@ora19c> select checkpoint_change#, scn_to_timestamp(checkpoint_change#) from v$datafile;
    
    CHECKPOINT_CHANGE#
    ------------------
    SCN_TO_TIMESTAMP(CHECKPOINT_CHANGE#)
    ---------------------------------------------------------------------------
               2834609
    14-JUL-25 11.43.14.000000000 PM
    
               2834609
    14-JUL-25 11.43.14.000000000 PM
    
               2834609
    14-JUL-25 11.43.14.000000000 PM
    
    CHECKPOINT_CHANGE#
    ------------------
    SCN_TO_TIMESTAMP(CHECKPOINT_CHANGE#)
    ---------------------------------------------------------------------------
               2834609
    14-JUL-25 11.43.14.000000000 PM
    
    -- 마지막 체크포인트 , 2번 리두 로그 파일에 존재
    SYS@ora19c> select * from v$log;
    
        GROUP#    THREAD#  SEQUENCE#      BYTES  BLOCKSIZE    MEMBERS ARC
    ---------- ---------- ---------- ---------- ---------- ---------- ---
    STATUS           FIRST_CHANGE# FIRST_TIM NEXT_CHANGE# NEXT_TIME     CON_ID
    ---------------- ------------- --------- ------------ --------- ----------
             1          1         10  209715200        512          1 NO
    INACTIVE               2661346 09-JUL-25      2767692 14-JUL-25          0
    
             2          1         11  209715200        512          1 NO
    CURRENT                2767692 14-JUL-25   1.8447E+19                    0
    
             3          1          9  209715200        512          1 NO
    INACTIVE               2527330 09-JUL-25      2661346 09-JUL-25          0
    
    -- 로그 스위치 유발
    SYS@ora19c> alter system switch logfile;
    
    System altered.
    
    ! rm /u01/app/oracle/oradata/ORA19C/system01.dbf
    
    -- 에러가 나야하는데 system altered 됨 (오라클이 인식 안된다.)
    SYS@ora19c> alter system checkpoint;
    
    System altered.
    
    -- 인식되게 하려면 정상적 종료해서 full checkpoint 유발 
    -- 이제서야 에러난다.
    -- alert.log에도 에러
    
    SYS@ora19c> shutdown immediate;
    ORA-01116: error in opening database file 1
    ORA-01110: data file 1: '/u01/app/oracle/oradata/ORA19C/system01.dbf'
    ORA-27041: unable to open file
    Linux-x86_64 Error: 2: No such file or directory
    Additional information: 3
    
    shutdown abort
    [oracle@oracle ~]$ sqlplus / as sysdba
    
    SQL*Plus: Release 19.0.0.0.0 - Production on Wed Jul 16 11:27:17 2025
    Version 19.3.0.0.0
    
    Copyright (c) 1982, 2019, Oracle.  All rights reserved.
    
    Connected to:
    Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
    Version 19.3.0.0.0
    
    SYS@ora19c> startup
    ORA-01081: cannot start already-running ORACLE - shut it down first
    SYS@ora19c> shutdown abort
    ORACLE instance shut down.
    SYS@ora19c> startup
    ORACLE instance started.
    
    Total System Global Area  713027608 bytes
    Fixed Size                  8900632 bytes
    Variable Size             507510784 bytes
    Database Buffers          188743680 bytes
    Redo Buffers                7872512 bytes
    Database mounted.
    ORA-01157: cannot identify/lock data file 1 - see DBWR trace file
    ORA-01110: data file 1: '/u01/app/oracle/oradata/ORA19C/system01.dbf'
    
    SYS@ora19c> select status from v$instance;
    
    STATUS
    ------------
    MOUNTED
    
    SYS@ora19c> ! cp -v backup/noarch/system01.dbf  /u01/app/oracle/oradata/ORA19C
    ‘backup/noarch/system01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/system01.dbf’
    
    -- 완전 복구 실패
    SYS@ora19c> recover tablespace system
    ORA-00279: change 2834606 generated at 07/14/2025 23:43:17 needed for thread 1
    ORA-00289: suggestion :
    /u01/app/oracle/fast_recovery_area/ORA19C/archivelog/2025_07_16/o1_mf_1_11_%u_.a
    rc
    ORA-00280: change 2834606 for thread 1 is in sequence #11
    
    Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
    
    -- 불완전 복구 다시 
    -- 모든 데이터 파일, 컨트롤 파일, 리두로그 파일 restore
    SYS@ora19c> shutdown abort
    ORACLE instance shut down.
    SYS@ora19c> cp -v *.* /u01/app/oracle/oradata/ORA19C/
    SP2-0734: unknown command beginning "cp -v *.* ..." - rest of line ignored.
    SYS@ora19c> !
    [oracle@oracle ~]$ cd backup/noarch
    [oracle@oracle noarch]$ cp -v *.* /u01/app/oracle/oradata/ORA19C/
    ‘control01.ctl’ -> ‘/u01/app/oracle/oradata/ORA19C/control01.ctl’
    ‘redo01.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo01.log’
    ‘redo02.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo02.log’
    ‘redo03.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo03.log’
    ‘sysaux01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/sysaux01.dbf’
    ‘system01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/system01.dbf’
    ‘temp01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/temp01.dbf’
    ‘undotbs01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/undotbs01.dbf’
    ‘users01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/users01.dbf’
    [oracle@oracle noarch]$ sqlplus / as sysdba
    
    SQL*Plus: Release 19.0.0.0.0 - Production on Wed Jul 16 11:32:26 2025
    Version 19.3.0.0.0
    
    Copyright (c) 1982, 2019, Oracle.  All rights reserved.
    
    Connected to an idle instance.
    
    SYS@ora19c> startup
    ORACLE instance started.
    
    Total System Global Area  713027608 bytes
    Fixed Size                  8900632 bytes
    Variable Size             507510784 bytes
    Database Buffers          188743680 bytes
    Redo Buffers                7872512 bytes
    Database mounted.
    Database opened.
    
    ```
    
- **📍 undo datafile 손상되었을 경우 복구(백업 이후 리두정보가 있다.)**
    
    ```sql
    SYS@ora19c> select * from v$log;
    
        GROUP#    THREAD#  SEQUENCE#      BYTES  BLOCKSIZE    MEMBERS ARC
    ---------- ---------- ---------- ---------- ---------- ---------- ---
    STATUS           FIRST_CHANGE# FIRST_TIM NEXT_CHANGE# NEXT_TIME     CON_ID
    ---------------- ------------- --------- ------------ --------- ----------
             1          1         10  209715200        512          1 NO
    INACTIVE               2661346 09-JUL-25      2767692 14-JUL-25          0
    
             2          1         11  209715200        512          1 NO
    CURRENT                2767692 14-JUL-25   1.8447E+19                    0
    
             3          1          9  209715200        512          1 NO
    INACTIVE               2527330 09-JUL-25      2661346 09-JUL-25          0
    
    SYS@ora19c> select a.file#, b.name tbs_name, a.name file_name, a.status, a.checkpoint_change#
    from v$datafile a, v$tablespace b
    where a.ts# = b.ts#;  
    
    SYS@ora19c> select segment_id, segment_name, owner, tablespace_name, status
    from dba_rollback_segs; 
    
    SYS@ora19c> select s.username, s.sid, s.serial#, r.name, t.xidusn, t.ubafil, t.ubablk, t.used_ublk
    from v$session s, v$transaction t, v$rollname r
    where s.taddr = t.addr
    and t.xidusn = r.usn; 
    
    -- ▶️ hr 세션
    HR@ora19c> select salary from hr.employees where employee_id = 200;
    
        SALARY
    ----------
          4400
    
    HR@ora19c> update hr.employees set salary = salary * 1.1 where employee_id = 200;
    
    1 row updated.
    
    -- ▶️ sys 세션
    SYS@ora19c> select s.username, s.sid, s.serial#, r.name, t.xidusn, t.ubafil, t.ubablk, t.used_ublk
    from v$session s, v$transaction t, v$rollname r
    where s.taddr = t.addr
    and t.xidusn = r.usn;   2    3    4
    
    USERNAME
    --------------------------------------------------------------------------------
           SID    SERIAL# NAME                               XIDUSN     UBAFIL
    ---------- ---------- ------------------------------ ---------- ----------
        UBABLK  USED_UBLK
    ---------- ----------
    HR
           275      24360 _SYSSMU8_399776867$                     8          4
          4628          1
    
    --📍 session kill 시키는 방법
    alter system kill session 'sid,serial#' immediate;
    
    -- 파일확인
    SYS@ora19c> ! ls /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
    /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
    
    -- undo datafile 손상
    SYS@ora19c> ! rm /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
    
    SYS@ora19c> alter system checkpoint;
    alter system checkpoint
    *
    ERROR at line 1:
    ORA-03113: end-of-file on communication channel
    Process ID: 30129
    Session ID: 237 Serial number: 52600
    
    SYS@ora19c> conn / as sysdba
    Connected to an idle instance.
    SYS@ora19c> startup
    ORACLE instance started.
    
    Total System Global Area  713027608 bytes
    Fixed Size                  8900632 bytes
    Variable Size             507510784 bytes
    Database Buffers          188743680 bytes
    Redo Buffers                7872512 bytes
    Database mounted.
    ORA-01157: cannot identify/lock data file 4 - see DBWR trace file
    ORA-01110: data file 4: '/u01/app/oracle/oradata/ORA19C/undotbs01.dbf'
    
    SYS@ora19c> select * from v$recover_file;
    
         FILE# ONLINE  ONLINE_
    ---------- ------- -------
    ERROR                                                                CHANGE#
    ----------------------------------------------------------------- ----------
    TIME          CON_ID
    --------- ----------
             4 ONLINE  ONLINE
    FILE NOT FOUND                                                             0
                       0
    
    -- undo datafile 백업본을 찾아서 restore
    SYS@ora19c>  ! cp -v backup/noarch/undotbs01.dbf  /u01/app/oracle/oradata/ORA19C
    ‘backup/noarch/undotbs01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/undotbs01.dbf’
    
    -- 백업 이후 변경 정보를 적용(리두 적용)
    
    -- 데이터베이스 open
    
    SYS@ora19c> recover datafile 4;
    Media recovery complete.
    
    ```
    
- **📍 undo datafile 손상되었을 경우 복구(백업 이후 리두정보가 없다.)**
    
    새로운 언두 테이블 스페이스를 생성해서 백업
    
    ```sql
    select * from v$log;
    
    SYS@ora19c> alter system switch logfile;
    
    System altered.
    
    ! rm /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
    ! cp -v backup/noarch/undotbs01.dbf  /u01/app/oracle/oradata/ORA19C
    
    shutdown abort
    
    [oracle@oracle ~]$ sqlplus / as sysdba
    
    SQL*Plus: Release 19.0.0.0.0 - Production on Wed Jul 16 13:49:58 2025
    Version 19.3.0.0.0
    
    Copyright (c) 1982, 2019, Oracle.  All rights reserved.
    
    Connected to an idle instance.
    
    SYS@ora19c> startup
    ORACLE instance started.
    
    Total System Global Area  713027608 bytes
    Fixed Size                  8900632 bytes
    Variable Size             507510784 bytes
    Database Buffers          188743680 bytes
    Redo Buffers                7872512 bytes
    Database mounted.
    ORA-01157: cannot identify/lock data file 4 - see DBWR trace file
    ORA-01110: data file 4: '/u01/app/oracle/oradata/ORA19C/undotbs01.dbf'
    
    SYS@ora19c> select status from v$instance;
    
    STATUS
    ------------
    MOUNTED
    
    SYS@ora19c>  ! cp -v backup/noarch/undotbs01.dbf  /u01/app/oracle/oradata/ORA19C
    ‘backup/noarch/undotbs01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/undotbs01.dbf’
    
    -- 백업 이후 변경 정보를 적용하려고 했지만 리두 정보가 없어서 복구 실
    SYS@ora19c> recover datafile 4;
    ORA-00279: change 2834606 generated at 07/14/2025 23:43:17 needed for thread 1
    ORA-00289: suggestion :
    /u01/app/oracle/fast_recovery_area/ORA19C/archivelog/2025_07_16/o1_mf_1_11_%u_.a
    rc
    ORA-00280: change 2834606 for thread 1 is in sequence #11
    
    Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
    
    select a.file#, b.name tbs_name, a.name file_name, a.status, a.checkpoint_change#
    from v$datafile a, v$tablespace b
    where a.ts# = b.ts#;
    
    -- 문제되는 undo datafile offline drop
    alter database datafile '/u01/app/oracle/oradata/ORA19C/undotbs01.dbf' offline drop;
    -- 📍 또는
    SYS@ora19c> alter database datafile 4 offline drop;
    
    Database altered.
    
    SYS@ora19c> alter database open;
    
    Database altered.
    
    -- active undo에서는 이런 행위 금지
    
    -- undo 잠시 offline 상태이므로,
    -- hr 유저로 접속해서
    -- select -> 0
    -- DML -> X 여야되는데 오류가 안난다. 확인해보기
    select s.username, s.sid, s.serial#, r.name, t.xidusn, t.ubafil, t.ubablk, t.used_ublk
    from v$session s, v$transaction t, v$rollname r
    where s.taddr = t.addr
    and t.xidusn = r.usn; 
    
    -- 시스템 undo 사용하고 있었음
    USERNAME
    -----------------------------------------------------------------------
           SID    SERIAL# NAME      XIDUSN     UBAFIL     UBABLK  USED_UBLK
    ---------- ---------- ------------------------------ ---------- -------
    
          SYS    237      SYSTEM     0          1
    			       547 
    				     61711   
    				     
    -- 1. HR 스키마에 테이블 생성 (테이블스페이스 USERS)
    create table hr.new(id number) tablespace users;
    
    -- 2. 데이터 삽입 시도
    insert into hr.new(id) values(1);	
    ORA-01552: cannot use system rollback segment for non-system tablespace 'USERS'	     
    
    -- 3. EMPLOYEES 테이블이 위치한 테이블스페이스 확인
    select tablespace_name from user_tables where table_name = 'EMPLOYEES';
    
    TABLESPACE_NAME
    ------------------------------
    SYSAUX
    
    drop table hr.new purge;
    
    -- 다시 sysaux : system tablespace 보조 
    -- 그런데 sysaux에 유저 정보 포함되면 좋지 않다.
    create table hr.new(id number) tablespace sysaux;
    Table created.
    
    HR@ora19c> insert into hr.new(id) values(1);   -- 잘 들어간다.
    
    1 row created.
    
    -- 테이블 드랍하
    -- sys session 
    
    create undo tablespace undo1 
    datafile '/u01/app/oracle/oradata/ORA19C/undo01.dbf' 
    size 10m autoextend on;
    
    select a.file#, b.name tbs_name, a.name file_name, a.status, a.checkpoint_change#
    from v$datafile a, v$tablespace b
    where a.ts# = b.ts#;
    
    select segment_id, segment_name, owner, tablespace_name, status
    from dba_rollback_segs;
    
    show parameter undo_tablespace
    
    NAME                                 TYPE        VALUE
    ------------------------------------ ----------- ------------------------------
    undo_tablespace                      string      UNDOTBS1
    
    alter system set undo_tablespace
    
    SYS@ora19c> alter system set undo_tablespace = undo1;
    
    System altered.
    
    NAME                                 TYPE        VALUE
    ------------------------------------ ----------- ------------------------------
    undo_tablespace                      string      UNDO1
    
    select s.username, s.sid, s.serial#, r.name, t.xidusn, t.ubafil, t.ubablk, t.used_ublk
    from v$session s, v$transaction t, v$rollname r
    where s.taddr = t.addr
    and t.xidusn = r.usn; 
    
    select a.file#, b.name tbs_name, a.name file_name, a.status, a.checkpoint_change#
    from v$datafile a, v$tablespace b
    where a.ts# = b.ts#;
    
    -- 문제가 생긴 undotbs1은 삭제.
    drop tablespace undotbs1 including contents and datafiles;
    ```
    
    📍 다시 처음부터 수행하기 위해서
    
    ```sql
    alter system set undo_tablespace = undotbs1 scope = spfile;
    shutdown abort;
    ! rm /u01/app/oracle/oradata/ORA19C/*.*
    ! ls /u01/app/oracle/oradata/ORA19C/*.*
    ! cp -v backup/noarch/*.* /u01/app/oracle/oradata/ORA19C/
    
    -- sys session
    select segment_id, segment_name, owner, tablespace_name,status
    from dba_rollback_segs;
    
    -- hr session
    update hr.employees set salary = salary * 1.1 where employee_id = 200;
    
    -- sys session
    -- 트랜잭션 정보 조회
    select s.username, s.sid, s.serial#, r.name, t.xidusn, t.ubafil, t.ubablk, t.used_ublk
    from v$session s, v$transaction t, v$rollname r
    where s.taddr = t.addr
    and t.xidusn = r.usn; 
    
    USERNAME
    --------------------------------------------------------------------------------
           SID    SERIAL# NAME                               XIDUSN     UBAFIL
    ---------- ---------- ------------------------------ ---------- ----------
        UBABLK  USED_UBLK
    ---------- ----------
    HR
           255      14578 _SYSSMU5_2101348960$                    5          4
          5902          1
    
    -- 트랜잭션 진행되고 있는데 파일 손상시킬 것
    ! ls /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
    ✅ 파일 존재 확인
    
    ! rm /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
    🗑️ Undo 데이터파일 삭제
    
    !ls /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
    ❌ 파일 없음 확인 (No such file or directory)
    
    alter system checkpoint;
    ❌ 오류 발생
    
    ORA-03113: end-of-file on communication channel
    
    SYS@ora19c> select * from v$recover_file;
    
         FILE# ONLINE  ONLINE_
    ---------- ------- -------
    ERROR                                                                CHANGE#
    ----------------------------------------------------------------- ----------
    TIME          CON_ID
    --------- ----------
             4 ONLINE  ONLINE
    FILE NOT FOUND                                                             0
                       0
    
    alter database datafile 4 offline drop;
    
    select a.file#, b.name tbs_name, a.name file_name, a.status, a.checkpoint_change#
    from v$datafile a, v$tablespace b
    where a.ts# = b.ts#;
    
    SYS@ora19c> alter database open;
    
    Database altered.
    
    SYS@ora19c> select count(*) from hr.employees;
    
      COUNT(*)
    ----------
           107
           
    create undo tablespace undo1 
    datafile '/u01/app/oracle/oradata/ORA19C/undo01.dbf' 
    size 10m autoextend on;       
    
    select segment_id, segment_name, owner, tablespace_name, status
    from dba_rollback_segs; 
    ```
    
    ![image.png](/assets/20250716/1.png)
    
    ```sql
    -- 새로운 undo 테이블 스페이스 설정
    alter system set undo_tablespace = undo1;
    
    -- 사용하고 있는 undo 테이블 스페이스
    show parameter undo_tablespace;
    
    NAME                                 TYPE        VALUE
    ------------------------------------ ----------- ------------------------------
    undo_tablespace                      string      UNDO1
    
    -- hr
    
    -- 🔍 undo가 필요한 행에 대한 조회 시도
    select salary from hr.employees where employee_id = 200;
    
    -- ❌ 오류 발생
    ORA-00376: file 4 cannot be read at this time  
    ORA-01110: data file 4: '/u01/app/oracle/oradata/ORA19C/undotbs01.dbf'
    
    -- ✅ 다른 행 조회는 정상 작동
    select salary from hr.employees where employee_id = 100;
    -- 결과: 24000
    
    -- 업데이트 문장 수행
    HR@ora19c> update hr.employees set salary = salary * 1.1 where employee_id = 100;
    
    -- sys
    -- 트랜잭션 작업 확인
    select s.username, s.sid, s.serial#, r.name, t.xidusn, t.ubafil, t.ubablk, t.used_ublk
    from v$session s, v$transaction t, v$rollname r
    where s.taddr = t.addr
    and t.xidusn = r.usn; 
    
    USERNAME
    --------------------------------------------------------------------------------
           SID    SERIAL# NAME                               XIDUSN     UBAFIL
    ---------- ---------- ------------------------------ ---------- ----------
        UBABLK  USED_UBLK
    ---------- ----------
    HR
            26       8693 _SYSSMU16_1792621023$                  16          5
           258          1
    
    -- undotbs01 테이블스페이스 삭제 시도
    -- 업데이트 문장 수행해서 이미 엮여있음. 오류가 난다.
    -- db 껐다 켜도 에러 해결 안된다.
    -- 밑의 저것들을 수동으로 offline 모드로 떨어뜨리고 online 으로 올려야지 해결된다.
    ```
    
   ![image.png](/assets/20250716/2.png)
    
    ```sql
    select segment_name||',' from dba_rollback_segs where status = 'NEEDS RECOVERY';
    
    --> 요렇게
    _offline_rollback_segments=(
    _SYSSMU1_1261223759$,
    _SYSSMU2_27624015$,
    _SYSSMU3_2421748942$,
    _SYSSMU4_625702278$,
    _SYSSMU5_2101348960$,
    _SYSSMU6_813816332$,
    _SYSSMU7_2329891355$,
    _SYSSMU8_399776867$,
    _SYSSMU9_1692468413$,
    _SYSSMU10_930580995$)
    
    -- pfile을 이용
    -- pfile 생성
    create pfile from spfile;
    shutdown immediate;
    
    SYS@ora19c> ! vi $ORACLE_HOME/dbs/initora19c.ora
    ```
    
   ![image.png](/assets/20250716/3.png)
    
    ```sql
    startup pfile='$ORACLE_HOME/dbs/initora19c.ora';
    
    SYS@ora19c> drop tablespace UNDOTBS1 including contents and datafiles;
    
    Tablespace dropped.
    
    select segment_id, segment_name, owner, tablespace_name, status
    from dba_rollback_segs; 
    
    -- 원복
    shutdown immediate
    alter system set undo_tablespace = undotbs1 scope = spfile;
    shutdown abort;
    ! rm /u01/app/oracle/oradata/ORA19C/*.*
    ! ls /u01/app/oracle/oradata/ORA19C/*.*
    ! cp -v backup/noarch/*.* /u01/app/oracle/oradata/ORA19C/
    ```
    
- **📍 temp 파일이 손상되었을 경우 복구**
    
    ```sql
    SYS@ora19c> select checkpoint_change#, scn_to_timestamp(checkpoint_change#) from v$database;
    
    CHECKPOINT_CHANGE#
    ------------------
    SCN_TO_TIMESTAMP(CHECKPOINT_CHANGE#)
    ---------------------------------------------------------------------------
               2834609
    14-JUL-25 11.43.14.000000000 PM
    
    SYS@ora19c> select * from v$tempfile;
    
         FILE# CREATION_CHANGE# CREATION_        TS#     RFILE# STATUS  ENABLED
    ---------- ---------------- --------- ---------- ---------- ------- ----------
         BYTES     BLOCKS CREATE_BYTES BLOCK_SIZE
    ---------- ---------- ------------ ----------
    NAME
    --------------------------------------------------------------------------------
        CON_ID
    ----------
             1          1921108 01-JUL-25          3          1 ONLINE  READ WRITE
      33554432       4096     20971520       8192
    /u01/app/oracle/oradata/ORA19C/temp01.dbf
             0
    
    SYS@ora19c> select tablespace_name, contents from dba_tablespaces;
    
    TABLESPACE_NAME                CONTENTS
    ------------------------------ ---------------------
    SYSTEM                         PERMANENT
    SYSAUX                         PERMANENT
    UNDOTBS1                       UNDO
    TEMP                           TEMPORARY
    USERS                          PERMANENT
    
    SYS@ora19c> select username, temporary_tablespace from dba_users;
    
    -- default 테이블스페이스 확인하는 방법
    SYS@ora19c> select property_value from database_properties where property_name = 'DEFAULT_TEMP_TABLESPACE';
    
    PROPERTY_VALUE
    ----------------
    TEMP
    
    SYS@ora19c> select * from dba_temp_files;
    
    FILE_NAME
    --------------------------------------------------------------------------------
       FILE_ID TABLESPACE_NAME                     BYTES     BLOCKS STATUS
    ---------- ------------------------------ ---------- ---------- -------
    RELATIVE_FNO AUT   MAXBYTES  MAXBLOCKS INCREMENT_BY USER_BYTES USER_BLOCKS
    ------------ --- ---------- ---------- ------------ ---------- -----------
    SHARED           INST_ID
    ------------- ----------
    /u01/app/oracle/oradata/ORA19C/temp01.dbf
             1 TEMP                             33554432       4096 ONLINE
               1 YES 3.4360E+10    4194302           80   32505856        3968
    SHARED
    
    SYS@ora19c> show parameter sort_area_size
    
    NAME                                 TYPE        VALUE
    ------------------------------------ ----------- ------------------------------
    sort_area_size                       integer     65536
    
    -- temp file 손상
    ! rm  /u01/app/oracle/oradata/ORA19C/temp01.dbf
    
    -- hr 세션
    select employee_id, salary from hr.employees order by 2 desc;
    
    HR@ora19c> select s.*, b.*
    from all_objects s, all_objects b
    order by 1,2,3,4;
    
    -- 계속 돌아가다가 오류날
    
    -- sys 세션
    -- temp 파일 삭제해서 오류
    
    SYS@ora19c> select s.*, b.*
    from all_objects s, all_objects b
    order by 1,2,3,4;  2    3
    from all_objects s, all_objects b
         *
    ERROR at line 2:
    ORA-01565: error in identifying file
    '/u01/app/oracle/oradata/ORA19C/temp01.dbf'
    ORA-27037: unable to obtain file status
    Linux-x86_64 Error: 2: No such file or directory
    Additional information: 7
    
    select name from v$tempfile;
    -- 해결 방법 : 새로운 템프 파일을 추가하자.
    alter tablespace temp add tempfile '/u01/app/oracle/oradata/ORA19C/temp02.dbf' size 10m;
    
    SYS@ora19c> select name from v$tempfile;
    
    NAME
    --------------------------------------------------------------------------------
    /u01/app/oracle/oradata/ORA19C/temp01.dbf
    /u01/app/oracle/oradata/ORA19C/temp02.dbf
    
    -- 논리적으로는 있지만 물리적으로는 존재하지 않은 temp01 파일...
    
    alter tablespace temp drop tempfile '/u01/app/oracle/oradata/ORA19C/temp01.dbf';
    
    SYS@ora19c> select * from v$tempfile;  -- temp02 파일만 있다.
    
    -- sys 세션
    -- temp file 크기가 작게 구성되어 있어서 temp segment를 생성할 수 없어서 오류 발
    select s.*, b.*
    from all_objects s, all_objects b
    order by 1,2,3,4;  
    
    ORA-01652: unable to extend temp segment by 128 in tablespace TEMP
    
    -- 해결 방안 : 1. 기존 temp file 크기를 조정  1152 -> 12672
    SYS@ora19c> select * from dba_temp_files;
    alter database tempfile '/u01/app/oracle/oradata/ORA19C/temp02.dbf' resize 100m;
    
    -- 2) 또는 기존 temp file 자동확장기능 활성화
    alter database tempfile '/u01/app/oracle/oradata/ORA19C/temp02.dbf' autoextend on;
    
    -- 3) 새로운 temp file 추가
    alter tablespace temp add tempfile '/u01/app/oracle/oradata/ORA19C/temp01.dbf' size 10m autoextend on; 
    
    ```
    
- **📍 temp file 손상되었을 경우 새로운 temp tablespace 생성하고  default**
    
    ```sql
    SYS@ora19c> ! ls /u01/app/oracle/oradata/ORA19C/temp{01,02}.dbf
    /u01/app/oracle/oradata/ORA19C/temp01.dbf  /u01/app/oracle/oradata/ORA19C/temp02.dbf
    
    SYS@ora19c> ! rm /u01/app/oracle/oradata/ORA19C/temp{01,02}.dbf
    
    SYS@ora19c> ! ls /u01/app/oracle/oradata/ORA19C/temp{01,02}.dbf
    ls: cannot access /u01/app/oracle/oradata/ORA19C/temp01.dbf: No such file or directory
    ls: cannot access /u01/app/oracle/oradata/ORA19C/temp02.dbf: No such file or directory
    
    -- temp file 를 사용해야 할 문장에 대해서만 오류 발생
    SYS@ora19c> select s.*, b.*
    from all_objects s, all_objects b
    order by 1,2,3,4;  2    3
    from all_objects s, all_objects b
         *
    ERROR at line 2:
    ORA-01565: error in identifying file '/u01/app/oracle/oradata/ORA19C/temp01.dbf'
    ORA-27037: unable to obtain file status
    Linux-x86_64 Error: 2: No such file or directory
    Additional information: 7
    
    -- 새로운 temp tablespace 생성
    create temporary tablespace temp_new tempfile '/u01/app/oracle/oradata/ORA19C/temp_new01.dbf' size 10m autoextend on;
    
    -- 새로운 temp tablespace를 default temp tablespace 지정
    alter database default temporary tablespace temp_new;
    
    SYS@ora19c> select property_value from database_properties where property_name = 'DEFAULT_TEMP_TABLESPACE';
    
    PROPERTY_VALUE
    --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
    TEMP_NEW
    
    -- 기존 temp tablespace 삭제
    drop tablespace temp including contents and datafiles;
    
    Tablespace dropped.
    
    --> 이렇게 정상 종료되었는데 다른 사람들은 삭제가 안된다고 한다.
    --> 해결 방법은? 
    --> temp를 사용하고 있는 sort usage를 조회해본다.
    select * from v$sort_usage;  -- 조회되는 행이 존재하면 사용하는 곳이 존재하는 것
    
    --> 존재한다면 이 쿼리를 수행해보라
    SELECT b.tablespace,b.segfile#,b.segblk#,b.blocks,a.sid,a.serial#,c.spid,a.username,a.osuser,a.status,a.sql_hash_value,c.pname
    FROM v$session a,v$sort_usage b,v$process c
    WHERE a.saddr = b.session_addr and a.paddr=c.addr
    ORDER BY b.tablespace, b.segfile#, b.segblk#, b.blocks;
    
    --> 돌고있는 세션 보이면 kill 시키기
    alter system kill session 'sid, serial#' immediate;
    
    --> 다시 drop문장을 수행해보면 정상적 수행된다.
    drop tablespace temp including contents and datafiles;
    ```
    
    📍 원상복구
    
    ```sql
    create temporary tablespace temp 
    tempfile '/u01/app/oracle/oradata/ORA19C/temp01.dbf' 
    size 10m autoextend on;
    
    -- 🔧 기본 TEMP 테이블스페이스 지정
    alter database default temporary tablespace temp;
    
    -- 🔍 현재 기본 TEMP 테이블스페이스 확인
    select property_value 
    from database_properties 
    where property_name = 'DEFAULT_TEMP_TABLESPACE';
    
    SYS@ora19c> select username, temporary_tablespace from dba_users;
    
    drop tablespace temp_new including contents and datafiles;
    ```