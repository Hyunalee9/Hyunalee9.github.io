---
title: "Autonomous Transaction"
excerpt: "아이티윌 0616_(2) "
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-06-16T18:37
---

# Autonomous Transaction

➡  **아래처럼** **다른 DML 에 영향을 미치는 트랜잭션 제어는 어떻게 해야 할까?**

```sql
CREATE TABLE hr.log_table
(username varchar2(30),
date_time timestamp,
message varchar2(4000));

desc hr.log_table;

CREATE TABLE hr.temp_table(n number);

desc temp_table;

CREATE OR REPLACE PROCEDURE hr.log_message(p_message IN varchar2)
IS 
BEGIN
    INSERT INTO hr.log_table(username,date_time,message)
    VALUES(user, current_date, p_message);
    COMMIT;
END log_message;
/

BEGIN
    hr.log_message();
    -- 프로시저 내의 COMMIT이 하기 INSERT 까지도 영향을 미친다.
    INSERT INTO hr.temp_table(n) VALUES(12345);
    hr.log_message('두번째 입력'); 
    ROLLBACK;
END;
/
```

**Autonomous Transaction (자율적인 (독립) 트랜잭션)**

- 트랜잭션 제어를  "부분적으로" 하게 만든다.  (트리거 내에서 TCL 쓰지 못하기 때문)
- 프로시저 내에서도 만들 수 있음 (COMMIT, ROLLBACK을 하든 무조건 흔적이 남을 수 있도록 할 수 있음)

➡ 트리거 안에서 자율 트랜잭션 작업을 수행

```sql
CREATE TABLE hr.trigger_tab(id number, name varchar2(10), day timestamp default systimestamp);

-- 로그용.
CREATE TABLE hr.trigger_log(id number, name varchar2(10), log_day timestamp default systimestamp);
```

```sql
-- INSERT/DELETE/UPDATE 수행할 때, trigger_log 테이블에 로그가 쌓인다.
CREATE OR REPLACE TRIGGER hr.log_trigger
AFTER
INSERT OR DELETE OR UPDATE ON hr.trigger_tab
FOR EACH ROW
DECLARE
	PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
	INSERT INTO hr.trigger_log(id,name,log_day) VALUES( :new.id, :new.name, default);
END;
/
```

➡ 바깥 DML에 영향을 주지 않음.

➡ 롤백해도 감사 로그 정보는 그대로 쌓인다.

```sql
SELECT * FROM hr.trigger_log
```