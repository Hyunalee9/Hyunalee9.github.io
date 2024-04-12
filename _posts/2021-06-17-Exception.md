---
title: "Exception"
excerpt: "예외처리에 대해 알아보자."
categories :
   - JAVA
tags :
   - TIL       
   - JAVA
last_modified_at : 2021-06-17 T21:09
---

### **Exception**

자바 예외 핸들링

- 목적에 따라서 처리 하도록 설계
- 주로 실행시에 발생되는 모든 에러 상황
- 던진다라고 표현
- 객체이기 때문에 클래스로 정의

### **자바에서의 예외 발생 순서**

컴파일 → 실행 → 실행중 예외 발생 → VM이 발생한 예외 종류 및 내용을 파악한 뒤 예외 객체 생성

→ 발생된 코드 밖으로 예외 던지기(Throw) → 예외의 콜 스택에 전이 → main 메소드 밖까지 던지게 되면 프로그램 비정상 종료

### **Error 하위 타입**

- 일반적으로 java 실행기 즉 VM에 관련된 에러 상황을 정의한 클래스
- 프로그래머가 처리할 수 없는 것들로 VM 즉 JRE 전반적인 문제
- 프로그래머는 Error 하위 타입들의 예외는 처리하지 않고 무시
- 컴퓨터 하드웨어의 오작동 또는 고장으로 인해 응용프로그램 실행오류가 발생하는 것

### **Exception 하위 타입**

- 프로그래머가 처리해야 할 예외 타입
- Throwable 클래스는 자식 클래스로 Error를 가지고 있기 때문에 예외의 최고 클래스로 표현하지 않는다.
- 사용자의 잘못된 조작 또는 개발자의 잘못된 코딩으로 인해 발생하는 프로그램 오류
- 예외가 발생하면 프로그램은 바로 종료
- 예외처리를 통해 프로그램을 종료하지 않고 정상 실행 상태를 유지

### **알려지지 않는 예외 (Unchecked Exception]**

컴파일러가 관여하지 않으면서 실행시에 예외가 발생할수도 발생하지 않을 수도 있는 예외

### **알려진 예외 [Checked Exception]**

컴파일러가 관여하는 예외

### **구분하는 방법**

알려지지 않은 예외: Exception 하위 클래스 중 RuntimeException의 자식 클래스

알려진 예외: 그외 나머지

### **예외처리 방법**

- 직접 처리 (try ~ catch)

```java
try{
	 예외가 발생할만한 코드;
} catch(Exception e) {
	 예외가 발생하면 실행할 코드;
} finally{
	 예외 발생유무와 상관없이 반드시 실행해야 할 코드;  // ex) 닫기();
}
```

- 던지기(throw)

메소드 뒷부분에 throw 처리할 예외타입을 적어 준다.

메소드가 실행되다가 예외를 만나면 메소드를 호출한 쪽으로 예외를 던진다.

main메소드는 VM으로 던진다.

```java
//메소드 뒷부분

```

- 직접 예외 객체 만들어서 처리하기

```java
throw 객체명;
		@oveeride
		public void printStrackTrace(){
				super.printstrackTrace();
				system.out.println("계산 불가");
   }

```

### **예외의 예시**

인덱스 넘어 가버렸을 때

![result05](/assets/result05.png)

try ~ catch 문을 사용해서 프로그램을 정상적으로  끝까지 실행할 수 있다.

```java
package june16;

public class Exception01 {
	public static void main(String[] args) {
		int[] ia = new int[5];
		try {
			for (int i = 0; i < 6; i++) {
				System.out.println(ia[i]);
			}
		} catch (Exception e) {
			System.out.println("예외 처리 완료");
		}

		System.out.println("정상적으로 종료");
	}
}
```

![result04](/assets/result04.png)
이상한 산술계산

![result04-2](/assets/result04-2.png)

```java
package june16;

public class Exception02 {

	public static void main(String[] args) {
		try {
			System.out.println(10 / 0);

			System.out.println("try문장 속"); // 화면에 안뜸
		} catch (Exception e) {
			System.out.println("0으로 나누려고 시도 중 예외 발생");
			//예외발생할 때만 실행
		}

		System.out.println("프로그램이 종료됩니다.");
	}

}
```

![result03](/assets/result03.png)

```java
package june16;

public class Exception02 {

	public static void main(String[] args) {
		try {

			int[] ia = { 10, 20, 30, 40, 50 };
			System.out.println(ia[5]);

			System.out.println(10 / 0);

			System.out.println("try문장 속"); // 화면에 안뜸
		} catch (ArithmeticException e) {
			System.out.println("산술계산 문제 발생할 때만.");
		} catch (ArrayIndexOutOfBoundsException e) {
			System.out.println("인덱스 넘어갈 때만");
		} finally {
			System.out.println("여기는 예외 발생 상관없이 실행할 문장");
		}

		System.out.println("프로그램이 종료됩니다.");
	}

}
```

![result02](/assets/result02.png)

파일 없음 오류

```java
package june16;

import java.io.FileNotFoundException;
import java.io.FileReader;
import java.util.Scanner;

public class Exception04 {

	public static void main(String[] args) {
		// 파일 없음 오류
		Scanner sc = new Scanner(System.in);
		System.out.println("c:\set.log라고 입력하세요");

		String fileName = sc.next();

		try {
			FileReader fr = new FileReader(fileName);
			System.out.println("파일있습니다.");
		} catch (FileNotFoundException e) {
			System.out.println("파일 없습니다.");
		} finally {
			sc.close();
		}

	}

}
```

```java
package june16;

import java.io.FileNotFoundException;
import java.io.FileReader;
import java.io.IOException;

public class Exception05 {

	public static void main(String[] args) {
		FileReader fr = null;
		try {
			fr = new FileReader("c:/temp/2021-06-02-second.md");
		} catch (FileNotFoundException e) {
			System.out.println("파일을 찾을 수 없습니다.");
		} finally {
			try {
				fr.close();
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
		System.out.println("끝");
	}

}
```

![result01](/assets/result01.png)
