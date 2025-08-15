---
title: "[49일차] 논리적 구조 :  tablespace"
excerpt: "아이티윌 0702_(3) "
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-07-02T23:00
---

▶️ 오라클은 데이터를 논리적으로 tablespace 에 저장하고 물리적으로 data file 저장한다.

🌳 tablespace

- 오라클 데이터베이스의 데이터는 tablespace에 저장
- tablespace는 하나 이상의 data file로 구성한다.
- tablespace는 하나 이상의 segment 로 구성한다.

🌳 SYSTEM tablespace

- 데이터베이스와 함께 생성됩니다.
- data dictionary를 포함하고 있다.
- system undo segment를 포함하고 있다.

🌳 Non System tablespace

- 사용자가 직접 생성하거나 운영하는 목적의 일반 테이블스페이스
- undo segment, temporary segment, 일반적인 data segment, index segment 가 저장되는 테이블 스페이스
- 데이터베이스 관리가 유연해진다.

🌳 data file

- data file은 오라클 서버에 실행중인 운영체제를 따르는 물리적 구조이다.
- 하나의 테이블 스페이스 및 하나의 데이터베이스에만 속할 수 있는 파일이다.

🌳 segment

- segment는 tablespace 안의 특정한 논리적 저장 영역 구조에 할당된 영역이다.
- 테이블, 인덱스, 언두, lob 등 오라클이 제공하는 논리적 공간을 의미한다.
- segment는 tablespace에 속한 여러 data file로 확장할 수 있다.

🌳 extent

- extent는 tablespace내에 segment가 사용하는 공간 영역이다.
- 하나 이상의 extent로 segment를 구성한다.
- extent는 연속된 오라클 블록으로 구성되어 있다.
- segment에서 extent가 할당
    - create(생성할 경우)
    - extent(확장될 경우)
    - alter(변경될 경우)
- segment에서 extent가 해제
    - drop(삭제할 경우)
    - alter(변경될 경우)
    - truncate(잘릴 경우)

🌳 block

- 오라클 데이터베이스의 데이터(row)는 블록에 저장된다.
- 오라클의 최소 I/O단위
- 오라클 블록 크기는 2K, 4K, 8K, 16K, 32K
- 기본 블록 크기는 db_block_size 파라미터로 결정된다. 기본 크기는 8k
- 오라클 블록은 OS 블록을 이용해서 생성한다.

▶️  딕셔너리 관리되는 테이블 스페이스 (dictionary managed tablespace) → **old 해서 10g 이후로는 로컬**

📌 **흐름**

> **emp 세그먼트 생성 → extent 할당→ FET$ (사용가능한 extent 정보) → UET$(사용가능한 extent 정보) → 성능은 나쁘다.**
> 
- 사용 가능한 extent에 대해서 시스템 테이블스페이스 데이터 딕셔너리에서 관리한다.
    - fet$ : free extent 정보
    - uet$ : used extent 정보
- extent 할당하거나 할당이 해제될 때 데이터 딕셔너리 테이블에 대해서 조회, 갱신이 발생한다.
- enq : ST - contention 이벤트가 발생

ex) 

```bash
CREATE TABLESPACE user_dict  -- 테이블스페이스 이름: user_dict
DATAFILE '/u01/app/oracle/oradata/ORA19C/user_dict01.dbf' SIZE 10M  -- 데이터파일 경로 및 크기: 10MB
EXTENT MANAGEMENT DICTIONARY  -- Dictionary Managed 테이블스페이스 (구식 방식)
BLOCKSIZE 8K  -- 블록 크기: 8KB
DEFAULT STORAGE (  -- 기본 스토리지 파라미터
  INITIAL 1M      -- 초기 익스텐트 크기: 1MB
  NEXT 1M         -- 다음 익스텐트 크기: 1MB
  PCTINCREASE 0   -- 다음 익스텐트 증가율: 0%
);
```

▶️  **로컬로 관리되는 테이블 스페이스 (locally managed tablespace)**

- 사용가능한 extent에 대해서 테이블스페이스 (각자 스스로)에서 관리하는 방식
- 비트맵 방식으로 사용가능한 extent 정보를 관리한다.
- 데이터 딕셔너리 테이블 경합을 줄였다.
- 공간을 할당하거나 해제할 때 undo 정보를 생성하지 않는다.

```bash
CREATE TABLESPACE userdata  -- 테이블스페이스 이름: userdata
DATAFILE '/u01/app/oracle/oradata/ORA19C/userdata01.dbf' SIZE 10M  -- 데이터파일 경로 및 크기: 10MB
EXTENT MANAGEMENT LOCAL  -- 로컬 관리 테이블스페이스 (시스템이 Extent 관리)
UNIFORM SIZE 64K;  -- Extent 크기 고정: 64KB

🌳 UNIFORM SIZE : 바이트 크기의 일정한 extent로 테이블 스페이스를 관리한다.
    kb, mb 단위의 extent 크기를 지정할 수 있다.
🌳 UNIFORM : extent 크기의 기본값은 1m 설정된다.
🌳 AUTOALLOCATE : extent 크기는 처음은 64k로 생성한 후 다음 extent 다음 크기는 오라클이 결정하는 방   
```

▶️ 이렇게 간편히 수행해도 가능

```bash
CREATE TABLESPACE userdata
DATAFILE '/u01/app/oracle/oradata/ORA19C/userdata01.dbf' SIZE 10M;
```

▶️ 테이블스페이스 삭제 

```bash
DROP TABLESPACE userdata INCLUDING CONTENTS AND DATAFILES;
```