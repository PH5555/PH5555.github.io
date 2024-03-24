---
layout: post
title: Spring - 스프링 트랜잭션
tags: [Spring, 데이터 접근 핵심 원리]
comments: true
---

## 문제점들?

이전에 트랜잭션을 직접 구현했을 때 여러가지 문제점이 있었다.

- 서비스 계층에는 비지니스 로직만이 존재해야 유지보수하기가 펀한데, 트랜잭션과 관련된 코드가 많다. 현재의 트랜잭션 코드는 JDBC와 관련되어있기 때문에 유지보수 하기가 까다로워진다.
- 트랜잭션 코드는 try 부분에 commit, catch 부분에 rollback 을 하는 패턴이 반복된다.

이런 문제점들을 스프링에서 제공하는 기능을 통해 해결할 것이다.

## 트랜잭션 추상화

현재 트랜잭션은 JDBC에서 제공하는 기술로 짜여져있다. 이 코드를 JPA 기술로 바꾸고자 하면 모든 코드를 수정해야 할 것이다. 

![abs](/assets/img/tranaction_abs.PNG)

서비스는 특정 트랜잭션 기술에 직접 접근하지 않고 `TxManager`라는 추상화된 인터페이스에 접근한다. DI를 통해 원하는 구현체를 넣어주면된다.

> 스프링은 이런 트랜잭션 추상화 기술을 구현해두었다. 스프링에서 구현하는 `PlatformTranactionManager`를 사용하면 된다.

## 트랜잭션 동기화

스프링에서 제공하는 트랜잭션 매니저는 크게 두가지 역할을 한다.

- 트랜잭션 추상화  
- 리소스 동기화

트랜잭션을 유지하려면 처음부터 끝까지 같은 커넥션을 유지해야한다. 앞에서는 같은 커넥션을 유지하기 위해서 파라미터로 커넥션을 전달하는 방법을 사용했다. 파라미터로 전달하면 코드가 지저분해지고
커넥션을 넘기는 메서드와 넘기지 않는 메서드를 중복으로 만들어야한다는 단점이 있다.

![manager](/assets/img/tran_manager.PNG)

스프링은 트랜잭션 동기화 매니저를 사용한다. 이것은 쓰레드 로컬을 사용해서 커넥션을 동기화한다.

> 쓰레드 로컬은 해당 쓰레드만 접근할 수 있는 특별한 저장소이다. 해당 쓰레드만 해당 저장소에 접근할 수 있다.

1. 트랜잭션을 시작하려면 커넥션이 필요하다. 트랜잭션 매니저는 데이터소스를 통해 커넥션을 만들고 트랜잭션을 시작한다.
2. 트랜잭션 매니저는 트랜잭션이 시작된 커넥션을 트랜잭션 동기화 매니저에 보관한다.
3. 리포지토리는 트랜잭션 동기화 매니저에 보관된 커넥션을 꺼내서 사용한다. 따라서 파라미터로 커넥션을 전달하지 않아도 된다.
4. 트랜잭션이 종료되면 트랜잭션 매니저는 트랜잭션 동기화 매니저에 보관된 커넥션을 통해 트랜잭션을 종료하고, 커넥션도 닫는다.

```java
private void close(Connection connection, PreparedStatement pstm, ResultSet resultSet) {
    JdbcUtils.closeResultSet(resultSet);
    JdbcUtils.closeStatement(pstm);

    DataSourceUtils.releaseConnection(connection, dataSource);
}

private Connection getConnection() throws SQLException {
    Connection con = DataSourceUtils.getConnection(dataSource);
    return con;
}
```

- `DataSourceUtils.getConnection`을 하면 커넥션을 반환한다. 트랜잭션 동기화 매니저가 관리하는 커넥션이 있으면 그 커넥션을 사용하고, 아니면 새로운 커넥션을 생성해서 반환한다.
- 그냥 close를 통해서 닫으면 커넥션이 유지가 되지 않기 때문에 `DataSourceUtils.releaseConnection`으로 닫아준다.

```java
private final PlatformTransactionManager transactionManager;
private final MemberRepositoryV3 memberRepository;

public void accountTransfer(String fromId, String toId, int money) throws SQLException {
    TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());

    try{
        bizLogic(fromId, toId, money);
        transactionManager.commit(status);
    }catch (Exception e){
        transactionManager.rollback(status);
        throw new IllegalStateException();
    }

}

private void bizLogic(String fromId, String toId, int money) {
    Member fromMember = memberRepository.findById(fromId);
    Member toMember = memberRepository.findById(toId);

    memberRepository.update(fromId, fromMember.getMoney() - money);
    validation(fromMember);
    memberRepository.update(toId, toMember.getMoney() + money);
}

private static void validation(Member fromMember) {
    if(fromMember.getMemberId().equals("ex")){
        throw new IllegalStateException();
    }
}
```

- `transactionManager.getTransaction`을 이용해서 트랜잭션을 시작할 수 있다. status가 반환되는데 이 값을 이용해서 commit, rollback을 실행한다.
- `DefaultTransactionDefinition`은 트랜잭션과 관련된 기본 옵션이다.

```java
@BeforeEach
public void before(){
    DriverManagerDataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);
    memberRepository = new MemberRepositoryV3(dataSource);
    PlatformTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);
    memberService = new MemberServiceV3_1(transactionManager, memberRepository);

}
```

PlatformTransactionManager를 만들어서 레파지토리에 넘겨준다. 구현체는 JDBC를 사용하기 때문에 `DataSourceTransactionManager`를 사용한다.

## 트랜잭션 템플릿

트랜잭션을 사용하는 로직을 보면 try, catch가 반복되는것을 볼 수 있다. 이런 문제를 템플릿 콜백 패턴을 활용해서 해결할 수 있다. 스프링은 `TransactionTemplate`이라고 하는 템플릿 클래스를 제공한다.

```java
public class TransactionTemplate {
 private PlatformTransactionManager transactionManager;
 public <T> T execute(TransactionCallback<T> action){..}
 void executeWithoutResult(Consumer<TransactionStatus> action){..}
}
```

결과를 반환하는 execute 메서드가 있고, 반환하지 않는 executeWithoutResult가 있다.

```java
private final TransactionTemplate txTemplate;
private final MemberRepositoryV3 memberRepository;

public MemberServiceV3_2(PlatformTransactionManager manager, MemberRepositoryV3 memberRepository) {
    this.txTemplate = new TransactionTemplate(manager);
    this.memberRepository = memberRepository;
}

public void accountTransfer(String fromId, String toId, int money) throws SQLException {
    txTemplate.executeWithoutResult((status) -> {
        bizLogic(fromId, toId, money);
    });
}
```

이전과 달리 try, catch 로직이 다 없어진 것을 볼 수 있다. 

- `TransactionTemplate`을 사용하려면 트랜잭션 매니저가 필요하다.
- 트랜잭션 템플릿은 정상 실행되면 커밋을 하고, 언체크 예외가 발생하면 롤백한다(체크 예외의 경우에는 커밋한다.)

> 체크 예외는 RuntimeException을 상속받지 않는다. 따라서 복구가능성이 있다고 보고 예외 처리를 강제한다. 언체크 예외는 RuntimeException을 상속받으며 예외처리를 강제하지 않는다.

이렇게 트랜잭션을 진행하는 코드의 반복을 제거했다. 하지만 서비스 계층에서 트랜잭션을 처리하는 코드가 있다는 사실은 바뀌지 않는다.

## 트랜잭션 AOP

![proxy](/assets/img/tran_aop.PNG)

트랜잭션 프록시가 트랜잭션에 관련된 로직을 다 가져가고, 실제 서비스를 대신 호출해준다.

스프링에서 제공하는 AOP를 사용하면 편리하게 작성할 수 있다. `@Transactional` 을 붙여주기만 하면 된다.

```java
private final MemberRepositoryV3 memberRepository;

public MemberServiceV3_3(MemberRepositoryV3 memberRepository) {
    this.memberRepository = memberRepository;
}

@Transactional
public void accountTransfer(String fromId, String toId, int money) throws SQLException {
    bizLogic(fromId, toId, money);
}
```

- @Transactional코드를 클래스레벨에 붙여도 되고 메서드 레벨에 붙여도 된다. 클래스 레벨에 붙이면 해당 클래스의 public 메서드는 모두 트랜잭션을 사용하게 된다.

```java
@SpringBootTest
public class MemberServiceTestV3_3 {
    @Autowired
    private MemberRepositoryV3 memberRepository;

    @Autowired
    private MemberServiceV3_3 memberService;

    @TestConfiguration
    static class testConfig{
        @Bean
        public DataSource dataSource(){
            return new DriverManagerDataSource(URL, USERNAME, PASSWORD);
        }

        @Bean
        public PlatformTransactionManager transactionManager() {
            return new DataSourceTransactionManager(dataSource());
        }

        @Bean
        public MemberRepositoryV3 memberRepositoryV3() {
            return new MemberRepositoryV3(dataSource());
        }

        @Bean
        public MemberServiceV3_3 memberServiceV3(){
            return new MemberServiceV3_3(memberRepositoryV3());
        }
    }
    ....
}
```

- 스프링 AOP를 적용하려면 스프링 컨테이너가 필요하기 때문에 `SpringBootTest` 어노테이션이 있으면 해당 테스트를 실행할 때 스프링부트를 사용해 스프링컨테이너를 생성한다.
- `@TestConfiguration` 어노테이션을 적용하면 테스트 안에서 내부 설정 클래스를 만들 수 있다. 이 클래스 안에서 필요한 빈들을 등록할 수 있다.

> 트랜잭션 AOP는 스프링 빈에 등록된 트랜잭션 매니저를 찾아서 사용하기 때문에 빈으로 등록해주어야한다.

## 자동 리소스 등록

데이터 소스와 트랜잭션 매니저는 트랜잭션을 사용한다면 모든 사람들이 다 등록하여 사용할 것이다. 그래서 스프링에서는 이 두 빈을 자동으로 등록해준다.

이 때 스프링 부트는 `application.properties`에 있는 내용을 보고 데이터소스를 만든다.

```
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa
spring.datasource.password=
```

> 스프링부트에서 만드는 데이터소스는 기본적으로 `HikariDataSource`이다.

```java
@TestConfiguration
static class testConfig{
    private final DataSource dataSource;

    public testConfig(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Bean
    public MemberRepositoryV3 memberRepositoryV3() {
        return new MemberRepositoryV3(dataSource);
    }

    @Bean
    public MemberServiceV3_3 memberServiceV3(){
        return new MemberServiceV3_3(memberRepositoryV3());
    }
}
```

스프링 부트가 자동으로 생성해준 dataSource를 가져와서 사용할 수 있다.

> 스프링 부트에서 자동으로 생성해주는 DataSouce와 TransactionManager의 기본 이름은 dataSource, transactionManager이다.
