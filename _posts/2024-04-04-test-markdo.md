---
layout: post
title: Spring - 스프링 트랜잭션
tags: [Spring, 데이터 접근 핵심 원리]
comments: true
---

## 트랜잭션 프록시 작동 방식

![proxy](/assets/img/transaction_proxy.PNG)

`@Transaction`을 메서드카 늘래스에 붙이면 해당 객체는 트랜잭션 AOP 적용의 대상이 되고, 결과적으로 실제 객체 대신에 트랜잭션을 처리해주는 프록시
객체가 스프링 빈에 등록된다. 프록시는 실제 객체를 상속해서 만들어지기 때문에 다형성을 활용할수 있다. 따라서 `BasicService` 대신에 `BasicService$$CGLIB`
를 주입할 수 있다. 

> 객체 안에 있는 메서드가 호출되면 프록시는 해당 메서드가 트랜잭션을 사용할 수 있는지 확인하고 트랜잭션 로직을 실행한다.

## 트랜잭션 우선순위

```java
@SpringBootTest
public class TxLevelTest {
    @Autowired
    LevelService levelService;

    @Test
    void orderTest(){
        levelService.write();
        levelService.read();
    }

    @TestConfiguration
    static class LevelTestConfig{
        @Bean
        LevelService levelService(){
            return new LevelService();
        }
    }
    @Slf4j
    @Transactional(readOnly = true)
    static class LevelService{
        @Transactional(readOnly = false)
        public void write(){
            log.info("write");
            printTxInfo();
        }

        public void read(){
            log.info("call read");
            printTxInfo();

        }

        private void printTxInfo() {
            boolean actualTransactionActive = TransactionSynchronizationManager.isActualTransactionActive();
            log.info("active = {}", actualTransactionActive);
            boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
            log.info("readonly = {}", readOnly);
        }
    }
}

```

스프링에서는 항상 자세한것을 우선순위로 적용한다. `LevelService`에 `@Transactional(readOnly = true)`가 붙어있어서 해당 클래스의 모든
메서드가 해당 트랜잭션 옵션이 적용되어야한다. 하지만 write 메서드에 `@Transactional(readOnly = false)`가 붙어있으므로 해당 메서드는
다른 옵션을 가진다.

## 프록시 내부 호출

`@Transactional` 방식은 AOP를 통해서 트랜잭션을 실행하기 때문에 만약 프록시를 거치지 않고 객체를 직접 호출하게 되면
어노테이션이 붙어있더라도 트랜잭션이 실행되지 않는다.

```java
@Slf4j
static class CallService{
    public void external(){
        log.info("call external");
        printTxInfo();
        internal();
    }

    @Transactional
    public void internal(){
        log.info("call internal");
        printTxInfo();
    }
    private void printTxInfo() {
        boolean actualTransactionActive = TransactionSynchronizationManager.isActualTransactionActive();
        log.info("active = {}", actualTransactionActive);
        boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        log.info("readonly = {}", readOnly);
    }
}
```

external 메서드에는 `@Transactional`이 없다. 따라서 트랜잭션 프록시는 트랜잭션을 적용하지 않는다. 트랜잭션을 적용하지 않고
메서드를 실행하게 되는데, external() 내부에서 internal() 메서드를 호출하면 프록시를 거치지 않아서 트랜잭션이 적용되지 않는다.

> 자바 언어에서 메서드 앞에 별도의 참조가 없으면 `this`가 붙는다. 여기서 `this`는 자기 자신을 가리키므로 프록시를 거치지 않고
> 객체에서 바로 메서드를 실행한다.

![internal](/assets/img/proxy_internal.PNG)

이를 해결하기 위해서는 메서드를 분리하면 된다.

```java
@Slf4j
@RequiredArgsConstructor
static class CallService{
    private final InternalService internalService;
    public void external(){
        log.info("call external");
        printTxInfo();
        internalService.internal();
    }

    private void printTxInfo() {
        boolean actualTransactionActive = TransactionSynchronizationManager.isActualTransactionActive();
        log.info("active = {}", actualTransactionActive);
        boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        log.info("readonly = {}", readOnly);
    }
}

static class InternalService{
    @Transactional
    public void internal(){
        log.info("call internal");
        printTxInfo();
    }

    private void printTxInfo() {
        boolean actualTransactionActive = TransactionSynchronizationManager.isActualTransactionActive();
        log.info("active = {}", actualTransactionActive);
        boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        log.info("readonly = {}", readOnly);
    }
}
```

callService에는 트랜잭션 관련 코드가 아예 없으므로 프록시가 적용되지 않는다. InternalService에는 트랜잭션 관련
코드가 있으므로 트랜잭션 프록시가 적용된다. 주입받은 InternalService가 프록시이기 때문에 트랜잭션이 적용된다.

![ex](/assets/img/proxy_ex.PNG)

## 트랜잭션 적용 시점

```java
@Slf4j
static class Hello{
    @PostConstruct
    @Transactional
    public void initV1(){
        boolean actualTransactionActive = TransactionSynchronizationManager.isActualTransactionActive();
        log.info("hello init @Post tx active={}", actualTransactionActive);
    }

    @EventListener(ApplicationReadyEvent.class)
    @Transactional
    public void initV2(){
        boolean actualTransactionActive = TransactionSynchronizationManager.isActualTransactionActive();
        log.info("hello init ApplicationReadyEvent tx active={}", actualTransactionActive);
    }
}
```

초기화코드 `@PostConstruct`를 `@Transactional`와 함께 사용하면 트랜잭션이 적용되지 않는다. 왜냐하면 초기화 코드가
먼저 실행되고 그 다음에 트랜잭션 AOP가 적용되기 때문이다.

`@EventListener(ApplicationReadyEvent.class)`를 사용하면 스프링 컨테이너가 완전히 생성되고 이벤트가 붙은 메서드를
호출해준다. 따라서 이 메서드는 트랜잭션이 적용된다.

