---
title: sql 쿼리 연습
excerpt: ""
categories:
  - MariaDB
tags:
  - MariaDB
  - TIL
last_modified_at: '2021-06-18 T14:28'
---

# SELECT

```sql
SELECT *
FROM member
LIMIT 10;     -- 상위 10개까지만 조회
```

![0](/assets/20210618/0.png)

```sql
SELECT *
FROM employees
LIMIT 100,10; -- 100순위 다음 부터 10개
```

```sql
SELECT *
FROM employees
ORDER BY first_name -- first_name 오름차순으로 정렬
LIMIT 10;
```

![1](/assets/20210618/1.png)

```sql
SELECT *
FROM employees
ORDER BY first_name DESC --내림차순 정렬
LIMIT 10;
```

![2](/assets/20210618/2.png)

```sql
SELECT *
FROM employees
ORDER BY last_name DESC, first_name ASC -- last_name Z부터 정렬 후 first_name 오름차순 정렬
LIMIT 10;
```

![3](/assets/20210618/3.png)

```sql
SELECT COUNT(*) AS '성이 Zykh인 사람'
FROM employees
WHERE last_name = 'Zykh'
ORDER BY last_name DESC, first_name ASC -- last_name Z부터 정렬 후 first_name 오름차순 정렬
;
```

![4](/assets/20210618/4.png)

```sql
-- NOW  현재 년월일시간
SELECT NOW()
FROM DUAL;  -- 없는 테이블
```

![5](/assets/20210618/5.png)

```sql
SELECT YEAR(birth_date) AS '출생년도'  ,YEAR(NOW())-YEAR(birth_date) AS '나이'
FROM employees
LIMIT 10;
```

![6](/assets/20210618/6.png)

```sql
SELECT YEAR(hire_date) AS '입사년도'  ,YEAR(NOW())-YEAR(hire_date) AS '근속년도'
FROM employees
LIMIT 10;
```

![7](/assets/20210618/7.png)

```sql
SELECT DISTINCT title    -- DISTINCT 고유한 값 중복 없이 보기
FROM titles;
```

![8](/assets/20210618/8.png)

```sql
SELECT *
FROM salaries
WHERE to_date LIKE '9999%'   -- 9999 (현재) 년도에도 같은 금액을 갖는 사람 찾기
;
```

```sql
SELECT first_name, title     -- 아우터 조인!
FROM titles AS A LEFT OUTER JOIN employees
ON A.emp_no = employees.emp_no
LIMIT 10
;
```

![9](/assets/20210618/9.png)

```sql
SELECT COUNT(DISTINCT dept_no) AS 부서수
FROM dept_manager
;
```

![10](/assets/20210618/10.png)

```sql
-- 1999년 입사한 여 직원을 모두 뽑기
SELECT *
FROM employees
WHERE hire_date LIKE ('1999%') AND gender = 'F';
```

![11](/assets/20210618/11.png)

```sql
SELECT emp_no,MAX(salary)  -- 이 회사에서 현재 제일 연봉이 높은 사람
FROM salaries
WHERE to_date LIKE ('9999%')
;

SELECT * , YEAR(NOW()) - YEAR(hire_date) AS '근속 년수'
FROM employees
WHERE emp_no = 10002;     -- Bazalel Simmel 이라는 사람이 제일 많이 받는다. 이 여성은 64년 생으로 근속년도가 무려 36년이나 된다.

-- 이 여자는 dept manager일거라 추측

SELECT *
FROM dept_manager
WHERE emp_no = 10002;    -- 음 놀랍게도 dept_manager 아님

SELECT *
FROM titles
WHERE emp_no = 10002;     -- 직책은 Staff
```

```sql
SELECT COUNT(*) AS '19995년 ~ 2000 까지 입사한 직원의 수'
FROM employees
WHERE YEAR(hire_date) BETWEEN '1995' AND '2000';
```

![12](/assets/20210618/12.png)

```sql
-- IN 함수 써서 last_name이 Makrucki,Zockler, Blokdijk 인 사람 모두 출력해보기

SELECT *
FROM employees
WHERE last_name IN('Makrucki','Zockler', 'Blokdijk');
```

```sql
SELECT title, COUNT(*)
FROM titles
WHERE YEAR(to_date) = '9999'
GROUP BY title; -- 직책 별로 묶어서 볼 수 있다.
```

![13](/assets/20210618/13.png)

```sql
-- 남여 직원의 수를 뽑기

SELECT gender, COUNT(*) AS '(명)'     
FROM employees
GROUP BY gender;
```

![14](/assets/20210618/14.png)

```sql
-- if(조건식, 참,거짓)
-- if (gender = 'M' , '남자','여자')

SELECT if (gender = 'M' , '남자','여자') AS '성별', COUNT(*) AS '(명)'     
FROM employees
GROUP BY gender;
```

![15](/assets/20210618/15.png)

```sql
-- 부서별 현 인원수를 구해주세요.
-- dept_emp에서 찾아주세요.

SELECT dept_no, COUNT(*) AS '부서별 현 인원수'
FROM dept_emp
WHERE YEAR(to_date) = '9999'
GROUP BY dept_no;
```

![16](/assets/20210618/16.png)

```sql
SELECT *    -- 이너 조인!    -- 알리아스를 이용해서 이렇게 !
FROM salaries A INNER JOIN employees B
ON A.emp_no = B.emp_no
LIMIT 10
;
```

```sql
SELECT B.emp_no, B.first_name, A.salary    -- 이너 조인!    -- 알리아스를 이용해서 이렇게 !
FROM salaries A INNER JOIN employees B     -- INNER JOIN 대신 JOIN을 사용해도 된다.
ON A.emp_no = B.emp_no
WHERE A.to_date LIKE ('9999%')
LIMIT 10;
;
```

![17](/assets/20210618/17.png)

```sql
SELECT A.emp_no, B.dept_name
FROM dept_emp A JOIN departments B
ON A.dept_no = B.dept_no
WHERE A.to_date LIKE ('9999%')
GROUP BY dept_name
ORDER BY A.dept_no;
```

![18](/assets/20210618/18.png)

```sql
SELECT C.first_name , D.*
FROM employees C JOIN (SELECT A.emp_no, B.dept_name
FROM dept_emp A JOIN departments B
ON A.dept_no = B.dept_no
WHERE A.to_date LIKE ('9999%')
GROUP BY dept_name
ORDER BY A.dept_no) D
ON C.emp_no = D.emp_no;
```

![19](/assets/20210618/19.png)

```sql
-- 부서별 직원수

SELECT A.dept_name, COUNT(*) AS '부서별 직원수'
FROM departments A  JOIN dept_emp B
ON A.dept_no = B.dept_no
WHERE B.to_date LIKE ('9999%')
GROUP BY A.dept_name
ORDER BY A.dept_name;
```

![20](/assets/20210618/20.png)

```sql
-- dept_manager관리자 , 관리자 이름

SELECT A.emp_no , A.dept_no, B.first_name, B.last_name
FROM dept_manager A JOIN employees B
ON A.emp_no = B.emp_no
WHERE A.to_date LIKE ('9999%')
GROUP BY A.dept_no;
```

![21](/assets/20210618/21.png)
