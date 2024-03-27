---
layout: post
title: Spring - 예외 처리
tags: [Spring, 데이터 접근 핵심 원리]
comments: true
---

## 체크예외와 인터페이스

레파지토리 인터페이스를 만들어서 레파지토리의 구현체를 쉽게 바꿀 수 있도록 설계할 것이다.

```java
public interface MemberRepository {
 Member save(Member member);
 Member findById(String memberId);
 void update(String memberId, int money);
 void delete(String memberId);
}
```

이런 인터페이스를 만들어서 적용시킬 것인데 현재 레파지토리에는 적용시킬 수가 없다.

```java
public Member save(Member member) throws SQLException {
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
        throw new SQLException(e);
    } finally {
        close(con, pstm, null);
    }
}
```

현재 구현체를 보면 `throws SQLException`라는 부분이 있다. 구현체에서 예외를 던지기 때문에 인터페이스도 예외를 던지게 설계해야한다. 하지만 그렇게 설계하게 되면 인터페이스를 설계하는 이유가 없다. 왜냐하면 어떤 구현체를 적용하더라도 무조건적으로
예외를 던져주어야하기 때문이다.(체크 예외이기 때문) 런타임 예외는 이런 부분에서 자유롭다.

## 런타임 예외 적용

```java
public class MyDbException extends RuntimeException {
    public MyDbException() {
    }

    public MyDbException(String message) {
        super(message);
    }

    public MyDbException(String message, Throwable cause) {
        super(message, cause);
    }

    public MyDbException(Throwable cause) {
        super(cause);
    }
}

`RuntimeException`예외를 상속받는 MyDbException을 만들어준다.

```java
@Override
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
        throw new MyDbException(e);
    } finally {
        close(con, pstm, null);
    }
}
```

SQLException 예외를 MyDbException으로 바꾸어서 던져준다.

```java
private final MemberRepository memberRepository;

public MemberServiceV4(MemberRepository memberRepository) {
    this.memberRepository = memberRepository;
}

@Transactional
public void accountTransfer(String fromId, String toId, int money){
    bizLogic(fromId, toId, money);
}

private void bizLogic(String fromId, String toId, int money) {
    Member fromMember = memberRepository.findById(fromId);
    Member toMember = memberRepository.findById(toId);

    memberRepository.update(fromId, fromMember.getMoney() - money);
    validation(fromMember);
    memberRepository.update(toId, toMember.getMoney() + money);
}

private static void validation(Member fromMember) {
    if (fromMember.getMemberId().equals("ex")) {
        throw new IllegalStateException();
    }
}
```

구현체를 인터페이스를 상속받도록 만들었기 때문에 서비스에 레파지토리를 인터페이스를 통해 접근할 수 있다.

체크 예외를 언체크 예외로 바꿈으로써 코드에서 SQLException이라는 종속성을 제거할 수 있었다. 하지만 모든 에러가 다 MyDbException가 넘어와서 어떤 에러인지 구분이 잘 안간다는 단점이 있다.

## 데이터 접근 예외 처리

데이터베이스 오류에 따라서 특정 에러는 복구하고 싶을 수 있다. 예를 들어서 회원을 추가하는데 같은 id이면 id뒤에 랜덤한 값을 붙여서 id가 안 겹치도록 할 수 있을것이다.

![error](/assets/img/data_error.PNG)

데이터베이스에 같은 id값이 있으면 데이터베이스는 오류코드를 반환하고, 이 오류 코드를 받은 JDBC는 SQLException을 던진다. 이때 이 에러코드를 이용하면 어떤 에러인지 판별할 수 있다.

> 참고로 같은 에러라도 데이터베이스마다 정의된 코드가 다르다.

```java
public class MyDuplicateKeyException extends MyDbException{
    public MyDuplicateKeyException() {
    }

    public MyDuplicateKeyException(String message) {
        super(message);
    }

    public MyDuplicateKeyException(String message, Throwable cause) {
        super(message, cause);
    }

    public MyDuplicateKeyException(Throwable cause) {
        super(cause);
    }
}
```

MyDuplicateKeyException이라는 예외를 새로 정의해준다. 이 에러는 데이터베이스 에러의 한 종류이므로 MyDbException를 상속받도록해준다.

```java
public Member save(Member member){
    String sql = "insert into member(member_id, money) values(?, ?)";
    Connection con = null;
    PreparedStatement pstm = null;

    try{
        con = dataSource.getConnection();
        pstm = con.prepareStatement(sql);
        pstm.setString(1, member.getMemberId());
        pstm.setInt(2, member.getMoney());
        pstm.executeUpdate();
        return member;
    }catch (SQLException e){
        if(e.getErrorCode() == 23505){
            throw new MyDuplicateKeyException(e);
        }

        throw new MyDbException(e);
    }finally {
        JdbcUtils.closeStatement(pstm);
        JdbcUtils.closeConnection(con);
    }
}
```

```java
static class Service{
  private final Repository repository;
  
  public void create(String memberId){
      try{
          repository.save(new Member(memberId, 0));
          log.info("save id = {}", memberId);
      }catch (MyDuplicateKeyException e){
          log.info("키 중복 다시 시도");
          String retryId = generateNewId(memberId);
          log.info("retry id = {}", retryId);
          repository.save(new Member(retryId, 0));
      }
  }
  
  private String generateNewId(String memberId) {
      return memberId + new Random().nextInt(100000);
  }
}
```

- 에러코드가 23505(키 중복 오류)이면 MyDuplicateKeyException예외를 던진다. 그 이외의 경우에는 MyDbException를 던진다.
- 서비스 계층에서 MyDuplicateKeyException예외를 잡는 부분에서 키 뒤에 새로운 값을 붙여서 다시 시도한다.
- MyDuplicateKeyException은 우리가 만든 예외이므로 레파지토리와 서비스의 순수성을 유지할 수 있음

## 스프링 예외 추상화

하지만 에러코드는 데이터베이스마다 모두 다르다. 게다가 오류 코드는 수없이 많은데 그거마다 에러를 새롭게 정의해줄 수는 없다. 

![springerror](/assets/img/spring_error.PNG)

그래서 스프링은 데이터 접근 계층에 대한 수십개의 예외를 정의해두고 모든 데이터베이스에서 접근할 수 있도록 해주었다. 각 에러는 특정기술에 종속되지 않는다.

- 최상위 계층은 런타임에러를 상속받기 때문에 모든 에러는 런타임에러다.
- Transient 에러는 일시적인 에러이다. 같은 sql로 다시 시도했을 때 성공할 가능성이 있다. 예를 들어 타임아웃, 락과 관련된 에러이다.
- NonTransient 에러는 일시적이지 않은 에러이다. sql 문법이 잘못되었거나, 데이터베이스 규약에 위반된다.

스프링은 데이터베이스에서 발생하는 에러코드를 스프링이 정의한 에러로 변환하는 변환기를 제공한다.

```java
private final DataSource dataSource;
private final SQLExceptionTranslator exTranslator;

public MemberRepositoryV4_2(DataSource dataSource) {
    this.dataSource = dataSource;
    this.exTranslator = new SQLErrorCodeSQLExceptionTranslator();
}

@Override
public Member save(Member member){
    String sql = "insert into membasder(member_id, money) values (?, ?)";

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
        throw exTranslator.translate("save", sql, e);
    } finally {
        close(con, pstm, null);
    }
}
```

`SQLExceptionTranslator` 을 만들어주고 exTranslator.translate 메서드를 이용해서 에러를 변환해준다.

첫번째 파라미터는 읽을 수 있는 설명이고, 두번째 파라미터는 sql, 세번째 파라미터는 발생된 SQLException을 넣어주면된다. 이렇게 해주면 적절한 에러로 변환해준다.

## JdbcTemplate

레파지토리에서 JDBC를 사용하기 때문에 반복되는 부분이 많이 발생한다. 커넥션을 만들고 쿼리를 만들고 결과를 바인딩하고 다 코드가 비슷하다. 실질적으로 다른것은 sql 밖에 없다.

```java
public class MemberRepositoryV5 implements MemberRepository{
    private final JdbcTemplate template;

    public MemberRepositoryV5(DataSource dataSource) {
        this.template = new JdbcTemplate(dataSource);
    }

    @Override
    public Member save(Member member){
        String sql = "insert into membasder(member_id, money) values (?, ?)";
        template.update(sql, member.getMemberId(), member.getMoney());
        return member;
    }

    @Override
    public Member findById(String id){
        String sql = "Select * from Member where member_id = ?";
        return template.queryForObject(sql, memberRowMapper(), id);
    }

    private RowMapper<Member> memberRowMapper() {
        return ((rs, rowNum) -> {
            Member member = new Member();
            member.setMemberId(rs.getString("member_id"));
            member.setMoney(rs.getInt("money"));
            return member;
        });
    }

    @Override
    public void update(String memberId, int money) {
        String sql = "update member set money=? where member_id=?";
        template.update(sql, money, memberId);
    }

    @Override
    public void delete(String memberId) {
        String sql = "delete from Member where member_id = ?";
        template.update(sql, memberId);
    }

}
```

- update 메서드를 통해 결과값이 반환되지 않는 sql을 실행할 수 있다. 첫번째 파라미터에 sql을 넣고 두번째 파라미터부터 인자를 넘겨주면 된다.
- queryForObject 메서드를 통해 결과값을 받을 수 있다. 첫번째 파라미터에 sql을 넣고, 두번째 파라미터에 처리로직, 세번째 파라미터에 인자를 넘겨주면 된다.
