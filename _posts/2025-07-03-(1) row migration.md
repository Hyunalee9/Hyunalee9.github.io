---
title: "[50일차] 세그먼트 관리 방식과 Row Migration"
excerpt: "아이티윌 0703_(1) "
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-07-03T23:00
---

# 오라클 블록 공간 관리 & Row Migration

1️⃣ **데이터 블록의 기본 구조**

![image.png](/assets/20250703/1.png)

| 위치 | 내용 |
| --- | --- |
| **Block Header (위에서 아래로)** | 트랜잭션 슬롯, SCN, ITL (Interested Transaction List) |
| **Row Data (아래에서 위로)** | 실제 데이터가 저장됨 |

2️⃣ 데이터 수정 & Row Migration 발생 과정

- `400번 사원`의 **주소(10Byte)** → **100Byte** 로 **업데이트** 시도
- → 블록 내 **여유 공간 부족** → **다른 블록으로 row 이동 (Row Migration)**

| 단계 | 설명 |
| --- | --- |
| 트랜잭션 시작 | 트랜잭션 슬롯 생성 (Block Header에) |
| 수정 시도 | 공간 부족 시 Row Migration 발생 |
| 새로운 블록으로 이동 | 데이터 이동 후, 기존 블록에 **포인터(주소)** 남김 |
| Commit | SCN 할당, Undo Segment 기록, Redo 기록 (LGWR 작동) |
| 부작용 | 행 찾을 때 **I/O 2번 발생** (성능 저하) |

3️⃣ Row Migration 해결 방법

| 시기 | 방법 | 설명 |
| --- | --- | --- |
| **사전 예방** | `PCT_FREE` 값 증가. block 안에 row들을 작게하여 buffer busy wait 줄인다. | 여유 공간 확보하여 row migration 예방 (단, 공간 낭비 위험) |
| **사후 조치** | 테이블 재구성 | `ALTER TABLE MOVE` 또는 **온라인 재구성 (11g~)** 사용 |

4️⃣ 블록 파라미터

| 파라미터 | 역할 |
| --- | --- |
| `PCT_FREE` | 행의 업데이트 공간 확보 비율 (행 크기 증가 대비) |
| `PCT_USED` | 블록이 다시 Free List에 추가되는 임계점 |

5️⃣ 공간 관리 방식

| 방식 | 특징 | 관리 대상 | 재사용 블록 판단 |
| --- | --- | --- | --- |
| **FLM (Free List Management)** | 수동 공간 관리 | Linked List | `PCT_USED` 기준 |
| **ASSM (Automatic Segment Space Management)** | 자동 공간 관리 | Bitmap | 비트맵 상태 |

```sql
-- 📌 참고 : 각각의 테이블스페이스들의 공간 관리 방식 | MANUAL : FLM | AUTO : ASSM  
select * from dba_tablespaces;
```

![image.png](/assets/20250703/2.png)

### 📌 FLM (Free List Management)

```sql
CREATE TABLESPACE flm_tab
DATAFILE '/u01/app/oracle/oradata/ORA19C/flm_tab01.dbf' SIZE 5M
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M
SEGMENT SPACE MANAGEMENT MANUAL;
```

- 딕셔너리 테이블 - System, undo, temp은 FLM 방식 **(MANUAL)**
- pctused, freelists, freelists groups 사용하는 방식
- Free Block ➔ Linked List 관리
- 동시에 Insert 시 Wait 발생 (Buffer Busy Wait)
- FreeLists, FreeLists Groups 등 스토리지 파라미터 사용
- free lists란 free block을 linked list 알고리즘으로 관리하는 기법

---

📌 **그림 정리**

![image.png](/assets/20250703/3.png)

📌 **data buffer busy wait 를 설명하기 위한 가정**

- A 세션: INSERT 시도
- B 세션: 동시에 INSERT 시도
- 같은 테이블, 같은 세그먼트, 같은 테이블스페이스

| 단계 | 설명 |
| --- | --- |
| 1️⃣ | A와 B가 동시에 INSERT 시도 |
| 2️⃣ | A가 먼저 Free List의 블록을 점유 |
| 3️⃣ | B는 같은 Free List에 접근하려다가 대기 (👉 **`buffer busy wait` 이벤트 발생**) |
| 4️⃣ | Free List가 **단 1개**밖에 없어서 동시에 처리 불가 |
| 5️⃣ | 결과적으로 **INSERT 시 성능 저하 + 대기 발생** |
- **Free List 수가 부족** (기본값: 1)
- **동시 다중 사용자 환경에서 Free List 경합** 발생

| 방법 | 설명 |
| --- | --- |
| ✅ Free List 개수 증가 | `FREELISTS` 스토리지 파라미터 사용 (예: `FREELISTS 2` 설정) |
| ✅ Free List Group 사용 (RAC 환경) | `FREELIST GROUPS` 설정 |
| ✅ ASSM 사용 (Automatic Segment Space Management) | Free List 자체 관리 (경합 완화) |

### 📌 ASSM (Automatic Segment Space Management)

```sql
CREATE TABLESPACE assm_tab
DATAFILE '/u01/app/oracle/oradata/ORA19C/assm_tab01.dbf' SIZE 5M
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M
SEGMENT SPACE MANAGEMENT AUTO;
```

- 각 블록의 상태를 비트맵(bitmap)값으로 관리하는 방법
- 공간 사용 상태 → **6단계 비트맵** 으로 관리
- **Insert 성능 향상**, **자동 공간 관리**
- `PCT_FREE`, `PCT_USED` 비활성화됨

6️⃣ ASSM 블록 상태 (6단계)

| 단계 | 상태 | 설명 |
| --- | --- | --- |
| #1 | Unformatted | HWM 뒤 블록 (미사용) |
| #2 | Full | 공간 없음 |
| #3 | 0~25% Free |  |
| #4 | 25~50% Free |  |
| #5 | 50~75% Free |  |
| #6 | 75~100% Free |  |

7️⃣ High Water Mark (HWM)

| 용어 | 설명 |
| --- | --- |
| **HWM** | 테이블/세그먼트에서 마지막으로 사용된 블록의 경계 |
| **FLM HWM 조정** | 수동, 5배수 단위로 조정 |
| **ASSM HWM 조정** | 자동 |

📍 ASSM 방식 

| #1 | #2 (PCT_USED 기준 0 ~ 25% free 공간 상태) | #3  | #4 | #5 | #6 (HWM)  | #7 | #8 |
| --- | --- | --- | --- | --- | --- | --- | --- |

▶️ 비트맵 방식으로 관리

| #1 | #2 3️⃣ | #3 | #4 |
| --- | --- | --- | --- |
| #5 | #6 | #7 unformatted 블록 | #8 unformatted 블록 |

문제점 : 비트맵 안에 새로운 데이터가 입력되는 순간 free 공간 사라짐 → access 못해서 buffer busy wait 발생. ASSM 방식이라고 해서 buffer busy wait이 발생하지 않은 건 아님 

**실습**

| 구분 | FLM (MANUAL) | ASSM (AUTO) |
| --- | --- | --- |
| 공간 관리 | Free List 직접 설정 | 자동 관리 |
| 경합 방지 | `FREELISTS` 수동 설정 필요 | 자동 분산 |
| 블록 헤더 | 항상 고정 위치 (처음부터) | 가변적 (블록 구조 다름) |
| 권장 사용 | OLTP 시스템, 수동 최적화 | 대부분의 현대 시스템 (기본값) |

```bash
-- 📌 테이블스페이스 생성 (FLM: Free List Management 방식)
CREATE TABLESPACE flm_tab
DATAFILE '/u01/app/oracle/oradata/ORA19C/flm_tab01.dbf' SIZE 5M
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M
SEGMENT SPACE MANAGEMENT MANUAL;  -- ❗ 수동 공간 관리 (FLM 방식)

-- 📌 테이블스페이스 생성 (ASSM: Automatic Segment Space Management 방식)
CREATE TABLESPACE assm_tab
DATAFILE '/u01/app/oracle/oradata/ORA19C/assm_tab01.dbf' SIZE 5M
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M
SEGMENT SPACE MANAGEMENT AUTO;  -- ✅ 자동 공간 관리 (ASSM 방식)

-- 기존 emp 테이블 삭제 (안전하게 PURGE까지)
DROP TABLE hr.emp PURGE;

-- 📌 FLM 테이블스페이스에 emp 테이블 생성 (Free List 사용)
CREATE TABLE hr.emp
PCTFREE 20                      -- 📌 블록당 20% 공간 확보 (UPDATE 대비)
PCTUSED 30                      -- 📌 사용률 30% 미만이면 Free List 등록
STORAGE(FREELISTS 2)            -- 📌 Free List 2개 → 동시 INSERT 시 대기 방지
TABLESPACE flm_tab
AS SELECT * FROM hr.employees;   -- 📌 데이터 복사

-- 📌 블록 파라미터 수정 (기존 테이블에 적용)
ALTER TABLE hr.emp
PCTFREE 30                      -- 📌 PCTFREE 30%로 증가
PCTUSED 60                      -- 📌 PCTUSED 60%로 증가
STORAGE(FREELISTS 5);           -- 📌 Free List 5개로 확장 → 경합 방지 강화

-- 📌 테이블 정보 확인
SELECT * FROM dba_tables WHERE owner = 'HR' and table_name = 'EMP';

-- 📌 세그먼트 정보 확인
SELECT * FROM dba_segments WHERE owner = 'HR' and segment_name = 'EMP';

-- 📌 익스텐트 정보 확인
SELECT * FROM dba_extents WHERE owner = 'HR' and segment_name = 'EMP';

-- 기존 dept 테이블 삭제 (안전하게 PURGE까지)
DROP TABLE hr.dept PURGE;

-- 📌 ASSM 테이블스페이스에 dept 테이블 생성 (자동 공간 관리)
CREATE TABLE hr.dept
TABLESPACE assm_tab
AS SELECT * FROM hr.departments;

-- 📌 테이블 정보 확인 (ASSM)
SELECT * FROM dba_tables WHERE owner = 'HR' and table_name = 'DEPT';

-- 📌 ASSM 세그먼트 정보 (헤더가 FLM처럼 처음부터 시작하지 않음 → ASSM의 특징)
SELECT * FROM dba_segments WHERE owner = 'HR' and segment_name = 'DEPT';

-- 📌 ASSM 익스텐트 정보 확인
SELECT * FROM dba_extents WHERE owner = 'HR' and segment_name = 'DEPT';

-- 📌 테이블스페이스 삭제 (FLM)
DROP TABLESPACE flm_tab INCLUDING CONTENTS AND DATAFILES;

-- 📌 테이블스페이스 삭제 (ASSM)
DROP TABLESPACE assm_tab INCLUDING CONTENTS AND DATAFILES;

-- 📌 예시로 작성한 일반 유저 테이블스페이스 삭제 (필요시)
DROP TABLESPACE userdata INCLUDING CONTENTS AND DATAFILES;
```

**실습2 - 테이블 스페이스 크기 조정**

| 개념 | 설명 |
| --- | --- |
| **AUTOEXTEND ON** | 공간 부족 시 자동으로 데이터파일 크기 확장 |
| **INCREMENT BY** | AUTOEXTEND 시 얼마씩 확장할지 (단위: 블록) |
| **RESIZE** | DBA가 직접 데이터파일 크기를 수동으로 설정 |
| **EXTENT MANAGEMENT LOCAL UNIFORM SIZE** | 익스텐트 크기 고정 (여기선 64KB) |
| **SEGMENT SPACE MANAGEMENT AUTO** | ASSM 사용 (Free List 불필요, 자동 공간 관리) |

```bash
-- 📌 테이블스페이스 생성 (데이터파일: 5MB, 확장 기능 없음, ASSM 사용)
CREATE TABLESPACE insa_tab
DATAFILE '/u01/app/oracle/oradata/ORA19C/insa_tab01.dbf' SIZE 5M
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 64K         -- 📌 익스텐트 크기 고정: 64KB
SEGMENT SPACE MANAGEMENT AUTO;                   -- 📌 ASSM (자동 세그먼트 공간 관리)

-- 📌 현재 테이블스페이스 목록 확인
SELECT * FROM dba_tablespaces;

-- 📌 현재 데이터파일 목록 확인
SELECT * FROM dba_data_files;

-- 📌 테이블스페이스 생성 (데이터파일: 5MB, 자동확장 활성화, ASSM 사용)
CREATE TABLESPACE insa_tab
DATAFILE '/u01/app/oracle/oradata/ORA19C/insa_tab01.dbf' SIZE 5M AUTOEXTEND ON  -- ✅ 자동 확장 기능 켬
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 64K
SEGMENT SPACE MANAGEMENT AUTO;

-- ❗ incremental by: AUTOEXTEND 시 얼마나 확장할지 결정 (기본값: 1 블록 단위, 보통 8KB)

-- 📌 테이블스페이스 생성 후, 이미 존재하는 데이터파일에 대해 자동 확장 활성화
ALTER DATABASE DATAFILE '/u01/app/oracle/oradata/ORA19C/insa_tab01.dbf' AUTOEXTEND ON;

-- 📌 변경된 데이터파일 목록 확인 (AUTOEXTEND 속성 확인 가능)
SELECT * FROM dba_data_files;

-- 📌 데이터파일 크기를 수동으로 10MB로 변경 (확장)
ALTER DATABASE DATAFILE '/u01/app/oracle/oradata/ORA19C/insa_tab01.dbf' RESIZE 10M;

-- 📌 최종 데이터파일 목록 재확인
SELECT * FROM dba_data_files;
```

![image.png](/assets/20250703/4.png)

```bash
-- 테이블스페이스에 데이터 파일 추가
ALTER TABLESPACE insa_tab ADD DATAFILE '/u01/app/oracle/oradata/ORA19C/insa_tab02.dbf' SIZE 10M AUTOEXTEND ON;

select * from dba_data_files;
```

![image.png](/assets/20250703/5.png)

```bash
CREATE TABLE hr.insa
TABLESPACE insa_tab
AS SELECT * FROM hr.employees;

insert into hr.insa
select * from hr.insa;

-- extent : 사용하고 있는 extent 수 
SELECT * FROM dba_segments WHERE segment_name = 'INSA';
SELECT * FROM dba_extents WHERE segment_name = 'INSA';

SELECT * FROM v$datafile;
```

![image.png](/assets/20250703/6.png)

각각의 checkpoint 정보가 다를 수 있다. 동일하게 맞춰지는 시점은 full checkpoint가 발생할 때

```bash
ALTER SYSTEM checkpoint;
SELECT * FROM v$datafile;
```

![image.png](/assets/20250703/7.png)

```bash
SELECT d.file_id, d.tablespace_name, d.file_name, v.checkpoint_change#, v.enabled
FROM v$datafile v, dba_data_files d
WHERE v.file# = d.file_id;
```

▶️ read write 작업 이니까 위와 같은 작업 가능

| 상태 | 의미 | 사용 목적 |
| --- | --- | --- |
| **READ WRITE** | 기본 상태 (읽기/쓰기 모두 가능) | 일반적인 DML/DDL 작업 가능 |
| **READ ONLY** | 읽기만 가능, 쓰기 불가 | 보관용, 데이터 변경 방지 (데이터 안정화) |
| **OFFLINE** | 완전히 사용 불가 | 유지보수, 특정 테이블스페이스 비활성화 |
| **ONLINE** | 활성화 상태 | 정상 사용 가능 |

📌  **테이블스페이스 상태 변경 방법**

```sql
-- ✅ READ ONLY로 변경 (쓰기 금지)
ALTER TABLESPACE users READ ONLY;

-- ✅ 다시 READ WRITE로 변경 (쓰기 허용)
ALTER TABLESPACE users READ WRITE;

-- ✅ OFFLINE (테이블스페이스 완전 비활성화)
ALTER TABLESPACE users OFFLINE;

-- ✅ ONLINE (다시 활성화)
ALTER TABLESPACE users ONLINE;
```

📌 **왜?**

| 상태 | 활용 상황 |
| --- | --- |
| **READ ONLY** | 데이터 아카이브, 감사 목적, 실수로 데이터 변경 방지 |
| **OFFLINE** | 유지보수, 데이터 파일 이동, 백업 복원 시 |

📍 테이블스페이스 read only

- read only tablespace 안에 있는 테이블의 데이터를 읽을 수만 있다. (SELECT 문 수행)
- DML 불허
- 객체 삭제는 가능
- partial checkpoint 발생 (해당 테이블스페이스에 버퍼 캐시 블록만  flush하여 읽기 일관성)

```bash
ALTER TABLESPACE insa_tab READ ONLY;

SELECT d.file_id, d.tablespace_name, d.file_name, v.checkpoint_change#, v.enabled
FROM v$datafile v, dba_data_files d
WHERE v.file# = d.file_id;

--> 체크 포인트가 바뀐 것을 알 수 있다. 

UPDATE hr.insa SET salary = salary * 1.1 WHERE employee_id = 100;

DELETE FROM hr.insa WHERE employee_id = 100;

TRUNCATE TABLE hr.insa;
CREATE TABLE hr.dept
TABLESPACE insa_tab
AS SELECT * FROM hr.departments;

-- DML, DDL 작업은 안된다. 

ALTER TABLE hr.insa ADD dept_name varchar2(30);
ALTER TABLE hr.insa MODIFY last_name VARCHAR2(30);
-- 컬럼 추가는 아무런 문제가 되지 않는다. 

ALTER TABLE hr.insa DROP COLUMN last_name;
-- 컬럼 삭제는 안됨.

DROP TABLE hr.insa PURGE;
-- 테이블을 비롯한 객체 삭제는 가능하다. 
```

![image.png](/assets/20250703/8.png)

```sql
-- read write 모드로 바꿈
ALTER TABLESPACE insa_tab read write;

-- 확인 
SELECT d.file_id, d.tablespace_name, d.file_name, v.checkpoint_change#, v.enabled
FROM v$datafile v, dba_data_files d
WHERE v.file# = d.file_id;
```

📍 테이블 스페이스 offline

- 테이블 스페이스에 속한 객체들을 사용할 수 없다.
- partial checkpoint 발생
- offline 으로 설정할 수 없는 테이블 스페이스
    - system
    - active undo segment가 있는 tablespace
    - default temporary tablespace

```sql
SELECT * FROM dba_tablespaces;
```

- offline 옵션
    - normal           : partial checkpoint 발생
    - temporary     : 가능한 데이터 파일에 속한 dirty buffer만 디스크로 쓰는 작업 수행 → 복구 필요
    - immediate     :  checkpoint 발생하지 않고 offline으로 수행된다. archivelog mode에서 수행하는 옵션  → 복구 필요.

▶️ 테이블 스페이스 online으로 설정    

```sql
ALTER tablespace insa_tab online;
```

```sql
📌 offline 은 언제 사용할까?

■ 데이터파일 이관작업

DROP TABLESPACE insa_tab INCLUDING CONTENTS AND DATAFILES;

CREATE TABLESPACE insa_tab
DATAFILE '/home/oracle/insa_tab01.dbf' SIZE 5M
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 64k
SEGMENT SPACE MANAGEMENT AUTO;

select d.file_id, d.tablespace_name, d.file_name, v.checkpoint_change#, v.enabled, d.online_status
from v$datafile v, dba_data_files d
where v.file# = d.file_id;

select * from dba_tablespaces;

create table hr.insa
tablespace insa_tab
as select * from hr.employees;

select count(*) from hr.insa;
```

1️⃣ 테이블 스페이스를 offline → 체크 포인트 발생

```sql
ALTER TABLESPACE insa_tab offline normal;
```

2️⃣ 물리적으로 데이터 파일 이동 (putty) 

```sql
mv -v /home/oracle/insa_tab01.dbf /u01/app/oracle/oradata/ORA19C/
```

3️⃣ 기존 파일을 새로운 파일 위치로 수정 (sql developer)

(물리적인 구조 정보는 컨트롤 파일이 가지고 있다.)

```sql
ALTER TABLESPACE insa_tab RENAME datafile '/home/oracle/insa_tab01.dbf' to '/u01/app/oracle/oradata/ORA19C/insa_tab01.dbf';

select d.file_id, d.tablespace_name, d.file_name, v.checkpoint_change#, v.enabled, d.online_status
from v$datafile v, dba_data_files d
where v.file# = d.file_id;
```

![image.png](/assets/20250703/9.png)

4️⃣ 테이블 스페이스 online

```sql
ALTER TABLESPACE insa_tab online;

select d.file_id, d.tablespace_name, d.file_name, v.checkpoint_change#, v.enabled, d.online_status
from v$datafile v, dba_data_files d
where v.file# = d.file_id;
```

![image.png](/assets/20250703/10.png)