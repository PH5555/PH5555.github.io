---
layout: post
title: Spring - 컴포넌트 스캔
tags: [Spring]
comments: true
---

## 컴포넌트 스캔

지금까지 스프링 빈을 등록할 때 자바코드에서 @Bean을 쓰거나 xml 코드를 사용했다. 한 두개의 빈이 있을 때는 괜찮지만 100개 1000개가 되면 일일이 붙이기에는 너무 힘들것이다. 이를 위해서 스프링에서는 설정 정보가 없어도 자동으로 빈을 등록해주는 컴포넌트 스캔을 제공한다. 

```java
@Configuration
@ComponentScan(
    excludeFilters = @Filter(type = FilterType.ANNOTATION, classes = Configuration.class)
)
public class AutoAppConfig {
    
}
```

먼저 AutoAppConfig라는 파일을 만들어준다. 기존의 `@Configuration` 어노테이션에다가 `@ComponentScan` 어노테이션을 붙이면 된다. 해당 예제에서 filter를 붙인 이유는 아직 삭제하지 않은 AppConfig의 설정 정보가 같이 등록될 수 있기 때문이다.

```java
@Component
public class MemberServiceImpl implements MemberService{
    
    private MemberRepository memberRepository;
    
    @Autowired
    public MemberServiceImpl(MemberRepository memberRepository){
        this.memberRepository = memberRepository;
    }
    
    @Override 
    public void join(Member member){
        memberRepository.save(member);
    }    
    
    @Override 
    public Member findMember(Long id){
        return memberRepository.findById(id);
    }
}
```

그 후 빈으로 등록될 클래스에 `@Component`어노테이션을 붙이면 된다. 그러면 @Component가 붙은 클래스를 스캔해서 스프링 빈으로 등록해준다.

> 이 때 `Autowired`를 붙인 이유는 의존관계를 설정하기 위해서다. 기존 설정 파일에서는 해당 생성자를 실행해서 의존관계를 직접 해주지만 컴포넌트 스캔으로 할 경우에는 코드를 직접 실행하지 않기 때문에 의존관계를 설정할 수 없다. 따라서 Autowired어노테이션을 붙여 의존관계를 설정해준다.

그리고 `AnnotationConfigApplicationContext`의 인자로 AutoAppConfig를 넘겨주면 앱이 정상적으로 실행된다. 

## 컴포넌트 스캔 범위

우리가 컴포넌트 스캔을 할 때 모든 자바 파일을 스캔하면 속도가 굉장히 느려질 것이다. 그러므로 한정된 파일만을 스캔하는것이 중요하다.

```java
@ComponentScan(
 basePackages = "hello.core",
}
```

ComponentScan의 옵션으로 basePackages을 지정하면 지정한 패키지를 포함한 하위 패키지를 스캔한다. 

> ComponentScan의 basePackages를 지정하지 않으면 ComponentScan이 적용된 파일의 패키지부터 스캔을 시작한다. 해당 예시에서는 AppConfig 파일이 최상단에 있으므로 모든 파일을 스캔한다. 이렇게 따로 스캔정보를 입력하지 않고 설정정보를 최상단에 놓는 방법이 권장된다.

> 스프링 부트의 시작 정보인 `@SpringBootApplication`안에는 `@ComponentScan`이 포함되어있다. 즉 ComponentScan을 따로 지정하지 않아도 실행된다. AnnotationConfigApplicationContext app = new AnnotationConfigApplicationContext(Application.class); 해당 코드를 작성하면 된다. (ComponentScan은 기본적으로 @Configuration 클래스를 모두 스캔하기 때문에 Config 파일 전부 삭제해야됨)

## 컴포넌트 스캔 대상

`@ComponentScan`은 Component만 스캔하는 것이 아니다.

스캔 대상

- Component
- Repository
- Controller
- Service
- Configuration     


