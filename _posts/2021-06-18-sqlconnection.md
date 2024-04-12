---
title: "sql과 이클립스 연결하기"
excerpt : "떡잎방범대 로그인 구현 "
categories:
      - MariaDB
tags:
      - MariaDB
      - TIL  
last_modified_at: 2021-06-18 T16:52
---

__HeidlSQL에서 login 테이블 만들고 떡잎방범대 데이터 입력하기__

![떡잎방범 데이터](/assets/20210618/login.PNG)

위와 같이 만들었다.

__이클립스와 연결하기__

```java
package login;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.Scanner;

class DBConnection {
	public static Connection getConn() { // static 때문에 객체 없이 불러올 수 있음
		Connection conn = null;

		try {
			Class.forName("org.mariadb.jdbc.Driver");
			String url = "jdbc:mariadb://ip:port/DB명";
      //보안을 위해 url과 아이디, 비밀번호는 지웠다.
			conn = DriverManager.getConnection(url, "아이디", "비밀번호");

		} catch (Exception e) {
			e.printStackTrace();
		}

		return conn;
	}
}

public class Login {

	public static void main(String[] args) {
		Scanner sc = new Scanner(System.in);
		Statement stmt = null;
		Connection conn = null;
		ResultSet rs = null;
		String id, pw, query, name;

		System.out.println("아이디를 입력하세요.");
		id = sc.next();

		System.out.println("비밀번호를 입력하세요.");
		pw = sc.next();

		// 입력 끝, 데이터베이스로 가기 = conn 정보 넣기
		conn = DBConnection.getConn();
		query = "SELECT name FROM login WHERE id='" + id + "' AND pw= '" + pw + "'";

		try {
			stmt = conn.createStatement(); // 접속정보를 stmt에 넣기
			rs = stmt.executeQuery(query);
			// 결과 뽑기 => id, pw가 일치하는 것은 하나이므로
			if (rs.next()) {
				System.out.println("데이터가 있습니다.");
				name = rs.getString("name");
				System.out.println(name + "님 반갑습니다.");
			} else {
				System.out.println("데이터가 없습니다.");
			}

		} catch (SQLException e) {
			e.printStackTrace();
		} finally {
			try {
				rs.close();
				stmt.close();
				conn.close();

			} catch (SQLException e) {
				e.printStackTrace();
			}

		}
	}

}
```

__결과__

![짱구로그인](/assets/20210618/짱구.png)
