---
layout: post
title: Spring - 스프링 타입 컨버터
tags: [Spring, 스프링 MVC]
comments: true
---

## 형변환

앱을 개발하다 보면 형변환을 해야하는 경우가 많다.

```java
public String helloV1(HttpServletRequest request) {
  String data = request.getParameter("data"); //문자 타입 조회
  Integer intValue = Integer.valueOf(data); //숫자 타입으로 변경
  System.out.println("intValue = " + intValue);
  return "ok";
}
```

이렇게 valueOf로 형변환을 할 수 있다.

```java
@GetMapping("/hello")
public String hello(@RequestParam Integer data){
    return "good";
}
```

스프링에서 제공하는 `@RequestParam`을 사용하면 자동으로 형변환이 일어난다. url에 `data=10`을 입력하면 10은 문자열이다.(파라미터는 항상 문자열) 중간에서 스프링이 문자열을 정수형으로 형변환을 해준다.

> `@RequestParam`과 마찬가지로 `@ModelAttribute` `@PathVariable`도 형변환이 일어난다.

## 타입 컨버터

스프링이 기본 제공하는 컨버터말고 직접 구현하려면 타입 컨버터를 구현하면 된다.

```java
public class IntegerToStringConverter implements Converter<Integer, String>{
    @Override 
    public String convert(Integer source){
        return String.valueOf(source);
    }
}
```

- Converter 인터페이스를 구현한다.
- convert 메서드 안에서 타입변환 로직을 작성해준다.

사실 정수형과 문자열은 스프링에서 기본적으로 변환을 해주기 때문에 사용자 정의 클래스의 타입 컨버터를 만들어보겠다.

```java
@Getter 
@EqualsAndHashCode
public class IpPort {
    private String ip;
    private int port;
    
    public IpPort(String ip, int port){
    	  this.ip = ip;
        this.port = port;
    }
}
```

새로운 클래스를 하나 정의해준다.

> `@EqualsAndHashCode`를 넣으면 equals() 메서드를 만들어준다. 모든 필드가 같으면 같다고 판단된다.

```java
public class StringToIpPortConverter implements Converter<String, IpPort>{
    @Override 
    public IpPort convert(String source){
        String[] split = source.split(":");
        String ip = split[0];
        int port = Integer.parseInt(split[1]);
    
        return new IpPort(ip, port);
    }
}
```

```java
public class IpPortToStringConverter implements Converter<IpPort, String>{
    @Override 
    public String convert(IpPort source){
        return source.getIp() + ":" + source.getPort();
    }
}
```

## 컨버전 서비스

타입 컨버터를 만들고 하나하나 찾아서 쓰기에는 불편하기 때문에 스프링에 등록해서 사용할 수 있다.

```java
DefaultConversionService service = new DefaultConversionService();
service.addConverter(new IpPortToStringConverter());
service.addConverter(new IntegerToStringConverter());
service.addConverter(new StringToIntegerConverter());
service.addConverter(new StringToIntegerConverter());

Integer result = service.convert("10", Integer.class);
System.out.println(result);
```

등록을 할 때는 `StringToIntegerConverter`같은 클래스를 알아야하지만 사용할 때에는 알 필요가 없다. (convert 하나로만 쓰기 때문) 따라서 타입 변환을 원하는 사용자는 인터페이스에만 의존하면 된다.
실제로 쓸 때에는 컨버전 서비스를 빈으로 등록해서 의존관계 주입을 받아서 사용하면 된다.

## 스프링에 컨버전 서비스 적용

@RequestParam 같은 어노테이션에서는 기본적으로 컨버전 서비스가 적용된다. 스프링에 컨버전 서비스를 등록하면 적용할 수 있다.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override 
    public void addFormatters(FormatterRegistry registry){
        registry.addConverter(new IpPortToStringConverter());
        registry.addConverter(new IntegerToStringConverter());
        registry.addConverter(new StringToIntegerConverter());
        registry.addConverter(new StringToIpPortConverter());
    }
}
```

```java
@GetMapping("/ip-port")
public String hello(@RequestParam IpPort ipPort){
    System.out.println("ip : " + ipPort.getIp());
    System.out.println("port : " + ipPort.getPort());

    return "good";
}
```

등록을 하고 실행을 해보면 문자열로 넘어온 파라미터가 성공적으로 우리가 정의한 타입으로 변환된다.

> @RequestParam 은 @RequestParam 을 처리하는 ArgumentResolver 인 RequestParamMethodArgumentResolver 에서 ConversionService 를 사용해서 타입을 변환한다

```html
<html xmlns:th="http://www.thymeleaf.org">
<head>
 <meta charset="UTF-8">
 <title>Title</title>
</head>
<body>
<ul>
 <li>${number}: <span th:text="${number}" ></span></li>
 <li>${{number}}: <span th:text="${{number}}" ></span></li>
 <li>${ipPort}: <span th:text="${ipPort}" ></span></li>
 <li>${{ipPort}}: <span th:text="${{ipPort}}" ></span></li>
</ul>
</body>
</html>
```

또한 타임리프에서 `${{}}`를 쓰면 컨버전 서비스가 적용된다.

```html
<html xmlns:th="http://www.thymeleaf.org">
<head>
 <meta charset="UTF-8">
 <title>Title</title>
</head>
<body>
<form th:object="${form}" th:method="post">
 th:field <input type="text" th:field="*{ipPort}"><br/>
 th:value <input type="text" th:value="*{ipPort}">(보여주기 용도)<br/>
 <input type="submit"/>
</form>
</body>
</html>
```

폼에 적용할 때는 `th:filed`를 사용하면 자동 적용된다. 

## 포메터

화면에 숫자를 출력할 때 1000을 1,000이런식으로 포멧을 하여 출력할 때가 많다. 이런과정을 편리하게 처리해줄 수 있는 컨버터라는 기능이 있다.

