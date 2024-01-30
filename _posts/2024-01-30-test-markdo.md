---
layout: post
title: Spring - 서블릿의 한계와 MVC
tags: [Spring, 스프링 MVC]
comments: true
---

## 서블릿의 한계

```java
@WebServlet(name="memberFormServlet", urlPatterns = "/servlet/members/new-form")
public class MemberFormServlet extends HttpServlet{
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse res) throws IOException{
        res.setContentType("text/html");
        res.setCharacterEncoding("utf-8");

        PrintWriter w = res.getWriter();
        w.write("<!DOCTYPE html>\n" +
                "<html>\n" +
                "<head>\n" +
                "    <meta charset=\"UTF-8\">\n" +
                "    <title>Title</title>\n" +
                "</head>\n" +
                "<body>\n" +
                "<form action=\"/servlet/members/save\" method=\"post\">\n" +
                "    username: <input type=\"text\" name=\"username\" />\n" +
                "    age:      <input type=\"text\" name=\"age\" />\n" +
                "    <button type=\"submit\">전송</button>\n" +
                "</form>\n" +
                "</body>\n" +
                "</html>\n");
    }
}
```

```java
@WebServlet(name="memberSaveServlet", urlPatterns = "/servlet/members/save")
public class MemberSaveServlet extends HttpServlet{
    private MemberRepository memberRepository = MemberRepository.getInstance();
    
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse res) throws IOException{
    	String name = req.getParameter("username");
        int age = Integer.parseInt(req.getParameter("age"));
        
        Member member = new Member(name, age);
        memberRepository.save(member);
        
        res.setContentType("text/html");
        res.setCharacterEncoding("utf-8");
        PrintWriter w = res.getWriter();
        w.write("<html>\n" +
                "<head>\n" +
                "    <meta charset=\"UTF-8\">\n" +
                "</head>\n" +
                "<body>\n" +
                "성공\n" +
                "<ul>\n" +
                "    <li>id="+member.getId()+"</li>\n" +
                "    <li>username="+member.getName()+"</li>\n" +
                "    <li>age="+member.getAge()+"</li>\n" +
                "</ul>\n" +
                "<a href=\"/index.html\">메인</a>\n" +
                "</body>\n" +
                "</html>");
    }
}
```

보기만해도 멀미가 난다... 서블릿 덕분에 동적으로 움직이는 HTML을 만들 수 있게 되었지만 코드에서 보이듯이 매우 비효율적이다. 자바코드에서 HTML을 입력하기란 너무 빡세다. HTML부분부분에 자바 코드를 넣어서 동적으로 움직이게 하면 더 편할 것이다. 이런 해결책이 JSP같은 템플릿 엔진이다.

## JSP로 변환

```java

 <%@ page import="hello.servlet.domain.member.MemberRepository" %>
 <%@ page import="hello.servlet.domain.member.Member" %>
 <%@ page contentType="text/html;charset=UTF-8" language="java" %>
 <%

MemberRepository memberRepository = MemberRepository.getInstance();
System.out.println("save.jsp");
String username = request.getParameter("username");
int age = Integer.parseInt(request.getParameter("age"));
Member member = new Member(username, age);
System.out.println("member = " + member);
memberRepository.save(member);
%>
<html>
<head>
    <meta charset="UTF-8">
</head>
<body> 성공
  <ul>
     <li>id=<%=member.getId()%></li>
     <li>username=<%=member.getUsername()%></li>
     <li>age=<%=member.getAge()%></li>
</ul>
<a href="/index.html">메인</a>
</body>
</html>
```

위 코드를 보면 HTML 중간중간에 `<li>id=<%=member.getId()%></li>` 와 같은 코드가 들어가있어서 동적으로 HTML을 조작할 수 있다. 하지만 여전히 다른 문제가 남아있다. 한 파일안에 JAVA코드, 리포지토리, 뷰등 다양한 코드가 JSP안에 들어있다. JSP가 너무 많은 역할을 하게 된다.

## MVC

MVC패턴은 Model, View, Controller 패턴이다.

- 컨트롤러 : HTTP요청을 받아서 파라미터를 검증하고 비지니스 로직을 실행해주는 역할
- 뷰 : 모델에 담겨있는 데이터를 사용해서 화면을 그린다.
- 모델 : 뷰에 출력할 데이터를 담아둔다. 

![mvc](/assets/img/mvc.png)

```java
@WebServlet(name = "mvcMemberSaveServlet", urlPatterns = "/servlet-mvc/members/save")
public class MvcMemberSaveServlet extends HttpServlet{
    
    private MemberRepository memberRepository = MemberRepository.getInstance();
    
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException{
        String name = req.getParameter("username");
        int age = Integer.parseInt(req.getParameter("age"));
        
        Member member = new Member(name, age);
        memberRepository.save(member);
        
        req.setAttribute("member", member);
        
        String viewPath = "/WEB-INF/views/save-result.jsp";
        RequestDispatcher dispatcher = req.getRequestDispatcher(viewPath);
        dispatcher.forward(req, res);
    }
}
```

```java
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Title</title>
</head>
<body>
성공
<ul>
    <li>id=${member.id}</li>
    <li>username=${member.name}</li>
    <li>age=${member.age}</li>
</ul>
<a href="/index.html">메인</a>
</body>
</html>

```

이런식으로 MVC패턴을 사용할 수 있다.

## MVC의 한계점

먼저 MVC를 사용하면 `Dispather`와 같은 반복되는 코드가 생긴다. 그리고 공통적인 부분을 앞에 따로 빼서 실행하기 쉽지 않다. 
