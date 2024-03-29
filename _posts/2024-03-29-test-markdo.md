---
layout: post
title: Spring - Mybatis
tags: [Spring, 데이터 접근 핵심 원리]
comments: true
---

## Mybatis

mybatis는 기본적으로 jdbcTemplate이 제공하는 기능을 거의 제공한다. 하지만 mybatis는 sql을 더 깔끔하게 작성할 수 있다.

```java
String sql = "update item " +
 "set item_name=:itemName, price=:price, quantity=:quantity " +
 "where id=:id";
```

```xml
<update id="update">
   update item
   set item_name=#{itemName},
   price=#{price},
   quantity=#{quantity}
   where id = #{id}
</update>
```

자바코드로 sql을 작성할 때는 줄이 바뀌면 + 연산자를 붙여야하는데 mybatis는 xml에 작성하기 때문에 그럴 필요가 없다.

그리고 동적쿼리를 쉽게 작성할 수 있다.

## Mybatis 설정

```
mybatis.type-aliases-package=hello.itemservice.domain
mybatis.configuration.map-underscore-to-camel-case=true
logging.level.hello.itemservice.repository.mybatis=trace
```

application.properties에 해당코드를 추가해준다.

- mybatis.type-aliases-package : 마이바티스에서 타입정보를 입력할 때 패키지이름까지 다 적어야하는데(xml이 java를 알 수 없으므로) 패키지 이름을 생략할 수 있게 해준다.
- mybatis.configuration.map-underscore-to-camel-case : 언더바를 카멜 표기법으로 변환할 수 있게 해준다.

## Mybatis 적용

```java
@Mapper
public interface ItemMapper {

    void save(Item item);

    void update(@Param("id") Long id, @Param("updateParam")ItemUpdateDto itemUpdateDto);

    List<Item> findAll(ItemSearchCond cond);

    Optional<Item> findById(Long id);
}

```

- 마이바티스 xml을 호출해주는 매퍼 인터페이스다.
- `@Mapper` 어노테이션을 붙이면 마이바티스에서 인식할 수 있게 된다.
- 파라미터가 하나가 아니면 `@Param`으로 이름을 지정해준다. 파라미터가 객체일 경우 필드이름으로 접근가능하다.

```java
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="hello.itemservice.repository.mybatis.ItemMapper">
    <insert id="save" useGeneratedKeys="true" keyProperty="id">
        insert into item(item_name, price, quantity)
        values (#{itemName}, #{price}, #{quantity})
    </insert>

    <update id="update">
        update item set item_name=#{updateParam.itemName}, price=#{updateParam.price}, quantity=#{updateParam.quantity}
        where id=#{id}
    </update>

    <select id="findById" resultType="Item">
        select id, item_name, price, quantity from item where id = #{id}
    </select>

    <select id="findAll" resultType="Item">
        select id, item_name, price, quantity from item
        <where>
            <if test="itemName != null and itemName != ''">
                and item_name like concat('%', #{itemName}, '%')
            </if>
            <if test="maxPrice != null">
                and price &lt;= #{maxPrice}
            </if>
        </where>
    </select>
</mapper>
```

- 매퍼클래스가 있는 패키지와 같은 곳에 파일을 생성해야한다. (hello.itemservice.repository.mybatis)
- namespace에 매퍼클래스를 넣으면 된다.

```java
<insert id="save" useGeneratedKeys="true" keyProperty="id">
    insert into item(item_name, price, quantity)
    values (#{itemName}, #{price}, #{quantity})
</insert>
```

- insert sql은 <insert>을 이용하면 된다.
- id에 매퍼 클래스에서 지정한 메서드 이름을 넣으면 된다.
- 파라미터는 #{} 문법을 사용한다. 매퍼에서 넘긴 파라미터 이름을 사용한다.
- useGeneratedKeys는 데이터베이스가 키를 생성해주는 전략일 때 사용한다.

```java
<select id="findById" resultType="Item">
    select id, item_name, price, quantity from item where id = #{id}
</select>
```

- select는 resultType을 지정해주어야한다. `mybatis.type-aliasespackage`속성을 지정해준 덕분에 패키지명을 생략할 수 있다.

```java
<select id="findAll" resultType="Item">
    select id, item_name, price, quantity from item
    <where>
        <if test="itemName != null and itemName != ''">
            and item_name like concat('%', #{itemName}, '%')
        </if>
        <if test="maxPrice != null">
            and price &lt;= #{maxPrice}
        </if>
    </where>
</select>
```

- `if`는 해당 조건을 만족하면 구문을 추가해준다.
- `where`은 적절하게 조건문을 만들어준다. 예를 들면 조건을 모두 만족하지 않으면 where을 삭제해주고 조건이 하나가 삭제되면 and를 삭제해준다.

> xml에서는 데이터 영역에 >, < 같은 특수문자를 사용할 수 없기 때문에 `&lt;`을 이용한다.

## 분석

매퍼 인터페이스를 보면 인터페이스만 있고 구현체가 없는데 어떻게 실행이 될 수 있는 것일까?

![my](/assets/img/mybatis.PNG)

1. 애플리케이션 실행시점에 `@Mapper`가 붙어있는 클래스를 조사한다.
2. 해당 인터페이스가 발견되면 동적 프록시 기술을 이용해서 구현체를 만든다.
3. 해당 구현체를 스프링빈으로 등록한다.

원래는 직접 xml과 연동해주어야하는데 프록시기술을 통해서 이를 편리하게 할 수 있다.



