---
title: "가입페이지 만들기, JSP,servlet 한글 인코딩 공부"
excerpt : "서블릿, 디스패처, JSP"
categories:
      - JSP
      - JAVA
      - HTML
      - TIL
tags:
      - JAVA
     
last_modified_at: 2021-07-08 T17:09
---

__오늘 한 일__

1. 회원이 로그인하면, Home 메뉴에 로그인 창 안보임 (대신 게시판이 보임)
2. 가입 페이지를 만들었다.
3. 가입 페이지에서 JSP를 사용하여 패스워드 일치 검사 후, 화면에 띄워주는 기능
4. 가입 페이지의 각 값들이 모두 들어와야하고 일정 길이 이상이여야 값을 전송하도록 설정
5. 가입 페이지에서 사용자가 입력한 값을 UTF-8로 인코딩하여 데이터베이스에 저장하는 방법 공부


### **가입하기 페이지 만들기**

```jsx
//join.jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>가입하기</title>
<style type="text/css">
@font-face {
    font-family: 'RIDIBatang';
    src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/noonfonts_twelve@1.0/RIDIBatang.woff') format('woff');
    font-weight: normal;
    font-style: normal;
}
@import url('https://fonts.googleapis.com/css2?family=Black+And+White+Picture&family=Roboto&family=Tourney:ital,wght@1,900&display=swap');
body{
font-family:"RIDIBatang";

}
input{
margin-top:5px;
width:50%;
}
button{
font-family:"RIDIBatang";
}
h1{
 font-family: 'RIDIBatang';
 color:black;
 text-shadow: 1px 1px 5px #B0ADAA;

}

table {
	font-family: 'RIDIBatang';
	width: 65%;
	height: 500px;
	/* background-color: #c0c0c0; */
	border-collapse: collapse;
	border: 1px solid;
	margin-left: 20%;
}
table td{
    font-family: 'RIDIBatang';
	text-align: center;
	border: 1px solid;

}
table th{
	background: #EDE9C1;
	font-weight: bold;
	border: 1px solid;
}
#join{
border: 1px dashed #EDE9C1;
width: 68%;
height:600px;
}
</style>
<script type="text/javascript">
function join(){
	var id = document.getElementById("id");
	var name = document.getElementById("name");
	var pw1 = document.getElementById("pw1");
	var email = document.getElementById("email");
//	alert(id.value + " : " + name.value + " : " + email.value + " : " + pw1.value);
	if(id.value.length < 3){
		alert("아이디는 3글자 이상이여야 합니다.")
		id.focus();
		return false;
	}
	if(name.value.length < 2 ){
		alert("이름은 2글자 이상이여야 합니다.")
		name.focus();
		return false;
	}
	if(pw1.value.length < 4){
		alert("비밀번호는 4글자 이상이여야 합니다.");
		pw1.focus();
		return false;
	}
}

function isSame() {
    if(document.getElementById('pw1').value!='' && document.getElementById('pw2').value!='') {
        if(document.getElementById('pw1').value==document.getElementById('pw2').value) {
            document.getElementById('same').innerHTML='비밀번호가 일치합니다.';
            document.getElementById('same').style.color='black';
        }
        else {
            document.getElementById('same').innerHTML='비밀번호가 일치하지 않습니다.';
            document.getElementById('same').style.color='black';
            document.getElementById('pw2').style.color = '#8F7CEE';
            document.getElementById('pw2').focus();
        }
    }
}
</script>
</head>
<body>
	<%@ include file="./menu.jsp"%>

<form action="./join" method="post" onsubmit="return join()">
	<h1>가입하기</h1>
	<div id="join">
	<table>
		<tr>
			<th>아이디</th>
			<td><input type="text" name = "id"  id = "id" required ></td>		
		</tr>
		<tr>
			<th>이름</th>
			<td><input type="text" name = "name"  id = "name" required ></td>		
		</tr>
		<tr>
			<th>암호</th>
			<td>
			<p><input type="password" name = "pw1"  id = "pw1" required></p>
			<p><input type="password" name = "pw2"  id = "pw2" required onchange="isSame()"></p>
			<span id="same"></span>		
			</td>		
		</tr>
		<tr>
			<th>email</th>
			<td><input type="email" name = "email"  id = "email"  required></td>		
		</tr>
		   <td colspan="2">
			<button type="reset">지우기</button>
		    <button type="submit">가입하기</button>
		   </td>
	</table>
	</div>
</form>

</body>
</html>
```

```java
//Join.java
package com.hyunalee.web;

import java.io.IOException;
import java.io.PrintWriter;

import javax.servlet.RequestDispatcher;
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import com.hyunalee.dao.LoginDAO;
import com.hyunalee.dto.LoginDTO;

@WebServlet("/join")
public class Join extends HttpServlet {
	private static final long serialVersionUID = 1L;


    public Join() {
        super();

    }


	protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {

	}


	protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		//한글 처리
		response.setCharacterEncoding("UTF-8");
		request.setCharacterEncoding("UTF-8");
		response.setContentType("text/html;charset=utf-8");

		if (request.getParameter("id") != null && request.getParameter("name") != null
				&& request.getParameter("pw1") != null && request.getParameter("pw2") != null
				&& request.getParameter("email") != null
				&& request.getParameter("pw1").equals(request.getParameter("pw2"))) {

			String id = request.getParameter("id");
			String name = request.getParameter("name");
			String pw1 = request.getParameter("pw1");
			String pw2 = request.getParameter("pw2");
			String email = request.getParameter("email");

			// dto로 담아주기
			LoginDTO dto = new LoginDTO();

			dto.setId(id);
			dto.setName(name);
			dto.setPw(pw1);
			dto.setEmail(email);

			// DB작업하기
			LoginDAO dao = new LoginDAO();
			int count = dao.join(dto);
//			System.out.println(count + "개 완료");

			RequestDispatcher rd  = request.getRequestDispatcher("./joinResult.jsp");
			request.setAttribute("count", count);
			request.setAttribute("id", id); //실제 사용자가 저장한 id값을 가지고 날라감

			rd.forward(request, response);

		} else {
			response.sendRedirect("error.jsp?error=5506");
		}

	}

}
```

```html
//joinResult.jsp

<%@page import="com.hyunalee.web.Util"%>
<%@page import="com.hyunalee.dto.BoardDTO"%>
<%@page import="java.util.ArrayList"%>
<%@page import="com.hyunalee.dao.BoardDAO"%>
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>가입 결과</title>
<style type="text/css">
@import url('https://fonts.googleapis.com/css2?family=Black+And+White+Picture&family=Roboto&family=Tourney:ital,wght@1,900&display=swap');
@font-face {
    font-family: 'RIDIBatang';
    src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/noonfonts_twelve@1.0/RIDIBatang.woff') format('woff');
    font-weight: normal;
    font-style: normal;
}
h2, h3{
 font-family:'RIDIBatang';
}

</style>
</head>
<body>
	<%@ include file="./menu.jsp"%>
	<h1>가입 결과</h1>
<%
if(request.getAttribute("count").equals(1)){
%>
	<h2><%=request.getAttribute("id") %>님, 가입이 완료되었습니다.</h2>
	<h3>로그인 해주세요.</h3>
	<button onclick = "location.href='./index.jsp'">로그인하기</button>
<% 	
} else{
%>
	<h2><%=request.getAttribute("id") %>님, 문제가 발생했습니다.</h2>
	<h3>가입을 완료하지 못했습니다. 다시 시도해주세요.</h3>

<%
}
%>

</body>
</html>
```

__결과물__

![1](/assets/20210708/1.png)
![2](/assets/20210708/2.png)
![3](/assets/20210708/3.png)
![4](/assets/20210708/4.png)


__UTF-8 인코딩__

### **form에 입력한 값을 전송할 때, 깨지지 않게 하기 위해서는(GET/POST 둘 다)**

>서블릿 규약에 따르면 GET은 request.setCharacterEncoding()이 적용안되지만
>server.xml파일의 <Connector> 태그의 useBodyEncodingForURI= 부분을 true로 바꿔주면 적용된다.

```jsx
request.setCharacterEncoding("UTF-8");
```

### **서블릿에서 한글 깨지지 않기 위해서는 (콘솔, 브라우저)**

```jsx
response.setCharacterEncoding("UTF-8");
response.setContentType("text/html;charset=utf-8");
```
