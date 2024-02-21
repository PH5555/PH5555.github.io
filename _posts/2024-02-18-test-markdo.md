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

bindingResult를 사용하면 타임리프에서 오류를 검증할 수 있는 편리한 기능을 사용할 수 있다.

- #fields : bindingResult 에서 에러를 가져올 수 있다.
  
- th:errors : 해당 필드에 에러가 있으면 에러메시지를 출력해준다. `th:if="${errors?.containsKey('quantity')}" th:text="${errors['quantity']}` 를 줄인 버전이다.
  
- th:errorClass : th:field에서 지정한 필드에 에러가 있으면 클래스를 추가해준다.

bindingResult를 추가하면 바인딩에러(형변환)가 발생해도 에러페이지로 가지 않고, 오류정보를 bindingResult에 추가해서 컨트롤러를 정상호출한다. 하지만 바인딩 에러가 발생하게 되면, modelAttribute 에 들어가지 않기 때문에 값이 저장되지는 않는다.

## FieldError, ObjectError 파라미터 추가

```java
@PostMapping("/add")
public String addItemV2(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {

    if(!StringUtils.hasText(item.getItemName())){
        bindingResult.addError(new FieldError("item", "itemName", item.getItemName(), false, null, null, "아이템 이름은 필수 요소입니다."));
    }
    if(item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000){
        bindingResult.addError(new FieldError("item", "price", item.getPrice(), false, null, null, "가격은 1,000원 이상 1,000,000원 이하입니다."));
    }
    if(item.getQuantity() == null || item.getQuantity() > 9999){
        bindingResult.addError(new FieldError("item", "quantity", item.getQuantity(), false, null, null, "수량은 9999개 이하입니다."));
    }
    
    if(item.getQuantity() != null && item.getPrice() != null){
        int result = item.getQuantity() * item.getPrice();
        if(result < 10000){
            bindingResult.addError(new ObjectError("item", null, null,  "수량 * 가격이 10,000원 이상이어야합니다."));
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

```java
public FieldError(String objectName, String field, String defaultMessage);
public FieldError(String objectName, String field, @Nullable Object
rejectedValue, boolean bindingFailure, @Nullable String[] codes, @Nullable
Object[] arguments, @Nullable String defaultMessage)
```

FieldError는 두가지 생성자를 제공한다.

- rejectValue : 거절된 값이다. 즉, 사용자가 입력한 값이다.
- bindingFailure : 바인딩 에러인지 아닌지
- codes : 메시지 코드이다.
- arguments : 메시지 코드에 들어가는 변수이다.
- defaultMessage : 기본 메시지이다. 메시지 코드가 존재하지 않는다면 기본 메시지가 출력된다.

```java
new FieldError("item", "price", item.getPrice(), false, null, null, "가격은 1,000 ~ 1,000,000 까지 허용합니다.")
```

3번째 파라미터에 값을 넣어주면 값을 저장할 수 있다. 

> th:field 는 에러가 없을 때는 Model 객체의 값을 출력하지만 에러가 있으면 bindingResult의 값을 출력한다.


## 에러 메시지 체계화

```
required.item.itemName=상품 이름은 필수입니다.
range.item.price=가격은 {0} ~ {1} 까지 허용합니다.
max.item.quantity=수량은 최대 {0} 까지 허용합니다.
totalPriceMin=가격 * 수량의 합은 {0}원 이상이어야 합니다. 현재 값 = {1}
```

errors.properties 파일을 생성하고 에러 메시지를 넣는다.

```java
@PostMapping("/add")
public String addItemV3(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {

    if(!StringUtils.hasText(item.getItemName())){
        bindingResult.addError(new FieldError("item", "itemName", item.getItemName(), false, new String[]{"required.item.itemName"}, null, null));
    }
    if(item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000){
        bindingResult.addError(new FieldError("item", "price", item.getPrice(), false, new String[]{"range.item.price"}, new Object[]{1000, 1000000}, null));
    }
    if(item.getQuantity() == null || item.getQuantity() > 9999){
        bindingResult.addError(new FieldError("item", "quantity", item.getQuantity(), false, new String[]{"max.item.quantity"}, new Object[]{1000000}, null));
    }
    
    if(item.getQuantity() != null && item.getPrice() != null){
        int result = item.getQuantity() * item.getPrice();
        if(result < 10000){
            bindingResult.addError(new ObjectError("item", new String[]{"totalPriceMin"}, new Object[]{1000, 1000000}, null));
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

에러메시지 코드는 배열로 넘겨서 여러값을 쓸 수 있는데, 처음부터 차례대로 쓰인다.(처음값이 존재하지 않으면 다음값 사용) 파라미터도 배열형식으로 넘겨주어서 메시지에 변수를 넣을 수 있다. 

## rejectValue

```java
@PostMapping("/add")
public String addItemV4(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {

    if(!StringUtils.hasText(item.getItemName())){
        bindingResult.rejectValue("itemName", "required");
    }
    if(item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000){
        bindingResult.rejectValue("price", "range", new Object[]{1000, 1000000}, null);

    }
    if(item.getQuantity() == null || item.getQuantity() > 9999){
        bindingResult.rejectValue("quantity", "max", new Object[]{1000000}, null);
    }
    
    if(item.getQuantity() != null && item.getPrice() != null){
        int result = item.getQuantity() * item.getPrice();
        if(result < 10000){
            bindingResult.rejectValue("quantity", "totalPriceMin", new Object[]{1000, 1000000}, null);
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

첫 인자로 필드, 두번째 인자로 메시지 코드, 세번째 인자로 메시지 코드 파라미터, 네번째 인자로 디폴트 메시지를 주면 된다. 근데 메시지 코드를 `required.item.itemName` 처럼 다쓰지 않았는데도 정상 출력된다. 그 비밀은 `MessageCodesResolver` 에 있다.

## 메시지 처리

에러코드를 자세하게 만들수도 있고, 간단하게 만들수도 있다.

`required.item.itemName` : 아이템 이름은 필수요소입니다. -> 자세함

`required` : 필수요소입니다. -> 간단함

자세하게 만들면 유저들에게 더 좋은 경험을 줄 수 있다. 하지만 개발을 할 때 범용성이 좋지않다. 간단하게 만들면 유저 경험이 좀 떨어지는 대신 개발을 할 때 범용성은 올라간다.

```
#Level1
required.item.itemName: 상품 이름은 필수 입니다.

#Level2
required: 필수 값 입니다.
```

그래서 우리는 메시지에 레벨을 둬서 범용성 있게 사용하다가 세밀한 메시지가 필요한 곳은 세밀한 메시지를 제공할 수 있다. 스프링에서는 이런 기능은 `MessageCodesResolver`를 통해서 제공한다. `MessageCodesResolver`는 다음과 같은 방법으로 메시지를 생성한다.

1.: code + "." + object name + "." + field
2.: code + "." + field
3.: code + "." + field type
4.: code

rejectValue는 내부에서 MessageCodesResolver를 사용한다. 그래서 `rejectValue("itemName", "required")` 라는 코드를 쓰면

1. required.item.itemName
2. required.itemName
3. required.java.lang.String
4. requried

이 4개의 에러메시지를 생성한다. 1부터 4까지 메시지코드가 있는지 검사하고 높은 우선순위부터 사용한다. 만약 다 없다면 디폴트메시지를 출력한다.

> BindingResult는 ModelAttribute 뒤에 쓰기 때문에 item 이라는 객체를 알고 있다. 따라서 rejectValue에 써주지 않아도 된다.

> 스프링에서 타입에러가 발생하면 typeMismatch 라는 에러가 발생하기 때문에 errors.properties 파일안에 typeMismatch 에러코드를 작성해주면 타입에러 메시지를 출력해줄 수 있다.

## Validator 분리

```java
@Component
public class ItemValidator implements Validator{
    @Override 
    public boolean supports(Class<?> clazz) {
        return Item.class.isAssignableFrom(clazz);
    }
    
    @Override 
    public void validate(Object target, Errors errors){
        Item item = (Item) target;
        
        if(!StringUtils.hasText(item.getItemName())){
        	errors.rejectValue("itemName", "required");
        }
        if(item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000){
        	errors.rejectValue("price", "range", new Object[]{1000, 1000000}, null);

        }
        if(item.getQuantity() == null || item.getQuantity() > 9999){
        	errors.rejectValue("quantity", "max", new Object[]{1000000}, null);
        }
        
        if(item.getQuantity() != null && item.getPrice() != null){
            int result = item.getQuantity() * item.getPrice();
            if(result < 10000){
                errors.rejectValue("quantity", "totalPriceMin", new Object[]{1000, 1000000}, null);
            }
        }
    }
}
```

컨트롤러에 검증을 하는 부분이 너무 많기 때문에 클래스로 따로 떼어줄 수 있다. Validator라는 인터페이스를 구현해준다. `supports`는 해당 검증기를 지원하는지 타입확인을 해준다. validate는 실제 검증로직이 들어가는 곳이다.

```java
private final ItemValidator itemValidator;

@PostMapping("/add")
public String addItemV5(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {

    itemValidator.validate(item, bindingResult);
    
    if(bindingResult.hasErrors()){
        return "validation/v2/addForm";
    }
    
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v2/items/{itemId}";
}
```

ItemValidator를 주입받고 validate메서드를 호출하면 된다.

하지만 클래스를 인터페이스를 통해 구현하지 않고 쌩으로 구현할 수 있지만 그렇게 하지 않는다. Validator를 구현함으로써 스프링에서 편리한 기능을 사용할 수 있다.

```java
@InitBinder
public void init(WebDataBinder webDataBinder){
    webDataBinder.addValidators(itemValidator);
}
```

`@InitBinder`를 통해서 검증기를 등록하면 해당 컨트롤러는 검증기를 자동으로 사용할 수 있다.

```java
@PostMapping("/add")
public String addItemV6(@Validated @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {

    if(bindingResult.hasErrors()){
        return "validation/v2/addForm";
    }
    
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v2/items/{itemId}";
}
```
 
validate 메서드를 따로 호출할 필요없이 `@Validated` 를 붙여주면 자동 검증이된다. 이 어노테이션을 붙이면 webDataBinder에 등록한 검증기를 찾아서 사용한다. 이때 webDataBinder에 여러 검증기가 등록되어있을 수도 있다. 이 때 내부에서 검증기 내부의 `supports` 메서드를 호출해주어서 알맞은 검증기인지 확인을 해준후 실행한다. 
