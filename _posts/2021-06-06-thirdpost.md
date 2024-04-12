---
title: "#TIL 20210606 오늘 한 일 "
excerpt: "JAVA String다루기 연습, 블로그 사이드바 만들기, 명세서 프로토타입 만들기 "
categories:
   - TIL
tags:
   - TIL
   - JAVA
   - Blog   
last_modified_at : 2021-06-06 T21:18
---

- 시저 암호 변형 문제 풀어보기
(입력받은 영문자열을  홀수는 +2, +3,... 짝수는 -2, -3,...  순으로 변형하기)

```java
package june04;

import java.util.Scanner;

public class Test02 {
	public static void main(String[] args) {

		Scanner sc = new Scanner(System.in);
		Solution mini = new Solution();
		System.out.println("암호화할 문자열을 입력해주세요.");
		String input = sc.next();
		System.out.println(mini.solution(input));

	}
}

class Solution {
	int number = 2;
	int number2 = 2;
	public String solution(String input) {
		String answer = "";

		for (int i = 0; i < input.length(); i++) {
			char ch = input.charAt(i);

			if (i % 2 == 0) {
				ch = (char) ((ch - 'a' + number) % 26 + 'a');  // 홀수일때
				number++;
			} else {
				ch = (char)((ch - 'a' - number2) * (-25)   + 'a'); //짝수일때
				number2++;
			}
			answer += ch;
		}

		return answer;
	}
}
```

- 블로그 사이드바 만들기 + tag /연도별로 나눈 topbar

![topbar](/assets/topbar.PNG) 
---
![sidebar](/assets/sidebar.PNG)

- 포스트 밑에 업데이트 날짜 넣기.

![date](/assets/date.PNG)

- 작은 발자국 명세서 + 프로토타입 만들기
