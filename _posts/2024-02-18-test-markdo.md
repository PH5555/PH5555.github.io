---
layout: post
title: Spring - 검증
tags: [Spring, 스프링 MVC]
comments: true
---

## 파라미터 검증

우리는 사용자로부터 값을 받아서 아이템을 생성, 수정하는 기능을 만들었다. 하지만 폼 데이터마다 제약조건이 붙는다고 해보자. 가격은 10000원 이상이어야하고 이름은 필수로 들어가야한다. 검증코드를 추가함으로써 이러한 요구사항을 처리할 수 있다.
컨트롤러의 중요한 기능중 하나가 HTTP요청이 정상인지 확인하는 것이다.

> 검증은 컨트롤러, 서비스, 레파지토리 모든 부분에서 하는게 제일 확실하다. 하지만 요청에 대한 부분은 최대한 컨트롤러에서 처리하고 서비스에 필요한 부분은 서비스에서 검증하는게 좋다.

> 프론트엔드에서도 검증을하고 백엔드에서도 검증을 해야한다. 프론트엔드에서 검증을 한다해도 검증은 신뢰되지않는다. 왜냐하면 프론트코드는 유저에게 노출되고, HTTP요청을 조작할 수 있기 때문이다. 그래서 반드시 이중검증을 해야한다.

## 검증 개발

HTTP요청을 검증하고 검증이 실패하면 고객에게 다시 등록 폼을 보여주고 에러메시지를 보여준다.

```java
@PostMapping("/add")
public String addItem(@ModelAttribute Item item, RedirectAttributes redirectAttributes, Model model) {
    
    Map<String, String> errors = new HashMap<>();
    
    if(!StringUtils.hasText(item.getItemName())){
        //stringutils 는 null 뿐만 아니라 빈 문자열도 포함하고 있음
        errors.put("itemName", "아이템 이름은 필수 요소입니다.");
    }
    if(item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000){
        errors.put("price", "가격은 1,000원 이상 1,000,000원 이하입니다.");
    }
    if(item.getQuantity() == null || item.getQuantity() > 9999){
        errors.put("quantity", "수량은 9999개 이하입니다.");
    }
    
    if(item.getQuantity() != null && item.getPrice() != null){
        int result = item.getQuantity() * item.getPrice();
        if(result < 10000){
          errors.put("global", "수량 * 가격이 10,000원 이상이어야합니다.");
        }
    }
    
    if(!errors.isEmpty()){
        model.addAttribute("errors", errors);
        return "validation/v1/addForm";
    }
    
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v1/items/{itemId}";
}
```

에러 내용을 저장하기 위해 `Map<String, String> errors = new HashMap<>();` 으로 맵데이터를 생성하고, 각 데이터마다의 조건이 부합하지 않으면 에러를 추가한다. 그리고 에러가 존재하면 `return "validation/v1/addForm";`를 하여 등록폼으로 돌아온다.

> Stringutils.hasText() 는 null 뿐만 아니라 비어있는 문자열(" ")인지도 확인해준다.

```html
<div class="container">

    <div class="py-5 text-center">
        <h2 th:text="#{page.addItem}">상품 등록</h2>
    </div>

    <form action="item.html" th:action th:object="${item}" method="post">
        <div th:if="${errors?.containsKey('global')}">
            <p class="field-error" th:text="${errors['global']}">
                글로벌 에러
            </p>
        </div>
        <div>
            <label for="itemName" th:text="#{label.item.itemName}">상품명</label>
            <input type="text" id="itemName" th:field="*{itemName}" class="form-control" placeholder="이름을 입력하세요" th.class="${errors?.containsKey('itemName')} ? 'form-control field-error' : 'form-control'">
            <div class="field-error" th:if="${errors?.containsKey('itemName')}" th:text="${errors['itemName']}">
                
            </div>
        </div>
        <div>
            <label for="price" th:text="#{label.item.price}">가격</label>
            <input type="text" id="price" th:field="*{price}" class="form-control" placeholder="가격을 입력하세요" th.class="${errors?.containsKey('price')} ? 'form-control field-error' : 'form-control'">
            <div class="field-error" th:if="${errors?.containsKey('price')}" th:text="${errors['price']}">
                
            </div>
        </div>
        <div>
            <label for="quantity" th:text="#{label.item.quantity}">수량</label>
            <input type="text" id="quantity" th:field="*{quantity}" class="form-control" placeholder="수량을 입력하세요" th.class="${errors?.containsKey('quantity')} ? 'form-control field-error' : 'form-control'">
            <div class="field-error" th:if="${errors?.containsKey('quantity')}" th:text="${errors['quantity']}">
                
            </div>
        </div>

        <hr class="my-4">

        <div class="row">
            <div class="col">
                <button class="w-100 btn btn-primary btn-lg" type="submit" th:text="#{button.save}">상품 등록</button>
            </div>
            <div class="col">
                <button class="w-100 btn btn-secondary btn-lg"
                        onclick="location.href='items.html'"
                        th:onclick="|location.href='@{/validation/v1/items}'|"
                        type="button" th:text="#{button.cancel}">취소</button>
            </div>
        </div>

    </form>

</div> <!-- /container -->
</body>
</html>
```

`errors?.containsKey('global')}` 를 이용해 Map에 해당 키가 있는지 확인하고 있다면 해당 에러의 메세지를 출력해준다. 

> errors 는 에러가 없다면 null 이 올 수도 있다. 따라서 `?`를 붙여서 null이 오면 무시하도록 할 수 있다.

위 코드를 실행해보면 값에 따라 에러 메시지가 잘 나오는 것을 볼 수 있다. 또한, 에러가 발생해서 등록폼으로 다시 돌아와도 값이 유지되는것을 볼 수 있다. 그 이유는 `@ModelAttribute`가 model에 값을 넣어주는 기능도 있기 때문에 넘어온 값을 그대로 model에 값을 주어서 뷰를 요청하기 때문이다.
하지만 데이터타입이 달라지면(Integer타입이 들어가는 곳에 문자열 입력) 400에러가 뜬다.

## BindingResult

```java
@PostMapping("/add")
public String addItemV1(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {

    if(!StringUtils.hasText(item.getItemName())){
        bindingResult.addError(new FieldError("item", "itemName", "아이템 이름은 필수 요소입니다."));
    }
    if(item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000){
        bindingResult.addError(new FieldError("item", "price", "가격은 1,000원 이상 1,000,000원 이하입니다."));
    }
    if(item.getQuantity() == null || item.getQuantity() > 9999){
        bindingResult.addError(new FieldError("item", "quantity", "수량은 9999개 이하입니다."));
    }
    
    if(item.getQuantity() != null && item.getPrice() != null){
        int result = item.getQuantity() * item.getPrice();
        if(result < 10000){
            bindingResult.addError(new ObjectError("item",  "수량 * 가격이 10,000원 이상이어야합니다."));
        }
    }
    
    if(bindingResult.hasErrors()){
        return "validation/v2/addForm";
    }
    
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v2/items/{itemId}";
}
```

BindingRusult를 사용해서 에러를 더 쉽게 처리할 수 있다.

> BindingResult의 위치는 @ModelAttribute 뒤에 위치해야한다.

BindingResult의 addError()를 이용해서 에러를 추가할 수 있다. addError에는 두가지 종류의 에러를 넣을 수 있는데 FieldError는 하나의 필드에 대한 에러이고, ObjectError는 복합 필드에 대한 에러이다.

- FieldError
  
  obejctName : @ModelAttribute 이름이다.
  
  field : 에러가 발생한 필드이다.
  
  defaultMessage : 에러가 발생했을 때 보여주는 메시지이다.
  
- ObjectError

  obejctName : @ModelAttribute 이름이다.
  
  defaultMessage : 에러가 발생했을 때 보여주는 메시지이다.

```html
<div class="container">

    <div class="py-5 text-center">
        <h2 th:text="#{page.addItem}">상품 등록</h2>
    </div>

    <form action="item.html" th:action th:object="${item}" method="post">
        <div th:if="${#fields.hasGlobalErrors()}">
            <p class="field-error" th:each="error : ${#fields.globalErrors()}" th:text="${error}">
                글로벌 에러
            </p>
        </div>
        <div>
            <label for="itemName" th:text="#{label.item.itemName}">상품명</label>
            <input type="text" id="itemName" th:field="*{itemName}" class="form-control" placeholder="이름을 입력하세요" th.errorClass="field-error">
            <div class="field-error" th:errors="*{itemName}">
                
            </div>
        </div>
        <div>
            <label for="price" th:text="#{label.item.price}">가격</label>
            <input type="text" id="price" th:field="*{price}" class="form-control" placeholder="가격을 입력하세요" th.errorClass="field-error">
            <div class="field-error" th:errors="*{price}">
                
            </div>
        </div>
        <div>
            <label for="quantity" th:text="#{label.item.quantity}">수량</label>
            <input type="text" id="quantity" th:field="*{quantity}" class="form-control" placeholder="수량을 입력하세요" th.errorClass="field-error">
            <div class="field-error" th:errors="*{quantity}">
                
            </div>
        </div>

        <hr class="my-4">

        <div class="row">
            <div class="col">
                <button class="w-100 btn btn-primary btn-lg" type="submit" th:text="#{button.save}">상품 등록</button>
            </div>
            <div class="col">
                <button class="w-100 btn btn-secondary btn-lg"
                        onclick="location.href='items.html'"
                        th:onclick="|location.href='@{/validation/v2/items}'|"
                        type="button" th:text="#{button.cancel}">취소</button>
            </div>
        </div>

    </form>

</div> <!-- /container -->
</body>
</html>
```

bindingResult를 사용하면 
