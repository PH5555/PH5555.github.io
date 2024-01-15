---
layout: post
title: Spring - 싱글톤 컨테이너
tags: [Spring]
comments: true
---

## 싱글톤 패턴이란?

웹 애플리케이션은 보통 많은 유저들이 한번에 요청을 한다.

![singleton1](/assets/img/sigleton.PNG)

만약 요청을 할 때마다 인스턴스를 새로 만들게 되면 많은 메모리 사용으로 인해 과부화가 올것이다. 어짜피 인스턴스의 내부는 다 똑같은 내용이기 때문에 여러개를 만들필요가 없기 때문에 하나의 인스턴스를 공유해서쓰는 `싱글톤 패턴`이 나오게 되었다.

```java
public class Singleton {
    private static final Singleton instance = new Singleton();
    
    public static Singleton getInstance(){
        return instance;
    }
    
    private Singleton(){
        
    }
    
    public void function(){
        //로직
    }
}

```

싱글톤 패턴은 다음과 같은 특징이 있다.

- Singleton 객체를 static으로 만듦으로써 하나의 인스턴스만을 생성하게 한다.
  
- 생성자를 private로 해서 개발자가 실수로 새로운 인스턴스를 만들수 없게 한다.

![singleton2](/assets/img/singleton2.PNG)

이런식으로 싱글톤 패턴을 적용하면 위의 그림과 같이 하나의 인스턴스를 공유하게 되어서 고객의 요청이 올때마다 인스터스를 생성하지 않게 된다.

하지만 싱글톤패턴을 위의 코드로 직접 구현하면 문제가 있다.

1. 코드 복잡성 (function이라는 로직만 써야할 코드에다가 다른 코드도 추가하게 됨)
2. 구현체에 의존하게 됨 -> DIP 위반
3. DIP를 위반 하므로 OCP도 위반할 확률이 높아짐
4. 유연성이 떨어진다.

`스프링 컨테이너`는 이런 단점을 다 보완하면서 싱글톤 패턴의 장점만을 살려 `싱글톤 컨테이너를 생성한다.`


  
