---
title: "[07] Subquery를 사용하여 Query 해결"
excerpt: ""
categories:
      - ORACLE12c
tags:
      - ORACLE12c
      - TIL
last-modified-at: 2025-04-25T20:14
---

# [07] Subquery를 사용하여 Query 해결

# Subquery

- subquery는 main query 전에 실행
- subquery 결과는 main query에서 사용됨
- 괄호로 묶는다.
- 가독성을 위해 비교 조건의 오른쪽에 배치한다. (양쪽 어디에나 사용가능)
- single-row subquery에는 **단일 행 연산자**를 사용 / multiple-row subquery에는 **다중 행 연산자** 사용

```sql
select employee_id, last_name, hire_date, job_id, salary
  from employees
 where salary > (select salary
                   from employees
                  where employee_id = 151);
```

# Subquery 에서 그룹 함수 사용

```sql
select last_name, job_id, salary
  from employees
  where salary = (select min(salary)
                from   employees);
```

# Subquery가 있는 HAVING 절

- subquery를 먼저 실행
- main query의 HAVING 절로 결과를 만들어 낼 수 있다.

# Multiple-Row-Subquery

- 두 개 이상의 행을 반환
- 다중 행 비교 연산자 사용
- **연산자**
    - IN : 리스트의 임의 멤버와 같음
    - ANY : =, !, >,<, <=, >=연산자가 앞에 있어야 한다.
    
    만족하는 결과 집합에 요소가 1개 이상 있을때 TRUE 반환 
    
    - ALL: =, !, >,<, <=, >=연산자가 앞에 있어야 한다.
    
    결과 집합의 모든 요소에 대한 관계가 TRUE인 경우 TRUE 반환 
    

```sql
-- all, any -> max, min으로 대체 가능
select employee_id, last_name, salary, department_id
  from employees
 where salary >ALL (select avg(salary)
                   from employees
                   group by department_id) ;

select employee_id, last_name, salary, department_id
  from employees
 where salary > (select max(avg(salary))
                   from employees
                   group by department_id) ;
```

# Multiple-Column Subquery

- outer query에 두 개 이상의 열을 반환
- 열 비교는 쌍 방식 or 비쌍 방식
- SELECT문의 FROM 절에 사용될 수도 있다.
- 비교 연산자는 오직 IN만 가능

```sql
-- 각 부서에서 급여가 가장 낮은 모든 사원 표시
select first_name, department_id, salary
from employees
where (salary,department_id) IN
      (select min(salary), department_id
       from   employees
       group by department_id)
order by department_id; 
```

# Subquery의 null값

서브 쿼리가 NOT IN 뒤에 올 때 null이 반환되지 않게 주의해야한다.

NVL로 처리