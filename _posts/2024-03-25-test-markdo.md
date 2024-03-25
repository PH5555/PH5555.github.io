---
layout: post
title: Spring - 자바 예외
tags: [Spring, 데이터 접근 핵심 원리]
comments: true
---

## 예외계층

![exception](/assets/img/exception.PNG)

- 자바의 모든 객체의 조상은 `Object`이다.
- `Error`는 메모리 부족이나 심각한 시스템 오류같이 애플리케이션에서 복구가 불가능한 예외이다. 개발자는 이 예외를 잡으려고 하면 안된다.
- `Exception`은 체크 예외이다. Exception과 그 하위 예외는 모두 컴파일러가 체크하는 체크예외이다. 여기에서 RuntimeException은 예외로 한다.
- `RuntimeException`은 컴파일러가 체크하지 않는 언체크예외이다.

## 예외 기본 규칙

예외는 폭탄돌리기와 같다. 잡아서 처리하거나, 잡을 수 없다면 밖으로 던져야한다.

![exception_catch](/assets/img/exception_catch.PNG)

5번에서 예외를 처리하면 그 이후로 애플리케이션 로직이 정상 흐름으로 바뀐다. 하지만 예외를 처리하지 못하면 호출한 곳으로 계속 예외를 던지게 된다.

> 예외를 처리하지 못하고 자바 main 쓰레드까지 전달됐을 경우 예외 로그를 출력하면서 시스템이 종료된다. 웹애플리케이션에서는 종료가 되면 안되기 때문에 WAS가 지정된 오류 페이지를 보여준다.

## 체크 예외 처리

```java
public class CheckedTest {
    @Test
    void call_test() {
        Service service = new Service();
        service.callCatch();
    }

    @Test
    void call_throw() throws MyCheckedException {
        Service service = new Service();
        Assertions.assertThatThrownBy(()-> service.callThrow()).isInstanceOf(MyCheckedException.class);
    }

    static class MyCheckedException extends Exception {
        public MyCheckedException(String message) {
            super(message);
        }
    }

    static class Service {
        Repository repository = new Repository();

        public void callCatch() {
            try {
                repository.call();
            } catch (MyCheckedException e) {
                log.info(e.getMessage());
            }
        }

        public void callThrow() throws MyCheckedException {
            repository.call();
        }
    }

    static class Repository {
        public void call() throws MyCheckedException {
            throw new MyCheckedException("hy");
        }
    }
}
```

- `Exception`을 상속받은 `MyCheckedException` 이라는 예외를 만든다. 이 예외는 체크예외가 된다.
- 레파지토리에서 call 메서드를 호출하면 MyCheckedException에러를 던진다.
- 서비스에서 레파지토리의 메서드를 호출한다.

callCatch 메서드를 실행할 경우 해당 메서드에서 이미 예외를 잡았기 때문에 호출하는 쪽에서 처리할 필요가 없다. 하지만 callThrow 같은 경우에는 예외를 던지고 있기 때문에 실행하는 쪽에서 잡아주거나 던져주어야 한다.

- 장점

개발자가 실수로 예외를 누락하지 않도록해주는 안전장치가 될 수 있다.

- 단점

개발자가 모든 예외를 처리하도록 하기 때문에 너무 번거로운 일이된다.

## 언체크 예외 처리

```java
public class UnCheckedTest {
    @Test
    void call_test() {
        Service service = new Service();
        service.callCatch();
    }

    @Test
    void call_throw() {
        Service service = new Service();
        Assertions.assertThatThrownBy(()-> service.callThrow()).isInstanceOf(MyUnCheckedException.class);
    }

    static class MyUnCheckedException extends RuntimeException {
        public MyUnCheckedException(String message) {
            super(message);
        }
    }

    static class Service {
        Repository repository = new Repository();

        public void callCatch() {
            try {
                repository.call();
            } catch (MyUnCheckedException e) {
                log.info(e.getMessage());
            }
        }

        public void callThrow() throws MyUnCheckedException {
            repository.call();
        }
    }

    static class Repository {
        public void call() {
            throw new MyUnCheckedException("hi");
        }
    }
}
```

체크 예외와 달리 예외를 잡거나 던지는 것이 강제되지 않는다. 따라서 개발자는 `throws MyUnCheckedException`를 적어도 되고 적지 않아도 된다.

- 장점

신경 쓰고 싶지 않은 예외의 의존관계를 참조하지 않아도 된다.

- 단점

개발자가 실수로 예외를 누락할 수 있다.

## 예외처리 방법

말만들어서는 체크예외를 더 많이 사용할 것 같지만 실제로는 언체크예외를 훨씬 많이 쓴다.

기본적으로는 언체크 예외를 사용하고 심각한 예외 문제라 반드시 처리해야하는 경우에만 체크 예외를 사용한다. 예를 들어서 계좌이체 실패 예외, 로그인 실패 예외 등이 있다.

```java
public class CheckedAppTest {

    @Test
    void test(){
        Controller controller = new Controller();
        Assertions.assertThatThrownBy(()->controller.call()).isInstanceOf(Exception.class);
    }
    static class Controller{
        Service service = new Service();

        public void call() throws SQLException, ConnectException {
            service.call();
        }
    }
    static class Service {
        Repository repository = new Repository();
        NetworkClient networkClient = new NetworkClient();

        public void call() throws ConnectException, SQLException {
            repository.call();
            networkClient.call();
        }
    }

    static class NetworkClient{
        public void call() throws ConnectException {
            throw new ConnectException();
        }
    }

    static class Repository{
        public void call() throws SQLException {
            throw new SQLException();
        }
    }
}
```

만약 체크예외를 사용한다면 예외를 다루는 부분에서 `throws SQLException, ConnectException`을 다 적어줘야한다. 사실 SQLException과 ConnectException 과 같은 에러는 애플리케이션에서 처리를 할 수 없다. 서비스, 레파지토리 모두 
이 에러를 해결할 수 없기 때문에 모두 `throws SQLException, ConnectException`을 적어서 예외를 던져야한다.

그리고 의존관계에 대한 문제도 발생한다. SQLException 은 JDBC에 관련된 예외인데 만약 JPA로 바꾼다고 하면 해당 예외를 다 바꿔줘야할것이다.

> 최상위 예외인 Exception을 던져서 의존관계에 대한 문제를 해결할 수 있지만 이렇게 되면 중요한 예외를 놓치게 되는 단점이 있다.

## 언체크 예외 처리

```java
public class UnCheckedAppTest {

    @Test
    void test(){
        Controller controller = new Controller();
        try{
            controller.call();
        }catch (Exception e){
            log.info("ex", e);
        }
//        Assertions.assertThatThrownBy(()->controller.call()).isInstanceOf(RuntimeException.class);
    }
    static class Controller{
        Service service = new Service();

        public void call() {
            service.call();
        }
    }
    static class Service {
        Repository repository = new Repository();
        NetworkClient networkClient = new NetworkClient();

        public void call() {
            repository.call();
            networkClient.call();
        }
    }

    static class NetworkClient{
        public void call() {
            throw new RuntimeConnectException("runtime connect");
        }
    }

    static class Repository{
        public void call() {
            try {
                runSql();
            } catch (SQLException e) {
                throw new RuntimeSqlException(e);
            }
        }

        public void runSql() throws SQLException {
            throw new SQLException("runtime sql");
        }
    }

    static class RuntimeConnectException extends RuntimeException{
        public RuntimeConnectException(String message) {
            super(message);
        }
    }

    static class RuntimeSqlException extends RuntimeException {
        public RuntimeSqlException(Throwable cause) {
            super(cause);
        }
    }
}
```

만약 체크예외가 발생하면 새로 정의한 RuntimeConnectException, RuntimeSqlException예외로 전환을 한다. 이렇게 하면 체크예외에서 언체크예외로 바뀌게 된다.

어차피 컨트롤러, 서비스에서 해당 예외를 처리할 수 없기 때문에 무시해도 된다. 이전 예시처럼 `throws SQLException, ConnectException` 부분이 없어진것을 볼 수 있다.

> 추가로 런타임 에러는 놓칠 수 있기 때문에 문서화를 잘 해두는것이 중요하다.

## 스택 트레이스

```java
public void call() {
    try {
        runSql();
    } catch (SQLException e) {
        throw new RuntimeSqlException(e);
    }
}
```

우리가 예외를 전환할 때 기존에러를 넘긴것을 볼 수 있다. 만약에 넘기지 않는다면 에러가 난 부분을 정확히 찾을 수 없게 된다.
