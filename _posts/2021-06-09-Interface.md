---
title: "인터페이스?"
excerpt: "자바에서 중요한 인터페이스 활용."
categories :
   - JAVA
tags :
   - TIL       
   - JAVA
last_modified_at : 2021-06-09 T23:17
---

__Interface__

- 인터페이스

   * 다른 언어에서 찾기 힘든 자바의 고급 기능
   * 단일 상속의 한계 극복
   * 오로지 추상 메소드와 상수만 가질 수 있다.
   * 인터페이스 내에 존재하는 메소드는 무조건 public abstract로 선언 된다.
   * 인터페이스 내에 존재하는 변수는 무조건 public static
   final 로 선언 된다. (상수)
   * 다중상속과 비슷한 기능을 제공
   * 외부의 것들을 서로 이어주는 통로 역할

  __인터페이스 생성하기__

  ```java

  interface SayHi{
  	//정적 변수
      public static final int NUMBER = 0; //상수
  	public void sayHi();
  	public void sayBye();
  }

  ```

  __인터페이스 사용하기__

```java

class Say implements SayHi{
	@Override
	public void sayHi() {//인터페이스에 있는 미구현 메소드 구현하기
	}

	@Override
	public void sayBye() {
	}

}
  }
```

  
