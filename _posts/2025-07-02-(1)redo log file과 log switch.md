---
title: "[49일차] redo log file과 log switch"
excerpt: "아이티윌 0702_(1) "
categories:
      - ORACLE19c
tags:
      - ORACLE19c
      - TIL
last-modified-at: 2025-07-02T23:00
---

# redo log file과 log switch

📍 **redo log file ( 트랜잭션의 변경 사항을 기록하는 파일 )**

- 이 녀석은 i/o가 비교적 적게 든다.
- 데이터의 모든 변경 사항을 기록한다
- 복구 방식 제공, instance failure, media failure 시 복구 방식을 제공
- 최소 두 개 이상의 그룹이 필요하다. 순환 형식의 구조이기 때문에

▶️ 변경 사항은 디스크에 저장되기 전에 **미리 로그에 기록(Write-Ahead Logging)**돼, 장애 발생 시 데이터를 **복구(Recovery)**하는 데 사용

🌳 **Redo Log 구조**

| 구성 요소 | 설명 |
| --- | --- |
| **Redo Log Group** | 하나 이상의 로그 파일을 포함하는 그룹 (예: GROUP 1, 2, 3) |
| **Redo Log Member** | Redo Log Group 안에 있는 실제 파일 (복제본, 미러링 용도) |
| **Log Sequence Number** | 로그 변경 순서를 식별하는 번호 |

🌳 **작동 흐름**

- 사용자가 DML(INSERT, UPDATE 등) 수행
- 변경 내용은 메모리(DB Buffer Cache)에 저장됨
- 동시에 **변경 사항은 Redo Log Buffer에 기록**
- 커밋 또는 타이밍이 되면 → Redo Log File로 기록됨 (`LGWR` 프로세스가 기록)
- 장애 발생 시 → Redo Log File을 기반으로 복구 수행

🌳 **파일 위치**

```bash
SELECT GROUP#, MEMBER FROM V$LOGFILE;
```

![image.png](/assets/20250702/1.png)

## log switch

**online redo log file이 가득 차면 LGWR는 다음 redo group(inactive) 으로 이동해서 write 한다.**

➡ log switch → 체크 포인트 발생(partial) : redo log 는 순환 형식 → control file 정보를 기록한다.

➡ current (active, inactive) 상태는 overwrite 할 수 없다.

물리적인 redo log file이 있는 위치 정보

```bash
select * from v$logfile;
```

![image.png](/assets/20250702/2.png)

수동으로 로그 스위치 발생. 바꾸기 

```bash
alter system switch logfile;   -- x3 3번 함
```

![image.png](/assets/20250702/3.png)

뭐 체크 포인트가 덜 끝났기때문에 alert.log에 다음과 같이 보인다.

다음 그룹이 체크 포인트 아직 끝나지 않았기 때문에 

![image.png](/assets/20250702/4.png)

log switch 가 너무 많이 발생한다면? → redo log file이 잘못 구성되어있어서 

▶️ Checkpoint not complete

- LGWR 이 redo log file를 재사용하려고 하는데 아직 그 redo log file이 checkpoint가 끝나지 않은 상태임을 나타내는 이벤트이다.
- redo log file의 크기나 갯수가 작아 log switch가 빈번히 발생하는 경우 한바퀴 도는 시간이 너무 빠르게 되면 이 현상이 쉽게 발생한다.
- Checkpoint not complete 이벤트가 자주 발생할 경우 성능상 문제가 발생하는 시점이니 redo log file 의 크기를 개선한 그룹을 추가하는 게 좋다.

▶️ redo log group 추가 : log switch 너무 자주 발생하기 때문에 

```bash
select * from v$log;

-- redo log group 추가 
ALTER DATABASE ADD LOGFILE GROUP 4 '/u01/app/oracle/oradata/ORA19C/redo04.log' SIZE 200M;
ALTER DATABASE ADD LOGFILE GROUP 5 '/u01/app/oracle/oradata/ORA19C/redo05.log' SIZE 200M;

select * from v$logfile;
```

![image.png](/assets/20250702/5.png)

▶️ redo log group member 추가

```bash
-- 위치
cd /u01/app/oracle/fast_recovery_area/ORA19C/onlinelog 

-- 아래에 있는 것 추가할 것
redo01.log 추가

ALTER DATABASE ADD LOGFILE MEMBER
  '/u01/app/oracle/fast_recovery_area/ORA19C/onlinelog/redo01.log' TO GROUP 1,
  '/u01/app/oracle/fast_recovery_area/ORA19C/onlinelog/redo02.log' TO GROUP 2,
  '/u01/app/oracle/fast_recovery_area/ORA19C/onlinelog/redo03.log' TO GROUP 3,
  '/u01/app/oracle/fast_recovery_area/ORA19C/onlinelog/redo04.log' TO GROUP 4,
  '/u01/app/oracle/fast_recovery_area/ORA19C/onlinelog/redo05.log' TO GROUP 5;
  
  
 select * from v$logfile;
  
 ALTER SYSTEM SWITCH LOGFILE;
  
 select * from v$logfile;
 select * from v$log

```

아카이브 모드 : 로그 스위치 발생 → 다른 그룹에 copy 해놓겠다. → copy 가 다 끝내야지만 inactive → 파일 크기 너무 크면 copy 하는 거 오래걸리니까  

노아카이브 모드 : 깨진 시점까지 복구 못해 → 백업 시점까지밖에 복구 못함. → 데이터 손실 발생

```bash
-- 그룹 추가
ALTER DATABASE ADD LOGFILE GROUP 6 ('/u01/app/oracle/oradata/ORA19C/redo06.log', '/u01/app/oracle/fast_recovery_area/ORA19C/onlinelog/redo06.log') SIZE 100M;

select * from v$logfile;
select * from v$log;
```

▶️ redo log group 삭제

```bash
ALTER DATABASE DROP LOGFILE GROUP 6;   

-- 꼭 물리적 위치에서도 제거해줘야 함.
rm /u01/app/oracle/oradata/ORA19C/redo06.log
rm /u01/app/oracle/fast_recovery_area/ORA19C/onlinelog/redo06.log
```

▶️ redo log group 삭제 시 제한 사항

- redo log group 이 최소 두 개 필요하다.
- current, active 상태 그룹에 대해서는 삭제를 할 수 없다.
- 그룹을 삭제한 후 실제 파일은 꼭 OS에서 파일 삭제해야 한다.

▶️ redo log group member 삭제

```bash
ALTER DATABASE DROP LOGFILE MEMBER '/u01/app/oracle/fast_recovery_area/ORA19C/onlinelog/redo05.log';
```

▶️ redo log group member 삭제시 제한사항

- member를 삭제한 후 실제 파일은 꼭 OS에서 파일 삭제해야 한다.
- current, active 상태 member에 대해서는 삭제를 할 수 없다.
- 삭제하려는 member가 마지막 유효한 group member 인 경우는 그 member를 삭제할 수 없다.

```bash
-- 그룹으로 삭제하기
ALTER DATABASE DROP LOGFILE GROUP 5;

select * from v$log;
select * from v$logfile;
```