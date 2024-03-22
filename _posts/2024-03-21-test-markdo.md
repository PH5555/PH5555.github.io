---
layout: post
title: Spring - 트랜잭션
tags: [Spring, 데이터 접근 핵심 원리]
comments: true
---

## 트랜잭션 이해

우리가 데이터를 저장할 때 그냥 파일에 저장해도 되는데 데이터베이스에 저장하는 이유중 하나는 트랜잭션 때문이다. 우리가 계좌 이체 서비스를 만든다고 해보자. A이용자가 B이용자에게 2000원을 송금하면 
A이용자 계좌에서 돈이 2000원 빠져나가야하고, B이용자 계좌에서 돈이 2000원 추가 되어야한다. 이렇게 서비스의 일련의 과정을 트랜잭션이라고 하고, 데이터베이스는 이런 트랜잭션이 안정적으로 처리되도록 도와준다.
만약 에러가 발생하면 트랜잭션을 실행하기 전으로 돌아가는 `롤백`을 실행하고 정상적으로 실행되면 최종적으로 데이터를 저장하는 `커밋`을 실행한다.

## 트랜잭션 ACID

- 원자성: 트랜잭션 내에서 실행한 작업들은 마치 하나의 작업인 것처럼 모두 성공 하거나 모두 실패해야 한다.
- 일관성: 모든 트랜잭션은 일관성 있는 데이터베이스 상태를 유지해야 한다. 예를 들어 데이터베이스에서 정한 무
결성 제약 조건을 항상 만족해야 한다.
- 격리성: 동시에 실행되는 트랜잭션들이 서로에게 영향을 미치지 않도록 격리한다. 예를 들어 동시에 같은 데이터
를 수정하지 못하도록 해야 한다. 격리성은 동시성과 관련된 성능 이슈로 인해 트랜잭션 격리 수준(Isolation
level)을 선택할 수 있다.
- 지속성: 트랜잭션을 성공적으로 끝내면 그 결과가 항상 기록되어야 한다. 중간에 시스템에 문제가 발생해도 데이
터베이스 로그 등을 사용해서 성공한 트랜잭션 내용을 복구해야 한다.

트랜잭션은 원자성, 일관성, 지속성을 보장한다. 문제는 격리성이다. 격리성을 완전히 지키려면 트랜잭션을 차례대로 실행되어야하는데 그렇게 되면 성능이 매우 떨어질 것이다. 그래서 ANSI에서는 트랜잭션의 격리수준을 4단계로 나눈다.

- READ UNCOMMITED(커밋되지 않은 읽기)
- READ COMMITTED(커밋된 읽기)
- REPEATABLE READ(반복 가능한 읽기)
- SERIALIZABLE(직렬화 가능)

## 데이터베이스 연결 구조

![con](/assets/img/connection_session.PNG)

- 사용자는 웹 애플리케이션 서버(WAS)나 DB 접근 툴 같은 클라이언트를 사용해서 데이터베이스 서버에 접근할
수 있다. 클라이언트는 데이터베이스 서버에 연결을 요청하고 커넥션을 맺게 된다. 이때 데이터베이스 서버는 내
부에 세션이라는 것을 만든다. 그리고 앞으로 해당 커넥션을 통한 모든 요청은 이 세션을 통해서 실행하게 된다.

- 세션은 트랜잭션을 시작하고, 커밋 또는 롤백을 통해 트랜잭션을 종료한다. 그리고 이후에 새로운 트랜잭션을 다
시 시작할 수 있다.

## 트랜잭션

![transaction](/assets/img/tranaction.PNG)

데이터를 변경하고 그 변경 데이터를 최종적으로 저장하고 싶으면 `commit`을 실행하고 되돌리고 싶으면 `rollback`을 실행하면 된다. 이 때 커밋을 실행하기 전까지는 데이터를 임시로 보관한다.

그림처럼 한 사용자가 데이터를 변경하면 해당 세션에서만 select를 써서 조회를 할 수 있다. 

> 다른 세션에서도 해당 데이터를 접근할 수 있으면 데이터 정합성에 문제가 생긴다.

commit을 실행하면 다른 세션에서도 해당 데이터를 조회할 수 있다.

> 트랜잭션을 실행하기 전에 `Auto commit`을 false로 해주어야한다. 기본값은 true이다.

## DB락

세션1이 커밋을 완료하기 전에 세션2가 동일 데이터를 수정하게 되면 원자성이 깨지게 된다. 세션1이 중간에 데이터를 롤백하게 되면 세션2는 잘못된 데이터를 얻게 된다. 이런 문제를 해결하려면 세션이 트랜잭션을 시작하고 커밋이나 롤백을 완료하기 전에는 해당 데이터를 수정할 수 없도록 막아야한다.

![lock](/assets/img/dblock.PNG)

데이터를 수정하고 싶으면 락을 획득해야하고, 만약 락이 없으면 락을 얻을 때까지 대기해야한다.

> 락을 얻을 때까지 무한정 대기하지는 않는다. 락 대기 시간을 설정해두고 대기 시간이 넘어가면 타임아웃 오류가 발생한다.

> 일반적으로 기본적인 조회는 락을 획득하지 않아도 조회를 할 수 있다. 조회를 할 때도 락을 획득하게 하고 싶으면 `select for update` 구문을 이용하면 된다.

## 트랜잭션 적용전

```java
@RequiredArgsConstructor
public class MemberServiceV1 {
  private final MemberRepositoryV1 memberRepository;

  public void accountTransfer(String fromId, String toId, int money) throws SQLException {
    Member fromMember = memberRepository.findById(fromId);
    Member toMember = memberRepository.findById(toId);
    memberRepository.update(fromId, fromMember.getMoney() - money);
    validation(toMember);
    memberRepository.update(toId, toMember.getMoney() + money);
  }

  private void validation(Member toMember) {
    if (toMember.getMemberId().equals("ex")) {
      throw new IllegalStateException("이체중 예외 발생");
    }
  }
}
```

먼저 트랜잭션을 사용하지 않고 일반적으로 계좌이체를 하는 로직을 짰다.

```java
@Test
@DisplayName("정상 이체")
void accountTransfer() throws SQLException {

  Member memberA = new Member(MEMBER_A, 10000);
  Member memberB = new Member(MEMBER_B, 10000);
  memberRepository.save(memberA);
  memberRepository.save(memberB);

  memberService.accountTransfer(memberA.getMemberId(),
  memberB.getMemberId(), 2000);

  Member findMemberA = memberRepository.findById(memberA.getMemberId());
  Member findMemberB = memberRepository.findById(memberB.getMemberId());
  assertThat(findMemberA.getMoney()).isEqualTo(8000);
  assertThat(findMemberB.getMoney()).isEqualTo(12000);
}

@Test
@DisplayName("이체중 예외 발생")
void accountTransferEx() throws SQLException {

  Member memberA = new Member(MEMBER_A, 10000);
  Member memberEx = new Member(MEMBER_EX, 10000);
  memberRepository.save(memberA);
  memberRepository.save(memberEx);

  assertThatThrownBy(() -> memberService.accountTransfer(memberA.getMemberId(), memberEx.getMemberId(), 2000))
    .isInstanceOf(IllegalStateException.class);
 
  Member findMemberA = memberRepository.findById(memberA.getMemberId());
  Member findMemberEx = memberRepository.findById(memberEx.getMemberId());

  assertThat(findMemberA.getMoney()).isEqualTo(8000);
  assertThat(findMemberEx.getMoney()).isEqualTo(10000);
  }
}
```

정상적인 데이터를 입력했을 때는 우리가 원하는 대로 출력이 된다. 하지만 중간에 에러를 발생시켜보면 한쪽의 돈만 줄어들고 다른 쪽의 돈은 늘어나지 않는다.
계좌 이체 과정이 하나로 관리되지 않고 다 따로따로 관리되기 때문이다.

## 트랜잭션 적용

트랜잭션을 이용해서 앞선 문제를 해결할 것이다. 애플리케이션의 어떤 계층에 트랜잭션을 걸어야할까?

![transaction_structure](/assets/img/transaction_structure.PNG)

- 트랜잭션은 서비스 로직이 시작되는 서비스계층에서 시작되어서 끝나야한다. 비지니스 로직이 잘못되면 해당 비지니스 로직을 전부 롤백시켜야한다. 
- 트랜잭션을 사용하려면 커넥션이 필요하다. 서비스 계층에서 커넥션을 만들고 트랜잭션 커밋이후에 커넥션을 종료해야한다.
- 트랜잭션을 사용하려면 같은 커넥션을 유지해야한다.

트랜잭션에서 같은 커넥션을 유지하려면 제일 쉬운 방법은 커넥션을 파라미터로 전달하는 것이다.

```java
public Member findById(Connection con, String memberId) throws SQLException {
  String sql = "select * from member where member_id = ?";
  PreparedStatement pstmt = null;
  ResultSet rs = null;

  try {
    pstmt = con.prepareStatement(sql);
    pstmt.setString(1, memberId);
    rs = pstmt.executeQuery();

    if (rs.next()) {
    Member member = new Member();
    member.setMemberId(rs.getString("member_id"));
    member.setMoney(rs.getInt("money"));
    return member;
    } else {
      throw new NoSuchElementException("member not found memberId=" + memberId);
    }
  } catch (SQLException e) {
    log.error("db error", e);
    throw e;
  } finally {
    JdbcUtils.closeResultSet(rs);
    JdbcUtils.closeStatement(pstmt);
  }
}
```

- 서비스 로직에서 사용하는 findById와 update메서드는 커넥션을 유지해야하기 때문에 `con = getConnection()` 을 사용하면 안되고 파라미터의 커넥션을 사용해야한다.
- finally에서 커넥션은 종료하면 안된다. 해당 커넥션을 서비스에서 계속 사용해야하기 때문이다.

```java
public class MemberServiceV2 {
  private final DataSource dataSource;
  private final MemberRepositoryV2 memberRepository;

  public void accountTransfer(String fromId, String toId, int money) throws SQLException {
    Connection con = dataSource.getConnection();
    try {
      con.setAutoCommit(false);
      bizLogic(con, fromId, toId, money);
      con.commit();
    } catch (Exception e) {
      con.rollback();
      throw new IllegalStateException(e);
    } finally {
      release(con);
    }
  }

  private void bizLogic(Connection con, String fromId, String toId, int money) throws SQLException {
    Member fromMember = memberRepository.findById(con, fromId);
    Member toMember = memberRepository.findById(con, toId);
    memberRepository.update(con, fromId, fromMember.getMoney() - money);
    validation(toMember);
    memberRepository.update(con, toId, toMember.getMoney() + money);
  }

  private void validation(Member toMember) {
    if (toMember.getMemberId().equals("ex")) {
      throw new IllegalStateException("이체중 예외 발생");
    }
  }

  private void release(Connection con) {
    if (con != null) {
      try {
        con.setAutoCommit(true);
        con.close();
      } catch (Exception e) {
        log.info("error", e);
      }
    }
  }
}
```

- 트랜잭션을 시작하려면 자동커밋을 꺼야한다. `con.setAutoCommit(false)`을 이용하여 트랜잭션을 시작한다.
- 비지니스 로직이 에러없이 정상 작동하면 `con.commit()`을 이용해 커밋을 한다.
- 만약 로직 실행중에 에러가 발생하면 `con.rollback()`을 이용해 롤백을 한다.
- 에러가 발생하던 안하던 `finally`를 이용하여 트랜잭션을 종료시키고 커넥션을 종료한다. 이 때 트랜잭션을 종료시키지 않으면 자동 커밋이 꺼져있는채로 커넥션풀에 커넥션이 반환되기 때문에 다른 사용자가 사용할 때 혼란이 야기될 수 있다.

애플리케이션에 트랜잭션을 적용하려면 서비스계층이 매우 더러워지고 커넥션을 관리하기가 쉽지 않다는 단점이 있다. 스프링을 사용하면 이런 단점을 없앨 수 있다. 
