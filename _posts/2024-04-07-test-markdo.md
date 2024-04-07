---
layout: post
title: Spring - 트랜잭션 전파
tags: [Spring, 데이터 접근 핵심 원리]
comments: true
---

## 전파 기본

트랜잭션을 각각 사용하는 것이 아니라 트랜잭션이 이미 진행중인데 추가로 트랜잭션 실행을 하면 어떻게 될까

![phy](/assets/img/transaction_phy.PNG)

먼저 실행한 트랜잭션을 외부 트랜잭션이라고 하고 나중에 실행한 트랜잭션을 내부 트랜잭션이라고 하면 
내부 트랜잭션이 외부 트랜잭션에 참여하게 된다. 그리고 이 외부, 내부 트랜잭션을 합쳐서 하나의 물리 트랜잭션을 만들어준다. 

<b>원칙</b>

- 모든 논리 트랜잭션이 커밋 되어야 물리 트랜잭션이 커밋 될 수 있다.
- 하나의 논래 트랜잭션이라도 롤백되면 물리 트랜잭션은 롤백된다.

## 전파 예제 (커밋)

![commit](/assets/img/trans_commit.PNG)

```java
@Test
void inner(){
    log.info("외부 시작");
    TransactionStatus outer = manager.getTransaction(new DefaultTransactionAttribute());
    log.info("is first = {}", outer.isNewTransaction());

    log.info("내부 시작");
    TransactionStatus inner = manager.getTransaction(new DefaultTransactionAttribute());
    log.info("is first = {}", inner.isNewTransaction());
    
    log.info("내부 커밋");
    manager.commit(inner);
    
    log.info("외부 커밋");
    manager.commit(outer);
}
```

- 외부 트랜잭션이 실행되고 있는데 내부 트랜잭션을 시작했다.
- 외부 트랜잭션의 경우에는 새롭게 생성되는 트랜잭션이므로 `isNewTransaction()`의 결과가 true가 된다.
- 내부 트랜잭션은 트랜잭션이 새롭게 만들어지는것이 아니라 외부 트랜잭션에 참여하는것이기 때문에 `isNewTransaction()`의 결과가 false가 된다.

외부 트랜잭션과 내부 트랜잭션이 하나로 묶여서 `commit`이 한번만 실행되야할거 같지만 예제를 보면 커밋이 두번 실행된다. 실제로 실행을 시켜해보면 내부 트랜잭션의 커밋은 실행되지 않는다.
처음 트랜잭션을 실행한 외부 트랜잭션이 실제 물리 트랜잭션을 관리하도록 한다.

![flow1](/assets/img/trans_flow1.PNG)

요청 흐름을 보면 내부 트랜잭션은 기존 트랜잭션이 있는지 확인하고 기존 트랜잭션이 있으면 해당 트랜잭션에 참여하게 된다.

![flow2](/assets/img/trans_flow2.PNG)

커밋을 할때에는 신규 트랜잭션이 아니면 커밋을 실제로 호출하지 않게 되고 외부 트랜잭션에서 물리 커밋을 실행하게 된다.

## 전파 예제 (롤백)

![rollback](/assets/img/trans_rollback.PNG)

외부 트랜잭션이 롤백되면 원칙에 따라 물리 트랜잭션도 롤백된다.

```java
@Test
void outer_rollback(){
    log.info("외부 시작");
    TransactionStatus outer = manager.getTransaction(new DefaultTransactionAttribute());

    log.info("내부 시작");
    TransactionStatus inner = manager.getTransaction(new DefaultTransactionAttribute());
    log.info("내부 커밋");
    manager.commit(inner);
    log.info("외부 롤백");
    manager.rollback(outer);
}
```

- 내부 커밋은 이전 커밋 예제처럼 직접 물리 트랜잭션에 관여하지 않는다.

외부 롤백의 경우에는 커밋과 비슷하지만 내부에서 롤백이 일어나면 쉽지 않아진다.

![inner_roll](/assets/img/trans_rollback_inner.PNG)

내부 트랜잭션이 롤백을 해도 내부 트랜잭션은 실제 물리 트랜잭션에 영향을 안 미치기 때문에 실제로는 커밋이 될거 같다.

```java
@Test
void inner_rollback(){
    log.info("외부 시작");
    TransactionStatus outer = manager.getTransaction(new DefaultTransactionAttribute());

    log.info("내부 시작");
    TransactionStatus inner = manager.getTransaction(new DefaultTransactionAttribute());
    log.info("내부 롤백");
    manager.rollback(inner);
    log.info("외부 커밋");
    manager.commit(outer);
}
```

하지만 실제로 실행해보면 `UnexpectedRollbackException`이 발생한다. 왜냐하면 내부 트랜잭션에서 롤백을 실행하면 실제로 롤백이 일어나지 않고 기존 트랜잭션을 롤백전용으로 표시를한다.
근데 외부 트랜잭션에서 롤백을 실핻하지 않고 커밋을 실행하면 런타임 예외를 던진다.

## REQUIRES_NEW

위와 같은 문제를 해결하기 위해서는 외부 트랜잭션과 내부 트랜잭션을 분리해야한다. 각각 별도의 트랜잭션으로 관리되면 커밋과 롤백도 별도로 되어 서로 영향을 주지 않는다.

![new](/assets/img/trans_commit.PNG)

- 이렇게 외부트랜잭션과 내부트랜잭션을 분리하려면 `REQUIRES_NEW`옵션을 사용하면 된다.
- 별도의 물리 트랜잭션을 가진다는것은 DB커넥션을 따로 사용한다는 뜻이다.

```java
@Test
void new_required(){
    log.info("외부 시작");
    TransactionStatus outer = manager.getTransaction(new DefaultTransactionAttribute());
    log.info("is first = {}", outer.isNewTransaction());

    log.info("내부 시작");
    DefaultTransactionAttribute definition = new DefaultTransactionAttribute();
    definition.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    TransactionStatus inner = manager.getTransaction(definition);
    log.info("is first = {}", inner.isNewTransaction());

    log.info("내부 롤백");
    manager.rollback(inner);

    log.info("외부 커밋");
    manager.commit(outer);
}
```

내부 트랜잭션을 시작할 때 전파 옵션인 `propagationBehavior`을 `PROPAGATION_REQUIRES_NEW`으로 설정해준다.
이 전파 옵션을 사용하면 내부 트랜잭션을 사용할 때 기존 트랜잭션에 참여하는 것이 아니라 새로운 트랜잭션을 만들어서 시작하게 된다.

