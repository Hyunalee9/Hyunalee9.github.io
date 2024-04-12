---
title: "Inner Class"
excerpt : "Static Class, Member Class, Local Class, Anonymous Class "
categories:
      - JAVA
tags:
      - JAVA
      - TIL  
last_modified_at: 2021-06-22 T17:04
---
### **Inner Class**

- 클래스 내부에 선언된 클래스
- 외 내부 클래스가 서로 긴밀한 관계를 맺고 있다.
- 외부 클래스에 비해 상대적으로 잘 사용되지 않는다.

```java
class A {

   class B { //외부에서 클래스 B를 절대 볼 수 없다.

    }

}
```

### **장점**

- 내부 클래스에서 외부 클래스의 멤버들을 쉽게 접근 할 수 있다.
- 캡슐화, 코드의 복잡성을 줄여준다.

### **종류**

- Static Class

```java
- 외부 클래스의 멤버 변수 선언 위치에서 선언한다.
- static 멤버처럼 다뤄진다.
- 주로 외부 클래스의 static 멤버, static 메소드에서 사용될 목적으로 선언
```

- Member Class

```java
- 외부 클래스의 멤버 변수 선언 위치에서 선언한다.
- 외부 클래스의 인스턴스 멤버처럼 사용
- 주로 외부 클래스의 인스턴스 멤버들과 관련된 작업에서 사용
```

- Local Class

```java
- 외부 클래스의 메소드나 초기화 블럭 안에서 선언
- 선언된 영역 내부에서만 사용
```

- Anonymous Class

```java
클래스 선언과 객체의 생성을 동시에 하는 이름 없는 클래스 (일회용)
```

### **Static Class**

- 내부 클래스안에 static이 붙은 클래스
- static때문에 객체 생성없이 사용 가능하다.
- 클래스의 static 변수처럼 사용된다.
- 외 내부 클래스가 서로 다르게 동작한다.
- 외 내부 클래스의 멤버가 private라도 상호 접근 가능하다.
- 경로만 지정된다면 단독으로 직접 사용할 수 있다.

```java
class AAA{
	static class BBB{

	}
}
```

```java
package innerClass;

public class StaticClass01 {
	private int num = 1; // 객체 생성해만 사용 가능 , 외부 X  , (외부 클래스 인스턴스 변수)
	int sc = num; // 객체 생성해야만 사용 가능 , 외부 O
	private static int outterSI = 0; // 클래스명. 으로 호출 , 객체X, 외부 X

	public static void outterMethod() { //외부 클래스 메소드
		System.out.println(SInner.innerSM);
                 // 외부 클래스 메소드에서 내부 클래스 멤버 변수 호출 가능
	}

	public static class SInner{ // 내부 클래스
		private int innerMember = 200;
		private static int innerSM = 300;
		final int LV = 100;

		//내부 클래스 안에서 메소드 정의하기
		public static void innerMethod() { // 내부 클래스의 메소드
			System.out.println("내부 클래스 안의 static 메소드");  
		}

		public void innerM() {
			System.out.println("내부 클래스 안의 static 메소드2");
			System.out.println("this.innerSM:" + this.innerSM); //private라도 접근 가능하다.
			System.out.println("outterSI :" + outterSI); //private라도 접근 가능하다.
		}
	}

	public static void main(String[] args) {
		StaticClass01 staticClass01 = new StaticClass01();

		StaticClass01.SInner si = new SInner(); // ->호출할 때 이렇게 해야한다.
		StaticClass01.SInner si2 = new StaticClass01.SInner();

		staticClass01.outterMethod(); //300

		si.innerMethod(); //이렇게 호출 가능

		si.innerM();

		StaticClass01.SInner.innerMethod();

		//		System.out.println(staticClass01.SInner);  이건 안됨  

	}
}
```

![1](/assets/20210622/1.PNG)

### **Member Class**

- 멤버 클래스는 static 클래스와 같은 위치에서 선언되며 비슷한 성질을 가진다.
- static 선언이 없는 내부 클래스
- 외부 클래스의 인스턴스 속성 변수처럼 활용됨 ( 메소드 {} 안에서만 유효함)
- 클래스 내에서 선언되지만 메소드 밖에서 , 생성자 밖에서, 다른 블록 밖에서 선언되는 경우 반드시 초기화가 필요
- 멤버 클래스는 외부 클래스의 멤버를 활용할 수 있지만, 외부 클래스는 멤버 클래스의 멤버 변수를 활용할 수 없다.
- static이 붙은 메소드 내에서는 멤버 클래스의 객체 선언 X

```java
package innerClass;

public class MemberClass01 {
		private int outerDf = 10;
		private static int osi = 55;
		int number = 777;

		void outerMethod() {
			System.out.println(number); //동적 메소드에서 동적 멤버 변수 호출 가능
			System.out.println(osi);  // 동적 메소드에서 정적 멤버 변수 호출 가능
			System.out.println(this.osi);
		}

		static void outterSM() {
			System.out.println(osi); //객체 생성 없이 호출! (정적 메소드에서 )
//			System.out.println(number); 정적 메소드에서 동적 멤버 변수 호출 불가
		}

		public class InnerClass{ //내부 클래스
			private int x = 100; // 내부 정적 멤버 변수
			int innerDf = 100;
			static final int ISI = 123; // 상수

			public void innerMethod() {
				int imnum = osi; // 외부 클래스 정적 멤버 변수 호출 가능
				System.out.println(x); //내부 클래스 private 멤버 변수 호출 가능
				System.out.println(innerDf);
				System.out.println(ISI);//상수 호출 가능
				System.out.println(number); // 외부 클래스 멤버 변수 호출 가능
				System.out.println(outerDf);

			}
		}

		public static void main(String[] args) {
			MemberClass01 memberClass01 = new MemberClass01();
//			MemberClass01.InnerClass in = new MemberClass01().InnerClass(); 이렇게 호출 못함

			MemberClass01.InnerClass in = memberClass01.new InnerClass();
			in.innerMethod(); //이렇게 호출

		}
}
```

### **Local Class**

- 메소드 안에 선언한 클래스
- 선언한 메소드 내에서만 사용하기 위해서 내부에 선언
- 메소드 안에서만 지역 변수처럼 클래스를 활용하므로 메소드의 실행이 끝나면 제거됨
- 외부에서 인스턴스를 생성할 수 없다.
- static을 사용할 수 없다.
- instance 변수 또는 메소드는 사용할 수 있다.
- final 붙은 지역 변수 (상수처리) 나 매개 변수는 지역 클래스의 메소드에서 접근 가능
- 객체를 생성해서 활용
- 컴파일 하면 외부 클래스$숫자+로컬 클래스명.class로 만들어짐
- 서로 다른 메소드일 때 같은 이름의 클래스가 존재할 수 있어서 구분하기 위해 숫자를 붙임.

```java
package innerClass;

public class LocalClass01 {
		private int a = 10;
		final int LV = 100;

		void method() {
			int in = 100;
			final int inD = 1000;

			class LocalClass{
				int no = 99;
				void msg() { //지역 변수 메소드
					no = no + 10;
					System.out.println("외부 a : " + a );
					System.out.println(LocalClass01.this.a);
					System.out.println(in);
					System.out.println(inD);
				}
			} // LocalClass 끝
			//지역 클래스가 선언된 메소드 안에서 객체를 만듭니다.

			LocalClass local = new LocalClass();
			local.msg(); // 호출

		}// 메소드 끝

		public static void main(String[] args) {
			LocalClass01 localClass01 = new LocalClass01();
			localClass01.method(); //메소드 호출
		}
}
```

![9](/assets/20210622/9.png)

### **Anonymous Class**

- 클래스 명이 없는 클래스
- 선언과 동시에 인스턴스 생성을 한다
- 클래스를 인수의 값으로 활용하는 클래스
- 객체를 한 번만 사용할 경우 사용함
- 클래스의 선언부가 없기 때문에 이름이 없으므로 생성자를 가질 수 없다.
- 둘 이상의 슈퍼 클래스의 상속이나 인터페이스를 구현하는 행위를  할 수 없다.
- 오직 하나의 클래스에서 상속 받거나 하나의 인터페이스를 구현
- 코드 블럭에 클래스 선언을 하는 것 제외하고는  생성자 호출과 방법이 동일
- 객체를 구성하는 new 문장 뒤에 클래스의 블럭 {}을 첨부하여 몸통을 닫는 형식
- new 슈퍼 클래스 또는 인터페이스명(){};
- 객체를 생성한 후에 {};
- new 뒤에 오는 생성자명이 기존 클래스명이면 익명클래스가 자동으로 클래스의 하위 클래스가 됨
- 인터페이스인 경우에는 인터페이스를 상속하는 부모 클래스가 Object가 됨

```java
package innerClass;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Iterator;
import java.util.List;
import java.util.ListIterator;

class AMClass{
	public void method() {
		System.out.println("method");
	}
}

public class Anonymous {
		public static void main(String[] args) {
//			List<String> list = new List<String>(); List는 인터페이스기때문에 이렇게 사용X
//			List<String> list = new List<String>() {}; // Anonymous Class
		 // 여기서 오버라이드 까지 해줘야 오류 안난다.
			AMClass amc = new AMClass() {
				@Override
				public void method() {
					System.out.println("오버라이드 했습니다."); //요게 나온다.
				}
			};

			amc.method();
 		}
}
```
