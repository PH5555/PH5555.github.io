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
