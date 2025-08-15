---
title: "[73일차] table reorganization (테이블 재구성) "
excerpt: "아이티윌 0807_(2)"
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-08-07T21:08
---

## table reorganization (테이블 재구성)

▶️ 대상 테이블 : full table scan (듬성 듬성 데이터가 쌓여있는 테이블을 full table scan)

▶️ 방법

| 방법 | 특징 | 온라인 가능? | 인덱스 상태 |
| --- | --- | --- | --- |
| 1. `ALTER TABLE MOVE` | 가장 간단, 빠름 | ❌ No | UNUSABLE됨 |
| 2. `CTAS (Create Table As Select)` | 새 테이블로 복사 | ❌ No | 새로 생성 필요 |
| 3. `DBMS_REDEFINITION` | 온라인 재정의 (무중단) | ✅ Yes | 자동 복원 가능 |

▶️ shrink

▶️ reorg 온라인 패키지

- 📍 shrink ?
    
    ```sql
    -- 그룹 커밋이 수행되어 commit이 for loop 안에 들어갔다 하더라도 10000번이 아닌 1번이 수행된다.
    create table hr.reorg_test(id number,name varchar(60)) tablespace users;
    begin
    		for i in 1..10000 loop
    			insert into hr.reorg_test(id, name) values(i, 'table/index reorganization example');
    		end loop;
    		commit;
    end;
    /
    
    select extents, blocks, bytes/1024 
    from dba_segments
    where owner = 'HR'
    and segment_name = 'REORG_TEST';
    
    /*
      EXTENTS     BLOCKS BYTES/1024
    ---------- ---------- ----------
             9         72        576
    
    */
    
    select tablespace_name, extent_id, bytes from dba_extents where owner = 'HR' and segment_name = 'REORG_TEST';
    
    /*
    TABLESPACE_NAME                 EXTENT_ID      BYTES
    ------------------------------ ---------- ----------
    USERS                                   0      65536
    USERS                                   1      65536
    USERS                                   2      65536
    USERS                                   3      65536
    USERS                                   4      65536
    USERS                                   5      65536
    USERS                                   6      65536
    USERS                                   7      65536
    USERS                                   8      65536
    */
    
    alter table hr.reorg_test add constraint reorg_id_pk primary key(id);
    -- rowid 기반이므로 ACCESS 술어. 
    select constraint_name, constraint_type, index_name 
    from dba_constraints 
    where table_name = 'REORG_TEST';
    
    /*
    REORG_ID_PK	P	REORG_ID_PK
    */
    ```
    
    ▶️ 테이블 통계 정보 - 실시간이 아니다. gathering해야 한다. 
    
    ```sql
    select num_rows, blocks, avg_row_len from dba_tables where owner ='HR' and table_name = 'REORG_TEST';
    ```
    
    ▶️ 테이블 통계 수집
    
    ```sql
    execute dbms_stats.gather_table_stats('hr','reorg_test')
    select num_rows, blocks, avg_row_len from dba_tables where owner ='HR' and table_name = 'REORG_TEST';
    
    /*
      NUM_ROWS     BLOCKS AVG_ROW_LEN
    ---------- ---------- -----------
         10000         65          39
    */
    ```
    
    ▶️ SQL Developer - F10 랑 같은 기능 
    
    ```sql
    -- SQL Plus 
    explain plan for select * from hr.reorg_test where id = 100;
    select * from table(dbms_xplan.display);
    
    -- primary key 적용 -> rowid -> access 
    -- 아무런 제약 조건 x -> filter 
    ```
    
   ![image.png](/assets/20250807/1.png)
    
    ▶️ 삭제
    
    ```sql
    delete from hr.reorg_test where id > 100;
    commit;
    ```
    
    ▶️ 정보 조회 
    
    ```sql
    select num_rows, blocks, avg_row_len from dba_tables where owner ='HR' and table_name = 'REORG_TEST';
    ```
    
    ▶️ 테이블 통계 수집
    
    ```sql
    execute dbms_stats.gather_table_stats('hr','reorg_test')
    select num_rows, blocks, avg_row_len from dba_tables where owner ='HR' and table_name = 'REORG_TEST';
    
    /*
      NUM_ROWS     BLOCKS AVG_ROW_LEN
    ---------- ---------- -----------
         10000         65          39
    */
    
    select extents, blocks, bytes/1024 
    from dba_segments
    where owner = 'HR'
    and segment_name = 'REORG_TEST';
    
    /*
    
    -- ▶️ 세그먼트에서 저장하고 있는 byte 값 계산
    -- NUM_ROWS * AVG_ROW_LEN ,, 
       EXTENTS     BLOCKS BYTES/1024
    ---------- ---------- ----------
             9         72        576
    */
    ```
    
    ▶️ 데이터 펌프 이용
    
    ```sql
    select * from dba_directories;
    ```
    
   ![image.png](/assets/20250807/2.png)
    
    ▶️ 테이블에 대해서 export
    
    ```sql
    ! expdp system/oracle directory=pump_dir dumpfile=reorg_test.dmp tables=hr.reorg_test
    ```
    
    ▶️ 
    
    ```sql
    ! ls data_pump
    truncate table hr.reorg_test;
    select extents, blocks, bytes/1024
    from dba_segments
    where owner = 'HR'
    AND segment_name = 'REORG_TEST';
    
    /*
    
       EXTENTS     BLOCKS BYTES/1024
    ---------- ---------- ----------
             1          8         64
    
    */
    
    ! impdp system/oracle directory=pump_dir dumpfile=reorg_test.dmp tables=hr.reorg_test content=data_only
    select count(*) from hr.reorg_test
    
    /*
    
      COUNT(*)
    ----------
           100
    
    */
    
    execute dbms_stats.gather_table_stats('hr','reorg_test')
    select num_rows, blocks, avg_row_len from dba_tables where owner ='HR' and table_name = 'REORG_TEST';
    
    /*
      NUM_ROWS     BLOCKS AVG_ROW_LEN
    ---------- ---------- -----------
           100          4          38
    
    */
    
    ```
    
- 📍 대상 테이블 mv
    
    ▶️ 공간에 free 공간이 있어야 한다.
    
    ```sql
    -- 그룹 커밋이 수행되어 commit이 for loop 안에 들어갔다 하더라도 10000번이 아닌 1번이 수행된다.
    create table hr.reorg_test(id number,name varchar(60)) tablespace users;
    begin
    		for i in 1..10000 loop
    			insert into hr.reorg_test(id, name) values(i, 'table/index reorganization example');
    		end loop;
    		commit;
    end;
    /
    
    select extents, blocks, bytes/1024 
    from dba_segments
    where owner = 'HR'
    and segment_name = 'REORG_TEST';
    
    /*
      EXTENTS     BLOCKS BYTES/1024
    ---------- ---------- ----------
             9         72        576
    
    */
    
    select tablespace_name, extent_id, bytes from dba_extents where owner = 'HR' and segment_name = 'REORG_TEST';
    
    /*
    TABLESPACE_NAME                 EXTENT_ID      BYTES
    ------------------------------ ---------- ----------
    USERS                                   0      65536
    USERS                                   1      65536
    USERS                                   2      65536
    USERS                                   3      65536
    USERS                                   4      65536
    USERS                                   5      65536
    USERS                                   6      65536
    USERS                                   7      65536
    USERS                                   8      65536
    */
    
    alter table hr.reorg_test add constraint reorg_id_pk primary key(id);
    -- rowid 기반이므로 ACCESS 술어. 
    select constraint_name, constraint_type, index_name 
    from dba_constraints 
    where table_name = 'REORG_TEST';
    
    /*
    REORG_ID_PK	P	REORG_ID_PK
    */
    ```
    
    ▶️ 테이블 통계 정보 - 실시간이 아니다. gathering해야 한다. 
    
    ```sql
    select num_rows, blocks, avg_row_len from dba_tables where owner ='HR' and table_name = 'REORG_TEST';
    ```
    
    ▶️ 테이블 통계 수집
    
    ```sql
    execute dbms_stats.gather_table_stats('hr','reorg_test')
    select num_rows, blocks, avg_row_len from dba_tables where owner ='HR' and table_name = 'REORG_TEST';
    
    /*
      NUM_ROWS     BLOCKS AVG_ROW_LEN
    ---------- ---------- -----------
         10000         65          39
    */
    ```
    
    ▶️ SQL Developer - F10 랑 같은 기능 
    
    ```sql
    -- SQL Plus 
    explain plan for select * from hr.reorg_test where id = 100;
    select * from table(dbms_xplan.display);
    
    -- primary key 적용 -> rowid -> access 
    -- 아무런 제약 조건 x -> filter 
    ```
    
    ![image.png](/assets/20250807/1.png)
    
    ▶️ 삭제
    
    ```sql
    delete from hr.reorg_test where id > 100;
    commit;
    ```
    
    ▶️ 통계 정보 조회 
    
    ```sql
    select num_rows, blocks, avg_row_len from dba_tables where owner ='HR' and table_name = 'REORG_TEST';
    ```
    
    ▶️ 인덱스에 대한 상태 정보
    
    ```sql
    select index_name, status from dba_indexes where table_name = 'REORG_TEST';
    explain plan for select * from hr.reorg_test where id = 100;
    select * from table(dbms_xplan.display);
    
    /*
    PLAN_TABLE_OUTPUT
    --------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT            |             |     1 |    38 |     1   (0)|
     00:00:01 |
    
    |   1 |  TABLE ACCESS BY INDEX ROWID| REORG_TEST  |     1 |    38 |     1   (0)|
     00:00:01 |
    
    |*  2 |   INDEX UNIQUE SCAN         | REORG_ID_PK |     1 |       |     0   (0)|
     00:00:01 |
    */ 
    ```
    
    ▶️ 테이블 재구성
    
    - 이동 장소를 지정하지 않으면 원래 테이블스페이스 에 새로운 세그먼트를 생성하여 이동
    
    ```sql
    alter table hr.reorg_test move;
    ```
    
    ▶️ 테이블을 다른 테이블스페이스로 이관
    
    ```sql
    alter table hr.reorg_test move tablespace users;
    ```
    
    ▶️ 통계 수집
    
    ```sql
    execute dbms_stats.gather_table_stats('hr','reorg_test')
    select num_rows, blocks, avg_row_len from dba_tables where owner ='HR' and table_name = 'REORG_TEST';
    ```
    
    ▶️ 인덱스 조회 (테이블을 재구성한다면, 생성되어있는 인덱스들은 모두 UNUSABLE 상태로 바뀜
    
    → rowid 바뀌니까  → 꼭 인덱스도 재구성해주자.
    
    ```sql
    select index_name, status from dba_indexes where table_name = 'REORG_TEST';
    
    /*
    REORG_ID_PK	UNUSABLE
    */
    ```
    
    ▶️ 인덱스 재구성
    
    ```sql
    -- 이렇게 하면 운영중에도 가능.
    alter index hr.reorg_id_pk rebuild online;
    select index_name, status from dba_indexes where table_name = 'REORG_TEST';
    
    /*
    REORG_ID_PK	VALID
    */
    ```
    
    ▶️ 인덱스를 다른 테이블스페이스로 이관
    
    ```sql
    alter index hr.reorg_id_pk rebuild online tablespace users;
    ```
    
    ▶️ hr 유저가 가지고 있는 세그먼트(저장 공간이 필요한 객체)들
    
    ```sql
    select * from dba_segments where owner = 'HR';
    select segment_name,segment_type, tablespace_name from dba_segments where owner = 'HR';
    ```
    
    ![image.png](/assets/20250807/3.png)
    
    ▶️ 딕셔너리가 사용해야 할 SYSAUX (시스템 테이블스페이스 보조) 에 유저 관련 정보가 저장되어있어서 문제가 있다. 다른 테이블스페이스 이관 작업하자. 
    
    ```sql
    select 
      'alter table ' || owner || '.' || segment_name || ' move tablespace users;' 
    from 
      dba_segments 
    where 
      owner = 'HR' 
      and segment_type = 'TABLE';
      
    select 
      'alter index ' || owner || '.' || segment_name || ' rebuild online tablespace users;' 
    from 
      dba_segments 
    where 
      owner = 'HR' 
      and segment_type = 'INDEX';
    ```
    
    ▶️
    
    ```sql
    alter table HR.REORG_TEST move tablespace users;
    alter table HR.TIME move tablespace users;
    alter table HR.DEPARTMENTS move tablespace users;
    alter table HR.EMPLOYEES move tablespace users;
    alter table HR.ITWILL move tablespace users;
    alter table HR.LOCATIONS move tablespace users;
    alter table HR.REGIONS move tablespace users;
    alter table HR.JOB_HISTORY move tablespace users;
    alter table HR.DEPT move tablespace users;
    alter table HR.JOBS move tablespace users;
    alter table HR.INC_EMP move tablespace users;
    alter table HR.EMP_TEMP move tablespace users;
    alter table HR.EMP move tablespace users;
    
    alter index HR.REORG_ID_PK rebuild online tablespace users;
    alter index HR.COUNTRY_C_ID_PK rebuild online tablespace users;
    alter index HR.DEPT_LOCATION_IX rebuild online tablespace users;
    alter index HR.DEPT_ID_PK rebuild online tablespace users;
    alter index HR.EMP_JOB_IX rebuild online tablespace users;
    alter index HR.EMP_DEPARTMENT_IX rebuild online tablespace users;
    alter index HR.EMP_MANAGER_IX rebuild online tablespace users;
    alter index HR.EMP_EMP_ID_PK rebuild online tablespace users;
    alter index HR.EMP_NAME_IX rebuild online tablespace users;
    alter index HR.LOC_ID_PK rebuild online tablespace users;
    alter index HR.LOC_COUNTRY_IX rebuild online tablespace users;
    alter index HR.LOC_STATE_PROVINCE_IX rebuild online tablespace users;
    alter index HR.LOC_CITY_IX rebuild online tablespace users;
    alter index HR.REG_ID_PK rebuild online tablespace users;
    alter index HR.JHIST_EMP_ID_ST_DATE_PK rebuild online tablespace users;
    alter index HR.JHIST_EMPLOYEE_IX rebuild online tablespace users;
    alter index HR.JHIST_DEPARTMENT_IX rebuild online tablespace users;
    
    alter index HR.JHIST_JOB_IX rebuild online tablespace users;
    alter table hr.countries move tablespace users;
    ```
    
    ▶️ 
    
    ```sql
    select segment_name,segment_type, tablespace_name from dba_segments where owner = 'HR';
    ```
    
    ![image.png](/assets/20250807/4.png)
    
    ![image.png](/assets/20250807/5.png)
    
    ▶️ 코드성 테이블 (만약 자주 사용한다면 인덱스를 걸어 조회) - IOT ( 인덱스 구성 테이블) 
    
   ![image.png](/assets/20250807/6.png)
    
- 📍행 이동 활성화
    
    ```sql
    select f.tablespace_name, f.file_name, count(*)
    from dba_extents e, dba_data_files f
    where e.file_id = f.file_id
      and e.segment_name = 'REORG_TEST'
      and e.owner = 'HR'
    group by f.tablespace_name, f.file_name;
    ```
    
    ▶️ 행 이동 활성화
    
    ```sql
    alter table hr.reorg_test enable row movement;
    ```
    
    ▶️ 행 이동 작업 진행 (온라인 상태)
    
    ▶️ 행이 세그먼트 왼쪽 부분으로 최대한 많이 이동한다.
    
    ▶️ 행 이동 중에 DML 작업과 query를 실행할 수 있다.
    
    ```sql
    -- 촘촘하게 제일 왼쪽으로 당긴다. 
    alter table hr.reorg_test shrink space compact;
    select extents, blocks, bytes/1024 from dba_segments where owner = 'HR' and segment_name = 'REORG_TEST';
    ```
    
    ▶️ HWM(High Water Mark) 조정
    
    ▶️ hwm 조정 시점에는 DML작업은 수행할 수 없다.
    
    ```sql
    alter table hr.reorg_test shrink space;
    select extents, blocks, bytes/1024 from dba_segments where owner='HR' and segment_name = 'REORG_TEST';
    ```
    
    ▶️ 행 이동 비활성화
    
    ```sql
    alter table hr.reorg_test disable row movement;
    select owner, index_name, status from dba_indexes where table_name = 'REORG_TEST';
    
    /*
    HR	REORG_ID_PK	VALID
    */
    ```
    
    ▶️ shrink 작업
    
    - 현재 위치(in-place)에서 처리되는 온라인 작업
    - 인덱스 유지 관리 된다.
    - 트리거는 실행되지 않습니다.
    - ASSM 테이블스페이스에 상주하는 세그먼트에만 수행할 수 있다.
    
    ```sql
    select * from dba_tablespaces;
    ```
    
- 📍ctas → drop → rename, 제약 추가
    
    ▶️ 삭제
    
    ```sql
    drop table hr.reorg_test purge;
    ```
    
    ▶️
    
    ```sql
    -- 그룹 커밋이 수행되어 commit이 for loop 안에 들어갔다 하더라도 10000번이 아닌 1번이 수행된다.
    create table hr.reorg_test(id number,name varchar(60)) tablespace users;
    begin
    		for i in 1..10000 loop
    			insert into hr.reorg_test(id, name) values(i, 'table/index reorganization example');
    		end loop;
    		commit;
    end;
    /
    
    select extents, blocks, bytes/1024 
    from dba_segments
    where owner = 'HR'
    and segment_name = 'REORG_TEST';
    
    /*
      EXTENTS     BLOCKS BYTES/1024
    ---------- ---------- ----------
             9         72        576
    
    */
    
    select tablespace_name, extent_id, bytes from dba_extents where owner = 'HR' and segment_name = 'REORG_TEST';
    
    /*
    TABLESPACE_NAME                 EXTENT_ID      BYTES
    ------------------------------ ---------- ----------
    USERS                                   0      65536
    USERS                                   1      65536
    USERS                                   2      65536
    USERS                                   3      65536
    USERS                                   4      65536
    USERS                                   5      65536
    USERS                                   6      65536
    USERS                                   7      65536
    USERS                                   8      65536
    */
    
    alter table hr.reorg_test add constraint reorg_id_pk primary key(id);
    -- rowid 기반이므로 ACCESS 술어. 
    select constraint_name, constraint_type, index_name 
    from dba_constraints 
    where table_name = 'REORG_TEST';
    
    /*
    REORG_ID_PK	P	REORG_ID_PK
    */
    ```
    
    ▶️ 테이블 통계 정보 - 실시간이 아니다. gathering해야 한다. 
    
    ```sql
    select num_rows, blocks, avg_row_len from dba_tables where owner ='HR' and table_name = 'REORG_TEST';
    ```
    
    ▶️ 테이블 통계 수집
    
    ```sql
    execute dbms_stats.gather_table_stats('hr','reorg_test')
    select num_rows, blocks, avg_row_len from dba_tables where owner ='HR' and table_name = 'REORG_TEST';
    
    /*
      NUM_ROWS     BLOCKS AVG_ROW_LEN
    ---------- ---------- -----------
         10000         65          39
    */
    ```
    
    ▶️ SQL Developer - F10 랑 같은 기능 
    
    ```sql
    -- SQL Plus 
    explain plan for select * from hr.reorg_test where id = 100;
    select * from table(dbms_xplan.display);
    
    -- primary key 적용 -> rowid -> access 
    -- 아무런 제약 조건 x -> filter 
    ```
    
   ![image.png](/assets/20250807/7.png)
    
    ▶️ 삭제
    
    ```sql
    delete from hr.reorg_test where id > 100;
    commit;
    ```
    
    ▶️ ctas → drop → rename, 제약 추가
    
    ```sql
    create table hr.reorg_test_temp tablespace users as select * from hr.reorg_test;
    ```
    
    ▶️ 
    
    ```sql
    select extents, blocks, bytes/1024 from dba_segments where owner = 'HR' and segment_name = 'REORG_TEST_TEMP';
    
    /*
     EXTENTS     BLOCKS BYTES/1024
    ---------- ---------- ----------
             1          8         64
    
    */
    
    drop table hr.reorg_test purge;
    
    -- rename을 하기 위해서는 hr유저로 해야한다.
    conn hr/hr
    rename reorg_test_temp to reorg_test;
    HR@ora19c> select count(*) from hr.reorg_test;
    
      COUNT(*)
    ----------
           100
    
    conn / as sysdba
    
    -- 제약 조건 추가
    alter table hr.reorg_test add constraint reorg_id_pk primary key(id);
    execute dbms_stats.gather_table_stats('hr','reorg_test')
    select num_rows, blocks, avg_row_len from dba_tables where owner ='HR' and table_name = 'REORG_TEST';
    ```
    
    ▶️ 제약조건 여러 개 있을 땐, 메타데이터 뽑아서 하자.
    
- 📍dbms_redefinition 사용 (온라인 재정의 (무중단))
    
    ▶️ 삭제
    
    ```sql
    drop table hr.reorg_test purge;
    ```
    
    ▶️
    
    ```sql
    -- 그룹 커밋이 수행되어 commit이 for loop 안에 들어갔다 하더라도 10000번이 아닌 1번이 수행된다.
    create table hr.reorg_test(id number,name varchar(60)) tablespace users;
    begin
    		for i in 1..10000 loop
    			insert into hr.reorg_test(id, name) values(i, 'table/index reorganization example');
    		end loop;
    		commit;
    end;
    /
    
    select extents, blocks, bytes/1024 
    from dba_segments
    where owner = 'HR'
    and segment_name = 'REORG_TEST';
    
    /*
      EXTENTS     BLOCKS BYTES/1024
    ---------- ---------- ----------
             9         72        576
    
    */
    
    select tablespace_name, extent_id, bytes from dba_extents where owner = 'HR' and segment_name = 'REORG_TEST';
    
    /*
    TABLESPACE_NAME                 EXTENT_ID      BYTES
    ------------------------------ ---------- ----------
    USERS                                   0      65536
    USERS                                   1      65536
    USERS                                   2      65536
    USERS                                   3      65536
    USERS                                   4      65536
    USERS                                   5      65536
    USERS                                   6      65536
    USERS                                   7      65536
    USERS                                   8      65536
    */
    
    alter table hr.reorg_test add constraint reorg_id_pk primary key(id);
    -- rowid 기반이므로 ACCESS 술어. 
    select constraint_name, constraint_type, index_name 
    from dba_constraints 
    where table_name = 'REORG_TEST';
    
    /*
    REORG_ID_PK	P	REORG_ID_PK
    */
    ```
    
    ▶️ 테이블 통계 정보 - 실시간이 아니다. gathering해야 한다. 
    
    ```sql
    select num_rows, blocks, avg_row_len from dba_tables where owner ='HR' and table_name = 'REORG_TEST';
    ```
    
    ▶️ 테이블 통계 수집
    
    ```sql
    execute dbms_stats.gather_table_stats('hr','reorg_test')
    select num_rows, blocks, avg_row_len from dba_tables where owner ='HR' and table_name = 'REORG_TEST';
    
    /*
      NUM_ROWS     BLOCKS AVG_ROW_LEN
    ---------- ---------- -----------
         10000         65          39
    */
    ```
    
    ▶️ 임시 테이블 생성
    
    ```sql
    create table hr.reorg_test_temp(id number constraint reorg_temp_id_pk primary key, name varchar2(30)) 
    tablespace users;
    
     select constraint_name, constraint_type, index_name
    from dba_constraints
    where table_name = 'REORG_TEST_TEMP';  
    
    /*
    REORG_TEMP_ID_PK	P	REORG_TEMP_ID_PK
    */
    
    select object_name, object_id, data_object_id, 
           to_char(created, 'yyyy-mm-dd hh24:mi:ss') create_date
    from dba_objects 
    where owner = 'HR' 
      and object_name in ('REORG_TEST', 'REORG_TEST_TEMP');
      
    /*
    REORG_TEST	75138	75138	2025-08-07 14:54:04	2025-08-07 14:54:04	2025-08-07:14:54:04	VALID
    REORG_TEST_TEMP	75139	75139	2025-08-07 14:57:56	2025-08-07 14:57:56	2025-08-07:14:57:56	VALID
    
    */  
    
    -- pk 가 없으면 dbms_redefinition.cons_use_rowid
    exec dbms_redefinition.can_redef_table('hr','reorg_test', dbms_redefinition.cons_use_pk);
    
    --
    exec dbms_redefinition.start_redef_table('hr','reorg_test', 'reorg_test_temp', 'id id,name name')
    
    select extents, blocks, bytes/1024 from dba_segments where owner = 'HR' and segment_name = 'REORG_TEST';
    select extents, blocks, bytes/1024 from dba_segments where owner = 'HR' and segment_name = 'REORG_TEST_TEMP';
    select * from hr.reorg_test;
    select count(*) from hr.reorg_test_temp;
    select * from hr.reorg_test where id = 50;
    select * from hr.reorg_test_temp where id = 50;
    update hr.reorg_test set name = 'oracle' where id = 50;
    select * from hr.reorg_test where id = 50;
    
    -- temp 테이블은 예전 값을 가지고 있을 것 
    -- 운영 테이블
    delete from hr.reorg_test where id = 1;
    commit;
    
    select * from hr.reorg_test where id = 1;
    select * from hr.reorg_test_temp where id = 1;
    exec dbms_redefinition.sync_interim_table('hr','reorg_test','reorg_test_temp')
    select * from hr.reorg_test where id = 50;
    select * from hr.reorg_test_temp where id = 50;
    
    select * from hr.reorg_test where id = 1;
    select * from hr.reorg_test_temp where id = 1;
    exec dbms_redefinition.finish_redef_table('hr','reorg_test','reorg_test_temp')
    
    select object_name, object_id, data_object_id, 
           to_char(created, 'yyyy-mm-dd hh24:mi:ss') create_date
    from dba_objects 
    where owner = 'HR' 
      and object_name in ('REORG_TEST', 'REORG_TEST_TEMP');
    ```
    
    ![image.png](/assets/20250807/8.png)
    
   ![image.png](/assets/20250807/9.png)
    
    ▶️ 이렇게 바뀌었다.
    
    ▶️ 테이블에 대한 구조를 변경하는 작업에서도 사용된다..
    
    ```sql
    -- 제약조건 이름은 바뀌지 않았다. 
     select constraint_name, constraint_type, index_name
    from dba_constraints
    where table_name = 'REORG_TEST_TEMP';  
    
     select constraint_name, constraint_type, index_name
    from dba_constraints
    where table_name = 'REORG_TEST';  
    
    ```
    
    ![image.png](/assets/20250807/10.png)
    
    
    ```sql
    select * from hr.reorg_test;
    ```
    
- 📍 테이블 구조 바꿀 때 이 패키지를 이용해서 쉽게 하자. (수동으로 CTAS 하면 제약조건 다 사라지는데 이 패키지를 이용하면 사라지지 않는다.) (11g) → 온라인 중에서는 가급적 하지 말자.
    
    ▶️ 삭제
    
    ```sql
    drop table hr.reorg_test_temp purge;
    create table hr.reorg_test_temp(content varchar2(60), emp_id number constraint reorg_emp_id_pk primary key) tablespace users;
    execute dbms_redefinition.can_redef_table('hr','reorg_test', dbms_redefinition.cons_use_pk);
    execute dbms_redefinition.start_redef_table('hr','reorg_test','reorg_test_temp','id emp_id, name content');
    execute dbms_redefinition.sync_interim_table('hr','reorg_test','reorg_test_temp')
    execute dbms_redefinition.finish_redef_table('hr','reorg_test', 'reorg_test_temp')
    ```
    
    ▶️ 컬럼의 이름이랑 위치 바뀌었음
    
    ```sql
    SYS@ora19c> select * from hr.reorg_test where emp_id = 2;
    
    CONTENT                                                          EMP_ID
    ------------------------------------------------------------ ----------
    table/index reorganization example                                    2
    
    select constraint_name, constraint_type, index_name from dba_constraints where table_name = 'REORG_TEST';
    /*
    REORG_EMP_ID_PK	P	REORG_EMP_ID_PK
    */
    select constraint_name, constraint_type, index_name from dba_constraints where table_name = 'REORG_TEST_TEMP';
    /*
    REORG_ID_PK	P	REORG_ID_PK
    컬럼 구조 중요!
    */
    ```
    

## segment deffered

- partition되지 않은 일반 테이블(힙 테이블)을 생성하는 경우 첫 번째 행을 입력할 때까지는 세그먼트 생성이 지연된다.
- 이 기능은 기본적으로 deferred_segment_creation 파라미터를 true 설정하여 활성화된다.
- 세그먼트 생성단계
    - 테이블 생성 : 데이터 딕셔너리 작업
    - insert : 세그먼트 생성
- 이점
    - 디스크 공간 절약
    - 응용 프로그램의 설치 시간 향상
- deferred_segment_creation
    - alter system
    - alter session
- segment creation immediate
- segment creation deferred(기본값)

```sql
show parameter deferred_segment_creation

/*
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
deferred_segment_creation            boolean     TRUE

*/
create table hr.seg_test(id number, name varchar2(30)) tablespace users;
-- 세그먼트 할당 : 여유 공간 확인
-- 뼈대 정보만 딕셔너리 테이블에 생성
select segment_name, tablespace_name 
from dba_segments 
where owner = 'HR' 
  and segment_name = 'SEG_TEST';

select table_name, tablespace_name from dba_tables where owner = 'HR' and table_name = 'SEG_TEST';
/*
SEG_TEST	USERS
*/
select column_name, data_type, data_length from dba_tab columns where owner = 'HR' and table_name = 'SEG_TEST';
insert into hr.seg_test(id,name) values(1,'oracle');
SELECT * FROM dba_tables 
WHERE owner = 'HR' AND table_name = 'SEG_TEST';
rollback;
show parameter deferred_segment_creation
drop table hr.seg_test purge;
create table hr.seg_test(id number, name varchar2(30)) segment creation immediate tablespace users;

```

▶️ deferred segment creation 제한

- iot, , 클러스터화된 테이블에서는 사용할 수 없다.
- 딕셔너리 관리 방식 테이블스페이스에서는 사용할 수 없다.
- 인덱스와 같은 종속 객체에 대한 deferred segment creation을 직접 수행할 수 없다.