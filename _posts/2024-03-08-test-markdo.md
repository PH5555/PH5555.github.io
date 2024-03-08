---
layout: post
title: Spring - API 예외 처리
tags: [Spring, 스프링 MVC]
comments: true
---

## API 예외 처리

우리는 지금까지 MVC의 예외처리에 대해서 알아봤다 MVC에서 예외를 처리할 때는 단순히 예외 페이지만 보여주면 된다. 하지만 API 에서는 좀 다르다. 요청마다 응답 스펙을 다르게 해야하고, 데이터를 JSON으로 내려주어야한다. 

```java
@RestController
public class ApiExceptionController {
  @GetMapping("/api/members/{id}")
  public MemberDto getMember(@PathVariable("id") String id) {
    if (id.equals("ex")) {
      throw new RuntimeException("잘못된 사용자");
    }
    return new MemberDto(id, "hello " + id);
  }

  @Data
  @AllArgsConstructor
  static class MemberDto {
    private String memberId;
    private String name;
  }
}
```

- MemberDto를 정의하고 컨트롤러에서 알맞은 요청이 들어오면 MemberDto를 반환하고 아니라면 `RuntimeException`을 반환한다.

API이기 때문에 JSON으로 반환될것을 기대했지만 HTML코드(이전에 만들어놨던 오류 페이지)로 반환이 된다. 

```java
@GetMapping("error-page/500")
public String error500(HttpServletRequest request, HttpServletResponse response) {
    return "error-page/500";
}
```

현재는 에러를 처리하는 컨트롤러가 뷰를 반환하게 되어있기 때문에 API를 처리하는 부분을 따로 만들어줘야 한다.

```java
@GetMapping(value = "error-page/500", produces = MediaType.APPLICATION_JSON_VALUE)
public ResponseEntity<Map<String, Object>> error500Api(HttpServletRequest request, HttpServletResponse response) {
    Map<String, Object> result = new HashMap<>();
    Exception ex = (Exception)request.getAttribute(ERROR_EXCEPTION);
    result.put("status", request.getAttribute(ERROR_STATUS_CODE));
    result.put("message",ex.getMessage());
  
    Integer statusCode = (Integer) request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
    return new ResponseEntity<>(result, HttpStatus.valueOf(statusCode));
}
```

- `produces`에  MediaType.APPLICATION_JSON_VALUE 를 넣어주면 클라이언트에서 `application/json`으로 요청을 보내면 해당 컨트롤러가 실행된다.
- 에러코드와 에러 내용을 포함해서 Map을 만들고 ResponseEntity로 반환한다.

```
{
 "message": "잘못된 사용자",
 "status": 500
}
```

그러면 이렇게 성공적인 결과가 나온다.

## 스프링 부트 기본 오류 처리

근데 사실 스프링 부트에서 기본적으로 오류 처리를 해준다. 우리가 작성했던 customize를 지우고 실행해보면

```java

 "timestamp": "2021-04-28T00:00:00.000+00:00",
 "status": 500,
 "error": "Internal Server Error",
 "exception": "java.lang.RuntimeException",
 "trace": "java.lang.RuntimeException: 잘못된 사용자\n\tat
hello.exception.web.api.ApiExceptionController.getMember(ApiExceptionController.
java:19...,
 "message": "잘못된 사용자",
 "path": "/api/members/ex"
}
```

이렇게 기본적으로 설정되어있는 정보들이 나온다. 하지만 API오류 처리를 할 때는 컨트롤러에 따라 다양한 스펙으로 처리해주어야하기 때문에 좋은 방법은 아니다.(오류 페이지를 할때만 유용함)

## HandlerExceptionResolver

오류를 잡지 못하고 WAS 밖까지 오류가 던져지면 기본적으로 500에러가 나온다. 하지만 우리는 400, 404 에러등 다양한 에러에 따라 다른 스펙을 적용하고 싶다.

```java
@GetMapping("/api/members/{id}")
public MemberDto getMember(@PathVariable("id") String id){
    if(id.equals("ex")){
        throw new RuntimeException("잘못된 사용자");
    }
    if(id.equals("bad")){
        throw new IllegalArgumentException("잘못된 인자");
    }
    return new MemberDto(id, "hello" + id);
}
```

bad 를 파라미터로 넣어서 요청하면 

```java
```json
{
 "status": 500,
 "error": "Internal Server Error",
 "exception": "java.lang.IllegalArgumentException",
 "path": "/api/members/bad"
}
```

400에러가 나와야하는데 500에러가 나오게 된다.

![exceptionResolver](/assets/img/exceptionResolver.PNG)

이렇게 컨트롤러 밖으로 던져진 에러의 처리 방식을 바구고 싶으면 `ExceptionResolver`를 이용하면 된다.

- 컨트롤러에서 에러가 발생하면 ExceptionResolver에서 예외해결을 시도한다.
- WAS까지 예외가 전달되지 않고 에러를 해결할 수 있다.
- 예외를 해결해도 postHandle()은 실행되지 않는다.

```java
public class MyHandlerExceptionResolver implements HandlerExceptionResolver{
    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
    	try{
            if(ex instanceof IllegalArgumentException){
                log.info("illegal");
                response.sendError(HttpServletResponse.SC_BAD_REQUEST, ex.getMessage());
                return new ModelAndView();
            }
        }catch(IOException e){
            log.info("error");

        }
        return null;
    }
}
```

- HandlerExceptionResolver를 구현하면 된다.
- ex에 예외가 담겨서 오는데 이 예외가 어떤 예외인지 판단해서 `sendError`해주면 된다.
- ModelAndView를 반환하는 이유는 예외 상황에서 정상 상황으로 돌아오기 위해서이다. (WAS까지 예외가 안 던져짐)

> 빈 ModelAndView 를 반환하면 뷰를 렌더링 하지 않고 정상흐름으로 간다. null이 반환되면 다음 handlerExceptionResolver를 찾아서 실행한다. 만약에 더 이상 실행할 수 있는 resolver가 없으면 서블릿 밖으로 예외를 던진다.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void extendHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {
        resolvers.add(new MyHandlerExceptionResolver());
    }
}
```

리졸버를 만든후 등록을 시키면 된다.

## HandlerExceptionResolver 활용

```java
public class UserException extends RuntimeException {
     public UserException() {
     	super();
     }
     public UserException(String message) {
     	super(message);
     }
     public UserException(String message, Throwable cause) {
     	super(message, cause);
     }
     public UserException(Throwable cause) {
     	super(cause);
     }
     protected UserException(String message, Throwable cause, boolean
    				enableSuppression, boolean writableStackTrace) {
     	super(message, cause, enableSuppression, writableStackTrace);
 	 }
}
```

RuntimeException를 상속받는 새로운 exception을 정의한다.

```java
@GetMapping("/api/members/{id}")
public MemberDto getMember(@PathVariable("id") String id){
    if(id.equals("ex")){
        throw new RuntimeException("잘못된 사용자");
    }
    if(id.equals("bad")){
        throw new IllegalArgumentException("잘못된 인자");
    }
    if(id.equals("user-ex")){
        throw new UserException("잘못된 유저");
    }
    return new MemberDto(id, "hello" + id);
}
```

정의한 `UserException`을 처리하는 로직을 추가한다.

```
public class UserHandlerExceptionResolver implements HandlerExceptionResolver{
    private static ObjectMapper objectMapper = new ObjectMapper();
    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
    	try{
            if(ex instanceof UserException){

                String acceptHeader = request.getHeader("accept");
                log.info(acceptHeader);
                response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
               
                if(acceptHeader.equals("application/json")){
                	Map<String, Object> errorResult = new HashMap<>();
                    errorResult.put("ex", ex.getClass());
                    errorResult.put("message", ex.getMessage());
                    String result = objectMapper.writeValueAsString(errorResult);
                    response.setContentType("application/json");
                    response.setCharacterEncoding("utf-8");
                    response.getWriter().println(result);
                    return new ModelAndView();
                }
                else {
                    return new ModelAndView("error/500");
                }
         
            }
        }catch(IOException e){
   

        }
        
        return null;
    }
}
```

`HandlerExceptionResolver`를 구현하여 해당 에러에 맞는 스펙의 JSON을 생성하여 반환할 수 있다.

## 스프링이 제공하는 ExceptionResolver

하지만 이렇게 ExceptionResolver를 직접 구현하는것은 너무 귀찮고 API형식에서 ModelAndView를 반환하는것은 사실 맞지 않다. 스프링에서는 이런것을 기본적으로 제공해준다.

1. ExceptionHandlerExceptionResolver
2. ResponseStatusExceptionResolver
3. DefaultHandlerExceptionResolver

스프링에서 제공하는 ExceptionResolver는 이렇게 3가지 종류가 있는데 1번이 제일 우선순위가 높다.

## ResponseStatusExceptionResolver

ResponseStatusExceptionResolver는 예외에 따라서 HTTP상태코드를 다르게 적용시켜주는 역할을 한다.

- @ResponseStatus가 달려있는 예외
- ResponseStatusException 예외

이렇게 두가지 예외를 처리한다.

```java
@ResponseStatus(code = HttpStatus.BAD_REQUEST, reason = "잘못된 요청 오류")
public class BadRequestException extends RuntimeException {
    
}
```

`@ResponseStatus`를 붙여서 새로운 예외를 정의해준다. 이 예외가 발생하면 ResponseStatusExceptionResolver는 해당 예외를 400으로 처리해준다.(원래는 500으로 처리해주어야함)

```java
@GetMapping("/api/response-status/ex1")
public void responseStatusEx1() {
    throw new BadRequestException();
}
```

컨트롤러에서 해당 예외를 발생시켜보면

```java
 "status": 400,
 "error": "Bad Request",
 "exception": "hello.exception.exception.BadRequestException",
 "message": "잘못된 요청 오류",
 "path": "/api/response-status-ex1"
}
```

이렇게 400에러가 뜨는것을 볼 수 있다.

> messages.properties 에 에러 메시지를 정의하고 responseStatus의 reason에 에러 메시지 코드를 넣어주어도 된다.

하지만 라이브러리 예외와 같은 경우 우리가 함부로 바꿀 수 `@ResponseStatus` 코드를 넣을 수 없다.

```java
@GetMapping("api/response-status/ex2")
public void responseStatusEx2(){
    throw new ResponseStatusException(HttpStatus.NOT_IMPLEMENTED, "오류오류", new IllegalArgumentException());
}
```

그럴 때 직접 `ResponseStatusException`에러를 발생시키면 된다. 

## DefaultHandlerExceptionResolver

```java
@GetMapping("api/default-error-ex1")
public String defaultErrorEx1(@RequestParam Integer data) {
    return "ok";
}
```

요청을 보낼 때 파라미터에 정수형이 아닌 문자열을 보냈다고 하면 타입 에러가 발생해서 500에러가 발생할 것이다. 하지만 이런경우 클라이언트가 잘못넣어준 거기 때문에 400 에러가 발생하는 것이 정상적이다.

```
{
 "status": 400,
 "error": "Bad Request",
 "exception":
"org.springframework.web.method.annotation.MethodArgumentTypeMismatchException",
 "message": "Failed to convert value of type 'java.lang.String' to required
type 'java.lang.Integer'; nested exception is java.lang.NumberFormatException:
For input string: \"hello\"",
 "path": "/api/default-handler-ex"
}
```

근데 실제로 실행해보면 500 대신 정상적으로 400이 뜨는것을 볼 수 있다. DefaultHandlerExceptionResolver.handleTypeMismatch 에서 예외가 발생하면 400으로 처리해주기 때문이다.

## @ExceptionHandler

```java
@Data 
@AllArgsConstructor 
public class ErrorResult {
    private String code;
    private String message;
}
```

다음과 같이 응답 객체를 만들어준다.

```java
@ExceptionHandler(IllegalArgumentException.class)
public ErrorResult illegalHandler(IllegalArgumentException e){
    return new ErrorResult("BAD", e.getMessage());
}

@ExceptionHandler
public ResponseEntity<ErrorResult> userExHandler(UserException e){
    ErrorResult result = new ErrorResult("bad", e.getMessage());
    return new ResponseEntity(result, HttpStatus.BAD_REQUEST);
}

@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
@ExceptionHandler
public ErrorResult exHandler(Exception e){
    return new ErrorResult("bad", e.getMessage());
}

@GetMapping("/api2/members/{id}")
public MemberDto getMember(@PathVariable("id") String id){
    if(id.equals("ex")){
        throw new RuntimeException("잘못된 사용자");
    }
    if(id.equals("bad")){
        throw new IllegalArgumentException("잘못된 인자");
    }
    if(id.equals("user-ex")){
        throw new UserException("잘못된 유저");
    }
    return new MemberDto(id, "hello" + id);
}

@Data 
@AllArgsConstructor 
private class MemberDto {
    private String id;
  private String name;
}
```

- @ExceptionHandler 를 써서 예외 마다 처리하는 로직을 짜준다. 이때 아까 만들어준 객체를 반환한다.
- 기본적으로 기존 컨트롤러와 비슷한 구조이고, 쓸 수 있는 기능도 거의 같다.
- 그냥 객체만을 반환하면 에러인데 200 코드가 반환된다. @ResponseStatus를 통해서 에러코드를 바꿀 수 있다.
- @ExceptionHandler의 안에 들어가는 에러 내용은 함수의 인자로 넘어오는 데이터의 타입과 같기 때문에 생략가능하다.
- Exception 예외를 잡으면 모든 에러를 처리할 수 있다.

> 항상 자세한 것이 우선순위를 가진다. 자식 클래스 예외로 지정된것이 더 우선 처리된다.
> ({AException.class, BException.class}) 이렇게 예외를 복수 처리할 수 있다.

## @ControllerAdvice

`@ExceptionHandler`로 예외를 잘 처리할 수 있게 되었지만 컨트롤러와 같은 클래스에 예외처리를 해주는 로직이 있으니 마음이 좀 불편하다.

```java
@RestControllerAdvice 
public class ExControllerAdvice {
    @ExceptionHandler(IllegalArgumentException.class)
    public ErrorResult illegalHandler(IllegalArgumentException e){
        return new ErrorResult("BAD", e.getMessage());
    }
    
    @ExceptionHandler
    public ResponseEntity<ErrorResult> userExHandler(UserException e){
        ErrorResult result = new ErrorResult("bad", e.getMessage());
        return new ResponseEntity(result, HttpStatus.BAD_REQUEST);
    }
    
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler
    public ErrorResult exHandler(Exception e){
        return new ErrorResult("bad", e.getMessage());
    }
}
```

`@RestControllerAdvice`를 사용하여 따로 분리해줄 수 있다. 따로 범위를 지정해주지 않으면 전체 컨트롤러에 해당 내용이 다 적용된다.
