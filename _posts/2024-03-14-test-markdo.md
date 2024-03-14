---
layout: post
title: Spring - JDBC 이해
tags: [Spring, 데이터 접근 핵심 원리]
comments: true
---

## JDBC 등장이유

우리가 웹이나 모바일 앱을 사용할 때 중요한 데이터를 저장할 때 데이터베이스에 저장한다. 이 때 클라이언트에서 바로 데이터베이스에 접근하지 않고, 서버를 통해서 접근한다.

![stru](/assets/img/database_stru.PNG)

데이터베이스와의 연결은 주로 TCP/IP를 사용해서 연결되고, DB가 이해할 수 있는 sql문을 통해 데이터를 받아온다. 
근데 문제는 각각의 데이터베이스마다 사용법이 다르다는 것이다. 기본 기능은 거의 같지만 좀만 깊게 들어가면 문법이 다 달라진다. 그래서 데이터베이스를 바꾸려고 하면
모든 코드를 다 수정해야하는 불상사가 발생한다.

![jdbc](/assets/img/jdbc.PNG)

그래서 JDBC 표준 인터페이스로 구현을 하면 된다. 물론 인터페이스만 구현한다고 해결되지 않는다. 데이터베이스의 회사에서 JDBC 인터페이스에 맞게 개발을 하는데 이걸 JDBC 드라이버라고 한다.
이런 드라이버를 연결해서 사용해야한다. 그러면 서버 코드가 인터페이스에만 의존하기 때문에 모든 코드를 수정하지 않아도 된다.

## 데이터베이스 연결

테스트용 서버를 개발할 때 편리하게 사용할 수 있는 `H2` 데이터베이스를 사용한다. H2 데이터베이스를 실행하고 테이블을 미리 만들어두어야하는데 사지방 컴퓨터에서 이걸 막아놔서 `H2인메모리`를 사용했다.

```
spring:
  application:
    name: jdbc

  h2:
    console:
      enabled: true
      path: /h2-console

  datasource:
    driver-class-name: org.h2.Driver
    url: jdbc:h2:mem:test
    username: sa
    password:

  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
```

application.yml 파일에서 인메모리 데이터베이스를 설정할 수 있다. `datasource`를 통해서 데이터베이스 이름과 유저네임등을 설정할 수 있다.

```java
@TestPropertySource(locations = "classpath:application-test.yml")
class MemberRepositoryV0Test {

    @BeforeEach
    void setUp() throws SQLException {
        String sql = "create TABLE Member (member_id VARCHAR(255), money INt)";
        Connection con = DBConnectionUtil.getConnection();

        Statement statement = con.createStatement();
        statement.executeUpdate(sql);

    }
    private MemberRepositoryV0 repository = new MemberRepositoryV0();
}
```

인메모리 데이터베이스는 서버를 껐다 키면 초기화 되기 때문에 test부분에 필요한 테이블을 추가시켜주었다.

```java
@Slf4j
public class DBConnectionUtil {
    public static Connection getConnection() {
        try {
            Connection connection = DriverManager.getConnection(URL, USERNAME, PASSWORD);
            log.info("get connection={}, class={}", connection, connection.getClass());
            return connection;
        } catch (SQLException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

다시 본론으로 돌아와서 데이터베이스를 연결하려면 `DriverManager.getConnection`을 사용해서 Connect객체를 얻으면 된다. 이렇게 하면 라이브러리에 있는 데이터베이스 드라이브를 찾아서 해당 드라이버가 제공하는 커넥션을 준다.

```java
public class DBConnectionUtilTest {

    @Test
    void connection(){
        Connection connection = DBConnectionUtil.getConnection();
        Assertions.assertThat(connection).isNotNull();
    }
}
```

간단하게 테스트를 만들어봐서 실행해보았다.

![connection](/assets/img/connection.PNG)

만약 라이브러리에 드라이버가 여러개 있으면 각각의 드라이버에 url, 이름, 비밀번호등 접속에 필요한 정보를 넘겨주어서 본인이 처리할 수 있는 요청인지 확인해서 처리할 수 있으면 그 반환값을 넘겨준다.

## 회원 등록

```java
@Data
public class Member {
    private String memberId;
    private int money;

    public Member(){

    }

    public Member(String memberId, int money) {
        this.memberId = memberId;
        this.money = money;
    }
}

```

먼저 멤버를 저장할 때 쓸 객체를 만들어준다.

```java
public Member save(Member member){
        String sql = "insert into member(member_id, money) values (?, ?)";

        Connection con = null;
        PreparedStatement pstm = null;

        try {
            con = getConnection();
            pstm = con.prepareStatement(sql);
            pstm.setString(1, member.getMemberId());
            pstm.setInt(2, member.getMoney());
            pstm.executeUpdate();
            return member;
        } catch (SQLException e) {
            throw new RuntimeException(e);
        } finally {
            close(con, pstm, null);
        }
}
```

테이블에 데이터를 넣을 때 필요한 sql 문을 만들고 preparedStatement에 sql 문을 넣어서 필요한 파라미터들을 넣어주고 실행한다.

> 인덱스가 1부터 시작하는거에 유의한다.

```java
private void close(Connection connection, PreparedStatement pstm, ResultSet resultSet) {
        if(resultSet != null){
            try {
                resultSet.close();
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        }

        if(pstm != null){
            try {
                pstm.close();
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        }

        if(connection != null){
            try {
                connection.close();
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        }
}
```

connection객체와 statement객체, resultSet 객체는 사용하고 나서 항상 종료해주어야한다. 만든 순서의 반대로 종료시킨다.

## 멤버 조회

```java
public Member findById(String id){
        String sql = "Select * from Member where member_id = ?";
        Connection con = null;
        PreparedStatement pstm = null;
        ResultSet rs = null;

        try{
            con = getConnection();
            pstm = con.prepareStatement(sql);
            pstm.setString(1, id);
            rs = pstm.executeQuery();

            if(rs.next()){
                String memberId = rs.getString("member_id");
                int money = rs.getInt("money");
                Member member = new Member();
                member.setMemberId(memberId);
                member.setMoney(money);

                return member;
            }
            else{
                throw new NoSuchElementException("member not found");
            }
        }catch (SQLException e) {
            throw new RuntimeException(e);
        }finally {
            close(con, pstm, rs);
        }
}
```

크게 달라진것은 없지만 select문은 결과값을 반환하기 때문에 `executeQuery`메서드를 사용한다. 이 메서드의 반환값은 resultSet이다. 이 객체를 통해서 결과값을 조회할 수 있다.

![result](/assets/img/resultset.PNG)

resultSet은 다음과 같이 생긴 데이터구조이다. 보통 select문 결과의 순서대로 들어간다. 내부에 있는 cursor를 이동시켜서 데이터를 조회할 수 있다. cusrost는 `next()`를 이용해서 다음으로 이동시킬 수 있다.

> 초기단계에는 cursor가 아무것도 가리키기 않기 때문에 next()를 실행해야한다.

## 수정, 삭제

```java
public void update(String memberId, int money) {
      String sql = "update member set money=? where member_id=?";

      Connection con = null;
      PreparedStatement pstm = null;
      ResultSet rs = null;

      try{
          con = getConnection();
          pstm = con.prepareStatement(sql);
          pstm.setInt(1, money);
          pstm.setString(2, memberId);
          int resultSize = pstm.executeUpdate();
          log.info("result size = {}", resultSize);

      }catch (SQLException e) {
          throw new RuntimeException(e);
      }finally {
          close(con, pstm, rs);
      }
  }

  public void delete(String memberId) {
      String sql = "delete from Member where member_id = ?";

      Connection con = null;
      PreparedStatement pstm = null;
      ResultSet rs = null;

      try{
          con = getConnection();
          pstm = con.prepareStatement(sql);
          pstm.setString(1, memberId);
          pstm.executeUpdate();
      }catch (SQLException e) {
          throw new RuntimeException(e);
      }finally {
          close(con, pstm, rs);
      }
  }
}
```

수정과 삭제도 특별한 것 없이 sql만 바꿔서 실행해주면 된다.

```java
@Test
void saveTest(){
    Member member = new Member("asdf", 10000);
    repository.save(member);
    Member findMember = repository.findById(member.getMemberId());
    assertThat(findMember).isEqualTo(member);

    repository.update(member.getMemberId(), 20000);
    Member updateMember = repository.findById(member.getMemberId());
    assertThat(updateMember.getMoney()).isEqualTo(20000);

    repository.delete(member.getMemberId());
    assertThatThrownBy(()->repository.findById(member.getMemberId())).isInstanceOf(NoSuchElementException.class);
}
```

테스트 코드를 돌려서 확인한다. 처음에 만든 member 객체와 findbyid로 찾은 멤버 객체는 서로 다른(주소가 다름)객체이다. 하지만 우리가 Member객체를 생성할 때 `@Data`를 붙였다. @Data는 자동으로 equal 메서드를 재정의 해주기 때문에
안에 있는 값이 다 같으면 true가 된다.

delete같은 경우에는 멤버가 사라지기 때문에 값을 비교하여 검증할 수 없다. `assertThatThrownBy`를 통해 에러 내용을 검증하면 된다.

> JDBC 코드를 쓰다보면 겹치는 부분이 많다는 것을 느낄 것이다. 이러한 부분을 개선해서 jdbcTemplate이나 jpa가 나온것이다.
