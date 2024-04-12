---
title: "다형성"
excerpt: "다형성에 대해."
categories :
   - JAVA
tags :
   - TIL       
   - JAVA
last_modified_at : 2021-06-09 T23:03
---

__Polymorphism__

- 다형성

  * 다양한 형태로 변할 수 있는 것
  * 한 타입의 참조 변수로 여러 타입의 객체를 참조할 수 있음
  * 동적 바인딩이 지원되어야 한다.
  * 자바는 다형성을 위해 부모 클래스로의 타입변환을 허용
    (부모 타입에 모든 자식 객체가 대입 될 수 있다.)
  * 객체의 부품화 가능

  ```java
//Avengers

  class Hero{ //부모 클래스
  	public String name;
  	public void attack() {
  		System.out.println("공격");
  	}
  }

  class Ironman extends Hero{
  	public String spot = "하늘";
  	int suitCount;

  	public void makeSuit() {
  		System.out.println("Javis, 슈트 만들어 줘");
  	}

  	public void attack() {
  		System.out.println("javis, 공격");
  	}
  	}

  class Hulk extends Hero{
  	public String spot = "땅";

  }

  class Spiderman extends Hero{
  	public String spot = "건물 사이";


  	public void attack() {
  		System.out.println("거미줄공격!");
  	}
  }
  class Thor extends Hero{

  	public void attack() {
  		System.out.println("망치공격");
  	}
  	}    
```

```java

//형 변환

public class Polymorphism {
	public static void main(String[] args) {
//		Hero h1 = new Ironman();

		Hero h1;//컴파일 타임
		h1 = new Ironman(); //런타임
		//반드시 상위 클래스가 앞으로 와야 한다.

//		Ironman i = new Hero(); 안된다

		h1.attack(); //javis, 공격

		//h1.makeSuit(); 부를 수 없다.
		// 그럼 자식 클래스에서 정의한 멤버나 메소드는?
		//숨김 = 변경시켜서 이용

		//방법1 형변환
		((Ironman) h1).makeSuit(); //Javis, 슈트 만들어 줘
		((Ironman) h1).suitCount = 10;


		//방법 2
		int a = 10;
		byte ba = (byte)a;

		Ironman i2 = (Ironman)h1;
		i2.makeSuit();

		//Hulk
		//Spiderman     => 하나의 배열에 넣고 싶다면?
		//Thor    

		int[] intArray = new int[4];

	    Hero[] heroArray = new Hero[4];

	    heroArray[0] = new Ironman();
	    heroArray[1] = new Hulk();
	    heroArray[2] = new Spiderman();
	    heroArray[3] = new Thor();

	    for (Hero hero : heroArray) { //단체로 공격
			hero.attack();
		}

	}
}
```

__이런식으로 응용해서 사용할 수도 있다.__

```java

package polymorphism;

public class Polymorphism02 {
	public static Hero callHero(String spot) {
		if(spot.equals("하늘")) {
			return new Ironman();
		}else if(spot.equals("땅")){
			return new Hulk();
		}else if(spot.equals("바다")){
			return new Spiderman();
		}else {
			return new Thor();
		}		
	}

	public static void main(String[] args) {
			Hero h1 = callHero("바다");
			Hero h2 = callHero("땅");

			h1.attack();    //거미줄공격!
			h2.attack();    //주먹
	}
}

```  
