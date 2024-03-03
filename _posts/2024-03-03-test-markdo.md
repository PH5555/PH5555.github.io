---
layout: post
title: Spring - 로그인 처리
tags: [Spring, 스프링 MVC]
comments: true
---

## 로그인 기능 개발

```java
public Optional<Member> findByLoginId(String loginId){
    return findAll().stream().filter(m -> m.getLoginId().equals(loginId)).findFirst();
}
```

```java
@Service 
@RequiredArgsConstructor 
public class LoginService{
    private final MemberRepository memberRepository;
    
    public Member login(String loginId, String password){
        return memberRepository.findByLoginId(loginId).filter(m -> m.getPassword().equals(password)).orElse(null);
        
    }
}
```

회원들의 아이디 목록에서 입력한 아이디와 맞는 것을 찾아서 비밀번호도 일치하는지 확인하는 로직을 짰다.

아이디나 비밀번호가 일치하지 않으면 로그인이 되지 않는다. 하지만 페이지를 이동하면 로그인 상태가 유지되지 않는다. 로그인 상태를 유지하려면 쿠키나 세션을 사용해야 한다.

## 쿠키를 이용한 로그인 상태 유지

로그인 정보를 쿼리 파라미터에 포함시킬 수도 있지만 모든 요청마다 쿼리 파라미터에 로그인 정보를 포함시키는것은 쉬운일이 아니다. 로그인 정보를 쿠키에 넣게 되면 쿠키가 만료될때까지 모든 요청에 쿠키가 포함되게 된다. 

> 쿠키에는 영속쿠키와 세션쿠키가 있다. 만료날짜를 생략하면 세션쿠키가 되며 브라우저가 종료되면 없어진다.

```java
public String login(@Valid @ModelAttribute LoginForm form, BindingResult bindingResult, HttpServletResponse response){
    if(bindingResult.hasErrors()){
        return "login/loginForm";
    }
    
    Member member = loginService.login(form.getLoginId(), form.getPassword());
    
    if(member == null) {
    bindingResult.reject("loginFail", "아이디 또는 비밀번호가 맞지 않습니다.");
        return "login/loginForm";
    }
    
    Cookie cookie = new Cookie("memberId", String.valueOf(member.getId()));
    response.addCookie(cookie);
    
    return "redirect:/";
}
```

로그인이 완료되면 새로운 쿠키를 생성해주고, `HttpServletResponse` 객체를 이용해 쿠키를 추가해준다. 

```java
public String loginHome(@CookieValue(value = "memberId", required = false) Long id, Model model){
    if(id == null){
        return "home";
    }
    
    Member member = memberRepository.findById(id);
    
    if(member == null){
        return "home";
    }
    
    model.addAttribute("member", member);
    return "loginHome";
}
```

홈화면에서 @CookieValue로 쿠키정보를 가져오고 쿠키가 존재하면 사용자 정보를 model에 넣어준다.

> 처음에 홈화면에 진입하면 로그인이 되어있지 않은 사용자도 있기 때문에 `required = false`를 해준다.

## 쿠키의 보안문제

쿠키로 로그인을 구현하면 절대절대 안된다. 

- 쿠키 값은 임의로 변경가능
  
쿠키 값을 개발자 모드로 들어가서 쉽게 바꿀 수 있다. 현재 우리는 유저id 값을 쿠키에 넣게 되는데 유저id를 예측할 수 있다면 다른 사용자의 아이디로 로그인 할 수 있게된다.

- 쿠키 값은 탈취가능

개발자 모드에서 쿠키 값을 볼 수 있다. 만약 쿠키 정보에 나의 중요한 정보가 있다면 해당 정보를 악용할 수 있다. 또한 쿠키를 이용해서 해당 값으로 악의적인 요청을 계속 시도할 수 있다.

이런 문제점을 보완하기 위해 이와같은 해결방법이 있다.

- 쿠키에 중요한 정보를 노출하지 않고, 예측할 수 없게 랜덤한 토큰값을 주고 서버에서 토큰을 사용자와 매칭하여 사용한다.
- 해커가 만약 토큰을 탈취하더라도 사용할 수 없도록 만료기간을 짧게 생성한다.

## 세션을 이용한 로그인 상태 유지

![session](/assets/img/session.PNG)

사용자가 아이디, 비밀번호를 입력하여 요청을 보내면 서버에서 해당 정보가 맞는지 확인 후 랜덤한 값을 생성한다. 이를 sessionId로 넣고 사용자 정보와
매칭시켜준다. 그리고 쿠키에 sessionId를 넘겨준다. 랜덤한 값만 브라우저에 넘어가고 사용자와 관련된 정보는 하나도 넘겨지지 않는다. 이렇게 하면 위에서 언급한 문제를 모두 해결할 수 있다.

```java
@Component 
public class SessionManager {
    private Map<String, Object> sessionStore = new HashMap<>();
    private static final String SESSION_COOKIE_NAME = "mySessionId";
    
    public void createSession(Object value, HttpServletResponse response){
        String sessionId = UUID.randomUUID().toString();
        sessionStore.put(sessionId, value);
        
        Cookie cookie = new Cookie(SESSION_COOKIE_NAME, sessionId);
        response.addCookie(cookie);
    }
    
    public Object getSession(HttpServletRequest request){
        Cookie sessionCookie = findCookie(request, SESSION_COOKIE_NAME);
        if(sessionCookie == null){
            return null;
        }

        return sessionStore.get(sessionCookie.getValue());
    }
    
    public void expire(HttpServletRequest request){
        Cookie sessionCookie = findCookie(request, SESSION_COOKIE_NAME);
        if(sessionCookie != null){
            sessionStore.remove(sessionCookie.getValue());
        }
    }
    
    public Cookie findCookie(HttpServletRequest request, String cookieName){
        Cookie[] cookies = request.getCookies();
        if(cookies == null){
            return null;
        }

        return Arrays.stream(cookies).filter(c -> c.getName().equals(cookieName)).findAny().orElse(null);
    }
}
```

세션아이디와 사용자 정보를 매칭시켜주는 테이블인 `sessionStore`을 생성해준다. 

createSession 메서드에서는 `UUID`를 이용해서 랜덤한 값을 하나 생성해주고 이를 세션 아이디로 사용해 테이블에 넣어준다. 그리고 세션 아이디를 쿠키에 추가해준다.

getSession 메서드에서는 쿠키에서 sessionId를 찾고 그 값을 이용해서 사용자정보를 반환한다.

```java
public String loginV2(@Valid @ModelAttribute LoginForm form, BindingResult bindingResult, HttpServletResponse response){
    if(bindingResult.hasErrors()){
        return "login/loginForm";
    }
    
    Member member = loginService.login(form.getLoginId(), form.getPassword());
    
    if(member == null) {
    bindingResult.reject("loginFail", "아이디 또는 비밀번호가 맞지 않습니다.");
        return "login/loginForm";
    }
    
    sessionManager.createSession(member, response);
  
    return "redirect:/";
}
```

```java
public String loginHomeV2(HttpServletRequest request, Model model){
   
    Member member = (Member)sessionManager.getSession(request);
    if(member == null){
        return "home";
    }
    
    model.addAttribute("member", member);
    return "loginHome";
}
```

이렇게 만든 세션 기능을 이용해서 로그인컨트롤러와 홈컨트롤러를 수정한다.

## HttpSession

위에서 우리는 세션을 직접 구현해서 사용했다. 자바에서는 `HttpSession`을 제공하는데 이도 우리가 구현한 것과 거의 비슷하게 구조가 되어있다.

```java
public String loginV3(@Valid @ModelAttribute LoginForm form, BindingResult bindingResult, HttpServletRequest request){
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
  
    return "redirect:/";
}
```

request.getSession으로 세션을 가져오고 세션에 정보를 추가해줄 수 있다.

```java
public String loginHomeV3(HttpServletRequest request, Model model){
       
    HttpSession session = request.getSession(false);
    
    if(session == null){
        return "home";
    }
    
    Member member = (Member)session.getAttribute(SessionConst.LOGIN_MEMBER);
    
    if(member == null){
        return "home";
    }
    
    model.addAttribute("member", member);
    return "loginHome";
}
```

세션데이터를 가져와서 원하는 데이터를 뽑아쓸 수 있다. 이 때 getSession(false)를 하면 세션이 없으면 새로운 세션을 생성하지 않고 null을 반환해준다.

```java
public String logoutV3(HttpServletRequest request){
  
    HttpSession session = request.getSession(false);
    if(session != null) {
        session.invalidate();
    }
    
    return "redirect:/";
}
```

로그아웃을 할 때는 `invalidate`를 하여 세션을 제거하면 된다. 

스프링은 세션을 더 편리하게 사용할 수 있도록 `@SessionAttribute`를 제공한다.

```java
@GetMapping("/")
public String loginHomeV4(@SessionAttribute(name = SessionConst.LOGIN_MEMBER, required = false) Member member, Model model){
   
    if(member == null){
        return "home";
    }

    model.addAttribute("member", member);
    return "loginHome";
}
```

`@SessionAttribute`의 name에 해당 세션의 id를 넣어주면 편리하게 값을 가져올 수 있다.

## 세션 타임아웃

세션은 사용자가 로그아웃을 클릭하면 삭제된다. 하지만 우리는 실제로 로그아웃을 클릭하지 않고 그냥 웹브라우저를 종료한다. Http요청은 연결이 유지되지 않기 때문에 브라우저를 그냥 종료하면 로그아웃이 된상태인지 확인이 불가하다.
이렇게 남아있는 세션을 무한정 보관하면 다음과 같은 문제가 발생한다.

- 세션 데이터 악용 가능

세션 데이터가 탈취됐을 경우 해당 세션으로 악의적인 요청을 계속 보낼 수 있다.

- 메모리 부족

세션은 만들때마다 메모리를 사용한다. 메모리의 크기는 무한하지 않기 때문에 꼭 필요할 때만 세션을 생성해야한다.

그러면 세션을 언제 종료해주면 좋을까? 세션 생성으로부터 30분 정도 잡으면 적당할 것 같다. 하지만 30분이 지날때마다 로그인을 다시해줘야한다. 따라서 마지막 요청으로부터 30분으로 잡아주면 좋다. HttpSession은 이런 방식을 사용한다.

```
server.servlet.session.timeout=60
```

application.properties에서 세션유지시간을 정할 수 있다. 이렇게 하면 모든 세션에 해당 초가 적용된다.

```
session.setMaxInactiveInterval(1800);
```

특정 세션의 타임아웃을 설정하고 싶으면 이렇게 쓰면 된다.

> 세션 타임아웃 시간을 너무 길게 가져가면 세션이 메모리에 많이 쌓이게 되어, 서비스 장애가 발생할 수 있다.
