---
title: "[25일차] flashback "
excerpt: "아이티윌 0526_(8)"
categories:
      - ORACLE11g
tags:
      -  ORACLE11g
      - TIL
last-modified-at: 2025-05-26T9:38
---

# flashback - 복원

## Flashback table(10g)

- 삭제한 테이블을 복원하는 SQL문

```sql
-- SQL developer에선 지원하지x
-- SQL plus command line에서 해보기

show recyclebin;
```

```sql
-- storage 꽉 차면 PURGE 됨
DROP TABLE hr.emp;

-- SQL developer
SELECT * FROM user_recyclebin;

-- 휴지통 비우기
PURGE recyclebin;
```

```sql
-- 이름만 rename작업한다. emp -> BIN$Easb09dyT/i7oaZ3Z+xkaQ==$0
SELECT * FROM "BIN$Easb09dyT/i7oaZ3Z+xkaQ==$0";
```

```sql
SELECT * FROM user_recyclebin;

-- 삭제된 테이블 복원
-- 제약조건, 인덱스 복원되지만 원래 이름으로 복원되지 x , 수동으로 이름을 수정해야 한다.
FLASHBACK TABLE emp TO BEFORE DROP;
SELECT * FROM user_recyclebin;
SELECT * FROM hr.emp;
SELECT * FROM user_constraints WHERE table_name = 'EMP';
SELECT * FROM user_cons_columns WHERE table_name = 'EMP';
SELECT * FROM user_indexes WHERE table_name = 'EMP';
SELECT * FROM user_ind_columns WHERE table_name = 'EMP';
```

```sql
-- recyclebin에 동일한 테이블이 있는 경우는 가장 최근 삭제한 테이블을 복원시킨다.
-- stack구조 형식 후입선출(Last In First Out : LIFO)
FLASHBACK TABLE emp TO BEFORE DROP;
```

```sql
DROP TABLE hr.emp;

SELECT * FROM user_recyclebin;

-- recyclebin에 동일한 테이블이 있는 경우는 가장 최근 삭제한 테이블을 복원시킨다.
-- stack구조 형식 후입선출(Last In First Out : LIFO)
-- 복원해야할 테이블 이름과 동일한 이름이 있을 경우 오류 발생
FLASHBACK TABLE emp TO BEFORE DROP;

-- 새로운 이름으로 복원
FLASHBACK TABLE emp TO BEFORE DROP RENAME TO emp_new;

-- 테이블 영구히 삭제
DROP TABLE hr.emp_new PURGE;
SELECT * FROM user_recyclebin;

DROP TABLE hr.emp;
DROP TABLE hr.dept;

-- RECYCLEBIN안에 있는 전부 영구히 삭제
-- PURGE RECYCLEBIN

-- 특정한 테이블만 RECYCLEBIN에서 삭제
PURGE TABLE EMP;
SELECT * FROM user_recyclebin;
```

![image.png](/assets/20250526/5.png)