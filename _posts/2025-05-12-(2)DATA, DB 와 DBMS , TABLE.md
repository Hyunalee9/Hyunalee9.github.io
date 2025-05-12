---
title: " DATA, DB 와 DBMS , TABLE"
excerpt: ""
categories:
      - ORACLE11g
tags:
      - ORACLE11g
      - TIL
last-modified-at: 2025-05-12T20:52
---

# DATA, DB 와 DBMS , TABLE

## Data

- 관찰이나 측정을 통해서 수집된 사실(facts)이나 값(values)
- 컴퓨터가 처리할 수 있는 문자, 숫자, 소리, 그림의 형태로 된 자료

## DataBase

- 데이터 집합
- 여러 사람이나 프로그램이 데이터를 쉽게 공유하기 위해 데이터의 조직화된 모음으로

관리되는 데이터의 집합

## DBMS(DataBase Management System)

- 데이터베이스 소프트웨어를 의미한다.
- oracle, ms-sql, my-sql, mariadb, postgresql,sqlite

## RDBMS(Relational DBMS)

- 표 형식의 모델 (2차원)
- 2차원 테이블 형태로 표현한 뒤 각 테이블 간의 관계를 정의
- 행(ROW)와 열(COLUMN)으로 구성되어 있다.

**📍DB-Engines Ranking**

[https://db-engines.com/en/ranking](https://db-engines.com/en/ranking)

## 테이블(segment)

행(row)과 열(column)로 구성되어 있는 데이터의 저장 구조

**상단에 붙어있는 탭 설명**

![image.png](/assets/20250512/5.png)

- 열 : 컬럼들 이름, 타입, 결측값, 아이디, 주석등 정보
- Model : ERD
- 제약조건: 제약조건 정보 들어있는 탭
- SQL : 테이블 생성시 입력한 syntax

## 테이블 구조 확인

```sql
desc hr.employees;
```

![image.png](/assets/20250512/6.png)