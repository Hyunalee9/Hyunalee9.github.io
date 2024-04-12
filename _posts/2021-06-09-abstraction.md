---
title: "자바 - 추상화"
excerpt: "추상 클래스와 추상 메소드"
categories :
   - JAVA
tags :
   - TIL       
   - JAVA
last_modified_at : 2021-06-09 T17:23
---

__Abstraction Class__

- 추상 클래스

  * 현실화 되어질 필요가 없는 클래스
  * 인스턴스를 생성할 수 없다.
  * 생성자, 메소드, 필드 모두 선언/정의 가능하다.
  * 상속과 다형성도 적용 가능하다. (super type으로 전개가- )

__선언 규칙__

- 특정 클래스에 정의된 메소드중 추상 메소드가 하나라도
있다면 해당 클래스는 추상 클래스가 된다.

- 추상 메소드가 없더라도, abstact라는 키워드를 class 앞에
붙여 추상화 할 수 있다.

- 추상 메소드는 메소드 바디가 없는 형태이다.

- 추상 메소드는 abstact 라는 키워드를 리턴 타입 앞에 선언한다.


```java
abstract class Animal{//부모 클래스 / 슈퍼 클래스 / 상위 클래스
	String name;
	int age;

	public abstract void sleep(); //하위 클래스에서 무조건 재정의하도록 강제.

	public abstract void eat(); //인터페이스 ★
}

class Cat extends Animal{

	@Override  //상속관계에서만 가능하고 부모에게 상속받은 메소드를 재정의
	public void sleep() {
	}//자식 클래스

	@Override
	public void eat() {
	}

}

class Dog extends Animal{

	@Override
	public void sleep() {
	}//자식 클래스

	@Override
	public void eat() {
	}

}

public class Abstraction {
	public static void main(String[] args) {
		Cat cat = new Cat();
		Dog dog = new Dog();
		//추상화 하였기 때문에 Animal animal = new Animal(); 로 Animal의 객체 생성 불가

	}
}

```
