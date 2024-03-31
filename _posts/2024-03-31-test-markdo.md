---
layout: post
title: Spring - 스프링 데이터 JPA
tags: [Spring, 데이터 접근 핵심 원리]
comments: true
---

## 스프링 데이터 JPA 주요 기능

스프링 데이터 JPA는 JPA를 편리하게 쓸 수 있도록 도와주는 라이브러리다.

- 공통 인터페이스 기능

![jpa](/assets/img/japrepository.PNG)

`JpaRepository`를 통해서 공통적인 CRUD기능을 다 제공한다. 

> 스프링 데이터는 공통적인 기능들을 편리하게 쓰려고 만든 라이브러리이다. JPA 뿐만 아니라 JDBC, Redis 등등 많은 데이터베이스를 지원한다

```java
public interface SpringDataJpaRepository extends JpaRepository<Item, Long> {
}
```

다음과 같이 `JpaRepository` 인터페이스를 상속받으면 동적 프록시를 통해서 ItemRepository 구현체를 만들어주고 그것을 스프링빈으로 등록시켜준다. 따라서 인터페이스만 만들면 기본적인 CRUD기능을 사용할 수 있다.
  
- 쿼리 메서드 기능

스프링 데이터 JPA는 인터페이스에 메서드를 적어두면 메서드 이름을 분석해서 쿼리를 자동으로 만들고 실행해준다. 

```java
public List<Member> findByUsernameAndAgeGreaterThan(String username, int age) {
 return em.createQuery("select m from Member m where m.username = :username and m.age > :age")
   .setParameter("username", username)
   .setParameter("age", age)
   .getResultList();
}
```

원래 순수 JPA는 JPQL을 작성하고 파라미터도 직접 바인딩해야 한다.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
  List<Member> findByUsernameAndAgeGreaterThan(String username, int age);
}
```

스프링 데이터 JPA에서는 메서드만 적으면 분석해서 JPQL을 생성해준다. 물론 아무렇게나 적는것은 아니고 규칙에 맞춰서 작성해야한다.

근데 이렇게 되면 조건이 많아지면 많아질수록 메서드 이름이 길어진다. 따라서 복잡한 쿼리는 직접 작성하는게 더 좋다.

```java
@Query("select i from Item i where i.itemName like :itemName and i.price <= :price")
List<Item> findItems(@Param("itemName") String itemName, @Param("price") Integer price);
```

`@Quenry`어노테이션 안에 작성하면 된다. 그리고 파라미터명을 명시적으로 입력해야한다. `@Param`어노테이션을 이용해 파라미터 이름을 지정한다.

## 스프링 데이터 JPA 적용

```java
@Repository
@Slf4j
@Transactional
@RequiredArgsConstructor
public class JpaItemRepositoryV2 implements ItemRepository {
    private final SpringDataJpaRepository repository;

    @Override
    public Item save(Item item) {
        return repository.save(item);
    }

    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        Item item = repository.findById(itemId).orElseThrow();
        item.setItemName(updateParam.getItemName());
        item.setPrice(updateParam.getPrice());
        item.setQuantity(updateParam.getQuantity());
    }

    @Override
    public Optional<Item> findById(Long id) {
        return repository.findById(id);
    }

    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();

        if (StringUtils.hasText(itemName) && maxPrice != null) {
            return repository.findItems("%" + itemName + "%", maxPrice);
        }else if(StringUtils.hasText(itemName)){
            return repository.findByItemNameLike("%" + itemName + "%");
        }else if (maxPrice != null) {
            return repository.findByPriceLessThanEqual(maxPrice);
        }else {
            return repository.findAll();
        }
    }
}
```

itemService가 itemRepository에 의존하기 때문에 ItemService에서 SpringDataJpaItemRepository를 그대로 사용할 수 없다.
JpaItemRepositoryV2를 만들어서 SpringDataJpaItemRepository를 itemRepository로 바꾸는 어댑터처럼 사용한다.

![auto](/assets/img/jpa_auto.PNG)

- save
스프링 데이터 JPA가 제공하는 save를 호출한다.

- update
스프링 데이터 JPA가 제공하는 findById를 사용해서 엔티티를 찾고 setter 메서드를 통해 수정한다.

- findAll
조건에 따라 4가지로 분류하여 메서드를 실행한다.

```java
@Configuration
@RequiredArgsConstructor
public class SpringDataJpaConfig {

    private final SpringDataJpaRepository repository;

    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }

    @Bean
    public ItemRepository itemRepository() {
        return new JpaItemRepositoryV2(repository);
    }
}

```

> JpaRepository를 상속해서 만든 레파지토리는 애플리케이션 실행 중에 동적으로 구현체를 만들어서 빈을 등록한다. 따라서 빨간줄이 그어질 수 있다.

