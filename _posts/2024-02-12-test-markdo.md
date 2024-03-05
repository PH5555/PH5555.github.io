---
layout: post
title: Spring - 스프링 MVC 웹페이지 만들어보기
tags: [Spring, 스프링 MVC]
comments: true
---

## 요구사항 분석

![flow](/assets/img/mvc1_flow.PNG)

첫 화면으로 상품 목록이 나오고 각 상품을 클릭하면 상품 상세 페이지가 나온다. 상품 상세페이지에서 상품을 수정할 수 있다. 상품 목록 페이지에서 상품 등록을 할 수 있고, 상품 등록을 완료하면 상품 상세 페이지로 넘어가는 구조다.

```java
@Getter @Setter
public class Item{
    private Long id;
    private String itemName;
    private Integer price;
    private Integer quantity;
    
    public Item(){
        
    }
    
    public Item(String itemName, Integer price, Integer quantity){
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    }
}
```

```java
package project.itemservice.domain.item;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.springframework.stereotype.Repository;

@Repository 
public class ItemRepository{
    
    private static final Map<Long, Item> store = new HashMap<>();
    private static Long sequence = 0L;
    
    public Item save(Item item){
        item.setId(++sequence);
        store.put(item.getId(), item);
        return item;
    }
    
    public Item findById(Long id){
        return store.get(id);
    }
    
    public List<Item> findAll(){
        return new ArrayList<>(store.values());
    }
    
    public void update(Long id, Item updateParam){
        Item item = findById(id);
        item.setItemName(updateParam.getItemName());
    	item.setPrice(updateParam.getPrice());
        item.setQuantity(updateParam.getQuantity());
    }
    
    public void clearStore(){
        store.clear();
    }
}
```

기본적인 도메인과 레파지토리를 만든다.

## 상품 목록

```java
@Controller 
@RequestMapping("basic/items")
@RequiredArgsConstructor 
public class BasicItemController {
    
    private final ItemRepository itemRepository;
    
    @GetMapping 
    public String items(Model model){
        List<Item> items = itemRepository.findAll();
        model.addAttribute("items", items);
        return "basic/items";
    }
    
    @PostConstruct 
    public void init(){
        itemRepository.save(new Item("itemA", 100000, 1000));        
        itemRepository.save(new Item("itemB", 200000, 10));

    }
}
```

레파지토리를 통해 상품 목록을 가져오고 Model을 통해서 데이터를 뿌려준다.

```html
<!DOCTYPE HTML>
<html xmlns:th="http://thymeleaf.org">
<head>
 <meta charset="utf-8">
 <link th:href="@{/css/bootstrap.min.css}"
       href="../css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
  <div class="container" style="max-width: 600px">
     <div class="py-5 text-center">
       <h2>상품 목록</h2>
     </div>
     <div class="row">
       <div class="col">
         <button class="btn btn-primary float-end"
         onclick="location.href='addForm.html'"
                 th:onclick="|location.href='@{/basic/items/add}'|"
                 type="button">상품 등록
          </button>
        </div>
      </div>
      <hr class="my-4">
      <div>
      <table class="table">
      <thead>
      <tr>
      <th>ID</th>
      <th>상품명</th>
      <th>가격</th>
      <th>수량</th>
      </tr>
      </thead>
      <tbody>
      <tr th:each="item : ${items}">
         <td><a href="item.html" th:href="@{/basic/items/{itemId}(itemId = ${item.id})}" th:text="${item.id}">1</a></td>
         <td><a href="item.html" th:href="@{/basic/items/{itemId}(itemId = ${item.id})}" th:text="${item.itemName}">테스트 상품1</a></td>
         <td th:text="${item.price}">10000</td>
         <td th:text="${item.quantity}">10</td>
      </tr>
      </tbody>
      </table>
      </div>
  </div> <!-- /container -->
</body>
</html>
```

`thymeleaf`를 이용해 정적 html코드를 동적으로 만든다.

- `xmlns:th="http://thymeleaf.org"` 를 써서 thymeleaf를 쓴다는것을 선언한다.
  
- `||` 를 쓰면 `+`를 쓰지 않고도 문자열을 조합할 수 있다. `예시) "|location.href='@{/basic/items/add}'|"`
  
- thymeleaf에서 링크를 표현할때 `@{}`를 쓴다. 경로 변수나 쿼리 파라미터도 생성할 수 있다. `예시) th:href="@{/basic/items/{itemId}(itemId=${item.id}, query='test')}"`
  
- `th:each`를 쓰면 반복 출력을 할 수 있다.

> 타임리프는 th가 붙은 부분을 서버사이드로 렌더링하고 없는 부분은 기존 html 속성을 그대로 사용한다.

## 상품 상세

```java
@GetMapping("/{itemId}")
public String item(@PathVariable Long itemId, Model model){
    Item item = itemRepository.findById(itemId);
    model.addAttribute("item", item);
    return "basic/item";
}
```

상품 상세 페이지는 경로변수를 통해서 아이템을 조회하고 데이터를 넣어주면 된다. 

## 상품 등록

```java
@GetMapping("/add")
public String addForm(){
    return "basic/addForm";
}
```

상품 등록 페이지를 띄워주는 경로다.

```html
<!DOCTYPE HTML>
<html xmlns:th="http://thymeleaf.org">
<head>
 <meta charset="utf-8">
 <link th:href="@{/css/bootstrap.min.css}"
       href="../css/bootstrap.min.css" rel="stylesheet">
 <style>
 .container {
 max-width: 560px;
 }
 </style>
</head>
<body>
  <div class="container">
   <div class="py-5 text-center">
     <h2>상품 등록 폼</h2>
   </div>
   <h4 class="mb-3">상품 입력</h4>
   <form action="item.html" th:action method="post">
     <div>
       <label for="itemName">상품명</label>
       <input type="text" id="itemName" name="itemName" class="form-control" placeholder="이름을 입력하세요">
     </div>
     <div>
       <label for="price">가격</label>
       <input type="text" id="price" name="price" class="form-control" placeholder="가격을 입력하세요">
     </div>
     <div>
       <label for="quantity">수량</label>
       <input type="text" id="quantity" name="quantity" class="form-control" placeholder="수량을 입력하세요">
     </div>
     <hr class="my-4">
     <div class="row">
     <div class="col">
     <button class="w-100 btn btn-primary btn-lg"  type="submit">상품 등록</button>
     </div>
     <div class="col">
     <button class="w-100 btn btn-secondary btn-lg"
    onclick="location.href='items.html'" th:onclick="|location.href='@{/basic/items}'|" type="button">취소</button>
     </div>
     </div>
   </form>
  </div> <!-- /container -->
</body>
</html>
```

form 부분을 보면 `th:action` 이라고 쓰여있다. 요청 url을 설계할 때 우리는 같은 url을 Get, Post로 구분하여 다른 메서드가 실행되게 할 것이기 때문에 등록 폼에서 등록을 해도 같은 url을 post로 요청한다. action에 아무값도 넣어주지 않으면 현재 페이지를 요청하기 때문에 아무값도 넣어주지 않는다.

```java
@PostMapping("/add")
public String addItemV1(@RequestParam String itemName,
 @RequestParam int price,
 @RequestParam Integer quantity,
 Model model) {
   Item item = new Item();
   item.setItemName(itemName);
   item.setPrice(price);
   item.setQuantity(quantity);
   itemRepository.save(item);
   model.addAttribute("item", item);
   return "basic/item";
}
```

Post이지만 form요청은 쿼리 파라미터형태로 넘어오기 때문에 `@RequestParam`을 사용할 수 있다.

```java
@PostMapping("/add")
public String addItemV2(@ModelAttribute("item") Item item, Model model) {
  itemRepository.save(item);
  //model.addAttribute("item", item); //자동 추가, 생략 가능
  return "basic/item";
}
```

`@ModelAttribute`를 쓰면 객체를 만들고 객체에 값을 넣어주는 과정을 생략할 수 있다. 또한 `("item")` 부분도 생략할 수 있다. 

이때 @ModelAttribute 를 쓰면 자동으로 모델에 객체를 넣어준다. 예제에서는 item을 모델에 넣어준다. 만약 이름을 생락하면 클래스의 첫글자만 소문자로 바꾼 이름으로 모델에 저장해준다.

## 상품 수정

```java
@GetMapping("/{itemId}/edit")
public String editForm(@PathVariable Long itemId, Model model){
    Item item = itemRepository.findById(itemId);
    model.addAttribute("item", item);
    return "basic/editForm";
}

@PostMapping("/{itemId}/edit")
public String edit(@PathVariable Long itemId, @ModelAttribute Item item){
    itemRepository.update(itemId, item);
    return "redirect:/basic/items/{itemId}";
}
```

별다른 내용은 없다. 하지만 상품을 등록할 때에는 단순히 뷰를 요청했지만 수정에서는 `redirect`를 통해서 다시 요청을 하고 있다. 
어떤 방법이 맞는것일까?

## PRG 패턴

정답은 `redirect`를 통해 뷰를 렌더링하는것이다. 기존의 상품 등록 페이지에서 새로고침을 계속해보면 무한으로 상품이 등록되는걸 볼 수 있다.

![prg_pre](/assets/img/prg_pre.PNG)

우리가 상품 등록 페이지를 처음 들어올 때 Get요청을 통해 화면을 불러온다. 그리고 상품을 등록하기 위해 버튼을 클릭하면 Post 요청을 하게 되고, 새로운 페이지를 보여주게 된다.
웹 브라우저의 새로 고침은 마지막에 서버에 전송한 데이터를 다시 전송한다. 따라서 우리가 마지막에 Post요청을 통해서 상품등록을 했기 때문에
새로고침을 하면 상품등록이 계속 되는것이다.

![prg](/assets/img/prg.PNG)

이것을 PRG패턴을 통해서 해결할 수 있다. 상품을 등록하고 뷰 템플릿으로 이동하는 것이 아니라 `redirect`를 통해서 새로운 화면을 요청해주면된다.
redirect를 하면 새로운 화면을 Get으로 요청하기 때문에 새로고침을해도 문제가 없다. 

> Post, Redirect, Get 을 줄여서 PRG라고 한다.

```java
@PostMapping("/add")
public String add(@ModelAttribute Item item, RedirectAttributes redirectAttributes){
    itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", item.getId());
    redirectAttributes.addAttribute("status", true);

    return "redirect:/basic/items/{itemId}";
}
```

리다이렉트를 요청할 때 `"redirect:/basic/items/" + item.getId()" ` 이런식으로 해도 되지만 인코딩이 문제가 될 때도 있다. 
`redirectAttribute`를 사용하면 이런 문제를 해결할 수 있다. addAttribute를 통해서 path 파라미터를 추가할 수 있고, 만약에 path 파라미터에 들어가 있지 않으면
쿼리 파라미터에다가 추가를 해준다. 따라서 위의 예제는 `redirect:/basic/items/2?status=true` 이런식으로 리다이렉트가 된다.

```html
<div class="container">
<div class="py-5 text-center">
<h2>상품 상세</h2>
</div>
<!-- 추가 -->
<h2 th:if="${param.status}" th:text="'저장 완료!'"></h2> 
```

`th:if`를 통해서 조건에 맞으면 해당 요소를 렌더링해줄 수 있다.
