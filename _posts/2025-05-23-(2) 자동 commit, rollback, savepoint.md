---
title: "[24일차] 자동 commit, rollback, savepoint"
excerpt: "아이티윌 0523_(2) "
categories:
      - ORACLE11g
tags:
      -  ORACLE11g
      - TIL
last-modified-at: 2025-05-23T17:14
---


# 자동 COMMIT , ROLLBACK , SAVEPOINT

## 자동 commit 발생할 때

▶️ DML 후 바로 DDL (CREATE, ALTER, DROP, RENAME, TRUNCATE, COMMENT) 할 때

```sql
DELETE FROM hr.emp e
WHERE EXISTS (SELECT 'x'
                FROM hr.job_history
               WHERE employee_id = e.id);

-- 내부적 자동 commit 발생 
CREATE TABLE hr.new(id number);         
          
ROLLBACK;

-- DML과 DDL은 절대 같은 창에서 작업하지 x
-- 데이터 딕셔너리가 테이블을 관리한다. 
-- DELETE 후 CREATE TABLE 했다. ROLLBACK해도 조회되지 않는다.

SELECT *
FROM hr.emp e
WHERE EXISTS (SELECT 'x'
                FROM hr.job_history
               WHERE employee_id = e.id);
```

```sql
-- 새로운 테이블 하기 기술한 테이블에 다 들어있어야 함.
-- DBA user
SELECT * FROM tab$ WHERE obj$ = 20240;
SELECT * FROM obj$ WHERE name = 'NEW';
SELECT * FROM col$ WHERE obj$ = 20240;
```

▶️ DML후 바로 DCL (GRANT, REVOKE)

▶️ sqlplus 에서 exit를 수행하면서 종료하면 자동 commit 

![image.png](/assets/20250523/1.png)

▶️ sqlplus에서 connect(conn)를 수행하면 자동 commit

## 자동 rollback 발생할 때

▶️ sqlplus를 비정상으로 종료 

▶️ DML문을 수행하고 있는 컴퓨터가 비정상적으로 종료

▶️ client - server 환경에서 네트워크가 장애가 발생하는 경우 (ex. 은행 거래 중.. 네트워크 장애)

## SAVEPOINT

- DML 작업시에 **ROLLBACK** 도와주는 표시자

```sql
CREATE TABLE hr.emp_20
AS 
SELECT employee_id, last_name
FROM hr.employees
WHERE department_id = 20;

SELECT * FROM hr.emp_20;

-- Transaction 시작
INSERT INTO hr.emp_20(employee_id,last_name)   
VALUES(300, 'ORACLE');

SELECT * FROM hr.emp_20;

UPDATE hr.emp_20
SET last_name = 'ITWILL'
WHERE employee_id = 202;

SELECT * FROM hr.emp_20;

DELETE FROM hr.emp_20 WHERE employee_id = 201;

SELECT * FROM hr.emp_20;

ROLLBACK; -- Transaction 시작 시점까지 전부 취소
```

```sql
SAVEPOINT 표시자;
ROLLBACK TO 표시자   -- 표시자 밑에 있는 트랜잭션 취소
```

![image.png](/assets/20250523/2.png)

```sql
-- Transaction 시작
INSERT INTO hr.emp_20(employee_id,last_name)   
VALUES(300, 'ORACLE');

SELECT * FROM hr.emp_20;

SAVEPOINT A;

UPDATE hr.emp_20
SET last_name = 'ITWILL'
WHERE employee_id = 202;

SELECT * FROM hr.emp_20;

SAVEPOINT B;

DELETE FROM hr.emp_20 WHERE employee_id = 201;

SELECT * FROM hr.emp_20;

ROLLBACK TO B; -- B 아래의 트랜잭션만 취소.

SELECT * FROM hr.emp_20;

COMMIT;
```

![image.png](/assets/20250523/3.png)