---
layout: post
title: Spring - 웹애플리케이션의 이해
tags: [Spring, 스프링 MVC]
comments: true
---

## 웹서버와 웹애플리케이션서버의 차이

- 웹서버

웹서버는 정적리소스를 제공한다. 이 때 정적리소스란 html, css, js, 이미지, 영상같은 것들이다. `정적`이기 때문에 데이터에 따라 보여지는 화면이 다르게 하는 동적 화면을 보여주지 못한다.
  
- 웹애플리케이션 서버

웹애플리케이션 서버는 정적리소스에 프로그램 코드를 실행해서 서비스로직을 실행할 수 있게 해준다. 즉 동적 화면을 만들 수 있다.

![was](/assets/img/was.PNG)

즉, 웹애플리케이션 서버는 애플리케이션 코드, html, 이미지, 비디오등을 모두 처리하게 되는데 그러면 WAS가 너무 많은 역할을 담당하게 된다. 그렇게 되면 시스템 과부하가 올 확률이 높아진다.

![was](/assets/img/was_new.PNG)

그래서 정적 리소스는 웹서버가 담당하고 동적 화면을 요청할 일이 생기면 WAS에게 일을 위임하는 방식으로 서버를 설계한다. 이렇게 하면 부분적으로 서버를 확장할 때도 편하다.

## 서블릿

HTTP 요청이 들어와서 요청을 처리한다고 해보자 먼저 서버가 요청을 받을 수 있도록 TCP/IP연결을 대기해야할 것이다. 그리고 요청이 들어오면 파싱해서 읽고 타입을 확인하고 메시지를 파싱하고... 비지니스 로직을 실행하기 전 단계가 너무 많다. 이런과정을 직접 하드코딩하면 겁나 힘이든다. 이런 과정을 편리하게 해주는 것이 서블릿이다.

![was](/assets/img/servlet.PNG)

이 때 서블릿 컨테이너는 싱글톤이다. 최초 로딩에 서블릿 객체를 생성해두고 재사용하기 때문에 공유변수 사용에 주의해야한다. 

## 멀티 스레드

서블릿 객체는 스레드가 실행해준다. 한 스레드가 한 요청을 처리하고 있다가 처리가 지연될 때 새로운 요청이 들어오면 다른 스레드를 새로 생성하여 요청에 지연이 없도록 한다. 이렇게 요청마다 스레드를 생성하면 동시성을 높을 수 있고 CPU 사용율을 높일 수 있지만 CPU최대 사용률을 넘으면 과부하로 서버가 다운될 수 잇다. 그리고 스레드를 바꿀 때마다 컨텍스트스위칭 비용이 발생한다. 

그래서 스레드갯수를 정해놓고 사용할 수 있는 `스레드풀`을 만든다. 예를 들어 스레드풀을 200으로 설정하면 요청이 200개까지는 동시처리할 수 있고 201번째 요청은 처리지연되거나 처리거부를 할 수 있다. 

> 최대 쓰레드 개수를 정하는 기준은 그저 감이다. 감으로 이만큼 사용할거 같다를 생각해두고 모니터링을 계속해서 튜닝하는 수밖에 없다.

이런 멀티 스레드에 관련된 부분은 WAS에서 처리를 해준다. 따라서 개발자가 해당 부분을 신경쓰지않고 개발할 수 있다. 멀티쓰레딩이 일어나기 때문에 공유변수 사용에 유의해야한다.