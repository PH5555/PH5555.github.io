---
layout: post
title: Spring - QueryDsl
tags: [Spring, 데이터 접근 핵심 원리]
comments: true
---

## QueryDsl 적용

```java
private final EntityManager em;
private final JPAQueryFactory query;

public JpaItemRepositoryV3(EntityManager em) {
    this.em = em;
    this.query = new JPAQueryFactory(em);
}
```

QueryDsl을 사용하려면 `JPAQueryFactory`이 필요하다. QueryDsl은 JPA쿼리인 JPQL을 만들기 때문에 EntityManager을 필요로 한다.

```java
@Override
public List<Item> findAll(ItemSearchCond cond) {
    Integer maxPrice = cond.getMaxPrice();
    String itemName = cond.getItemName();

    BooleanBuilder builder = new BooleanBuilder();
    QItem item = QItem.item;
    if(StringUtils.hasText(itemName)){
        builder.and(item.itemName.like("%" + itemName + "%"));
    }
    if(maxPrice != null){
        builder.and(item.price.loe(maxPrice));
    }

    return query.select(item).from(item).where(builder).fetch();
}
```

`BooleanBuilder`를 통해서 조건을 넣어주면 된다.

```java
@Override
public List<Item> findAll(ItemSearchCond cond) {
    Integer maxPrice = cond.getMaxPrice();
    String itemName = cond.getItemName();

    return query.select(item).from(item).where(likeItemName(itemName), maxPrice(maxPrice)).fetch();
}

private BooleanExpression maxPrice(Integer maxPrice) {
    if(maxPrice != null){
        return item.price.loe(maxPrice);
    }
    return null;
}

private BooleanExpression likeItemName(String itemName) {
    if(StringUtils.hasText(itemName)){
        return item.itemName.like("%" + itemName + "%");
    }
    return null;
}
```

where의 인자로 넣은 조건들은 And로 처리가 된다. maxPrice와 likeItemName 메서드를 다른 쿼리에서도 활용할 수 있다.

QueryDsl덕분에 동적 쿼리를 깔끔하게 작성할 수 있다. 그리고 쿼리에 오타가 있어도 컴파일 시점에 알 수 있다는 장점이 있다.
