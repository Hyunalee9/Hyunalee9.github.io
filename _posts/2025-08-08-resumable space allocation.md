---
title: "[74일차] resumable space allocation "
excerpt: "아이티윌 0808"
categories:
      - ORACLE19c
tags:
      - ORACLE19c
      - TIL
last-modified-at: 2025-08-08T21:08
---

▶️ 대량의 데이터 입력 시.

## resumable space allocation

- 공간 할당이 실패한 경우 데이터베이스 작업 실행을 일시 중지 했다가 해결되면 자동으로 실행할 수 있는 기능
- 일시 중지 되는 경우
    - 공간 부족
    - max extent 수에 도달
    - user 가 사용할 수 있는 테이블스페이스 quota값 도달
        - select * from dba_ts_quotas where username =’HR”;

📍 특정한 테이블스페이스에 대해서 quota값 수정

```sql
alter user hr quota unlimited on users;
grant unlimited tablespace to hr; -- 어느 날짜에 부여했는지 꼭 적어놓기. (any 시스템 권한 따로 관리) 
select * from dba_sys_privs where grantee = 'HR';
```

📍 resumable space allocation 활성화

```sql
alter session enable resumable;
```

▶️ alter session 또는 alter system 문을 사용하여 resumable_timeout 파라미터를 0 이 아닌 값으로 설정

```sql
alter session set resumable_timeout = 3600; 
-- resumable space allocation 기능이 활성화되며 타임아웃이 발생하기 전까진 초단위로 대기한다.

alter session enable resumable timeout 3600;
alter session enable resumable; --📍기본 타임 아웃은 7200초(2시간) 
```

📍 resumable space allocation 비활성화

```sql
alter session disable resumable;
```

📍 활성화, 비활성화 확인

```sql
show parameter resumable_timeout
/*
NAME              TYPE    VALUE 
----------------- ------- ----- 
resumable_timeout integer 0  --> 비활성화
*/

```

- 📍 1
    
    ▶️ 테이블스페이스 생성 
    
    ```sql
    create tablespace insa_tbs
    datafile '/u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf' size 1m
    extent management local uniform size 64k
    segment space management auto;
    ```
    
    ▶️ 테이블 생성
    
    ```sql
    create table hr.test(id char(1000)) tablespace insa_tbs;
    
    begin
    	for i in 1..1000 loop
    		insert into hr.test(id) values(i);
    	end loop;
    end;
    /
    ```
    
     ![image.png](/assets/20250808/1.png)
    
    ▶️ 공간 부족, max extend에 도달하여 에러가 남.
    
    ```sql
    SYS@ora19c> select count(*) from hr.test;
    
      COUNT(*)
    ----------
             0
    
    -- 데이터도 입력되지 않음
    ```
    
    ▶️ resumable 기능 활성화
    
    ```sql
    alter session enable resumable; -- 활성화. 2시간동안 대기 
    
    begin
    	for i in 1..1000 loop
    		insert into hr.test(id) values(i);
    	end loop;
    end;
    /
    ```
    
   ![image.png](/assets/20250808/2.png)
    
    ▶️ alert.log
    
    ▶️ 다른 세션 창 - 문제 생긴 테이블 확인
    
    ```sql
    select * from dba_resumable;
    
    /*
    0	237	1			SUSPENDED	7200	08/07/25 19:14:01	08/07/25 19:14:01		User SYS(0), Session 237, Instance 1
    */
    
    select file_id, tablespace_name, file_name, bytes, maxbytes, autoextensible from dba_data_files;
    
    /*
    1	SYSTEM	/u01/app/oracle/oradata/ORA19C/system01.dbf	964689920	34359721984	YES
    3	SYSAUX	/u01/app/oracle/oradata/ORA19C/sysaux01.dbf	786432000	34359721984	YES
    5	FDA_TBS	/u01/app/oracle/oradata/ORA19C/fda_tbs01.dbf	18874368	34359721984	YES
    7	USERS	/u01/app/oracle/oradata/ORA19C/users01.dbf	15728640	34359721984	YES
    2	INSA_TBS	/u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf	1048576	0	NO
    4	UNDOTBS1	/u01/app/oracle/oradata/ORA19C/undotbs01.dbf	356515840	34359721984	YES
    
    */
    ```
    
    ▶️ 해결방법 
    
    1. autoextend 지정
    
    ```sql
    alter database datafile '/u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf' autoextend on next 1m;
    ```
    
    1. datafile resize
    
    ```sql
    alter database datafile '/u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf' resize 10m
    ```
    
    1. datafile 추가

```sql
alter tablespace insa_tbs add datafile '/u01/app/oracle/oradata/ORA19C/insa_tbs02.dbf' size 10m;
```

![image.png](/assets/20250808/3.png)

```sql
select * from dba_resumable;
```

▶️ 비활성화

```sql
alter session disable resumable;
-- 삭제
drop tablespace insa_tbs including contents and datafiles;
```

▶️ 일반 유저들은 권한이 없으면 수행할 수 없다. 

▶️ resumable 시스템 권한이 필요하다.

```sql
alter session enable resumable;
```

▶️ 권한 주기

```sql
grant resumable to hr;
```

▶️ 권한 조회

```sql
select * from dba_sys_privs where grantee = 'HR';

/*
GR PRIVILEGE                                ADM COM INH
-- ---------------------------------------- --- --- ---
HR CREATE DATABASE LINK                     NO  NO  NO
HR ALTER SESSION                            NO  NO  NO
HR CREATE VIEW                              NO  NO  NO
HR RESUMABLE                                NO  NO  NO
HR UNLIMITED TABLESPACE                     NO  NO  NO
HR CREATE SESSION                           NO  NO  NO
HR CREATE SEQUENCE                          NO  NO  NO
HR CREATE SYNONYM                           NO  NO  NO

*/

select * from user_sys_privs; -- hr

HR	**RESUMABLE**	NO	NO	NO
HR	UNLIMITED TABLESPACE	NO	NO	NO
HR	ALTER SESSION	NO	NO	NO
HR	CREATE SESSION	NO	NO	NO
HR	CREATE SYNONYM	NO	NO	NO
HR	CREATE DATABASE LINK	NO	NO	NO
HR	CREATE SEQUENCE	NO	NO	NO
HR	CREATE VIEW	NO	NO	NO/*

*/
```

▶️ 활성화

```sql
alter session neable resumable;
```

▶️ 비활성화

```sql
alter session disable resumable;
alter session set resumable_timeout = 3600;
revoke resumable from hr;
```

▶️ 테이블 삭제

```sql
drop table hr.test purge;

create table hr.test(
	id number constraint test_id_pk primary key,
	sal number constraint sal_ck check(sal > 1000)) tablespace users;

```

▶️ 제약 조건 조회

```sql
select constraint_name, constraint_type, search_condition, status, validated, index_name from dba_constraints where owner 'HR' and table_name = 'TEST';
select index_name, uniqueness, status from dba_indexes where owner = 'HR' and table_name = 'TEST' ;

/*
TEST_ID_PK	UNIQUE	VALID
*/
```

▶️ 데이터 삽입

```sql
insert into hr.test(id,sal) values(1,2000);
insert into hr.test(id,sal) values(1,2001);
insert into hr.test(id,sal) values(null,2001);
insert into hr.test(id,sal) values(2,1000);
rollback;

/*
1 행 이(가) 삽입되었습니다.

명령의 2 행에서 시작하는 중 오류 발생 -
insert into hr.test(id,sal) values(1,2001)
오류 보고 -
ORA-00001: 무결성 제약 조건(HR.TEST_ID_PK)에 위배됩니다

명령의 3 행에서 시작하는 중 오류 발생 -
insert into hr.test(id,sal) values(null,2001)
오류 보고 -
ORA-01400: NULL을 ("HR"."TEST"."ID") 안에 삽입할 수 없습니다

명령의 4 행에서 시작하는 중 오류 발생 -
insert into hr.test(id,sal) values(2,1000)
오류 보고 -
ORA-02290: 체크 제약조건(HR.SAL_CK)이 위배되었습니다

롤백 완료.
*/
```

▶️ disable novalidate(disable 기본값)

```sql
alter table hr.test disable novalidate constraint test_id_pk;
alter table hr.test disable constraint sal_ck;
select constraint_name, constraint_type, search_condition, status, validated, index_name from dba_constraints where owner = 'HR' and table_name = 'TEST';
```

![image.png](/assets/20250808/4.png)

## SQL*Loader

▶️ 외부 파일에서 오라클 데이터베이스의 테이블로 data load 하는 기능 

- Conventional  Load Path
    - 입력해야 할 데이터를 insert문을 사용하여 데이터를 load한다.
    - SGA 영역 data buffer cache를 이용한다.
    - commit을 사용하여 영구적으로 입력
    - redo entry 생성 ?
    
    📍 **장점**
    
    - 모든 제약 조건 시행
    - insert trigger 실행
    - 클러스터화된 테이블로 load할 수 있다.
    - 작업 진행 동안 다른 유저는 대상 테이블에 데이터를 변경할 수 있다.
    
    📍 단점
    
    ```sql
    drop table hr.test purge;
    
    create table hr.test(
    	id number constraint test_id_pk primary key,
    	name varchar2(30),
    	phone varchar2(151)) tablespace users;
    	
    vi insa.dat
    
    1,"james","010-9999-0000"
    2,"grace","010-7777-7777"
    3,"scott","010-8888-0000"
    3,"lucas","010-1004-1004"	
    ```
    
    ▶️ SQL*Loader을 제어할 수 있는 컨트롤 파일을 생성해야 한다.
    
    ```sql
    vi insa.ctl
    
    load data
    infile insa.dat
    badfile insa.bad
    - insert : 비어있는 테이블에 입력
    - replace : 기존 행을 삭제(delete)하고 입력
    - truncate : 기존 행을 truncate하고 입력
    - append : 기존 행 뒤에 추가 
    into table hr.test
    fields terminated by ',' optionally enclosed by '"'
    {id,name,phone}
    
    load data
    infile insa.dat
    badfile insa.bad
    truncate
    into table hr.test
    fields terminated by ',' optionally enclosed by '"'
    (id,name,phone)
    begindata
    1,"james","010-9999-0000"
    2,"grace","010-7777-7777"
    3,"scott","010-8888-0000"
    3,"lucas","010-1004-1004"	
    ```
    
    ![image.png](/assets/20250808/5.png)
    
   ![image.png](/assets/20250808/6.png)
    
    ```sql
    SQL> ! sqlldr hr/hr control=insa.ctl
    ```
    
    ![image.png](/assets/20250808/7.png)
    
- Direct Path : 다량의 데이터를 부어넣을 때.
    - unrecoverable load data 는 archive log mode에서 direct path load 사용 시에 리두 발생을 최소화할 수 있는 기능
        
        🌳 `unrecoverable`
        ▶️ Direct Path 로드 시 redo 로그를 최소화하여 로드 속도를 높인다..
        즉, 로드된 데이터에 대해 복구(recovery) 시 redo 로그로 복구할 수 없게 된다.
        따라서 백업이 반드시 필요하다. 
        
    - data block을 uga에서 생성하고 이 블록을 디스크에 있는 테이블 제일 뒤에 direct 하게 저장한다. 즉 hwm 뒤에 저장한 후 hwm는 조종된다.
    - data save 사용
    - 최소 redo 발생
    - 제약 조건을 무시한다.
    - insert trigger 실행하지 않습니다.
    
    📍 단점 : 클러스터화된 테이블에는 load되지 않습니다.
    
    - 다른 유저는 대상 테이블을 사용할 수 없다.
    
    ▶️ 삭제
    
    ```sql
    truncate table hr.test;
    ```
    
    ▶️ 로드 방식 (insa.ctl)
    
    ```sql
    unrecoverable load data
    infile insa.dat
    replace
    into table hr.test
    fields terminated by ',' optionally enclosed by '"'
    (id,name,phone)
    ```
    
    ▶️ 로드
    
    ```sql
     ! sqlldr hr/hr control=insa.ctl direct=true 
     
     -- 제약조건 확인 
     select constraint_name, constraint_type, search_condition,status, validated, index_name from dba_constraints where owner = 'HR' and table_name = 'TEST';
     /*
      TEST_ID_PK	UNIQUE	UNUSABLE
      
      TEST_PHONE_UNIQUE	UNIQUE	VALID
     */
     select index_name, uniqueness, status from dba_indexes where owner = 'HR' and table_name = 'TEST';
     explain plan for select * from hr.test where id = 1;
     select plan_table_output from table(dbms_xplan.display());
     
     /*
     PLAN_TABLE_OUTPUT
    --------------------------------------------------------------------------------
    Plan hash value: 1357081020
    
    --------------------------------------------------------------------------
    | Id  | Operation         | Name | Rows  | Bytes | Cost (%CPU)| Time     |
    --------------------------------------------------------------------------
    |   0 | SELECT STATEMENT  |      |     1 |    23 |     3   (0)| 00:00:01 |
    |*  1 |  TABLE ACCESS FULL| TEST |     1 |    23 |     3   (0)| 00:00:01 |
    --------------------------------------------------------------------------
    
    Predicate Information (identified by operation id):
    ---------------------------------------------------
    
    PLAN_TABLE_OUTPUT
    --------------------------------------------------------------------------------
    
       1 - filter("ID"=1) 
     */
     
     insert into hr.test(id,name,phone) values(5,'bella','010-2222-3333');
     
     /*
     SYS@ora19c> insert into hr.test(id,name,phone) values(5,'bella','010-2222-3333');
    insert into hr.test(id,name,phone) values(5,'bella','010-2222-3333')
    *
    ERROR at line 1:
    ORA-01502: index 'HR.TEST_ID_PK' or partition of such index is in unusable
    state 
     */
     
     delete from hr.test where name = 'lucas';
     
     /*
     SYS@ora19c>  delete from hr.test where name = 'lucas';
     delete from hr.test where name = 'lucas'
    *
    ERROR at line 1:
    ORA-01502: index 'HR.TEST_ID_PK' or partition of such index is in unusable
    state
     
     */
     
     update hr.test set phone=null where id = 1;
     update hr.test set id=4 where name = 'lucas';
     
     /*
    update hr.test set id=4 where name = 'lucas'
    *
    ERROR at line 1:
    ORA-01502: index 'HR.TEST_ID_PK' or partition of such index is in unusable
    state
      */
      
      -- unusable 인덱스 -> 1) 삭제 2) pk 위반되는 데이터 수정
    ```
    
    ▶️ 인덱스 삭제
    
    ```sql
    drop index hr.test_id_pk;
    conn hr/hr
    $ORACLE_HOME/rdbms/admin/utlexpt1.sql
    
    select constraint_name, constraint_type, search_condition, status, validated,index_name from dba_constraints where owner ='HR' and table_name ='TEST';
    alter table hr.test enable validate constraint test_id_pk exceptions into hr.exceptions;
    select * from hr.exceptions;
    col row_id forma a50
    
    ```
    

▶️ 문제가 발생한 데이터 발생

```sql
select rowid,id,name,phone from hr.test where rowid in (select row_id from hr.exceptions);
update hr.test set id = 4 where rowid = 문제가되는로우아이디;
commit;
select * from hr.test;
truncate table hr.exceptions;
alter table hr.test enable validate constraint test_id_pk exceptions into hr.exceptions;
select constraint_name, constraint_type, search_condition, status , validated, index_name from dba_constraints where owner = 'HR' and table_name = 'TEST';
```

▶️ pk 위반된 데이터 배드 파일로 떨어진다

▶️ 데이터 수정 (insa.dat)

```sql
1,"james","010-9999-0000",2000
2,"grace","010-7777-7777",500
3,"scott","010-8888-0000",3000
3,"lucas","010-1004-1004",4000
4,"010-1234-1004",6000
```

▶️ 테이블 다시 생성

```sql
drop table hr.test purge;

create table hr.test(
    id number constraint test_id_pk primary key,
    name varchar2(30) not null,
    phone varchar2(15) constraint test_phone_unique unique,
    sal number constraint test_sal_ck check(sal > 1000),
    mgr number constraint test_mgr_fk references hr.test(id)
) tablespace users;

-- 제약 조건 확인
/*
TEST_ID_PK	UNIQUE	VALID
TEST_PHONE_UNIQUE	UNIQUE	VALID

--> pk와 unique 제약조건 활성화되어있음 
ENABLED VALIDATE 상태 
*/

! cat insa.ctl

unrecoverable load data
infile insa.dat
truncate
into table hr.test
fields terminated by ',' optionally enclosed by '"'
(id, name, phone, sal, mgr)

! cat insa.dat
1,"james","010-9999-0000",2000,
2,"grace","010-7777-7777",500,1
3,"scott","010-8888-0000",3000,1
3,"lucas","010-1004-1004",4000,2
4,"","010-1234-1004",5000,1
5,"oracle","010-0001-0001",6000,6
```

일부 행은 제약 조건 위반 가능성이 높음:

- `2,"grace",...,500,1` → `sal`이 1000 이하 → CHECK 제약 조건 위반
- `3,"lucas",...` → `id`가 3은 이미 존재하므로 PK 위반
- `4,"",...` → `name`이 비어있어 NOT NULL 위반
- `5,"oracle",...` → `mgr=6`인데 6번 ID가 존재하지 않아 FK 위반

![image.png](/assets/20250808/8.png)

▶️ 1번 데이터는 테이블 구조와 맞지 않아서 입력 안됨

▶️ 이렇게 ctl 안에 수정하면 입력된다.

```sql
 unrecoverable load data
infile insa.dat
replace
into table hr.test
fields terminated by ',' optionally enclosed by '"'
(id,name,phone,sal,mgr)
```

▶️ enable : 새롭게 들어온 데이터 검증 

▶️ validate : 기존 데이터 검증 

```sql
! cat insa.log

-- 제약 조건 맞지 않으면 bad file로 
```

![image.png](/assets/20250808/9.png)

▶️ 인덱스 중복이어서 reject되었지만, 데이터는 save되긴 하였다.

📍 대량의 데이터 입력

```sql
1. 제약 조건 disabled
2. 데이터 입력
3. not null 조건 -> 어떤 컬럼이 적용되는 지 조회 후 업무팀과 상의 후 수정.
```

```sql
spool emp.dat

select employee_id ||','|| last_name ||','|| first_name ||','|| salary ||','|| department_id
from hr.employees
order by 1;

spool off;

-- ! vi emp.sql
set pagesize 0    -- 컬럼 이름 display 하지 않겠다.
set linesize 200  -- linesize 작게 
set echo off -- spool 에서만 사용 가능 화면상에 출력하겠나?  
set termout off --  결과집합을 출력하지 않겠다.
set trimspool on -- 라인 뒤에 공백 
set feedback off  -- 마지막에 결과물이 표시하지 않겠다.

spool emp.dat
select employee_id||','|| last_name||','|| first_name||',' ||salary||','|| department_id
from hr.employees
order by 1;
spool off;

@ emp.sql
! cat emp.sql

---------- 다시 
drop table hr.emp purge;
create table hr.emp as select employee_id, last_name, first_name, salary, department_id, hire_date from hr.employees where 1=2;
! cat insa.ctl

! sqlldr hr/hr control=emp.ctl direct=true

-- 컨트롤 파일 생성
! vi emp.ctl

unrecoverable load data
infile emp.dat
insert
into table hr.emp
fields terminated by ',' optionally enclosed by '"'
trailing nullcols
(employee_id, last_name, first_name, salary, department_id, hire_date)

! sqlldr hr/hr control=emp.ctl direct=true

spool emp.dat

select employee_id ||','|| last_name ||','|| first_name ||','|| salary ||','|| department_id ||','|| to_char(hire_date,'yyyy-mm-dd')
from hr.employees
order by 1;

spool off
set pagesize 1000
set termout on
set feedback on

spool off;

```

SQL*Loader 실행이 끝나면 터미널에 결과가 나오고,

동일 폴더에 `.log` 파일과 `.bad` 파일이 생성됩니다.

- **.log** : 로드 과정과 결과 로그
- **.bad** : 제약 조건 위반, 데이터 오류로 로드 실패한 행들
- **.discard** (있을 경우) : 로드 조건에 맞지 않아 버려진 행들

▶️ 다시

▶️ 테이블 준비

```sql
drop table hr.emp purge;

create table hr.emp as
select employee_id,last_name,first_name,salary,hire_date,department_id
from hr.employees where 1=2;
```

▶️ 제어 파일 작성 (! vi emp.ctl) 

```sql
unrecoverable load data
infile emp.dat
insert
into table hr.emp
fields terminated by ',' optionally enclosed by '"'
trailing nullcols
(employee_id,last_name,first_name,salary,hire_date date 'yyyy-mm-dd',department_id)
```

- **unrecoverable** : Direct Path에서 redo 최소화
- **infile** : 로드할 데이터 파일
- **insert** : 기존 데이터 유지 + 새 데이터 삽입 (truncate 아님)
- **fields terminated by ','** : CSV 형태 구분자
- **optionally enclosed by '"'** : 큰따옴표로 감싼 데이터 허용
- **trailing nullcols** : 데이터 파일에 컬럼 값이 없으면 NULL 허용
- **date 'yyyy-mm-dd'** : 날짜 포맷 지정

▶️ Direct-path load 실행

```sql
! sqlldr hr/hr control=emp.ctl direct=true
```

▶️ 결과 확인

```sql
select * from hr.emp;
```