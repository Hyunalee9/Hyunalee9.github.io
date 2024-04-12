---
title: "Small Steps (작은 발자국) 프로토타입2"
excerpt: "보다 더 나은 프로토타입 제작"
categories:
    - smallstep
tags :
    - smallstep
    - Django
    - TIL
last-modified-at: 2021-07-19 T20:53
---

###  Django 데이터베이스 MySQL로 교체

**장고와 데이터베이스 연결하기**

[참조블로그](https://velog.io/@devmin/Django-MySQL-Connect)

```python
python -m pip install mysqlclient
```

```python
# MySQL 데이터베이스로 교체 후 코드

# my_settings.py
# MySQL 데이터베이스 세팅 (나중에 이 파일을 .gitignore에 넣을 것)

DATABASES = {
    'default' : {
        'ENGINE' : 'django.db.backends.mysql',  # 사용할 엔진 설정
        'NAME' : '',  # 연동할 MySQL의 데이터베이스 이름  
        'USER' : 'root',  # DB 접속 계정명
        'PASSWORD' : '비밀',  # 해당 DB 접속 계정 비밀번호
        'HOST' : '127.0.0.1',  # 실제 DB 주소
        'PROT' : '3306',  # 포트번호
    }
}

# settings.py
from diary import my_settings

DATABASES = my_settings.DATABASES
```

### 20210719 추가 - 새로운 프로토타입

__Intro__

![1](/assets/Prototype2/프로토타입.jpg)

생각해보니 little step이 아니라 small steps 였는데.... 다음에 수정해야겠다.

__Calendar__

![2](/assets/Prototype2/7월달력.jpg)

__Emojies__

![3](/assets/Prototype2/이모지들.jpg)

__Write__

![4](/assets/Prototype2/일기화면.jpg)

__After__

![5](/assets/Prototype2/일기쓴후.jpg)
