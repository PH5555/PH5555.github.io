---
layout: post
title: Spring - 커넥션 풀과 데이터소스
tags: [Spring, 데이터 접근 핵심 원리]
comments: true
---

## 커넥션 풀

데이터베이스 커넥션을 획득할 때는 복잡한 과정을 거쳐야한다.

1. DB드라이버를 통해 커넥션을 조회
2. DB드라이버와 DB간의 TCP/IP 연결
3. 연결이 완료되면 ID, PW같은 부가정보 전달
4. 내부 인증을 완료하고 DB세션을 생성
5. 커넥션이 생성완료되었다고 응답을 보내고 커넥션을 반환

고객이 애플리케이션을 실행할 때 SQL 실행 시간과 더불어서 데이터베이스 커넥션 시간까지 매번 사용해야한다.

이런 문제를 해결하기 위해서 커넥션을 미리 생성해두고 사용하는 커넥션풀을 사용한다.

![pool](/assets/img/connectionpool.PNG)

애플리케이션이 실행되는 시점에 필요한 만큼 커넥션을 풀에 보관한다. 커넥션 풀에 보관되어 있는 커넥션은 TCP/IP로 DB에 연결되어 있는 상태이기 때문에 언제든 꺼내쓸 수 있다.

1. 커넥션 요청이 들어오면 커넥션 풀에 들어있는 커넥션 중 하나를 반환한다.
2. 커넥션을 모두 사용하면 커넥션을 종료하는 것이 아니라 다음에 다시 사용할 수 있도록 커넥션 풀에 커넥션이 살아있는 채로 반환한다.

# 데이터소스

우리는 앞서서 데이터베이스를 연결할 때 DriverManager를 사용했다. 커넥션을 얻는 방법에는 여러가지가 있다. 이런 요구사항은 언제든 바뀔 수 있다. 만약 클라이언트와 DriverManger가 직접적인 의존관계를 가지고 있다면 Hikari로 바꾸려고 했을 때 
모든 코드를 수정해야 할 것이다.

![datasource](/assets/img/datasource.PNG)

이러한 문제를 해결하기 위해서 `DataSource`인터페이스를 제공한다. 구현체에 직접 의존하는 것이 아니라 인터페이스에 의존하도록 해서 유연하게 사용할 수 있도록 한다.

```java
@Test
void driver() throws SQLException {
    Connection connection1 = DriverManager.getConnection(URL, USERNAME, PASSWORD);
    Connection connection2 = DriverManager.getConnection(URL, USERNAME, PASSWORD);

    print(connection1);
    print(connection2);
}

@Test
void datasource() throws SQLException {
    DriverManagerDataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);
    Connection connection1 = dataSource.getConnection();
    Connection connection2 = dataSource.getConnection();

    print(connection1);
    print(connection2);
}

private static void print(Connection connection) {
    log.info("connection={} class={}", connection, connection.getClass());
}
```

DriverManager는 요청을 할 때마다 파라미터를 넘겨줘야하는 반면 DataSource는 처음 한번만 넘겨주면 된다.
