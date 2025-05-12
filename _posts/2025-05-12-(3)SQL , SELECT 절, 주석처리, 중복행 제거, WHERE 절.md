---
title: "[15일차] SQL , SELECT 절, 주석처리, 중복행 제거, WHERE 절"
excerpt: "아이티윌 0512_(3)"
categories:
      - ORACLE11g
tags:
      -  ORACLE11g
      - TIL
last-modified-at: 2025-05-12T20:48
---

# SQL , SELECT 절, 주석처리, 중복행 제거, WHERE 절

## SQL(Structured Query Language)

→ DB 다루는 언어는 SQL 밖에 없다.

- 데이터베이스에서 데이터를 정의, 조작, 제어하기 위해 사용하는 언어
- ANSI(American National Standards Institute) 표준어인 SQL를 사용한다.
- DQL(Data Query Language ; 데이터 질의 언어)
    - 데이터베이스에 저장된 데이터를 조회하는 명령어
    - **SELECT (현장에서 조금 더 중요함…)**
- DML(Data Manipulation Language ; 데이터 조작 언어)
    - 단위 : 레코드 단위
    - COMMIT/ROLLBACK 가능
    - 데이터베이스에 데이터를 입력, 수정, 삭제하는 명령어
    - INSERT, UPDATE, DELETE, MERGE
- TCL (Transaction Control Language)
    - 데이터의 일관성을 유지하면서 안정적으로 데이터를 복구시키기 위한 명령어
    - COMMIT, ROLLBACK, SAVEPOINT
- DCL (Data Control Language)
    - 데이터베이스에 저장된 데이터를 관리하기 위해서 데이터의 보안성 제어하기 위한 명령어, 권한관리
    - GRANT, REVOKE
- DDL (Data Definition Language)
    - 데이터베이스를 생성 또는 객체들을 생성 유지 관리하는 명령어
    - CREATE, ALTER, DROP, TRUNCATE, RENAME, COMMENT

## SELECT 문

데이터베이스에서 데이터를 검색(조회)하는 문

```sql
SELECT 컬럼, 컬럼, 컬럼,...      -- 절
FROM 소유자 이름.테이블명;        -- 절 
```

**기능**

- projection  : 열 추출

```sql
-- Projection
SELECT employee_id
FROM hr.employees;
```

- selection     : 행 추출

```sql
-- Selection
SELECT  *
FROM hr.employees
WHERE employee_id = 100;
```

- join                : 서로 다른 테이블에 데이터를 추출

**SQL 문 작성시 주의점**

- SQL문은 대소문자를 구분하지 않습니다.
    - 성능으로 보면 SQL문의 대소문자를 규칙적으로 생성하면 좋다.
    - 공백 문자는 구분한다.
    
    ```sql
    SELECT last_name,first_name FROM hr.employees;
    SELECT last_name, first_name FROM hr.employees;
    -- 성능 관점으로 보면 엄청난 차이
    ```
    
- tab key, enter key 구분한다.
- 절은 서로 다른 줄에 작성하자.
- 주석처리 잘하자.
- SQL문 제일 뒤에서는 ; 종료하자.

→ 키워드는 대문자, 컬럼 등은 소문자로 규칙성 있게 작성하여

 **가독성**과 **성능**을 챙긴다.

→ 성능 측면:  사용자가 쿼리를 날리면 오라클 server는 실행 계획을 만들어두기 위해 쿼리 문장을 shared pool 내의 library(cache) 안에 저장시킨다.  그렇기 때문에 쿼리문을 규칙적으로 작성하는 것이 좋다… 모 다르면 그때마다 실행 계획을 만들기 때문

## 실행 계획 보기 (F10)

```sql
SELECT *    -- (1)
FROM hr.employees e
WHERE employee_id = 100;

SELECT /*+ full(e) */ *    -- (2) 풀 스캔을 의도적으로 유도
FROM hr.employees e
WHERE employee_id = 100;
```

![image.png](/assets/20250512/7.png)

(1)

![image.png](/assets/20250512/8.png)

(2)

## 주석

- 단일행 주석   :    - - 주석 내용
- 여러행 주석   :  /*  주석 내용 */

```sql
SELECT 
/* 사원 정보 */
     last_name,     -- 성
     first_name,    -- 이름
     salary         -- 급여
FROM hr.employees;   
```

## 중복 행 제거

- 기본적으로 query(select)결과에는 중복행을 포함한 모든 행이 출력된다.
- distinct, unique 키워드를 이용해서 중복을 제거할 수 있다.
- distinct, unique 키워드는 select절 제일 앞에 한 번만 사용한다.

```sql
SELECT    -- 중복 행 제거
    distinct department_id
FROM hr.employees;    
```

```sql
SELECT 
    unique department_id
FROM hr.employees;  
```

![image.png](/assets/20250512/9.png)

## WHERE 절

```sql
SELECT
FROM
WHERE 기준컬럼 비교연산자 비교값;
```

```sql
SELECT *
FROM hr.employees
WHERE employee_id = 100;
```

```sql
SELECT *
FROM hr.employees
WHERE last_name = 'King';
```

```sql
SELECT *
FROM hr.employees
WHERE last_name = 'KING';
```

- 행을 제한하는 절
- 조건절
- 기준 컬럼이 문자열,날짜열이면 비교값은 작은 따옴표로 묶어야 한다.
- 영문자는 대소문자를 구분한다.
- 날짜 형식은 지역에 따라 기본 날짜 형식이 다르다. 한국(RR/MM/DD), 미국(DD-MON-RR)

```sql
-- 지역에 
SELECT *
FROM nls_session_parameters;
```

```sql
SELECT *
FROM hr.employees
WHERE hire_date = '01/01/13';  -- RR/MON/DD
```

```sql
-- RR/MON/DD
--  미국식 날짜 형식에선 에러남
SELECT *
FROM hr.employees
WHERE hire_date = '01/01/13';
```

```sql
SELECT *
FROM hr.employees
WHERE hire_date = to_date('2001-01-13','yyyy-mm-dd');
```