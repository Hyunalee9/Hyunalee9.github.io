---
title: "[19일차] 연습문제"
excerpt: "아이티윌 0516_(3) group by,to_char,join"
categories:
      - ORACLE11g
tags:
      -  ORACLE11g
      - TIL
last-modified-at: 2025-05-17T18:19
---

# 연습문제

[문제22] 2008년도에 입사한 사원들의 job_id별 인원수를 구하고 인원수가 많은 순으로 출력하세요.

```sql
SELECT 
    job_id,
    count(*) 인원수
FROM hr.employees
WHERE hire_date >= to_date('2008/01/01','yyyy/mm/dd')
AND hire_date < to_date('2009/01/01' ,'yyyy/mm/dd')
GROUP BY job_id
ORDER BY 2 desc ;

```

[문제23] 년도별 입사 인원수를 출력해주세요.

```sql
-- 네자리이면 yyyy, 두자리이면 rr
SELECT
    to_char(hire_date, 'yyyy'),
    count(*)
FROM hr.employees
GROUP BY to_char(hire_date, 'yyyy')
ORDER BY 1;
```

[문제24] 년도별 입사 인원수를 아래 화면과 같이 출력해주세요.     

![image.png](/assets/20250517/15.PNG)


```sql
-- 나쁜 코드 : 성능 나빠지게 한다. 
SELECT 
    count(*) total,
    count(decode(to_char(hire_date, 'yyyy'),'2001','x')) "2001년",
    count(decode(to_char(hire_date, 'yyyy'),'2002','x')) "2002년",
    count(decode(to_char(hire_date, 'yyyy'),'2003','x')) "2003년",
    count(decode(to_char(hire_date, 'yyyy'),'2004','x')) "2004년",
    count(decode(to_char(hire_date, 'yyyy'),'2005','x')) "2005년",
    count(decode(to_char(hire_date, 'yyyy'),'2006','x')) "2006년",
    count(decode(to_char(hire_date, 'yyyy'),'2007','x')) "2007년",
    count(decode(to_char(hire_date, 'yyyy'),'2008','x')) "2008년"
FROM hr.employees;

```

[문제25] 80 부서에 근무하는 사원들의 last_name, job_id, department_name, city 출력해주세요.

```sql
-- employees , departments, locations 테이블 조인
SELECT
    e.last_name, e.job_id, d.department_name, l.city
FROM hr.employees e, hr.departments d, hr.locations l
WHERE e.department_id = d.department_id   -- 조인 조건 술어
AND d.location_id = l.location_id         -- 조인 조건 술어 
AND e.department_id = 80;                 -- 비조인조건술어
```

![image.png](/assets/20250517/1.png)

📍비조인조건술어부분 80인 부서 먼저 걸러내야할까? vs 조인을 하고 해야할까

![image.png](/assets/20250517/2.png)

→ e.department_id = d.department_id 조인 조건 술어가 사라졌다.

→ 1(d.department),  m(e.department) 이니까 **cartesian product**로 하는 게 성능 상 더 괜찮다. 

→ 

```sql
-- 어차피 결과 집합은 1:m 이므로 e.department에 따라가게 되어있음. 
-- 34개
SELECT count(*)
FROM hr.employees
WHERE department_id = 80;
```

![image.png](/assets/20250517/3.png)

## NESTED LOOPS JOIN

- 2 개 이상의 테이블에서 하나의 집합을 기준으로 순차적으로 상대방 row를 결합하여 원하는 결과를 조합하는 조인 방법
- 많은 양의 데이터를 조회하는 경우, 성능 떨어진다.
- Naive Nested Loop Join , Indexed Nested Loop Join, Temporary Index Nested Loop Join등이 있다.
    - Naive Nested Loop Join : 내부 테이블 전체 혹은 인덱스 전체를 조회
    - 내부 테이블의 인덱스만을 조회해서 조회 대상의 B-Tree를 찾을 때까지 조회
    - Indexed Nested Loop Join과 유사. 차이점은 실행 시간 동안에만 일시적으로 인덱스가 생성된다는 것.

조금 더 이해해보기 위해 파이썬으로.

**1️⃣hr 유저의 DB에 있는 테이블들을 csv 파일로 만들기(하나씩 하나씩..)**

**SQL Plus `SPOOL`이용** 

1. SQL Plus hr DB 연결하기

```sql
SQL> conn hr/hr
```

1. csv output을 위한 환경 세팅

```sql
   -- SET MARKUP CSV ON -> 12c 이상에서만 가능..
   
   SET HEADING ON      -- 컬럼명 표시 유무
   SET ECHO OFF        -- SQL 문 출력 x
   SET PAGESIZE 0      -- 한 페이지에 출력될 행의 갯수
```

1. 데이터를 저장할 파일 경로 지정

```sql
SQL> SPOOL c:\hr\employees\employees.csv
```

1. `SELECT` 로 데이터 추출

```sql
-- 12c 이하는 연결 연산자로 필드 구분자를 직접 생성해야 한다.
-- 그런데 이 방식으로 하면.. 데이터 타입이 반영이 원할하게 안된다.
SQL> SELECT 
				EMPLOYEE_ID ||','||
				FIRST_NAME  ||','||
				LAST_NAME   ||','||
				EMAIL       ||','||   
				PHONE_NUMBER ||','|| 
				HIRE_DATE    ||','|| 
				JOB_ID       ||','||
				SALARY       ||','||  
				COMMISSION_PCT  ||','||
				MANAGER_ID      ||','|| 
				DEPARTMENT_ID
	FROM hr.employees;
```

1. `SPOOL OFF` 명령어 실행해, 파일 저장 과정 종료

![image.png](/assets/20250517/4.png)

잘 뽑아졌음

**SQL Developer 의 export 기능 이용하기**

![image.png](/assets/20250517/5.png)

2️⃣ colab google drive에 파일들 올려놓고 마운트해서 불러오기.

```python
from google.colab import drive
import pandas as pd
import os
```

```python
drive.mount('/content/drive')
# os.chdir('/content/drive/MyDrive/') -> 요건 작업 디렉토리 변경할 때.
```

```python
file_path = '/content/drive/MyDrive/employees.csv'
file_path2 = '/content/drive/MyDrive/departments.csv'
file_path3 = '/content/drive/MyDrive/locations.csv'
```

```python
e_df = pd.read_csv(file_path)
d_df = pd.read_csv(file_path2)
l_df = pd.read_csv(file_path3)
```

![image.png](/assets/20250517/6.png)

```sql
-- 이 문장을 파이썬 언어로 구현해보자.
-- 80 부서에 근무하는 사원들의 last_name, job_id, department_name, city 출력해주세요.
SELECT
    e.last_name, e.job_id, d.department_name, l.city
FROM hr.employees e, hr.departments d, hr.locations l
WHERE e.department_id = d.department_id   -- 조인 조건 술어
AND d.location_id = l.location_id         -- 조인 조건 술어 
AND e.department_id = 80;     
```

```python
# 80 부서에 근무하는 사원들의 last_name, job_id, 먼저 구해보기
dept_80 = e_df[e_df.DEPARTMENT_ID == 80]

# 특정 컬럼 여러개 뽑기.
dept_80[['LAST_NAME','JOB_ID']]
```

![image.png](/assets/20250517/7.png)

```python
# department_id를 이용하여 DEPARTMENTS 테이블과 조인해보기
# (1) 결과 집합 중 LAST_NAME과 JOB_ID, DEPARTMENT_NAME 추출
result1 = pd.merge(dept_80,d_df, on='DEPARTMENT_ID')
result1[['LAST_NAME', 'JOB_ID','DEPARTMENT_NAME']].head()
```

![image.png](/assets/20250517/8.png)

```sql
-- 같은 결과가 실행될까?
-- equi join 

SELECT
    e.last_name, e.job_id, d.department_name
FROM hr.employees e, hr.departments d
WHERE e.department_id = d.department_id
AND e.department_id = 80;
```

![image.png](/assets/20250517/9.png)


```python
# 다시 파이썬으로 locations 테이블과 조인하고 city 결과값 가져오기
result2 = pd.merge(result1, l_df, on='LOCATION_ID')
result2[['LAST_NAME','JOB_ID','DEPARTMENT_NAME','CITY']].head()
```

![image.png](/assets/20250517/12.png)

```sql
-- cartesian product 과 비교

SELECT
    e.last_name, e.job_id, d.department_name, l.city
FROM hr.employees e, hr.departments d, hr.locations l
WHERE d.location_id = l.location_id        
AND e.department_id = 80    
AND d.department_id = 80;
```

![image.png](/assets/20250517/13.png)

## Query Plan 보는 방법

1️⃣ 위에서 아래로

2️⃣ 가장 안쪽 들여쓰기 존재한다면 그것부터 상위 레벨 순으로

- **Cost**
    - 누적된 값. 하위 Cost를 Roll up 함 (마지막 Cost가 최종)
    - 예측 비용
    - 사용된 자원, 작업의 단위
    - 숫자가 적을수록 좋은 성능의 쿼리
- **Cardinality**
    - 행 집합에서 행의 수를 표시
    - 숫자가 적을수록 속도가 빠를 수 있다.
    - 옵티마이저가 측정
- **Bytes**
    - 각 실행 계획 단계에서 access된 byte 수를 의미
    - 옵티마이저가 측정

📌 **cartesian product**

![image.png](/assets/20250517/14.png)