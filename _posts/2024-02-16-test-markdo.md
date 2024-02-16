---
layout: post
title: Spring - 메시지, 국제화
tags: [Spring, 스프링 MVC]
comments: true
---

## 메시지 처리

기획자가 hello라는 문자열을 hello!라고 바꿔달라고 했다. 그럼 우리는 html을 들어가서 해당 부분을 수정하면 된다. 프로젝트가 작은경우 가능하지만 만약 hello라는 문자열이 여기저기에 1000개 정도 존재한다고 하면 일일히 들어가서 바꾸기는 너무 힘들것이다.
그래서 문자열을 한 파일에 정의해두고 정의해둔 문자열을 가져다 쓴다. 

```java
item=상품
item.id=상품 ID
item.itemName=상품명
item.price=가격
item.quantity=수량
```

message.properties 라는 파일을 만들고 문자열을 정의해둔다. 

```html
<label for="itemName" th:text="#{item.itemName}"></label>
```

그리고 타임리프에서 제공하는`#{}` 기능을 이용해서 문자열을 처리해준다. 

## 국제화

선택하는 언어에 따라 다른 언어를 보여주는 웹을 만들고 싶다면 위 코드를 활용해서 국제화를 하면 된다.

```java
item=Item
item.id=Item ID
item.itemName=Item Name
item.price=price
item.quantity=quantity
```

message_en.properties 파일을 만들어서 해당 코드를 넣는다. 

```java
public interface MessageSource {
  String getMessage(String code, @Nullable Object[] args, @Nullable StringdefaultMessage, Locale locale);
  String getMessage(String code, @Nullable Object[] args, Locale locale) throws NoSuchMessageException;
}
```

스프링에서 제공하는 `MessageSource`를 이용해서 국제화를 처리할 수 있다. 인터페이스를 보면 첫번째 인자로는 메시지 코드가 들어가고 두번째 인자에는 파라미터가 들어간다. 
message.properties 파일에 `item = Item {0}` 이런식으로 적고 파라미터를 넘겨주면 `{0}` 자리에 파라미터가 들어간다. 세번째 인자는 디폴트 메시지다. 만약 메시지 코드로 메시지를 찾지 못하면 디폴트 메시지가 출력된다. 마지막 인자는 위치정보이다. 위치에 따라서 다른 properties 파일에서 문자열을 가져온다.

## 국제화 적용

스프링은 국가 정보를 변경할 수 있게 `LocaleResolver` 라는 인터페이스를 제공하는데 스프링은 기본적으로 `Accept-Language` 를 활용하는 `AcceptHeaderLocaleResolver`를 제공한다. 이는 브라우저의 설정에 들어가서 언어 정보를 바꿔야한다.
세션이나, 쿠키로도 국가 정보를 변경할 수 있다.

```java
@Bean
public LocaleResolver localeResolver() {
    SessionLocaleResolver localeResolver = new SessionLocaleResolver();
    localeResolver.setDefaultLocale(Locale.KOREA);  // 기본 값을 Locale.KOREA로 설정
    return localeResolver;
}
```

```java
@PostMapping
public String changeLocale(@ModelAttribute LocaleDto localeDto, HttpSession session) {
    Locale locale = localeDto.getLocale();
    session.setAttribute(SessionLocaleResolver.LOCALE_SESSION_ATTRIBUTE_NAME, locale);
    return "redirect:/";
}
```

localResolver를 `SessionLocalResolver` 구현체로 바꾸고 세션에 값을 넣으면 국제화를 할 수 있다.

> LocaleResolver를 빈으로 등록할 때, 메서는 이름은 항상 localResolver이다.
