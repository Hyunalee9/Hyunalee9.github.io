---
title: "Server Process"
excerpt: "아이티윌 0714_(1)"
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-07-14T17:13
---

# Server Process

📍 **server process 구성**

오라클은 server process를 생성하여 유저가 전달한 SQL문을 처리한다.

1️⃣ Dedicated server : 하나의 server process가 하나의 user process만 처리한다. **user process (1) : server process(1)**

2️⃣ Shared server : 하나의 server process가 여러 user process를 처리한다.  **user process(n) : server process(1)**

📍 **PGA(Program Global Area)**

- 서버 프로세스에 대한 데이터, 제어 정보를 포함하는 전용(private) 메모리 영역이다.

📍 UGA(User Global Area)

- 유저 세션 데이터 : 보안, 리소스 사용 정보를 포함
- cursor stat : SQL문 실행 메모리 영역
- SQL 작업 영역 : 정렬 영역, 해시 영역, 비트맵 인덱스 생성, 비트맵 인덱스 병합
- stack space : 프로세스 변수

![image.png](/assets/20250714/1.png)

클라이언트 (IP,SID,PORT 가지고) 접속 시 리스너가 클라이언트 정보 체크 후 서버 프로세스 생성해준다. 유저와 붙이는 작업까지 한다.

## Dedicated Server Process

📍 **PGA**

▶️ SQL Work Areas (SQL 작업 영역) :

- Soft Merge Join
- Hash Join
- SORT OPERATION
    - ORDER BY
    - 집합 연산자(UNION ALL 제외한 모든 집합 연산자)
    - GROUP BY ( 9i버전 이전, 이후에는 Hash 영역으로 들어간다.)
    - UNIQUE, DISTINCT 키워드도 이전에는 sort operation , 9i의 r2버전에서 Hash로 바뀜
    - 객체 생성 중 인덱스 생성(primary key, index key)
    - 통계 수집.
- Sort Merge Join

▶️ Session Memory ( User Session Data) 

resource 관리

▶️ Dedicated Server process(1:1)의 문제점 : 클라이언트 수 많으면 , 공간 부족 → **shared server processes 로 만들자.**

## Shared Server Processes

- PGA 메모리를 공유하지만 stack space를 제외한 나머지 UGA는 공유해서는 안된다.
- UGA는 SGA 영역에 Large pool이 설정되어 있으면 UGA가 이곳에 생성되고 Large pool이 설정되어 있지 않으면 shared pool 공간에 UGA가 생성된다.
- dispatcher 프로세스가 사용된다.

📍 **Shared server 환경 이점**

- server process 수를 줄일 수 있다.
- 메모리 사용량과 시스템 오버헤드를 줄일 수 있다.
- 허용되는 유저 수를 증가 시킬 수 있다.

📍 **Shared server 환경이지만 수행하지 않아야 하는 작업 (무조건 Dedicated server 환경에서 수행하는 작업)**

- 데이터베이스 관리 작업(DBA 작업)
- 백업, 복구
- 일괄처리, 대량 로드 작업
- 병렬 처리 작업

![image.png](/assets/20250714/2.png)

▶️ 문제 : 어떤 유저가 실행한 SQL인지 모른다. → dispatcher를 만들어 리스너를 시켜서 각 클라이언트에 분배시킨다.

📍 dispatcher : A 유저가 던진 SQL문을 가지고 Large pool 로 가서 (shared pool 의 Request Queue, Response Queue 공간 부족 때문에 ) request queue에 등록한다. 그럼 서버 프로세서가 그걸 보고 Parse- Bind- Execute (결과 집합을 Response Queue에 담음) -  fetch(디스패처가 결과 집합을 가지고 클라이언트에게 전달. )

📍 Response Queue 갯수는 dispatcher의 갯수와 1:1 대응된다. 

▶️ **shared server 환경 요청 처리**

1️⃣ user 가 요청한 SQL문을 dispatcher로 전송한다.

2️⃣ dispatcher는 SQL문을 request queue 에 저장

3️⃣ shared server process는 request queue 에 있는 SQL문을 처리한다.

4️⃣ shared server process는 처리한 SQL 정보를 dispatcher의 response queue 에 저장한다.

5️⃣ dispatcher는 response queue 에 있는 정보를 user한테 반환한다.

📍 **request queue**

- 모든 dispatcher는 하나의 request queue를 공유한다.
- 요청은 first-in  first-out (FIFO) 기준으로 처리한다.

📍 **respnse queue**

- 각 dispatcher별로 response queue를 사용한다.
- shared sever process는 처리한 정보를 dispatcher의 response queue에 저장한다.

🌳 local 환경

```sql
[oracle@oracle ~]$ lsnrctl service

LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 14-JUL-2025 11:52:23

Copyright (c) 1991, 2019, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=oracle)(PORT=1521)))
Services Summary...
Service "ora19c" has 1 instance(s).
  Instance "ora19c", status UNKNOWN, has 1 handler(s) for this service...
    Handler(s):
      "**DEDICATED**" established:14 refused:0
         LOCAL SERVER
The command completed successfully
```

📍 처음에 시작되는 dispatchers 수를 지정

```sql
SYS@ora19c> show parameter dispatchers  -- dynamic parameter

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
dispatchers                          string      (PROTOCOL=TCP) (SERVICE=ora19c
                                                 XDB)
max_dispatchers                      integer

select * from v$parameter where name = 'dispatchers';  -- 동적, 정적 파라미터 확인
```

![image.png](/assets/20250714/3.png)

```sql
alter system set dispatchers = "(PROTOCOL=TCP)(dispatchers=5)";

! lsnrctl service -- 확인했을 때 디스패처 갯수 확인
```

📍 최대로 생성할 수 있는 dispatchers 수

```sql
select * from v$parameter where name = 'max_dispatchers';
alter system set max_dispatchers = 10;
show parameter disparchers -- (1)
show parameter shared_servers -- (2) 
```

![(1)](/assets/20250714/4.png)

(1)

![(2)](/assets/20250714/5.png)

(2)

📍 시작시 생성할 shared server process 수

```sql
select * from v$parameter where name = 'shared_servers';
alter system set shared_servers = 2;
```

📍 최대로 생성할 수 있는 shared server process 수

```sql
select * from v$parameter where name = 'max_shared_servers';
alter system set max_shared_servers = 10;
```

📍 클라이언트 측에서 tnsnames.ora 구성

- dedicated server 환경

```sql
ORA19C =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.56.110)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ora19c)
    )
  )
```

- shared server 환경

```sql
-- tnsnames.ora 잠시 수정  
ORA19C =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.56.110)(PORT = 1521))
    (CONNECT_DATA =
      (SERVICE_NAME = ora19c)
    )
  )
```

📍 sawon01 → shared pool

📍 shared pool 해제

```sql
select * from v$session where username in ('HR','SAWON01', 'SAWON02');

alter system set shared_servers = 0;
alter system set dispatchers = '';

-- ▶️ sys 
! lsnrctl reload
! lsnrctl services

-- ▶️ 접속 끊고 다시 연결 
-- ▶️ sawon01 -> shared pool 이더라도 디스패처 존재하지 않으니까 dedicated 환경이 된다. 
```