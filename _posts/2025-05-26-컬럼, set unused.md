---
title: "[25일차] 컬럼 ADD, MODIFY, DROP/ SET UNUSED"
excerpt: "아이티윌 0526_(1) "
categories:
      - ORACLE11g
tags:
      -  ORACLE11g
      - TIL
last-modified-at: 2025-05-26T9:38
---


```sql
-- emp 테이블 삭제
DROP TABLE hr.emp PURGE;
```

```sql
-- 다시 emp 생성 
CREATE TABLE hr.emp
(id number, name varchar2(30), day date)
TABLESPACE users;
```

```sql
-- 권한 및 정보 확인
SELECT * FROM session_privs;   -- 시스템 권한 + 롤로 받은 시스템 권한 
SELECT * FROM user_ts_quotas;  -- user 기준 테이블 스페이스 권한 
SELECT * FROM user_tables WHERE table_name = 'EMP';  -- 테이블 통계 정보
SELECT * FROM user_tab_columns WHERE table_name = 'EMP'; -- 테이블 컬럼 통계 정보
```

## 컬럼 추가

```sql
desc hr.emp; --테이블 구성 확인

ALTER TABLE hr.emp ADD job_id varchar(20);
ALTER TABLE hr.emp ADD dept_id varchar2(20);
```

## 컬럼 수정

```sql
ALTER TABLE hr.emp MODIFY job_id varchar(30);
ALTER TABLE hr.emp MODIFY dept_id number(3);
```

## 컬럼 삭제

```sql
-- 영향 받는 프로그램 다 조사해야 한다.
-- 공유, 조회, DML 작업 중단
ALTER TABLE hr.emp DROP COLUMN job_id
```

```sql
-- 다시 조회
desc hr.emp
SELECT * FROM user_tab_columns WHERE table_name = 'EMP';
```

## SET UNUSED 옵션(8i에서 새롭게 나온 기능)

- 하나 이상의 열을 사용되지 않음으로 표시하여 시스템 리소스 요구량이 낮아질때 삭제할 수 있는 기능
- **테이블에서 컬럼을 "삭제"하는 것처럼 만들되, 실제로 데이터를 바로 지우진 않는** Oracle의 기능
- **컬럼을 데이터 딕셔너리(시스템 내부)에서 안 보이게 만들지만**, 데이터를 **즉시 삭제하지 않아**서 처리 속도가 빠름.
- **`DROP COLUMN`보다 빠르고 안전한 대안**
- unused 다시 되돌릴 수 있는 방법은 아직 없음..

```sql
-- 컬럼 삭제된 건 아니지만, 보이지는 않음
ALTER TABLE hr.emp SET UNUSED COLUMN dept_id;
-- ALTER TABLE hr.emp SET UNUSED (dept_id);

desc hr.emp; 
```

![image.png](/assets/20250526/1.png)

```sql
-- 테이블의 set unused 적용된 수 조회 (컬럼명은 따로 보이지 않으므로 따로 작업 이력 만들어 놔야함)
SELECT * FROM user_unused_col_tabs;
```

```sql
-- set unused 컬럼 삭제
ALTER TABLE hr.emp DROP UNUSED COLUMNS;
```

```sql
--id 중복, null 데이터 품질 나빠짐
INSERT INTO hr.emp(id,name,day) VALUES(1,'noah',sysdate);
INSERT INTO hr.emp(id,name,day) VALUES(1,'lim',sysdate);
INSERT INTO hr.emp(id,name,day) VALUES(null,'ethan',sysdate);
```