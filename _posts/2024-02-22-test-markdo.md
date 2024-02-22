---
layout: post
title: Spring - Bean Validation
tags: [Spring, 스프링 MVC]
comments: true
---

## Validation 로직의 문제점

지금까지 작성한 검증 코드를 보면 너무 길고 복잡하다. 우리는 대부분 값이 비어있는지, 값의 범위, 값의 최댓값 등을 검증하게 된다. 이런 공통적인 검증 로직을 표준화 한것이 `Bean Validation`이다. `Bean Validation`을 잘 활용하면 어노테이션 하나로 검증로직을 짤 수 있다.

## Bean Validation 적용

```java
@Data
public class Item {
    
    @NotNull
    private Long id;
    
    @NotBlank
    private String itemName;
    
    @NotNull
    @Range(min = 1000, max = 1000000)
    private Integer price;
    
    @NotNull
    @Max(value = 9999)
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

- NotNull : 널이 아닌지 확인한다.
- NotBlank : 공백, 비어있는 값 둘다 확인한다. ("", " ")
- Range : 최솟값과 최댓값으로 값의 범위를 정한다.
- Max : 쵯댓값을 정한다.

스프링 부트는 자동으로 LocalValidatorFactoryBean을 글로벌 Validator로 적용하기 때문에 `@Validated`를 붙이기만 하면  Bean validator를 사용할 수 있다. 

## 에러 코드

저번 시간에 에러 코드를 체계화해서 적용했다. Bean Validation 도 같은 방법으로 적용할 수 있다. 
`@NotNull`을 적용했다면 NotNull.Item.ItemName 등의 코드가 생성된다.

## 객체 Bean Validation

Bean Validation은 기본적으로 필드에만 적용가능하다. 글로벌 에러로 두개 이상의 필드에 에러를 적용하려면 어떻게 할까?

```java
@Data
@ScriptAssert(lang = "javascript", script = "_this.price * _this.quantity >=
10000")
public class Item {
 //...
}
```

`@ScriptAssert`를 사용해서 객체 검증을 할 수 있다. script 속성에 검증식을 넣어주면 된다.

> 근데 실제 사용해보면 제약이 많고 복잡해서 사용을 권장하지 않는다. 객체 검증을 컨트롤러 부분에서 rejectValue로 처리해주는것이 좋다.

## Bean Validation의 한계

우리가 상품을 등록할 때하고 수정할 때 다른 검증 방식을 사용한다면 지금의 방법으로는 처리할 수 없다. 한곳에서 적용을 시키면 다른 곳에서도 똑같이 적용된다.

```java
package hello.itemservice.domain.item;
public interface SaveCheck {
}
```

```java
package hello.itemservice.domain.item;
public interface UpdateCheck {
}

```
@Data
public class Item {
  @NotNull(groups = UpdateCheck.class)
  private Long id;
  
  @NotBlank(groups = {SaveCheck.class, UpdateCheck.class})
  private String itemName;
  
  @NotNull(groups = {SaveCheck.class, UpdateCheck.class})
  @Range(min = 1000, max = 1000000, groups = {SaveCheck.class,
  UpdateCheck.class})
  private Integer price;
  @NotNull(groups = {SaveC
  heck.class, UpdateCheck.class})
  @Max(value = 9999, groups = SaveCheck.class) //등록시에만 적용
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

수정과 저장의 검증방식을 달리 하기 위해서 그룹을 지정한다. 그룹용 클래스 UpdateCheck, SaveCheck를 만들고 어노테이션안에 사용할 그룹을 정한다.

```java
@PostMapping("/add")
public String addItemV2(@Validated(SaveCheck.class) @ModelAttribute Item item,
BindingResult bindingResult, RedirectAttributes redirectAttributes) {
 //...
}
```

컨트롤러 부분에 그룹 클래스를 지정해주면 해당 그룹 클래스를 넣은 필드만 검증이 실시된다.

## Form 분리

실무에서는 저장하고 수정의 폼을 분리한다. 예를 들어 회원가입을 할 때 개인정보 약관에 동의했는데 수정폼에서는 해당 내용이 필요없는 이런 경우가 많기 때문이다. 

```java
@Data 
public class ItemSaveForm {
    @NotBlank
    private String itemName;
    
    @NotNull
    @Range(min = 1000, max = 1000000)
    private Integer price;
    
    @NotNull
    @Max(value = 9999)
    private Integer quantity;
}
```

```java
@Data
public class ItemUpdateForm{
    @NotNull
    private Long id;
    
    @NotBlank
    private String itemName;
    
    @NotNull
    @Range(min = 1000, max = 1000000)
    private Integer price;
    
    private Integer quantity;
}
```

각 폼의 요구사항에 따라 필드를 정의하고 검증 로직을 넣어준다. 

```java
@PostMapping("/add")
public String addItem(@Validated @ModelAttribute("item") ItemSaveForm form, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {

    if(form.getQuantity() != null && form.getPrice() != null){
        int result = form.getQuantity() * form.getPrice();
        if(result < 10000){
            bindingResult.rejectValue("quantity", "totalPriceMin", new Object[]{1000, 1000000}, null);
        }
    }
    
    if(bindingResult.hasErrors()){
        return "validation/v3/addForm";
    }
    
    Item item = new Item();
    item.setItemName(form.getItemName());
    item.setPrice(form.getPrice());
    item.setQuantity(form.getQuantity());

    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v3/items/{itemId}";
}
```

@ModelAttribute의 클래스 타입을 위에서 정의한 클래스 타입으로 바꿔준다. 로직에 맞게 코드를 수정해주면 된다.
