---
layout: post
title: Spring - 예외 처리와 오류 페이지
tags: [Spring, 스프링 MVC]
comments: true
---

## 예외

자바의 메인 메서드를 직접 실행하면 main이라는 이름의 스레드를 실행한다. 실행 중에 예외를 잡지 못하고 main메서드 밖으로까지 예외가 던져지면 main 스레드를 종료한다.

웹애플리케이션에서는 사용자별로 다른 스레드가 할당된다. 예외가 발생했을 때 애플리케이션 어디에선가 예외를 잡아주면 아무런 문제가 없지만 잡지 못하면 서블릿 밖까지 예외가 전달된다.

```
WAS <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러
```

서블릿 밖에까지 예외가 전달되면 결국에는 WAS까지 예외가 전달된다.

```java
@RequestMapping("/error-ex")
public void ex() {
    throw new RuntimeException();
}
```

다음과 같이 아무런 에러를 내보면 기본적으로 tomcat에서 제공하는 에러페이지가 나온다.

```java
@RequestMapping("error-404")
public void ex404(HttpServletResponse response) throws IOException{
    response.sendError(404, "404error");
}

@RequestMapping("error-500")
public void ex500(HttpServletResponse response) throws IOException{
    response.sendError(500);
}
```

다음과 같이 `sendError`를 이용해서 에러를 코드와 메시지와 함께 전달할 수도 있다.

## 오류 페이지

하지만 Tomcat에서 기본으로 제공하는 오류 페이지는 너무 구리다. 유저에게 보여주기 위한 페이지를 우리가 새로 추가할 수 있다.

```java
@Component
public class WebServerCustomizer implements WebServerFactoryCustomizer<ConfigurableWebServerFactory> {
    @Override 
    public void customize (ConfigurableWebServerFactory factory){
        ErrorPage errorPage404 = new ErrorPage(HttpStatus.NOT_FOUND, "/error-page/404");
        ErrorPage errorPage500 = new ErrorPage(HttpStatus.INTERNAL_SERVER_ERROR, "/error-page/500");
        ErrorPage errorPageEx = new ErrorPage(RuntimeException.class, "/error-page/500");
        
        factory.addErrorPages(errorPage404, errorPage500, errorPageEx);
    }
}
```

- ErrorPage 의 첫번째 인자로 들어가는 에러가 발생하면 두번째 인자로 들어가는 경로의 컨트롤러를 실행해준다.
- 그래서 해당 경로의 컨트롤러를 새로 만들어준다.

```java
@Controller 
public class ErrorPageController {
    
    @GetMapping("error-page/404")
    public String error400(HttpServletRequest request, HttpServletResponse response) {
        return "error-page/404";
    }
    
    @GetMapping("error-page/500")
    public String error500(HttpServletRequest request, HttpServletResponse response) {
        return "error-page/500";
    }
}
```

## 오류 페이지 작동 원리

예외 발생 흐름

```
WAS <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(오류 발생)
```

오류 페이지 요청 흐름

```
WAS `/error-page/500` 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> View
```

오류가 발생하면 오류가 WAS까지 전달되고 WAS에서 오류 페이지 경로를 찾아서 해당 컨트롤러를 다시 요청한다. 즉, 유저가 한번 요청을 하고 에러가 나면 두번 요청이 되는것이다.

이 때 보면 필터나 인터셉터도 두번씩 실행되는것을 볼 수 있다. 2번 실행하면 안되는 필터가 있을 수도 있고 일단 굉장히 비효율적이다. 결국 우리는 요청이 사용자에 의한 요청인지 에러 페이지를 출력하기 위한 요청인지 구분할 수 있어야한다. 서블릿은 이런 문제를 
해결하기 위해서 `DispatcherTypes`라는 옵션을 제공한다.

> 오류 페이지를 요청할 때 request의 attribute에서 해당 내용을 확인할 수 있다.

```
public enum DispatcherType {
 FORWARD,
 INCLUDE,
 REQUEST,
 ASYNC,
 ERROR
}
```

자주 쓰는거 두개만 보면 REQUEST가 사용자가 요청하는 것이고, ERROR가 오류 페이지를 출력하기 위한 것이다.

## 필터와 DispatcherType

```java
@Bean
public FilterRegistrationBean logFilter() {
  FilterRegistrationBean<Filter> filterRegistrationBean = new FilterRegistrationBean<>();

  filterRegistrationBean.setFilter(new LogFilter());
  filterRegistrationBean.setOrder(1);
  filterRegistrationBean.addUrlPatterns("/*");
  filterRegistrationBean.setDispatcherTypes(DispatcherType.REQUEST, DispatcherType.ERROR);

  return filterRegistrationBean;
}
```

필터를 등록할 때 `setDispatcherTypes`를 추가로 등록한다. 어떤 요청 타입에서 해당 필터를 실행할지 정할 수 있다.(기본 설정은 클라이언트의 요청만 실행하는 것이다.) 

## 서블릿 예외 처리 - 인터셉터

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
  registry.addInterceptor(new LogInterceptor())
    .order(1)
    .addPathPatterns("/**")
    .excludePathPatterns("/css/**", "/*.ico", "/error", "/error-page/**");
}
```

인터셉터는 필터와는 달리 따로 지정해줄 수 있는 메서드가 없다. 대신 URL 패턴을 통해 구현할 수 있다. 그냥 error-page 일 때 해당 인터럽터를 실행하지 않으면 된다.

## 스프링 부트의 에러페이지

에러 페이지를 만들려면 위와 같은 복잡한 과정을 거쳐야한다. 하지만 스프링 부트에서는 더 편리한 기능을 제공한다. 스프링 부트에서는 기본적으로 ErrorPage를 자동으로 등록해주고 BasicErrorController라는 컨트롤러를 실행하게 된다. 그래서 우리는 
에러를 보여줄 페이지만 만들어서 정해져있는 루트에 넣기만하면 된다.

1. templates

뷰템플릿에 있는 에러페이지가 제일 우선순위가 높다.

2. static

3. error

위의 폴더에 파일이 없으면 error라는 뷰를 실행한다.

> 404.html, 4xx.html 이런식으로 파일이름을 자세히 작성할 수도 있고, 간략하게 작성할 수도 있다. 이 경우 자세한 파일이 더 우선순위가 높다.

```html
 <li th:text="|timestamp: ${timestamp}|"></li>
 <li th:text="|path: ${path}|"></li>
 <li th:text="|status: ${status}|"></li>
 <li th:text="|message: ${message}|"></li>
 <li th:text="|error: ${error}|"></li>
 <li th:text="|exception: ${exception}|"></li>
 <li th:text="|errors: ${errors}|"></li>
 <li th:text="|trace: ${trace}|"></li>
```

그리고 기본적으로 다음의 내용을 model에 담아서 전달한다. 물론 이런정보를 고객에게 노출하는것은 좋지않다.
