---
layout: post
title: Spring - 스프링 MVC 기본 기능
tags: [Spring, 스프링 MVC]
comments: true
---

## 요청 매핑

```java
@Slf4j
@RestController
@RequestMapping("/mapping/users")
public class MappingClassController{
    @GetMapping
    public String user(){
        log.info("get user");
        return "get user";
    }
    
    @PostMapping
    public String addUser(){
        log.info("add user");
        return "add user";
    }
    
    @GetMapping("/{userId}")
    public String findUser(@PathVariable String userId){
        log.info("find user = {}", userId);
        return "find user";
    }
    
    @PatchMapping("/{userId}")
    public String updateUser(@PathVariable String userId){
        log.info("update user = {}", userId);
        return "update user";
    }
    
    @DeleteMapping("/{userId}")
    public String deleteUser(@PathVariable String userId){
        log.info("delete user = {}", userId);
        return "delete user";
    }
    
}
```

`RestController`를 쓰면 뷰를 찾는것이 아니라 문자 그대로 반환해준다. `GetMapping` `PostMapping` 등은 해당 메서드로 호출하지 않으면 405에러를 낸다. 

> @PathVariable을 쓰면 경로변수를 사용할 수 있다. 최근에는 이런 경로 변수를 선호한다.

## 요청 파라미터

```java
@RequestMapping("/request-param-v1")
public void requestParamV1(HttpServletRequest request, HttpServletResponse response){
    String username = request.getParameter("username");
    int age = Integer.parseInt(request.getParameter("age")); 
    
    log.info("name = {}, age = {}", username, age);
}
```

서블릿과 마찬가지로 HttpServletRequest, HttpServletResponse를 사용해서 파라미터를 받을 수 있다.

```java
@ResponseBody
@RequestMapping("/request-param-v2")
public String requestParamV2(
  @RequestParam("username") String username,
    @RequestParam("age") int age
){

    log.info("name = {}, age = {}", username, age);
    return ("ok");
}
```

`@RequestParam` 어노테이션을 사용하면 이렇게 깔끔하게 코드를 작성할 수 있다. 이때 변수이름과 파라미터이름이 같으면 @RequestParam("username") 부분에서 ("username") 을 생략할 수 있다.

> 파라미터가 int, Integer, String 같은 단순타입이면 @RequestParam도 생략할 수 있지만 너무 생략해도 알아보기 힘들기 때문에 비추한다.

```java
@ResponseBody
@RequestMapping("/request-param-required")
public String requestParamRequired(
  @RequestParam(required = false) String username,
    @RequestParam(required = false) Integer age
){

    log.info("name = {}, age = {}", username, age);
    return ("ok");
}
```

`reqquired` 옵션을 false로 하면 파라미터 값을 주지 않으면 기본값으로 설정된다. 이때 int는 null이 들어갈 수 없기 때문에 Integer로 해주어야한다.

> defaultValue 옵션을 주면 null이 들어갈 일이 없기 때문에 int로 자료형을 설정해주어도 된다.

```java
@ResponseBody
@RequestMapping("/request-param-map")
public String requestParamMap(
  @RequestParam Map<String, String> map
){
    
    log.info("name = {}, age = {}", map.get("username"), map.get("age"));
    return ("ok");
}
```

파라미터를 Map 자료형으로 모두 가져올 수도 있다.

```java
@ResponseBody
@RequestMapping("/model-attribute-v1")
public String requestParamMap(
  @ModelAttribute HelloData helloData
){
    
    log.info("name = {}, age = {}", helloData.getUsername(), helloData.getAge());
    return ("ok");
}
```

원래 객체를 `new HelloData`를 해서 값을 일일히 다 설정해주어야하는데 `@ModelAttribute`를 사용해서 간단히 끝낼 수 있다. 

> @ModelAttribute도 생략가능하나 위와 같은 이유로 비추

## Http 요청 메시지

요청 파라미터와 다르게 요청 메시지는 body에 있는값을 `inputStream`으로 꺼내서 String으로 바꿔줘야하는 귀찮은 과정을 겪어야한다.

```java
@PostMapping("/request-body-string-v3")
public HttpEntity<String> requestBodyStringV3(HttpEntity<String> httpEntity){
    String body = httpEntity.getBody();
    
    log.info("body = {}", body);
    return new HttpEntity<>("ok");
}
```

`HttpEntity`는 요청바디를 편리하게 접근할 수 있도록 해준다. 또한 요청또한아니라 응답도 HttpEntity를 통해서 편리하게 할 수 있다.

```java
@ResponseBody
@PostMapping("/request-body-string-v4")
public String requestBodyStringV4(@RequestBody String body){

    log.info("body = {}", body);
    return "ok";
}
```

하지만 `@RequestBody`어노테이션을 쓰면 더 쉽게 사용할 수 있다. 응답도 `ResponseBody`를 사용하면 문자열만 반환하면 된다.

> ResponseBody는 응답코드를 설정할 수는 없기 때문에 응답코드를 설정해주려면 HttpEntity를 사용하면 된다.

```java
private ObjectMapper objectMapper = new ObjectMapper();
    
@ResponseBody
@PostMapping("/request-body-json-v2")
public String requestBodyJsonV2(@RequestBody String body) throws IOException {
    log.info("body = {}", body);
    HelloData helloData = objectMapper.readValue(body, HelloData.class);
    log.info("username = {}", helloData.getUsername());        
    log.info("age = {}", helloData.getAge());
    return "ok";
}
```

단순 텍스트가 아니라 json으로 올 때도 같은방법으로 받은후 `ObjectMapper`를 통해 json을 처리해주면 된다.

```java
@ResponseBody
@PostMapping("/request-body-json-v3")
public String requestBodyJsonV3(@RequestBody HelloData helloData) throws IOException {
    log.info("username = {}", helloData.getUsername());        
    log.info("age = {}", helloData.getAge());
    return "ok";
}
```

더 쉽게 하는 방법은 데이터 클래스를 데이터 타입으로 넣어주면 된다. 응답도 데이터 클래스로 반환하면 Json으로 응답된다.

## Http 응답

```java
@RequestMapping("response-view-v1")
public ModelAndView responseView(){
    ModelAndView model = new ModelAndView("response/hello").addObject("data", "asdf");
    return model;
}

@RequestMapping("response-view-v2")
public String responseViewV2(Model model){
    model.addAttribute("data", "asdfasdf");
    return "response/hello";
}
```

응답은 요청을 하면서 많이 해봤기 때문에 간단히 하고 넘어간다.

기본적으로 스프링은 `ModelAndView`를 통해서 뷰를 넘겨준다. ModelAndView를 생성하여 addObject로 정보를 넣어주면 된다. 또한, String으로 반환해주면(@ResponseBody 를 빼야함) 자동으로 뷰를 매핑하여 반환해주는데, 이때 Model을 받아서 attribute를 추가해주면 데이터를 넣어줄 수 있다.

## Http 메시지 컨버터

![message](/assets/img/message_converter.PNG)

`ResponseBody`를 적용하면 viewResolver가 작동하는 대신 메시지 컨버터가 작동하는데, 문자냐 객체냐에 따라 JsonConverter, StringConverter가 나뉘어 작동된다.

```java
package org.springframework.http.converter;
public interface HttpMessageConverter<T> {
  boolean canRead(Class<?> clazz, @Nullable MediaType mediaType);
  boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType);
  List<MediaType> getSupportedMediaTypes();

  T read(Class<? extends T> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException;

  void write(T t, @Nullable MediaType contentType, HttpOutputMessageoutputMessage) throws IOException, HttpMessageNotWritableException;
}
```

메시지 컨버터를 보면 `canRead()` `canWrite()` `read()` `write()` 메서드가 있다.

- canRead, canWrite : 메시지 컨버터가 해당 클래스, 미디어타입을 지원하는지 체크
- read, write : 메시지 컨버터를 통해서 메시지를 읽고 쓰는 기능

```
0 = ByteArrayHttpMessageConverter
1 = StringHttpMessageConverter
2 = MappingJackson2HttpMessageConverter
```

요청데이터, 반환 데이터 타입등에 따라 해당 컨버터가 실행된다.

![st](/assets/img/message_stru.PNG)

요청을 하게 되면 우선 해당 핸들러에 맞는 RequestMapping 핸들러 어댑터가 실행된다. 그리고 핸들러를 실행하기 위한 인자들이 들어오는데 이 인자들은 엄청나게 다양하다. 그래서 ArgumentResolver를 통해 이 인자들을 유연하게 처리해줄 수 있게 해준다. 이 때 메시지 컨버터는 ArgumentResolver 다음에 실행되어 Json및 String으로 변환을 해준다. 
