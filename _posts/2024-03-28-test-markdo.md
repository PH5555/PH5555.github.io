---
layout: post
title: Spring - JdbcTemplate
tags: [Spring, 데이터 접근 핵심 원리]
comments: true
---

## JdbcTemplate 적용

```java
@Override
public Item save(Item item) {
    String sql = "insert into item(item_name, price, quantity) values (?, ?, ?)";
    KeyHolder keyHolder = new GeneratedKeyHolder();
    template.update(con -> {
        PreparedStatement pstm = con.prepareStatement(sql, new String[]{"id"});
        pstm.setString(1, item.getItemName());
        pstm.setInt(2, item.getPrice());
        pstm.setInt(3, item.getQuantity());
        return pstm;
    }, keyHolder);

    long key = keyHolder.getKey().longValue();
    item.setId(key);
    return item;
}
```

- `con.prepareStatement` 에 id를 지정해줌으로써 쿼리 실행 이후에 생성된 id를 조회할 수 있다.

```java
@Override
public Optional<Item> findById(Long id) {
    String sql = "select id, item_name, price, quantity from item where id=?";
    try{
        Item item = template.queryForObject(sql, itemRowMapper(), id);
        return Optional.of(item);
    }catch (EmptyResultDataAccessException e) {
        return Optional.empty();
    }
}
```

- `queryForObject`는 쿼리 실행결과가 하나일 때 쓴다.

```java
@Override
public List<Item> findAll(ItemSearchCond cond) {
    String itemName = cond.getItemName();
    Integer maxPrice = cond.getMaxPrice();

    String sql = "select id, item_name, price, quantity from item";
    if(StringUtils.hasText(itemName) || maxPrice != null){
        sql += " where";
    }

    boolean andFlag = false;
    List<Object> param = new ArrayList<>();
    if(StringUtils.hasText(itemName)){
        sql += " item_name like concat('%', ?, '%')";
        param.add(itemName);
        andFlag = true;
    }

    if(maxPrice != null){
        if(andFlag) {
            sql += " and";
        }
        sql += " price <= ?";
        param.add(maxPrice);
    }
    return template.query(sql, itemRowMapper(), param.toArray());

}
```

- 아이템 조회는 조건에 따라 쿼리문이 달라져야하므로 조건에 따라 문자열 연산으로 쿼리를 추가해준다.
- 조건이 하나만 늘어나도 케이스가 많이 늘어나기 때문에 복잡해진다.
  
```java
@Configuration
@RequiredArgsConstructor
public class JdbcTemplateV1Config {

    private final DataSource dataSource;

    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }

    @Bean
    public ItemRepository itemRepository() {
        return new JdbcTemplateItemRepository(dataSource);
    }

}
```
```java
@Import(JdbcTemplateV3Config.class)
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
public class ItemServiceApplication {
```

- 만든 레파지토리와 서비스를 빈으로 등록한다.
- 컴포넌트스캔 범위를 web패키지 하위로 지정해주었기 때문에 `@Import`를 이용해서 스프링빈 등록을 해준다.

## 이름 지정 파라미터

jdbcTemplate를 사용하면 파라미터의 순서를 지켜야한다. 파라미터의 순서를 잘못넣으면 데이터베이스를 복구해야하는 대참사가 일어난다. 이런 문제를 보완하기 위해서
`NamedParameterJdbcTemplate`라는 기능을 사용한다.

```java
@Override
public Item save(Item item) {
    String sql = "insert into item(item_name, price, quantity) values (:itemName, :price, :quantity)";
    SqlParameterSource param = new BeanPropertySqlParameterSource(item);
    KeyHolder keyHolder = new GeneratedKeyHolder();
    template.update(sql, param, keyHolder);

    long key = keyHolder.getKey().longValue();
    item.setId(key);
    return item;
}
```

`?`부분을 `:itemName`같이 이름으로 바인딩해준다. 바인딩 방법도 여러가지가 있다.

`BeanPropertySqlParameterSource` 는 파라미터로 넣는 클래스의 프로퍼티를 이용해 자동으로 매핑을 해준다. getItemName이라는 메서드가 있으면 itemName으로 키를 만들어주고 getItemName 메서드의 결과값을 벨류로 사용한다.
간단하게 사용할 수 있지만 클래스에 포함되지 않는 파라미터가 있으면 사용할 수 없다.

```java
@Override
public Optional<Item> findById(Long id) {
    String sql = "select id, item_name, price, quantity from item where id=:id";
    try{
        Map<String, Object> param = Map.of("id", id);
        Item item = template.queryForObject(sql, param, itemRowMapper());
        return Optional.of(item);

    }catch (EmptyResultDataAccessException e){
        return Optional.empty();
    }
}
```

Map 객체를 만들어서 파라미터로 넣어줄 수 있다.

```java
@Override
public void update(Long itemId, ItemUpdateDto updateParam) {
    String sql = "update item set item_name=:itemName, price=:price, quantity=:quantity where id=:id";
    MapSqlParameterSource param = new MapSqlParameterSource()
            .addValue("itemName", updateParam.getItemName())
            .addValue("price", updateParam.getPrice())
            .addValue("quantity", updateParam.getQuantity())
            .addValue("id", itemId);

    template.update(sql, param);
}
```

Map과 유사한데 좀더 sql에 특화되어있는 기능을 제공하는 `MapSqlParameterSource`도 있다. sql타입을 지정할 수 있다.

## BeanPropertyRowMapper

```java
private RowMapper<Item> itemRowMapper() {
    return (rs, rowNum) -> {
        Item item = new Item();
        item.setItemName(rs.getString("item_name"));
        item.setId(rs.getLong("id"));
        item.setPrice(rs.getInt("price"));
        item.setQuantity(rs.getInt("quantity"));
        return item;
    };
}
```

rowMapper를 보면 사실 거의 반복되는 내용이다. 클래스의 프로퍼티에 기반해서 만들어지기 때문에 자동화할 수 있다.

```java
private RowMapper<Item> itemRowMapper() {
    return BeanPropertyRowMapper.newInstance(Item.class);
}
```

BeanPropertyRowMapper.newInstance의 파라미터로 클래스를 넣어주면 해당 클래스의 프로퍼티에 기반하여 변환해준다. 

근데 데이터베이스상에는 member_name 으로 나와있는데 객체에는 userName으로 되어 있으면 자동 변환이 되지 않는다. 자바 프로퍼티 규약에 의해서 변환되기 때문에 쿼리문을 짤때 `As user_name`으로 변환해주어야한다.

> 데이터베이스에는 `snake case`를 많이쓰고 자바 객체에는 `camel case`를 많이 쓰므로 자바에서 해당 부분을 자동으로 변환해준다.

## SimpleJdbcInsert

JdbcTemplate에서는 insert문을 직접 작성하지 않아도 되도록 SimpleJdbcInsert를 제공한다.

```java
private final SimpleJdbcInsert jdbcInsert;

public JdbcTemplateItemRepositoryV3(DataSource dataSource) {
    this.template = new NamedParameterJdbcTemplate(dataSource);
    this.jdbcInsert = new SimpleJdbcInsert(dataSource)
            .withTableName("item")
            .usingGeneratedKeyColumns("id");
}
```

먼저 테이블명과 키가 자동으로 생성되는 컬럼을 지정해서 객체를 만들어준다.

```java
@Override
public Item save(Item item) {

    SqlParameterSource param = new BeanPropertySqlParameterSource(item);
    Number key = jdbcInsert.executeAndReturnKey(param);
    item.setId(key.longValue());
    return item;
}
```

`executeAndReturnKey` 를 사용하면 반환되는 키를 매우 쉽게 찾을 수 있다.

