---
title: "DB, Listner 상태 확인"
excerpt: "아이티윌 0623_(1) "
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-06-23T17:34
---

# 매일 아침 루틴 - DB, Listner 상태 확인

**DBA daily check routine**

```bash
# 오라클 서버에서 oracle 프로세스 상태 체크
ps -ef | grep oracle
```

1. 정상 작동

![image.png](/assets/20250623/1.png)

1. 비정상 

![image.png](/assets/20250623/2.png)

```bash
# listener 상태 체크 
lsnrctl status

# 만약 리스너가 존재하지 않는다면 
# 1) 환경 변수를 체크해봐야 함
# 환경 변수 확인
# ORACLE_SID ⇒ 인스턴스 이름 
cat .bash_profile
```

![image.png](/assets/20250623/3.png)

```bash
# 2) 리스너 스타트 
[oracle@oracle ~]$ lsnrctl start

# 다시 한번 상태 체크
[oracle@oracle ~]$ lsnrctl status
```

![image.png](/assets/20250623/4.png)

```bash
# 리스너와 DB 정상적으로 start 되었다면 인스턴스 이름과, 상태 확인해보기

SELECT instance_name, status from v$instance;
```

![image.png](/assets/20250623/5.png)

```bash
sqlplus / as sysdba 
SELECT instance_name, status from v$instance;  # 인스턴스 상태 확인 

📍참고 
▶️ ! -> os로 나가는 것
▶️ exit -> 뒤에 했던 작업으로 돌아가기
▶️ lsnrctl stop -> 리스너 내리기
```

▶️ DB가 떠있지 않은 경우 아래처럼 나온다. 

![image.png](/assets/20250623/6.png)

```bash
ps -ef | grep oracle  -- 아침에 출근하자마자 DB 상태 확인
```

▶️ startup  (sqlplus에서)

![image.png](/assets/20250623/7.png)

▶️ diag

![image.png](/assets/20250623/8.png)

![image.png](/assets/20250623/9.png)

→ alert.log ⇒ 오류 확인

```bash
# 경로 꼭 외우기. 
[oracle@oracle trace]$ pwd
/u01/app/oracle/diag/rdbms/ora19c/ora19c/trace
                          ------------> 이 부분은 다를 수 있음.
                          
# alert.log 창 띄우기     
tail -F alert_ora19c.log
                      
```

![image.png](/assets/20250623/10.png)

▶️ shutdown 하면 alert log에 로그가 적힌다.

📍 cmd

```bash
C:\Users\ITWILL>ping 192.168.56.110
```

▶️ SQL Developer (클라이언트) 

![image.png](/assets/20250623/11.png)

![image.png](/assets/20250623/12.png)

![image.png](/assets/20250623/13.png)

→ 리스너가 내려가 있기 때문에 이런 오류가 뜬다. 

→ 로컬은 문제 없지만, 클라이언트는 불가능.

→ 클라이언트와 서버 관계에서는 리스너가 무조건 떠 있어야한다.

→ 클라이언트 환경에서 접속할 때는 리스너가 떠야 한다.

```bash
# 리스너 시작
lsnrctl start
```

![image.png](/assets/20250623/14.png)

→ oracle_hr 유저 생성 

이름, 비번 hr

```bash
lsnrctl stop 하면 클라이언트 작업 불가능 하기 오류가 뜬다.
start up 해서 DB와 리스너를 시작하자.
```

![image.png](/assets/20250623/15.png)

▶️ 프롬프트에 현재 로그인한 유저와 연결 식별자를 표시한다.

![image.png](/assets/20250623/16.png)

```bash
cd $ORACLE_HOME/sqlplus/admin/glogin.sql
vi glogin.sql

G (맨 밑으로) -> o (한 칸 아래 입력)
set sqlprompt "_user'@'_connect_identifier> "

-> esc -> :wq

# 확인
sqlplus / as sysdba

# 이런식으로 어떤 유저가 로그인했는지, 어떤 DB로 붙었는지 떠야한다.
SYS@ora19c>
```

▶️ 명령 프롬프트에서 바꾸기

```bash
# windows Xe 버전 sqlplus 
# C:\oraclexe\app\oracle\product\11.2.0\server\sqlplus\admin  경로 -> glogin.sql

sqlplus hr/hr@192.168.56.110/ora19c  
```

![image.png](/assets/20250623/17.png)

![image.png](/assets/20250623/18.png)