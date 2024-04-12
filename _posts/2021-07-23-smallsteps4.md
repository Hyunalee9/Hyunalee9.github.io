---
title: "작은 발자국 20210723 오늘 한 일"
excerpt: "프론트엔드에서 데이터베이스로 데이터 보내기"
categories:
      - smallstep
tags:
      - smallstep
      - Vue.js
      - RESTFramework
      - TIL
last-modified-at: 2021-07-23T19:35
---

### 20210723 글쓰기 화면 구현, 프론트엔드에서 데이터보내기 성공

- async, await를 이용해 form으로 입력받은 데이터를 데이터베이스에 저장했다.

- home model의 title 필드를 삭제했다.

- pub_date 필드를 DateTime ->Date 로 바꿔준 후, 하루에 하나씩만 일기를
 작성 할 수 있게 primary key로 지정했다.

 - WritePage (글쓰기 화면) 을 완성했다.

 - 해결해야할 문제: 글을 작성한 뒤 화면 전환 , 캘린더 컴포넌트 만들기

 __결과__

 - 글쓰기 페이지

 ![WritePage](/assets/20210723/20210723WritePage.PNG)


 - 날짜를 primary key로 설정 해주기 전, 글 보내보기

![20210723글쓰기부분](/assets/20210723/20210723글쓰기부분.PNG)


- 데이터베이스에 잘 저장된다.

 ![저장완료](/assets/20210723/20210723데이터받음.PNG)

- 날짜를 primary key로 만들어 하루에 하나만 작성할 수 있도록 한 후 (기존의 데이터들은 삭제했다.)

![하루에 하나만](/assets/20210723/20210723날짜를기본키로.PNG)           
