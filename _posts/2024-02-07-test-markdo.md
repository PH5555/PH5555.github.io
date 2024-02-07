---
layout: post
title: Spring - 스프링 MVC의 구조
tags: [Spring, 스프링 MVC]
comments: true
---

## 스프링 MVC의 기본 구조

![structure](/assets/img/mvc_structure.PNG)

스프링 MVC의 기본 구조를 살펴보면 우리가 저번 시간까지 만들었던 MVC와 그닥 다르지 않다. 다른점은 이름밖에 없다.

## RequestMapping 과 Controller

```java
@Controller 
public class SpringMemberFormControllerV1{
    @RequestMapping("/springmvc/v1/members/new-form")
    public ModelAndView process(){
        return new ModelAndView("new-form");
    }
}
```

스프링 MVC의 기본 구조는 다음과 같다. 스프링을 실행하게 되면 스프링은 `RequestMappingHandlerMapping`이라는 것을 자동으로 등록해서 매핑된 핸들러를 찾을 수 있도록 한다. 이 때 `Controller` 또는 `RequestMapping` 이 붙은 클래스를 찾을 수 있다.

- Controller

스프링 빈으로 등록해주고, 스프링에서 어노테이션 기반 컨트롤러로 인식한다.

- RequestMapping

요청 정보를 매핑하여 입력한 url정보로 요청하면 해당 메서드가 실행된다.

```java
@Component //컴포넌트 스캔을 통해 스프링 빈으로 등록
@RequestMapping
public class SpringMemberFormControllerV1 {
   @RequestMapping("/springmvc/v1/members/new-form")
     public ModelAndView process() {
     return new ModelAndView("new-form");
   }
}
```

`Controller` `RequestMapping` 중 하나만 있어도 되기 때문에 Controller대신 Component로 스프링빈을 등록해도 작동된다.

> Component로만 스프링 빈을 등록하면 URL과 클래스가 연결이 안되므로 RequestMapping을 추가로 클래스부분에 붙여줘야한다.

```java
@Controller
public class SpringMemberSaveControllerV1{
    
    MemberRepository memberRepository = MemberRepository.getInstance();
    @RequestMapping("/springmvc/v1/members/save")
    public ModelAndView process(HttpServletRequest request, HttpServletResponse response){
        String name = request.getParameter("username");
        int age = Integer.parseInt(request.getParameter("age"));

        Member member = new Member(name, age);
        memberRepository.save(member);
        
        ModelAndView mv = new ModelAndView("save-result");
        mv.addObject("member", member);
        return mv;
    } 
}
```

이전에 했던 회원정보를 저장하는 로직을 스프링 MVC로 구현하면 해당 코드가 된다. 이 때 우리가 구현했던 ModelView 대신 스프링에서 지원하는 ModelAndView를 쓰면 된다.

## 컨트롤러 통합

```java
@Controller
@RequestMapping("/springmvc/v2/members")
public class SpringMemberControllerV2 {
    MemberRepository memberRepository = MemberRepository.getInstance();

    @RequestMapping("/new-form")
    public ModelAndView process(){
        return new ModelAndView("new-form");
    }
    
    @RequestMapping("/save")
    public ModelAndView process(HttpServletRequest request, HttpServletResponse response){
        String name = request.getParameter("username");
        int age = Integer.parseInt(request.getParameter("age"));

        Member member = new Member(name, age);
        memberRepository.save(member);
        
        ModelAndView mv = new ModelAndView("save-result");
        mv.addObject("member", member);
        return mv;
    } 
}
```

컨트롤러는 클래스 단위가 아니고 메서드 단위로 이루어 지므로 한 클래스로 통합할 수 있다.

공통된 URL은 클래스 레벨에 적용시켜주고 세부 URL은 메서드에 적어주면 된다.

## 더 실용적이게 개션해보자!

```java
@Controller
@RequestMapping("/springmvc/v3/members")
public class SpringMemberControllerV3{
    
    MemberRepository memberRepository = MemberRepository.getInstance();

    @GetMapping("/new-form")
    public String process(){
        return "new-form";
    }
    
    @PostMapping("/save")
    public String process(@RequestParam("username") String name, @RequestParam("age") int age, Model model){
        Member member = new Member(name, age);
        memberRepository.save(member);

        model.addAttribute("member", member);
        return "save-result";
    } 
}
```

`HttpServletRequest` `HttpServletResponse` 대신 `@RequestParam`을 써서 간단하게 처리할 수 있다. 내가 자료형을 지정해주는 대로 알아서 변환해주기 때문에 엄청 편하다. `Model`이라는 객체를 넘겨받아서 필요한 정보를 넘겨줄 수도 있다. 그리고 ModelAndView대신 String을 넘겨주어도 스프링에서 알아서 문자열 정보로 ModelAndView를 만들어준다.

`RequestMapping`대신 `GetMapping` `PostMapping` 을 사용할 수 있다. RequestMapping은 어떤 방법으로든 접근할 수 있어서 좋은 설계가 아니다. GetMapping인 곳에 Post로 보내면 405에러를 띄워준다.

> PostMapping 이어도 폼으로 넘어오는 데이터는 쿼리로 넘어오기 때문에 RequestParam을 쓸 수 있다.
