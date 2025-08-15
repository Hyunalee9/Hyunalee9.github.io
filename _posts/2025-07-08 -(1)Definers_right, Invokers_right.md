---
title: "[53일차] Definer's right, Invoke's right"
excerpt: "아이티윌 0708_(1)"
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-07-08T18:01
---

# PL/SQL(Definer’s right, Invoke’s right )

- Definer’s right (만든 사람 입장에서 프로그램을 수행할 것인지)
- Invoke’s right (호출자 입장에서 프로그램 수행할 것인지)

```sql
▶️ sys session
-- 유저 생성
CREATE USER green IDENTIFIED BY oracle;
-- select 권한 주기 
GRANT SELECT ON hr.employees TO green;
SELECT * FROM dba_users WHERE username = 'GREEN';

-- connect  현장에선 사용하지 말라고 한다.
 
GRANT connect, resource TO green;

▶️ green session
INSA@ora19c> conn green/oracle
GREEN@ora19c> select * from session_roles;

/*
ROLE
---------
CONNECT
RESOURCE
SODA_APP
*/

-- 포맷팅해서 보기 좋게 만들기
COLUMN ROLE FORMAT A10
COLUMN PRIVILEGE FORMAT A30
COLUMN ADM FORMAT A3
COLUMN COM FORMAT A3
COLUMN INH FORMAT A3

select * from role_sys_privs;

/*

ROLE       PRIVILEGE                      ADM COM INH
---------- ------------------------------ --- --- ---
RESOURCE   CREATE SEQUENCE                NO  YES YES
RESOURCE   CREATE PROCEDURE               NO  YES YES
CONNECT    CREATE SESSION                 NO  YES YES
RESOURCE   CREATE CLUSTER                 NO  YES YES
CONNECT    SET CONTAINER                  NO  YES YES
RESOURCE   CREATE TABLE                   NO  YES YES
RESOURCE   CREATE TRIGGER                 NO  YES YES
RESOURCE   CREATE TYPE                    NO  YES YES
RESOURCE   CREATE OPERATOR                NO  YES YES
RESOURCE   CREATE INDEXTYPE               NO  YES YES

*/
```

📍테이블 삭제할 수 있는 유저

- 소유자
- drop any table 시스템 권한 가지고 있는 sys

📍테이블 생성할 수 있는 유저

- create table 시스템 권한
- 특정한 테이블스페이스를 사용할 수 있는 quota
- 앞에 언급한 것 소유한 유저가 테이블 생성할 수 있다.
- create any table, unlimited tablespace 시스템 권한 가지고 있는 sys

```sql
▶️ oracle_hr session 
drop table hr.emp purge;

-- 테이블 생성
CREATE TABLE hr.emp
TABLESPACE users
AS
SELECT employee_id, last_name, salary
FROM hr.employees
WHERE 1 = 2 ;  -- 구조만 가져오겠다.

▶️ green session
drop table green.emp purge;

-- 테이블 생성
CREATE TABLE green.emp
TABLESPACE users
AS
SELECT employee_id, last_name, salary
FROM hr.employees
WHERE 1 = 2 ;  -- 구조만 가져오겠다.

▶️ oracle_hr session

-- 프로시저 생성
-- 📍 Definer's right
CREATE OR REPLACE PROCEDURE insert_emp1
(p_id   in number,
 p_name in varchar2,
 p_sal  in number)
IS
BEGIN
	INSERT INTO emp(employee_id, last_name, salary)
	VALUES(p_id, p_name, p_sal);
	COMMIT;
END insert_emp1;
/ 

-- 📍 Invoker's right
CREATE OR REPLACE PROCEDURE insert_emp2
(p_id   in number,
 p_name in varchar2,
 p_sal  in number)
 authid current_user
IS
BEGIN
	INSERT INTO emp(employee_id, last_name, salary)
	VALUES(p_id, p_name, p_sal);
	COMMIT;
END insert_emp2;
/ 

-- source를 봐야지 Definer's 인지 Invoker's 인지 알 수 있다.
SELECT text FROM user_source WHERE name = 'INSERT_EMP1' ORDER BY line;
SELECT text FROM user_source WHERE name = 'INSERT_EMP2' ORDER BY line;

-- green 유저에게 프로시저 execute 권한 부여
grant execute on insert_emp1 to green;
grant execute on insert_emp2 to green;

-- 객체 권한 조회
SELECT * FROM user_tab_privs;

▶️ green session

SELECT * FROM green.emp;
SELECT * FROM user_tab_privs;
```

![image.png](/assets/20250708/1.png)

📍 INHERIT PRIVILEGES (12c) : 

authid current_user 지시어를 이용해서 생성한 프로시저를 만든 유저가 아닌 호출하는 사람 입장에서 수행하기 위한 객체 권한 (원래 가지고 있는 권한)

```sql
▶️ green session

desc hr.insert_emp2

/*
PROCEDURE hr.insert_emp2
인수 이름  유형       In/Out 기본값?    
------ -------- ------ ------- 
P_ID   NUMBER   IN     unknown 
P_NAME VARCHAR2 IN     unknown 
P_SAL  NUMBER   IN     unknown 

*/

SELECT text FROM all_source WHERE name = 'INSERT_EMP1' ORDER BY line;
SELECT text FROM all_source WHERE name = 'INSERT_EMP2' ORDER BY line;

-- 📍 Definer's right
-- hr이 가지고 있는 emp 테이블에 입력
execute hr.insert_emp1(1000,'oracle',5000);
```

![image.png](/assets/20250708/2.png)

```sql
▶️ oracle_hr session
select * from hr.emp;  -- user 테이블스페이스 사용 권한이 없어서 오류난다.

▶️ sys session
alter user green quota 10m on users; -- quota 값 할당

--📍 Invoker's right
-- green이 가지고 있는 emp 테이블에 입력
execute hr.insert_emp2(2000,'scott',3000);

select * from green.emp;

▶️ oracle_hr session
select * from hr.emp;

```

![image.png](/assets/20250708/3.png)

```sql

-- ▶️ green session
execute hr.insert_emp1(3000,'plsql',1000);

-- ▶️ oracle_hr session
select * from hr.emp;
```

![image.png](/assets/20250708/4.png)

📍INHERIT PRIVILEGES 관련 실습

```sql
-- ▶️ sys session
-- INHERIT PRIVILEGES 권한 취소
REVOKE INHERIT PRIVILEGES ON USER GREEN FROM PUBLIC;

-- ▶️ green session
SELECT * FROM user_tab_privs;
```

![INHERIT PRIVILEGES 권한 사라졌다.](/assets/20250708/5.png)

INHERIT PRIVILEGES 권한 사라졌다.

```sql
-- ▶️ green session
execute hr.insert_emp1(7000,'ethan',8000);

--  INHERIT PRIVILEGES 권한 취소되었기에 오류 난다. 
execute hr.insert_emp2(6000,'daniel',6000); 

▶️ sys session
GRANT INHERIT PRIVILEGES ON USER GREEN TO PUBLIC;

▶️ green
SELECT * FROM user_tab_privs;
execute hr.insert_emp2(6000,'daniel',6000); 
select * from green.emp;
```

![image.png](/assets/20250708/6.png)

→ 다시 INHERIT PRIVILEGES 권한 부여하고 프로시저 수행

![image.png](/assets/20250708/7.png)

```sql
▶️ oracle_hr
HR@ora19c> SELECT to_char(sysdate, 'yyyy-mm-dd hh24:mi:ss') FROM dual;

/*
TO_CHAR(SYSDATE,'YY
-------------------
2025-07-08 12:05:49
*/
```

📍시간 정보 맞추기

```sql
[root@oracle ~]# rdate -s time.bora.net
[root@oracle ~]# date
Tue Jul  8 12:03:23 KST 2025
```

📍 application role (응용 프로그램 롤) : PL/SQL 프로그램에 의해서 롤을 활성화, 비활성화 할 수 있다.

📍 DBMS_SESSION.SET_ROLE : 현재 사용자에게 부여된 롤을 활성화, 비활성화하는 프로그램

📍 SELECT ANY DICTIONARY : data dictonary table 를 조회할 수 있는 권한

```sql
▶️ sys 
CREATE OR REPLACE PROCEDURE priv_mgr

-- ✅ PL/SQL에서 프로시저, 함수, 패키지를 만들 때
-- 그 실행 권한 주체를 결정하는 옵션
AUTHID CURRENT_USER
IS
BEGIN
 -- 12:15 ~ 12:20 사이의 시간에서는
	IF to_char(sysdate, 'hh24:mi') BETWEEN '12:15' AND '12:20' THEN
	  -- sec_app_role을 활성화한다. 
		DBMS_SESSION.SET_ROLE('sec_app_role');
	ELSE 
		DBMS_SESSION.SET_ROLE('NONE');	
	END IF;	
END priv_mgr;
/

--  application role(응용 프로그램)을 통한 롤 생성 방법.
CREATE ROLE sec_app_role IDENTIFIED USING priv_mgr;

-- aplication role 권한 부여
GRANT SELECT ANY DICTIONARY TO sec_app_role;

-- priv_mgr 권한 부여
GRANT EXECUTE ON priv_mgr TO insa;

▶️ insa

INSA@ora19c> select * from session_roles;

/*
ROLE
-----
PROG
MGR
*/

select * from role_sys_privs;

/*

ROLE       PRIVILEGE                      ADM COM INH
---------- ------------------------------ --- --- ---
PROG       CREATE VIEW                    NO  NO  NO
PROG       CREATE SESSION                 NO  NO  NO
PROG       CREATE TRIGGER                 NO  NO  NO
MGR        SELECT ANY TABLE               NO  NO  NO
PROG       CREATE PROCEDURE               NO  NO  NO

*/

execute sys.priv_mgr
select * from sys.tab$;

INSA@ora19c> ! date
Tue Jul  8 12:21:33 KST 2025

INSA@ora19c> execute sys.priv_mgr

-- PL/SQL procedure successfully completed.

INSA@ora19c> select * from session_roles;

-- no rows selected

INSA@ora19c> select * from sys.tab$;

/*
select * from sys.tab$
                  *
ERROR at line 1:
ORA-00942: table or view does not exist
*/

INSA@ora19c> select * from session_roles;

-- 권한 활성화
execute dbms_session.set_role('all')

/*
ROLE
----------
PROG
MGR
*/

execute dbms_session.set_role('prog,mgr')

-- sys테이블 select 권한 없음 
INSA@ora19c> select count(*) from hr.locations;

-- 권한을 준다.
exec sys.priv_mgr
select count(*) from sys.obj$

▶️ sys

CREATE OR REPLACE PROCEDURE priv_mgr
AUTHID CURRENT_USER
IS
BEGIN
	IF to_char(sysdate, 'hh24:mi') BETWEEN '14:00' AND '14:05' THEN
	  -- sec_app_role을 활성화한다. 
		DBMS_SESSION.SET_ROLE('sec_app_role');
	ELSE 
		DBMS_SESSION.SET_ROLE('NONE');	
	END IF;	
END priv_mgr;
/
```