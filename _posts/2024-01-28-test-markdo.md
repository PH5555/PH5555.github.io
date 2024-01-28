---
layout: post
title: Spring - 서블릿
tags: [Spring, 스프링 MVC]
comments: true
---

## 서블릿 기본 구조

```java
@ServletComponentScan
@SpringBootApplication
public class Application{
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    	
    }
}
```

```java
@WebServlet(name = "helloServlet", urlPatterns = "/hello")
public class HelloServlet extends HttpServlet{
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse res) throws IOException{
        String name = req.getParameter("username");
    	  System.out.println(name);
        
        res.setContentType("text/plain");
        res.setCharacterEncoding("utf-8");
        res.getWriter().write(name);
    }
}
```

스프링의 `Componentscan`같이 서블릿도 서블릿 객체를 탐색할 수 있는 `ServletComponentScan`이 있다. 이 코드가 실행되면 `@WebServlet`어노테이션이 붙은 클래스를 서블릿 객체로 올린다. HttpServlet을 상속받은 클래스는 Service메서드를 구현해야한다. 인자로 request, response가 오는데 이는 Http 요청, 응답 메시지를 간편하게 작성할 수 있도록 도와주는 객체이다.

## Http 요청

Http 요청에는 크게 3가지로 나눌 수 있다.

1. Get 요청

데이터가 쿼리 파라미터로 전달된다.(/hello?username=kim) 

```java
@WebServlet(name = "requestParamrService", urlPatterns = "/request-param")
public class RequestParamService extends HttpServlet{
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse res) throws IOException{
        String username = req.getParameter("username");
        
        String[] names = req.getParameterValues("username");
    	
        for(String name : names) {
            System.out.println(name);
        }
    }
}
```

쿼리 파라미터로 보내지는 데이터를 추출하는 방법은 `response.getParameter()`을 하면 된다. 이 때 (/hello?username=kim&username=lee) 와 같이 같은 키가 여러가지 벨류를 가질수도 있다.(물론 그러면 별로 안좋다.) 그럴 때 `response.getParameterValues()`를 사용해서 for문을 돌리면 각각의 데이터를 얻을 수 있다.

> 값이 중복일 때 `response.getParameter`로 꺼내면 제일 먼저 온 값이 꺼내진다. 

2. HTML FORM

HTML FORM은 클라이언트에서 서버로 데이터를 보낼 때 쓰인다. 예를들어 로그인, 상품주문 등이 있다. 

```html
<!DOCTYPE html>
<html>
  <head>
   <meta charset="UTF-8">
   <title>Title</title>
  </head>
  <body>
    <form action="/request-param" method="post">
     username: <input type="text" name="username" />
     age: <input type="text" name="age" />
       <button type="submit">전송</button>
    </form>
  </body>
</html>
```

위 html을 보면 input으로 username과 age를 받고 /request-param 루트에 post방식으로 데이터를 보낸다. post방식이지만 실제로는 쿼리 파라미터로 `username=kim&age=30` 이런식으로 메시지바디에 실려서 데이터가 보내진다. 따라서 get방식에서 썼던 `getParameter` 방식을 그대로 사용하면 된다. 

3. HTTP Message body

HTTP Message body 방식은 API를 만들 때 쓰인다. API는 Json, Xml 방식등으로 데이터를 보낼 수 있다. (최근에는 xml은 거의 안쓴다.) 

```java
@WebServlet(name = "requestBodyJsonService", urlPatterns = "/request-body-json")
public class RequestBodyJsonService extends HttpServlet{
    
    private ObjectMapper objectMapper = new ObjectMapper();
    
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse res) throws IOException{
        ServletInputStream stream = req.getInputStream();
    	  String message = StreamUtils.copyToString(stream, StandardCharsets.UTF_8);
        System.out.println(message);
        HelloData helloData = objectMapper.readValue(message, HelloData.class);
        System.out.println(helloData.getUsername());
        System.out.println(helloData.getAge());

    }
}
```

먼저 `request.getInputStream()`으로 메시지바디에 있는 데이터를 읽어들인후, `StreamUtils.copyToString()`을 이용해서 문자열로 변환해준다. 순수 텍스트이면 이렇게 까지만 하면 사용할 수 있지만 Json은 여기에서 한번더 데이터 추출을 해줘야한다. `ObjectMapper`를 이용해서 데이터를 객체로 추출할 수 있다.

## HTTP 응답

HTTP 응답도 크게 3가지 방법으로 나뉜다.

1. 단순 텍스트 응답
   
```java
res.setContentType("text/plain");
res.setCharacterEncoding("utf-8");
res.getWriter().write(name);
```

맨 처음 예제처럼 `Writer()`를 이용하면 된다.

2. HTML 응답

```java
@WebServlet(name = "responseHtmlService", urlPatterns = "/response-html")
public class ResponseHtmlService extends HttpServlet{
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse res) throws IOException{
    	res.setContentType("text/html");
        res.setCharacterEncoding("utf-8");
        PrintWriter writer = res.getWriter();
        writer.println("<html>");
        writer.println("<body>");     
        writer.println("<div>안녕</div>");
        writer.println("</body>");
    	writer.println("</html>");
    }
}
```

동일한 방법으로 html을 작성해주면 된다. 이 때 다른점은 `contentType`을 text/html로 해주어야 한다는 것이다.

3. JSON 응답

```java
@WebServlet(name = "responseJsonService", urlPatterns = "/response-json")
public class ResponseJsonService extends HttpServlet{
    
    private ObjectMapper objectMapper = new ObjectMapper();
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse res) throws IOException{
    	res.setContentType("application/json");
        res.setCharacterEncoding("utf-8");
        PrintWriter writer = res.getWriter();
        
        HelloData helloData = new HelloData();
        helloData.setUsername("kim");
        helloData.setAge(20);
        
        String data = objectMapper.writeValueAsString(helloData);
        
        res.getWriter().write(data);	
        
    }
}
```

역시 `Writer()`를 이용해서 응답한다. 하지만 객체를 문자열로 바꿔야하기 때문에 `ObjectMapper.writeValueAsString()`를 이용해준다. 그리고 `contentType`을 application/json으로 해준다.
