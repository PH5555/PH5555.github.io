---
layout: post
title: Spring - JPA
tags: [Spring, 데이터 접근 핵심 원리]
comments: true
---

## JPA 적용

JPA에서 제일 중요한것은 객체와 테이블을 매핑하는것이다.

```java
@Data
@Entity
@Table(name = "item")
public class Item {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "item_name")
    private String itemName;
    private Integer price;
    private Integer quantity;

    public Item() {
    }

    public Item(String itemName, Integer price, Integer quantity) {
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    }
}
```

- `@Entity` 이 어노테이션이 있어야 JPA가 객체를 인식할 수 있다.
- `@Table` 테이블 이름을 지정할 수 있다. 클래스 이름과 테이블 이름이 같으면 생략할 수 있다.
- `@Id` 테이블의 PK를 지정할 수 있다.
- `@GenratedValue` PK 생성값의 전략을 정할 수 있다.
- `@Column` 컬럼명을 지정할 수 있다. 필드이름과 컬럼 이름이 같으면 생략할 수 있다. 이때 카멜 케이스인 itemName을 언더스코어인 item_name 으로 자동 변환해준다.

JPA는 public 기본 생성자가 기본으로 있어야한다.

```java
private final EntityManager em;

public JpaItemRepository(EntityManager em) {
    this.em = em;
}
```

엔티티매니저를 통해서 JPA의 모든 동작이 이루어진다. 엔티티매니저는 데이터소스를 가지고 있어서 데이터베이스에 접근할 수 있다.

```java
@Override
public Item save(Item item) {
    em.persist(item);
    return item;
}
```

데이터를 저장할 때는 `em.persist`를 사용한다. 

```java
@Override
public void update(Long itemId, ItemUpdateDto updateParam) {
    Item item = em.find(Item.class, itemId);
    item.setItemName(updateParam.getItemName());
    item.setPrice(updateParam.getPrice());
    item.setQuantity(updateParam.getQuantity());
}
```

업데이트 하고자하는 데이터를 `find`로 찾아서 setter 메서드로 값을 바꿔준다. 따로 데이터를 저장하지 않아도 자동 저장이 된다.
JPA는 트랜잭션이 커밋되는 시점에 변경된 엔티티가 있는지 확인하고 update sql을 실행해준다.

```java
@Override
public List<Item> findAll(ItemSearchCond cond) {
    String jpql = "select i from Item i";

    Integer maxPrice = cond.getMaxPrice();
    String itemName = cond.getItemName();
    if (StringUtils.hasText(itemName) || maxPrice != null) {
        jpql += " where";
    }
    boolean andFlag = false;
    if (StringUtils.hasText(itemName)) {
        jpql += " i.itemName like concat('%',:itemName,'%')";
        andFlag = true;
    }
    if (maxPrice != null) {
        if (andFlag) {
            jpql += " and";
        }
        jpql += " i.price <= :maxPrice";
    }
    log.info("jpql={}", jpql);
    TypedQuery<Item> query = em.createQuery(jpql, Item.class);
    if (StringUtils.hasText(itemName)) {
        query.setParameter("itemName", itemName);
    }
    if (maxPrice != null) {
        query.setParameter("maxPrice", maxPrice);
    }
    return query.getResultList();
}
```

jpql은 JPA에서 복잡한 조건의 데이터를 조회할 때 쓰인다. 엔티티 객체를 대상으로 하기 때문에 `from`다음에 객체 이름이 들어간다. 

> 파라미터 지정은 `:itemName` 같이 사용한다.

```java
@Configuration
public class JpaConfig {

    private final EntityManager em;

    public JpaConfig(EntityManager em) {
        this.em = em;
    }

    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }

    @Bean
    public ItemRepository itemRepository() {
        return new JpaItemRepository(em);
    }

}
```

스프링부트에서 설정정보를 이용해서 EntityManager를 자동생성한다. 이를 주입받아서 사용하면 된다.

## JPA 예외 변환

JPA 는 스프링과 관련이 없는 독립적인 기술이기 때문에 사용하다 에러가 발생하면 PersistenceException과 그 하위 예외를 발생시킨다. 이 JPA 예외를 `@Repository`가 스프링예외로 바꿀 수 있도록 해준다.

![error](/assets/img/jpa_error.PNG)

`@Repository`어노테이션과 `JPA`를 같이 사용하면 스프링이 예외를 변환할 수 있는 AOP를 만들어준다. 예외 변환 AOP 프록시는 JPA 관련 예외가 발생하면 JPA 예외 변환기를 통해 발생한 예외를 스프링 데이터 접근 예외로 변환한다.
