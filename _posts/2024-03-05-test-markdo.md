---
layout: post
title: Spring - 필터, 인터셉터
tags: [Spring, 스프링 MVC]
comments: true
---

## 공통 관심 사항

우리는 이전 시간까지 로그인 기능을 구현했다. 지금 상태에서는 로그인을 하지 않아도 모든 페이지에 접근가능하다. 이를 막기위해서 모든 컨트롤러에서 로그인이 되어있는 상태인지 확인하는 로직을 작성할 수 있을 것이다.
하지만 그렇게 되면 로직이 변경할 때마다 모든 곳을 수정해야하는 문제점이 있다. 이렇게 여러 로직에서 공통으로 필요한 것을 공통 관심 사항이라고 하는데, 이를 `AOP`로도 처리가능하지만 웹과 관련된 로직이면
`서블릿 필터` `인터셉터`로 처리할 수 있다.

## 서블릿 필터

```
HTTP 요청 -> WAS -> 필터 -> 서블릿 -> 컨트롤러
```

필터를 적용하면 필터가 실행된 후에 서블릿이 실행된다. 필터 단계에서 적절하지 않은 요청이라고 판단되면 서블릿을 실행하지 않는다.

```
HTTP 요청 -> WAS -> 필터1 -> 필터2 -> 필터3 -> 서블릿 -> 컨트롤러
```

필터는 체인을 통해서 필터를 자유롭게 추가할 수 있다. 

- 인터페이스

```java
public interface Filter {
  public default void init(FilterConfig filterConfig) throws ServletException;
  public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException;
  public default void destroy();
}
```

- init : 필터 초기화 메서드
- doFilter : 요청이 들어올 때마다 해당 메서드가 실행된다.
- destroy : 서블릿 컨테이너가 종료될 때 실행된다.

## 필터 구현

```java
@Slf4j
public class LoginCheckFilter implements Filter {
    private static final String[] whitelist = {"/", "/members/add", "/login", "/logout","/css/*"};
    
    public void init(FilterConfig filterConfig) throws ServletException{
        log.info("log init");
    }
    
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException{
     
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        String requestURI = httpRequest.getRequestURI();
        
        try{
            if(isLoginCheckPath(requestURI)){
                log.info("URI 있음");
            	HttpSession session = httpRequest.getSession(false);
                
                if(session == null || session.getAttribute(SessionConst.LOGIN_MEMBER) == null){
                    httpResponse.sendRedirect("/login?redirectURL=" + requestURI);
                    return; 
                }
            }
            chain.doFilter(request, response);
        }catch (Exception e) {
          
        }finally{
        	
        }
    }
    
    public void destroy() {
        
        log.info("log init");
    }
    
    public boolean isLoginCheckPath(String requestURI){
        return !PatternMatchUtils.simpleMatch(whitelist ,requestURI);
    }
}
```

- whitelist를 지정해서 필터가 적용되지 않을 URL을 지정한다. 요청되는 URL이 whitelist안에 있는지 검사하여 없을 때만 필터를 실행한다.

> chain.doFilter(request, response) 는 항상 실행해주어야한다. 실행해주지 않으면 다음 필터가 실행되지 않고, 결과적으로 컨트롤러가 실행되지 않는다.

- httpResponse.sendRedirect("/login?redirectURL=" + requestURI) 를 하여 로그인을 하면 마지막 URL로 돌아갈 수 있도록 한다.

```java
@PostMapping("/login")
public String loginV3(@Valid @ModelAttribute LoginForm form, BindingResult bindingResult, HttpServletRequest request, @RequestParam(defaultValue = "/") String redirectURL){
    if(bindingResult.hasErrors()){
        return "login/loginForm";
    }
    
    Member member = loginService.login(form.getLoginId(), form.getPassword());
    
    if(member == null) {
    bindingResult.reject("loginFail", "아이디 또는 비밀번호가 맞지 않습니다.");
        return "login/loginForm";
    }
    
    HttpSession session = request.getSession();
    session.setAttribute(SessionConst.LOGIN_MEMBER, member);
  
    return "redirect:" + redirectURL;
}
```

`@RequestParam(defaultValue = "/")` 를 추가해서 마지막 URL을 가져올 수 있다.

필터를 만들었으면 웹에 필터를 적용시키면 된다.

```java
@Configuration 
public class WebConfig{
    @Bean 
    public FilterRegistrationBean loginCheckFilter(){
        FilterRegistrationBean<Filter> filterRegistrationBean = new FilterRegistrationBean<>();
        filterRegistrationBean.setFilter(new LoginCheckFilter());
        filterRegistrationBean.setOrder(1);
        filterRegistrationBean.addUrlPatterns("/*");
        return filterRegistrationBean;
    }
}
```

설정 클래스를 만들고 빈으로 등록시켜준다. 이 때 `FilterRegistrationBean`을 쓴다. setFilter메서드로 필터를 적용시킬 수 있고, setOrder로 필터가 적용될 순서를 정할 수 있다. 그리고 addUrlPatterns로 필터가 적용될 URL을 정할 수 있다. 해당 예제에서는 모든 url에 필터를 적용한다.

## 인터셉트

```
HTTP요청 -> WAS -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러
```

스프링 인터셉터는 서블릿과 컨트롤러 사이에서 실행된다. 필터와 동일하게 적절하지 않은 요청이라고 판단되면 컨트롤러를 실행하지 않고, 여러 인터셉터를 적용할 수 있다.

```java
public interface HandlerInterceptor {
  default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {}
  default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable ModelAndView modelAndView) throws Exception {}
  default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception {}
}
```

인터셉터는 `HandlerInterceptor` 를 구현하면 된다.

![interceptor](/assets/img/interceptor.PNG)

- preHandle : 컨트롤러 실행전에 실행된다. 반환되는 값이 true이면 다음으로 진행하고 false일 경우에는 더 이상 진행하지 않는다.
- postHandle : 컨트롤러 실행후에 실행된다.
- afterCompletion : 뷰가 렌더링 된 이후에 실행된다.

> 컨트롤러에서 예외가 발생하면 postHandle은 실행되지 않는다. 이 때 afterComletion은 항상 실행된다 파라미터로 오는 ex를 통해서 에러메시지를 확인할 수 있다.

## 인터셉터 구현

```java
@Slf4j 
public class LoginCheckInterceptor implements HandlerInterceptor{
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String requestURI = request.getRequestURI();
        HttpSession session = request.getSession();
        
        if(session == null || session.getAttribute(SessionConst.LOGIN_MEMBER) == null) {
            log.info("미인증 사용자");
            
            response.sendRedirect("login?redirectURL=" + requestURI);
            return false;
        }
 		return true;
    }
    
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
     
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
	
    }
}
```

- 로그인만 체크하면 되기 때문에 preHandle만 구현하면 된다.
- 미인증 사용자인것으로 판단되면 더 이상 진행하지 않아도 되기 때문에 false를 반환한다.

인터셉터를 구현하고 마찬가지로 적용을 시켜주면 된다.

```java
@Configuration 
public class WebConfig implements WebMvcConfigurer{
    @Override 
    public void addInterceptors(InterceptorRegistry registry) {
    	registry.addInterceptor(new LogInterceptor()).order(1)
            .addPathPatterns("/**")
            .excludePathPatterns("/css/**", "/*.ico", "/error");
        
        registry.addInterceptor(new LoginCheckInterceptor()).order(2)
            .addPathPatterns("/**")
            .excludePathPatterns("/login", "/", "/logout", "members/add", "/css/**", "/*.ico", "/error");
    }
}
```

- WebMvcConfigurer의 addInterceptors를 구현하면 된다. 필터와 다르게 URL 패턴을 자세하게 작성할 수 있다. 인터셉터를 적용할 URL을 적어줄 수 있고, 적용되지 않을 URL도 지정해줄 수 있다.

## ArgumentResolver 활용

ArgumentResolver는 다양한 인자를 처리하기 위한 것이라고 했다. 

```java
public String loginHomeV4(@SessionAttribute(name = SessionConst.LOGIN_MEMBER, required = false) Member member, Model model){
    if(member == null){
        return "home";
    }

    model.addAttribute("member", member);
    return "loginHome";
}
```

홈컨트롤러 부분을 보면 세션을 확인하는 과정이 좀 귀찮기 때문에 이런 것을 단순화 시킬 수 있다.

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface Login {

}
```

`@Login`어노테이션을 만들어서 홈컨트롤러에 적용시킬 수 있게 할 것이다.

- Target : 어노테이션이 적용될 범위를 적용. 이 예제에서는 파라미터에만 적용할 수 있다.
- Retention : 어노테이션이 적용되는 기간을 정할 수 있다. 이 예제에서는 런타임까지 적용됨.

```java
public class LoginMemberArgumentResolver implements HandlerMethodArgumentResolver{
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        boolean hasLoginAnnotation = parameter.hasParameterAnnotation(Login.class);
        boolean hasMemberType = Member.class.isAssignableFrom(parameter.getParameterType());
        
        return hasLoginAnnotation && hasMemberType;
    }
    
    @Override
    public Object resolveArgument(MethodParameter parameter,
    	ModelAndViewContainer mavContainer, NativeWebRequest webRequest,
    	WebDataBinderFactory binderFactory) throws Exception {

        HttpServletRequest request = (HttpServletRequest) webRequest.getNativeRequest();
        HttpSession session = request.getSession();
        
        if(session == null){
            return null;
        }
        
        return session.getAttribute(SessionConst.LOGIN_MEMBER);
    }
}
```

- supportsParameter를 구현해서 리졸버가 적용될 곳인지 판단한다. `@Login` 어노테이션이 붙어 있고, `Member`타입의 파라미터에만 이 리졸버를 적용시킨다.
- resolveArgument에 실제 로직을 작성한다. webRequest를 HttpServletRequest로 바꿔주고 세션을 가져온다. 세션 값이 존재하면 그 값을 반환하면 된다.

```java

@Configuration 
public class WebConfig implements WebMvcConfigurer{
    
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
    	resolvers.add(new LoginMemberArgumentResolver());
    }
    
    @Override 
    public void addInterceptors(InterceptorRegistry registry) {
    	registry.addInterceptor(new LogInterceptor()).order(1)
            .addPathPatterns("/**")
            .excludePathPatterns("/css/**", "/*.ico", "/error");
        
        registry.addInterceptor(new LoginCheckInterceptor()).order(2)
            .addPathPatterns("/**")
            .excludePathPatterns("/login", "/", "/logout", "members/add", "/css/**", "/*.ico", "/error");
    }
}
```

`addArguemntResolver`를 구현하여 등록한다.

이렇게 ArgumentResolver를 적용하면 공통작업이 필요한곳에 편리하게 해당 내용을 적용할 수 있다.
