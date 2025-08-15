---
title: "[69일차] 복제 DB time recovery "
excerpt: "아이티윌 0801"
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-08-01T21:08
---

# 복제 DB time recovery

- 📍clone 1
    - ▶️ 확인 작업
        
        ```sql
        
        RMAN> report schema;
        -- 기본 테이블스페이스만 남기고 지우기.
        
        using target database control file instead of recovery catalog
        Report of database schema for database with db_unique_name ORA19C
        
        List of Permanent Datafiles
        ===========================
        File Size(MB) Tablespace           RB segs Datafile Name
        ---- -------- -------------------- ------- ------------------------
        1    920      SYSTEM               YES     /u01/app/oracle/oradata/ORA19C/system01.dbf
        3    720      SYSAUX               NO      /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
        4    340      UNDOTBS1             YES     /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
        7    5        USERS                NO      /u01/app/oracle/oradata/ORA19C/users01.dbf
        
        List of Temporary Files
        =======================
        File Size(MB) Tablespace           Maxsize(MB) Tempfile Name
        ---- -------- -------------------- ----------- --------------------
        1    20       TEMP                 32767       /u01/app/oracle/oradata/ORA19C/temp01.dbf
        
        -- 백업본 확인하기
        list backup;
        
        -- 백업셋 전부 삭제하기
        RMAN> delete backupset;
        
        allocated channel: ORA_DISK_1
        channel ORA_DISK_1: SID=273 device type=DISK
        
        List of Backup Pieces
        BP Key  BS Key  Pc# Cp# Status      Device Type Piece Name
        ------- ------- --- --- ----------- ----------- ----------
        25      25      1   1   AVAILABLE   DISK        /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_07_31/o1_mf_nnndf_TAG20250731T172500_n8pb2wyy_.bkp
        26      26      1   1   AVAILABLE   DISK        /u01/app/oracle/fast_recovery_area/ORA19C/autobackup/2025_07_31/o1_mf_s_1207934704_n8pb301m_.bkp
        
        Do you really want to delete the above objects (enter YES or NO)? y
        
        -- 특정한 백업셋 삭제
        delete backupset 10;
        
        ```
        
    - ▶️ 백업 받기
        
        ```sql
        backup database;
        
        -- 확인
        RMAN> list backup;
        
        List of Backup Sets
        ===================
        BS Key  Type LV Size       Device Type Elapsed Time Completion Time
        ------- ---- -- ---------- ----------- ------------ ---------------
        29      Full    1.32G      DISK        00:00:02     01-AUG-25
                BP Key: 29   Status: AVAILABLE  Compressed: NO  Tag: TAG20250801T**110546**
                Piece Name: /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_01/o1_mf_nnndf_TAG20250801T110546_n8r87t9g_.bkp
          List of Datafiles in backup set 29
          File LV Type Ckp SCN    Ckp Time  Abs Fuz SCN Sparse Name
          ---- -- ---- ---------- --------- ----------- ------ ----
          1       Full 4643487    01-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/system01.dbf
          3       Full 4643487    01-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
          4       Full 4643487    01-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
          7       Full 4643487    01-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/users01.dbf
        
        BS Key  Type LV Size       Device Type Elapsed Time Completion Time
        ------- ---- -- ---------- ----------- ------------ ---------------
        30      Full    10.30M     DISK        00:00:00     01-AUG-25
                BP Key: 30   Status: AVAILABLE  Compressed: NO  Tag: TAG20250801**T110549**
                Piece Name: /u01/app/oracle/fast_recovery_area/ORA19C/autobackup/2025_08_01/o1_mf_s_1207998349_n8r87xg1_.bkp
          SPFILE Included: Modification time: 01-AUG-25
          SPFILE db_unique_name: ORA19C
          Control File Included: Ckp SCN: 4643500      Ckp time: 01-AUG-25
        
          
        -- 리그 로그 정보 확인
        SELECT a.group#, b.sequence#, a.member, b.status, b.first_change#, to_char(b.first_time, 'yyyy-mm-dd hh24:mi:ss') first, b.next_change#, to_char(b.next_time, 'yyyy-mm-dd hh24:mi:ss') next  
        FROM v$logfile a, v$log b
        WHERE a.group# = b.group#
        ORDER BY b.group#;
        
        /*
        
        1	4	/u01/app/oracle/oradata/ORA19C/redo01.log	INACTIVE	4632790	2025-08-01 10:07:10	4637539	2025-08-01 10:26:58
        2	5	/u01/app/oracle/oradata/ORA19C/redo02.log	ACTIVE	4637539	2025-08-01 10:26:58	4643730	2025-08-01 11:07:18
        3	6	/u01/app/oracle/oradata/ORA19C/redo03.log	CURRENT	4643730	2025-08-01 11:07:18	18446744073709551615	
        */
        
        + 2차
        /*
        1	10	/u01/app/oracle/oradata/ORA19C/redo01.log	INACTIVE	4681875	2025-08-01 15:19:08	4795067	2025-08-01 16:54:46
        2	11	/u01/app/oracle/oradata/ORA19C/redo02.log	ACTIVE	4795067	2025-08-01 16:54:46	4805339	2025-08-01 17:36:03
        3	12	/u01/app/oracle/oradata/ORA19C/redo03.log	CURRENT	4805339	2025-08-01 17:36:03	18446744073709551615	
        */
        
        -- current 아카이브 받기
         alter system archive log current;
        
        -- 아카이브 로그 확인
        
        /*
        /home/oracle/arch1/arch_1_1_1207934646_.arc	4520750	4521899
        /home/oracle/arch2/arch_1_1_1207934646_.arc	4520750	4521899
        /home/oracle/arch1/arch_1_2_1207934646_.arc	4521899	4630193
        /home/oracle/arch2/arch_1_2_1207934646_.arc	4521899	4630193
        /home/oracle/arch1/arch_1_3_1207934646_.arc	4630193	4632790  -
        /home/oracle/arch2/arch_1_3_1207934646_.arc	4630193	4632790
        /home/oracle/arch1/arch_1_5_1207934646_.arc	4637539	2025-08-01 10:26:58	4643730	2025-08-01 11:07:18
        /home/oracle/arch2/arch_1_5_1207934646_.arc	4637539	2025-08-01 10:26:58	4643730	2025-08-01 11:07:18
        */
        
        + 2차
        /*
        /home/oracle/arch1/arch_1_11_1207934646_.arc	4795067	2025-08-01 16:54:46	4805339	2025-08-01 17:36:03
        /home/oracle/arch2/arch_1_11_1207934646_.arc	4795067	2025-08-01 16:54:46	4805339	2025-08-01 17:36:03
        ㅊ
        */
        
        -- 물리적 아카이브
        /*
        
        SYS@ora19c> ! ls arch*
        arch_1_1_1207934646_.arc  arch_1_3_1207934646_.arc  arch_1_5_1207934646_.arc  arch_1_7_1207927727_.arc
        arch_1_2_1207934646_.arc  arch_1_4_1207934646_.arc  arch_1_6_1207927727_.arc
        
        arch2:
        arch_1_1_1207934646_.arc  arch_1_3_1207934646_.arc  arch_1_5_1207934646_.arc  arch_1_7_1207927727_.arc
        arch_1_2_1207934646_.arc  arch_1_4_1207934646_.arc  arch_1_6_1207927727_.arc
        
        */
        
        ```
        
    - ▶️ 복제 데이터베이스 생성
        - ▶️ 초기 파라미터 파일(pfile) 생성
            
            ```sql
            create pfile='/home/oracle/clone/initclone.ora'from spfile;
            
            -- pfile 수정
            ! vi /home/oracle/clone/initclone.ora
            
            *.compatible='19.0.0'
            *.control_files='/home/oracle/clone/control01.ctl'
            *.db_name='clone'
            *.log_archive_dest_1='location=/home/oracle/clone mandatory'
            *.log_archive_format='arch_%t_%s_%r.arc'
            *.undo_tablespace='UNDOTBS1'
            ```
            
        - ▶️ 백업 (컨트롤 파일, 데이터 파일) 복사
            
            ```sql
            -- 데이터 파일 백업본
            /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_01/o1_mf_nnndf_TAG20250801T100426_n8r4ntjq_.bkp
            
            -- 컨트롤 파일 백업본
            /u01/app/oracle/fast_recovery_area/ORA19C/autobackup/2025_08_01/o1_mf_s_1207994669_n8r4nxos_.bkp
            
             cp -v /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_01/o1_mf_nnndf_TAG20250801T110546_n8r87t9g_.bkp /home/oracle/clone
            
             cp -v /u01/app/oracle/fast_recovery_area/ORA19C/autobackup/2025_08_01/o1_mf_s_1207998349_n8r87xg1_.bkp /home/oracle/clone
            
            ```
            
        - ▶️ 아카이브 파일 (백업 시점부터 목표시간까지 리두 정보 : 11:07:18)
            
            ```sql
            -- 
            select sequence#, name, first_change#, next_change# from v$archived_log;
            alter system archive log current
            
            /*
            /home/oracle/arch1/arch_1_3_1207934646_.arc	4630193	4632790  
            /home/oracle/arch2/arch_1_3_1207934646_.arc	4630193	4632790
            /home/oracle/arch1/arch_1_4_1207934646_.arc	4632790	4637539
            /home/oracle/arch2/arch_1_4_1207934646_.arc	4632790	4637539
            /home/oracle/arch2/arch_1_5_1207934646_.arc	4637539	2025-08-01 10:26:58	4643730	2025-08-01 11:07:18
            /home/oracle/arch1/arch_1_6_1207934646_.arc	4643730	2025-08-01 11:07:18	4645295	2025-08-01 11:13:10
            
            */
            
            ! cp -v /home/oracle/arch1/arch_1_3_1207934646_.arc clone/
            ! cp -v /home/oracle/arch2/arch_1_4_1207934646_.arc clone/
            ! cp -v /home/oracle/arch1/arch_1_5_1207934646_.arc clone/
            ! cp -v /home/oracle/arch1/arch_1_6_1207934646_.arc clone/
            
            ! ls clone/
            
            !
            cat /etc/oratab
            
            . oraenv
            clone
            sqlplus / as sysdba
            startup pfile=/home/oracle/clone/initclone.ora nomount;
            select instance_name, status from v$instance;
            exit
            
            rman auxiliary / 
            select instance_name, status from v$instance;
            
            ```
            
            ▶️ 목표 시간 
            
            ```sql
            run {
            set newname for datafile 1 to '/home/oracle/clone/system01.dbf';
            set newname for datafile 3 to '/home/oracle/clone/sysaux01.dbf';
            set newname for datafile 4 to '/home/oracle/clone/undotbs01.dbf';
            set newname for datafile 7 to '/home/oracle/clone/users01.dbf';
            set newname for tempfile 1 to '/home/oracle/clone/temp01.dbf';
            duplicate target database to 'clone'
            pfile='/home/oracle/clone/initclone.ora'
            nofilenamecheck
            backup location '/home/oracle/clone'
            	until time "to_date('2025-08-01 12:00:31','yyyy-mm-dd hh24:mi:ss')"
            logfile
            '/home/oracle/clone/redo01.log' size 50m,
            '/home/oracle/clone/redo02.log' size 50m,
            '/home/oracle/clone/redo03.log' size 50m;
            }
            
            ```
            
        
        - 잘 안되서 다시 했음.
        
        kill -9 필수 백그라운드 프로세서 중에 하나를 잡아서 죽이면 
        
        ```sql
        RMAN> list backup;
        
        List of Backup Sets
        ===================
        
        BS Key  Type LV Size       Device Type Elapsed Time Completion Time
        ------- ---- -- ---------- ----------- ------------ ---------------
        31      Full    1.32G      DISK        00:00:03     01-AUG-25
                BP Key: 31   Status: AVAILABLE  Compressed: NO  Tag: TAG20250801T115717
                Piece Name: /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_01/o1_mf_nnndf_TAG20250801T115717_n8rc8f7b_.bkp
          List of Datafiles in backup set 31
          File LV Type Ckp SCN    Ckp Time  Abs Fuz SCN Sparse Name
          ---- -- ---- ---------- --------- ----------- ------ ----
          1       Full 4651779    01-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/system01.dbf
          3       Full 4651779    01-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
          4       Full 4651779    01-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
          7       Full 4651779    01-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/users01.dbf
        
        BS Key  Type LV Size       Device Type Elapsed Time Completion Time
        ------- ---- -- ---------- ----------- ------------ ---------------
        32      Full    10.30M     DISK        00:00:00     01-AUG-25
                BP Key: 32   Status: AVAILABLE  Compressed: NO  Tag: TAG20250801T115724
                Piece Name: /u01/app/oracle/fast_recovery_area/ORA19C/autobackup/2025_08_01/o1_mf_s_1208001444_n8rc8ncl_.bkp
          SPFILE Included: Modification time: 01-AUG-25
          SPFILE db_unique_name: ORA19C
          Control File Included: Ckp SCN: 4651800      Ckp time: 01-AUG-25
        
        --> ls -l /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_01/o1_mf_nnndf_TAG20250801T115717_n8rc8f7b_.bkp
        -- 시간나온다.
        
        -- 리두 정보
        1	7	/u01/app/oracle/oradata/ORA19C/redo01.log	CURRENT	4645295	2025-08-01 **11:13:10**	18446744073709551615	
        2	5	/u01/app/oracle/oradata/ORA19C/redo02.log	INACTIVE	4637539	2025-08-01 10:26:58	4643730	2025-08-01 11:07:18
        3	6	/u01/app/oracle/oradata/ORA19C/redo03.log	INACTIVE	4643730	2025-08-01 11:07:18	4645295	2025-08-01 11:13:10
        
        --
        1	7	/u01/app/oracle/oradata/ORA19C/redo01.log	INACTIVE	4645295	2025-08-01 11:13:10	4652220	2025-08-01 12:00:31
        2	8	/u01/app/oracle/oradata/ORA19C/redo02.log	CURRENT	4652220	2025-08-01 12:00:31	18446744073709551615	
        3	6	/u01/app/oracle/oradata/ORA19C/redo03.log	INACTIVE	4643730	2025-08-01 11:07:18	4645295	2025-08-01 11:13:10
        
        /*
        
        /home/oracle/arch2/arch_1_4_1207934646_.arc	4632790	2025-08-01 10:07:10	4637539	2025-08-01 10:26:58
        /home/oracle/arch1/arch_1_5_1207934646_.arc	4637539	2025-08-01 10:26:58	4643730	2025-08-01 11:07:18
        /home/oracle/arch2/arch_1_5_1207934646_.arc	4637539	2025-08-01 10:26:58	4643730	2025-08-01 11:07:18
        /home/oracle/arch1/arch_1_6_1207934646_.arc	4643730	2025-08-01 11:07:18	4645295	2025-08-01 11:13:10
        /home/oracle/arch2/arch_1_6_1207934646_.arc	4643730	2025-08-01 11:07:18	4645295	2025-08-01 11:13:10
        /home/oracle/arch1/arch_1_7_1207934646_.arc	4645295	2025-08-01 11:13:10	4652220	2025-08-01 12:00:31
        /home/oracle/arch2/arch_1_7_1207934646_.arc	4645295	2025-08-01 11:13:10	4652220	2025-08-01 12:00:31
        
        */
        
        2025-08-01 12:00:31   --> 현재 리두로그를 보는 것임!
        ```
        
    
    🌳 clone db : rman auxiliary 한번만! 한번 만들어진 이후에는 rman target /
    
- 📍clone2
    
    ▶️ 초기 파라미터 파일
    
    ```sql
    /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_01/o1_mf_nnndf_TAG20250801T115717_n8rc8f7b_.bkp
    /u01/app/oracle/fast_recovery_area/ORA19C/autobackup/2025_08_01/o1_mf_s_1208001444_n8rc8ncl_.bkp
    
    # 초기 파라미터파일
    *.compatible='19.0.0'
    *.control_files='/home/oracle/clone/control01.ctl'
    *.db_name='clone'
    *.log_archive_dest_1='location=/home/oracle/clone mandatory'
    *.log_archive_format='arch_%t_%s_%r.arc'
    *.undo_tablespace='UNDOTBS1'
    *.db_file_name_convert=('/u01/app/oracle/oradata/ORA19C/','/home/oracle/clone/')
    *.log_file_name_convert=('/u01/app/oracle/oradata/ORA19C/','/home/oracle/clone/')
    
    run {
    duplicate target database to 'clone'
    pfile='/home/oracle/clone/initclone.ora'
    nofilenamecheck
    backup location '/home/oracle/clone'
    	until time "to_date('2025-08-01 12:00:31','yyyy-mm-dd hh24:mi:ss')"
    }
    ```
    
- 📍 clone3
    
    ```sql
    -- 아카이브만 가지고 복제 DB 만들기
    
    run {
    duplicate target database to 'clone'
    pfile='/home/oracle/clone/initclone.ora'
    nofilenamecheck
    backup location '/home/oracle/clone'
    }
    ```
    
- 📍 clone4
    - ▶️ 현재 테이블 정보
        
        ```sql
        SELECT a.file#, b.name tbs_name, a.name file_name, a.status, a.checkpoint_change#
        FROM v$datafile a, v$tablespace b
        WHERE a.ts# = b.ts#;
        
        create tablespace insa_tbs datafile '/u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf' size 5m;
        create table hr.insa_2025 tablespace insa_tbs as select * from hr.employees;
        select count(*) from hr.insa_2025;
        ```
        
    - ▶️ rman
        
        ```sql
        report schema
        backup database;
        ```
        
    - ▶️ 백업
        
        ```sql
        backup database;
        
        List of Backup Sets
        ===================
        
        BS Key  Type LV Size       Device Type Elapsed Time Completion Time
        ------- ---- -- ---------- ----------- ------------ ---------------
        35      Full    1.33G      DISK        00:00:03     01-AUG-25
                BP Key: 35   Status: AVAILABLE  Compressed: NO  Tag: TAG20250801T150212
                Piece Name: /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_01/o1_mf_nnndf_TAG20250801T150212_n8rp34jv_.bkp
          List of Datafiles in backup set 35
          File LV Type Ckp SCN    Ckp Time  Abs Fuz SCN Sparse Name
          ---- -- ---- ---------- --------- ----------- ------ ----
          1       Full 4679119    01-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/system01.dbf
          3       Full 4679119    01-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
          4       Full 4679119    01-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
          5       Full 4679119    01-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf
          7       Full 4679119    01-AUG-25              NO    /u01/app/oracle/oradata/ORA19C/users01.dbf
        
        BS Key  Type LV Size       Device Type Elapsed Time Completion Time
        ------- ---- -- ---------- ----------- ------------ ---------------
        36      Full    10.30M     DISK        00:00:00     01-AUG-25
                BP Key: 36   Status: AVAILABLE  Compressed: NO  Tag: TAG20250801T150219
                Piece Name: /u01/app/oracle/fast_recovery_area/ORA19C/autobackup/2025_08_01/o1_mf_s_1208012539_n8rp3cdz_.bkp
          SPFILE Included: Modification time: 01-AUG-25
          SPFILE db_unique_name: ORA19C
          Control File Included: Ckp SCN: 4679140      Ckp time: 01-AUG-25
          
          
          
          SELECT a.group#, b.sequence#, a.member, b.status, b.first_change#, b.first_time, b.next_change#, b.next_time
        FROM v$logfile a, v$log b
        WHERE a.group# = b.group#
        ORDER BY b.group#;
        
        /*
        1	7	/u01/app/oracle/oradata/ORA19C/redo01.log	INACTIVE	4645295	25/08/01	4652220	25/08/01
        2	8	/u01/app/oracle/oradata/ORA19C/redo02.log	CURRENT	4652220	25/08/01	18446744073709551615	
        3	6	/u01/app/oracle/oradata/ORA19C/redo03.log	INACTIVE	4643730	25/08/01	4645295	25/08/01
        
        */
        
        alter system archive log current;
        
        select a.name, a.checkpoint_change#, b.status, b.change#, b.time
        from v$datafile a, v$backup b
        where a.file# = b.file#;
        
        /*
        
        /u01/app/oracle/oradata/ORA19C/system01.dbf	4679119	NOT ACTIVE	4063220	25/07/29
        /u01/app/oracle/oradata/ORA19C/sysaux01.dbf	4679119	NOT ACTIVE	4063220	25/07/29
        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf	4679119	NOT ACTIVE	4063220	25/07/29
        /u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf	4679119	NOT ACTIVE	0	
        /u01/app/oracle/oradata/ORA19C/users01.dbf	4679119	NOT ACTIVE	4063220	25/07/29
        
        /home/oracle/arch1/arch_1_8_1207934646_.arc	4652220	4679751
        /home/oracle/arch2/arch_1_8_1207934646_.arc	4652220	4679751
        
        */
        
        SYS@ora19c> select systimestamp from dual;
        
        SYSTIMESTAMP
        ---------------------------------------------------------------------------
        01-AUG-25 03.09.08.458745 PM +09:00
        
        SYS@ora19c> drop table hr.insa_2025 purge;
        -- 어느시점?
        -- drop 제외 리두로그 파일, 아카이브 로그에서 찾는다.  
        
        Table dropped.
        
        SELECT a.group#, b.sequence#, a.member, b.status, b.first_change#, to_char(b.first_time, 'yyyy-mm-dd hh24:mi:ss') first, b.next_change#, to_char(b.next_time, 'yyyy-mm-dd hh24:mi:ss') next  
        FROM v$logfile a, v$log b
        WHERE a.group# = b.group#
        ORDER BY b.group#;
        
        /*
        
        1	7	/u01/app/oracle/oradata/ORA19C/redo01.log	INACTIVE	4645295	2025-08-01 11:13:10	4652220	2025-08-01 12:00:31
        2	8	/u01/app/oracle/oradata/ORA19C/redo02.log	INACTIVE	4652220	2025-08-01 12:00:31	4679751	2025-08-01 15:06:40
        3	9	/u01/app/oracle/oradata/ORA19C/redo03.log	CURRENT	4679751	2025-08-01 15:06:40	18446744073709551615	
        
        */
        
        select sequence#,  
               name,  
               first_change#,  
               to_char(first_time, 'yyyy-mm-dd hh24:mi:ss') first,  
               next_change#,  
               to_char(next_time, 'yyyy-mm-dd hh24:mi:ss') next  
        from   v$archived_log;
        
        /*
        
        /home/oracle/arch1/arch_1_9_1207934646_.arc	4679751	2025-08-01 15:06:40	4681875	2025-08-01 15:19:08
        /home/oracle/arch2/arch_1_9_1207934646_.arc	4679751	2025-08-01 15:06:40	4681875	2025-08-01 15:19:08
        
        */
        SYS@ora19c> ! ls arch*
        
        arch1:
        arch_1_6_1207927727_.arc  arch_1_7_1207927727_.arc  arch_1_8_1207934646_.arc  arch_1_9_1207934646_.arc
        
        arch2:
        arch_1_6_1207927727_.arc  arch_1_7_1207927727_.arc  arch_1_8_1207934646_.arc  arch_1_9_1207934646_.arc
        
        ```
        
    - ▶️ pfile 생성
        
        ```sql
        create pfile='/home/oracle/clone/initclone.ora'from spfile;
        
        # 초기 파라미터파일
        *.compatible='19.0.0'
        *.control_files='/home/oracle/clone/control01.ctl'
        *.db_name='clone'
        *.log_archive_dest_1='location=/home/oracle/clone mandatory'
        *.log_archive_format='arch_%t_%s_%r.arc'
        *.undo_tablespace='UNDOTBS1'
        *.db_file_name_convert=('/u01/app/oracle/oradata/ORA19C/','/home/oracle/clone/')
        *.log_file_name_convert=('/u01/app/oracle/oradata/ORA19C/','/home/oracle/clone/')
        ```
        
    - ▶️ 백업 컨트롤 파일, 데이터 파일, 아카이브 파일 복사.
        
        ```sql
        cp -v /u01/app/oracle/fast_recovery_area/ORA19C/backupset/2025_08_01/o1_mf_nnndf_TAG20250801T150212_n8rp34jv_.bkp .
        cp -v /u01/app/oracle/fast_recovery_area/ORA19C/autobackup/2025_08_01/o1_mf_s_1208012539_n8rp3cdz_.bkp .
        ```
        
    - ▶️ 아카이브 파일 (백업 시점부터 목표시간까지 리두 정보 : 15:06:40)
        
        ```sql
        
        cp -v /home/oracle/arch1/arch_1_9_1207934646_.arc .
        cp -v /home/oracle/arch1/arch_1_8_1207934646_.arc .
        
        ```
        
        ▶️ 목표 시간 
        
        ```sql
        run {
        	duplicate target database to 'clone'
        	tablespace insa_tbs
        	pfile='/home/oracle/clone/initclone.ora'
        	nofilenamecheck
        	backup location '/home/oracle/clone'
        	until time "to_date('2025-08-01 15:06:40','yyyy-mm-dd hh24:mi:ss')";
        }
        
        ```
        
        +▶️ skip 
        
        ```sql
        run {
        	duplicate target database to 'clone'
        	skip tablespace users
        	tablespace insa_tbs
        	pfile='/home/oracle/clone/initclone.ora'
        	nofilenamecheck
        	backup location '/home/oracle/clone'
        	until time "to_date('','yyyy-mm-dd hh24:mi:ss')";
        }
        ```
        
        +▶️ suplemental 활성화
        
        ```sql
        alter database add supplemental log data;
        ```
        
- 📍 image copy backup
    
    백업 대상 파일을 1:1로 받는 백업을 의미한다.
    
    ▶️ 데이터 파일 image copy로 백업 수행
    
    ```sql
    RMAN> backup as copy database;
    ```
    
    ▶️ 컨트롤 파일 image copy로 백업 수행
    
    ```sql
    -- 내부적으로 alter database backup controlfile to '';
    backup as copy current controlfile;
    ```
    
    ▶️ 백업 셋으로 받은 백업 정보
    
    ```sql
    list backup;
    ```
    
    ▶️ image copy 형식으로 받은 백업 정보
    
    ```sql
    list copy;
    ```
    
    ▶️ 특정한 테이블스페이스에 대해서 image copy로 백업 받은 정보 확인
    
    ```sql
    list copy of tablespace system
    ```
    
    ▶️ 특정한 데이터 파일 번호에 대해서 image copy로 백업 받은 정보 확인
    
    ```sql
    list copy of datafile 1;
    ```
    
    ▶️ 컨트롤 파일에 대해서 image copy로 백업 받은 정보 확인
    
    ```sql
    list copy of controlfile;
    ```
    
    ▶️ 장애 발생
    
    ```sql
    shutdown immediate;
    
    rm /u01/app/oracle/oradata/ORA19C/users01.dbf
    
    SYS@ora19c> startup
    ORA-01081: cannot start already-running ORACLE - shut it down first
    SYS@ora19c> shutdown abort
    ORACLE instance shut down.
    SYS@ora19c> startup
    ORACLE instance started.
    
    Total System Global Area  713027608 bytes
    Fixed Size                  8900632 bytes
    Variable Size             578813952 bytes
    Database Buffers          117440512 bytes
    Redo Buffers                7872512 bytes
    Database mounted.
    ORA-01157: cannot identify/lock data file 7 - see DBWR trace file
    ORA-01110: data file 7: '/u01/app/oracle/oradata/ORA19C/users01.dbf'
    
    SYS@ora19c> select * from v$recover_file;
    
         FILE# ONLINE  ONLINE_
    ---------- ------- -------
    ERROR                                                                CHANGE#
    ----------------------------------------------------------------- ----------
    TIME          CON_ID
    --------- ----------
             7 ONLINE  ONLINE
    FILE NOT FOUND                                                             0
                       0
    
    SYS@ora19c> alter database datafile 7 offline;
    
    Database altered.
    
    SYS@ora19c> alter database open;
    
    Database altered.
    
    SYS@ora19c> exit
    Disconnected from Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
    Version 19.3.0.0.0
    [oracle@oracle ~]$ rman target /
    
    Recovery Manager: Release 19.0.0.0.0 - Production on Fri Aug 1 16:54:55 2025
    Version 19.3.0.0.0
    
    Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.
    
    connected to target database: ORA19C (DBID=1258537005)
    
    RMAN> report schema;
    
    using target database control file instead of recovery catalog
    Report of database schema for database with db_unique_name ORA19C
    
    List of Permanent Datafiles
    ===========================
    File Size(MB) Tablespace           RB segs Datafile Name
    ---- -------- -------------------- ------- ------------------------
    1    920      SYSTEM               YES     /u01/app/oracle/oradata/ORA19C/system01.dbf
    3    720      SYSAUX               NO      /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
    4    340      UNDOTBS1             YES     /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
    5    5        INSA_TBS             NO      /u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf
    7    0        USERS                NO      /u01/app/oracle/oradata/ORA19C/users01.dbf
    
    List of Temporary Files
    =======================
    File Size(MB) Tablespace           Maxsize(MB) Tempfile Name
    ---- -------- -------------------- ----------- --------------------
    1    20       TEMP                 32767       /u01/app/oracle/oradata/ORA19C/temp01.dbf
    
    RMAN> list failure;
    
    Database Role: PRIMARY
    
    List of Database Failures
    =========================
    
    Failure ID Priority Status    Time Detected Summary
    ---------- -------- --------- ------------- -------
    82         HIGH     OPEN      31-JUL-25     One or more non-system datafiles are missing
    222        HIGH     OPEN      31-JUL-25     One or more non-system datafiles need media recovery
    
    RMAN> list failure 82;
    
    Database Role: PRIMARY
    
    List of Database Failures
    =========================
    
    Failure ID Priority Status    Time Detected Summary
    ---------- -------- --------- ------------- -------
    82         HIGH     OPEN      31-JUL-25     One or more non-system datafiles are missing
    
    RMAN> list failure 82 detail;
    
    Database Role: PRIMARY
    
    List of Database Failures
    =========================
    
    Failure ID Priority Status    Time Detected Summary
    ---------- -------- --------- ------------- -------
    82         HIGH     OPEN      31-JUL-25     One or more non-system datafiles are missing
      Impact: See impact for individual child failures
      List of child failures for parent failure ID 82
      Failure ID Priority Status    Time Detected Summary
      ---------- -------- --------- ------------- -------
      1585       HIGH     OPEN      01-AUG-25     Datafile 7: '/u01/app/oracle/oradata/ORA19C/users01.dbf' is missing
        Impact: Some objects in tablespace USERS might be unavailable
    
    RMAN> list failure 222 detail;
    
    Database Role: PRIMARY
    
    List of Database Failures
    =========================
    
    Failure ID Priority Status    Time Detected Summary
    ---------- -------- --------- ------------- -------
    222        HIGH     OPEN      31-JUL-25     One or more non-system datafiles need media recovery
      Impact: See impact for individual child failures
      List of child failures for parent failure ID 222
      Failure ID Priority Status    Time Detected Summary
      ---------- -------- --------- ------------- -------
      797        HIGH     OPEN      31-JUL-25     Datafile 7: '/u01/app/oracle/oradata/ORA19C/users01.dbf' needs media recovery
        Impact: Some objects in tablespace USERS might be unavailable
    
    ```
    
    ▶️ 백업 정보 확인
    
    ```sql
    RMAN> list copy of datafile 7;
    
    List of Datafile Copies
    =======================
    
    Key     File S Completion Time Ckp SCN    Ckp Time        Sparse
    ------- ---- - --------------- ---------- --------------- ------
    9       7    A 01-AUG-25       4693695    01-AUG-25       NO
            Name: /u01/app/oracle/fast_recovery_area/ORA19C/datafile/o1_mf_users_n8rw26rb_.dbf
            Tag: TAG20250801T164358
    
    ```
    
    ▶️ restore 
    
    ```sql
    restore datafile 7;
    ```
    
    ▶️ recover
    
    ```sql
    recover datafile 7;
    
    alter database datafile 7 online;
    
    List of Permanent Datafiles
    ===========================
    File Size(MB) Tablespace           RB segs Datafile Name
    ---- -------- -------------------- ------- ------------------------
    1    920      SYSTEM               YES     /u01/app/oracle/oradata/ORA19C/system01.dbf
    3    720      SYSAUX               NO      /u01/app/oracle/oradata/ORA19C/sysaux01.dbf
    4    340      UNDOTBS1             YES     /u01/app/oracle/oradata/ORA19C/undotbs01.dbf
    5    5        INSA_TBS             NO      /u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf
    7    5        USERS                NO      /u01/app/oracle/oradata/ORA19C/users01.dbf
    
    List of Temporary Files
    =======================
    File Size(MB) Tablespace           Maxsize(MB) Tempfile Name
    ---- -------- -------------------- ----------- --------------------
    1    20       TEMP                 32767       /u01/app/oracle/oradata/ORA19C/temp01.dbf
    
    ```
    
- 📍 모든 데이터 파일 손상
    - ▶️ 장애 발생
        
        ```sql
        RMAN> shutdown immediate;
        
        database closed
        database dismounted
        Oracle instance shut down
        
        RMAN> exit
        
        Recovery Manager complete.
        [oracle@oracle ~]$ rm  /u01/app/oracle/oradata/ORA19C/*.dbf
        ```
        
    - ▶️ 복구 필요 파일 확인
        
        ```sql
        sqlplus / as sysdba
        SYS@ora19c> startup
        select * from v$recover_file
        ```
        
    - ▶️ restore
        
        ```sql
        restore database;
        ```
        
    - ▶️ recover
        
        ```sql
        recover database;
        ```
        
    
- 📍 파일 switch
    
    ```sql
    
    drop tablespace insa_tbs including contents and datafiles;
    create tablespace insa_tbs datafile '/home/oracle/insa_tbs01.dbf' size 5m;
    create table hr.insa_2025 tablespace insa_tbs as select * from hr.employees;
    
    SELECT a.file#, b.name tbs_name, a.name file_name, a.status, a.checkpoint_change#
    FROM v$datafile a, v$tablespace b
    WHERE a.ts# = b.ts#;
    
    backup as copy datafile 5 format '/u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf';
    list copy of datafile 5;
    
    shutdown immediate;
    ! rm /home/oracle/insa_tbs01.dbf
    
    startup
    -- non system datafile 손상
    alter database datafile 5 offline;
    alter database open; -- 서비스 띄우는게 제일 우선
    select count(*) from hr.employees;
    
    > rman
    list copy of datafile 5;
    
    -- 백업 파일을 운영 데이터 파일로 사용하는 방법 (restore 시간을 줄이자)
    > rman 
    switch datafile 문제되는 데이터 파일 번호 ex) 5 to copy;
    
    recover datafile 5;
    alter database datafile 5 online;
    select count(*) from hr.insa_2025;
    
    backup as copy datafile 5 format '/u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf';
    
    ```