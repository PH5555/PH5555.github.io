---
layout: post
title: Spring - 스프링의 핵심 원리 이해
tags: [Spring]
comments: true
---

## Spring 은 왜 사용할까?

Spring은 `Java` 기반으로 돌아가는 프레임워크다. 

즉 Spring의 본질은 Java의 본질에서 찾을 수 있다. Java는 객체 지향 언어이다. 즉, Spring을 이용하면 객체 지향을 극대화할 수 있다는 장점이 있다. 객체 지향이 무엇인지 생각해보자. 객체 지향에서 제일 중요한 개념은 아마 다형성일 것이다. 

> 다형성(polymorphism)이란 하나의 객체가 여러 가지 타입을 가질 수 있는 것을 의미합니다.

다형성의 가장 큰 장점은 구현이 바뀌더라도 코드의 사용자는 인터페이스에 따라서 그대로 실행만 하면 된다는 것이다. 즉, 수정과 유지보수가 편리해진다. 우리가 객체 지향 프로그래밍을 했을 때 유지보수가 쉬워진다는 말이 이 `다형성`에서 나오는것이다.

## 프로젝트 생성

강의를 따라서 해당 코드를 작성하였다.

```java
package project.member;

public interface MemberRepository {
    
    void save(Member member);
    Member findById(Long id); 
}
```

```java
package project.member;

import java.util.HashMap;
import java.util.Map;

public class MemoryMemberRepository implements MemberRepository{
    
    private static Map<Long, Member> store = new HashMap<>();
    
    @Override
    public void save(Member member){
        store.put(member.getId(), member);
    }
    
    @Override 
    public Member findById(Long id){
        return store.get(id);
    }
}
```

```java
package project.member;

public interface MemberService {
    void join(Member member);
    Member findMember(Long id);
}
```

```java
package project.member;

public class MemberServiceImpl implements MemberService{
    
    private MemberRepository memberRepository = new MemoryMemberRepository();
    
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

무료 강의를 들으면서도 작성했던코드가 많아서 쉽게 작성했다. 멤버 레파지토리는 요구사항에 아직 데이터베이스를 붙일지 외부에서 들어올지 모르는 상황이라해서 변경이 용이하도록 인터페이스를 쓰지만 서비스까지 인터페이스를 쓰는것은 의문이었다.

## 인터페이스도 비용이다.

모든 설계는 이상적으로 역할과 구현을 분리해야한다. 즉, 인터페이스를 다 만드는것이 좋다. 하지만 인터페이스를 만드는것 또한 비용이다. 기능을 확장할 가능성이 없다면, 구체 클래스를 직접 사용하고, 향후 꼭 필요할 때 리팩터링해서 인터페이스를 도입하는 것도 방법이다. 실제로 실무에서는 교체 가능성이 없는 서비스 클래스는 구체 클래스로 바로 만드는 것을 선호한다고 한다. 
