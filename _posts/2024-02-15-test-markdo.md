---
layout: post
title: Spring - 타임리프
tags: [Spring, 스프링 MVC]
comments: true
---

## 타임리프가 뭘까?!

타임리프는 JSP와 마찬가지로 MVC에서 VIEW를 담당하는 부분이다. JSP는 서블릿이라는 형태로 바꿔서 실행하게 되는데 이 서블릿이 자바로 이루어져 있기 때문에 JSP코드안에 자바 코드를 포함시킬 수 있다. 하지만 타임리프는 HTML, css 등을 처리할 수 있는 독립된 자바 템플릿 엔진이기 때문에 자바코드를 넣을 수 없다.
독립되어 있기 때문에 서버가 돌아가지 않아도 뷰를 띄울 수 있다.

속도적인 부분에서는 JSP가 더 좋다는 평이 있지만 타임리프가 스프링과 통합이 잘 되어있기 때문에 타임리프를 많이 쓰는 추세이다.

## 입력 폼 처리

```html
<form action="item.html" th:action th:object="${item}" method="post">
  <div>
    <label for="itemName">상품명</label>
    <input type="text" th:field="*{itemName}" class="formcontrol" placeholder="이름을 입력하세요">
  </div>
  <div>
    <label for="price">가격</label>
    <input type="text" th:field="*{price}" class="form-control"
    placeholder="가격을 입력하세요">
  </div>
  <div>
    <label for="quantity">수량</label>
    <input type="text" th:field="*{quantity}" class="formcontrol" placeholder="수량을 입력하세요">
  </div>
```

`th:object`를 사용하면 해당 form에서 사용할 데이터를 지정해줄 수 있다. th:object를 지정해주면 선택변수 식을 지정할 수 있다. input부분을 보면 `th:field="*{price}"` 이렇게 넣은 것을 볼 수 있다. th:field 를 지정해주면 자동으로 `id` `name` `value` 을 field로 지정한 변수이름과 변수로 설정해준다.

```html
<input type="text" th:field="*{itemName}" class="formcontrol" placeholder="이름을 입력하세요">
```

```html
<input type="text" id="itemName" name="itemName" value="kim" class="formcontrol" placeholder="이름을 입력하세요">
```

위의 코드가 서버에서 실행해보면 아래의 코드로 바뀌게 된다. 이는 수정폼에서 유용하게 사용될 수 있다. 수정폼을 사용할 때는 기존의 데이터를 value 부분에 넣어줘야하는데 `th:field`는 이런 고민없이 이를 한번에 처리해준다.

## 체크박스

```html
<div>
  <div class="form-check">
    <input type="checkbox" id="open" name="open" class="form-check-input">
    <label for="open" class="form-check-label">판매 오픈</label>
  </div>
</div>
```

체크박스를 체크하면 value가 true, 체크박스를 해제하면 false가 나와야할 것 같지만 실제로는 그렇지 않다. 체크박스를 해제하면 value가 아예 넘어오지 않는다. 

```html
<input type="hidden" name="_open" value="on"/>
```

그래서 이렇게 hidden form을 추가해서 _open 의 값만 넘어오면 spring이 체크박스가 해제되었다고 인식해서 값을 처리해줄 수 있다. 하지만 타임리프에서는 더 쉬운 방법을 제공한다.

```html
<div>
  <div class="form-check">
    <input type="checkbox" id="open" th:field="${item.open}" class="formcheck-input" disabled>
    <label for="open" class="form-check-label">판매 오픈</label>
  </div>
</div>
```

th:field 를 추가하면 서버에서 띄워줄 때 자동으로 hidden 태그를 만들어준다. 

```java
@GetMapping("/add")
public String addForm(Model model) {
    Map<String, String> regions = new LinkedHashMap<>();
    regions.put("SEOUL", "서울");
    regions.put("BUSAN", "부산");
    regions.put("JEJU", "제주");
    model.addAttribute("regions", regions);
    model.addAttribute("item", new Item());
    return "form/addForm";
}
```

체크박스의 요소를 추가하기 위해서 위와 같은 코드를 작성한다. 하지만 이런식으로 작성하면 모든 요청을 처리해주는 메서드마다 Map을 만들어주고 데이터를 추가해주고 Model에 넣어주는 작업을 반복해야한다. 

```java
@ModelAttribute("regions")
public Map<String, String> regions(){
    Map<String, String> regions = new LinkedHashMap<>();
    regions.put("SEOUL", "서울");
    regions.put("BUSAN", "부산");
    regions.put("JEJU", "제주");
    return regions;
}
```

위와 같은 방법을 사용하면 반복을 없앨 수 있다. `@ModelAttribute`의 새로운 기능으로 모든 요청의 Model에 값을 추가해주는 기능이 있다. @ModelAttribute의 괄호안의 값이 Model의 key로 들어간다. 그리고 메서드안에서 반환하는 값이 value가 된다. 

> 물론 이 코드 그대로 사용하면 요청을 받을 때마다 regions를 생성해야하므로 메모리 낭비가 된다. 따라서 따로 클래스에서 static으로 생성해주고 그 데이터를 넘겨주면 한 데이터로 돌려막기를 할 수 있다.

```html
<div>
  <div>등록 지역</div>
  <div th:each="region : ${regions}" class="form-check form-check-inline">
    <input type="checkbox" th:field="*{regions}" th:value="${region.key}" class="form-check-input">
    <label th:for="${#ids.prev('regions')}" th:text="${region.value}" class="form-check-label">서울</label>
  </div>
</div>
```

체크박스 데이터를 처리할 때는 th:each를 사용하면 된다. 이때 th:field를 이용해서 id를 생성하면 타임리프가 자동으로 id를 다르게 생성해준다.(region1, region2 ...) 또한 label 태그는 input의 id와 연결이 되어야한다. 근데 input의 id는 동적으로 생성이 되므로 알 수 없다.
이 때 `#ids.prev('regions')`을 사용하게 되는데, regions를 이용한 제일 최근의 태그의 id를 현재 태그의 id로 넣어준다. 제일 최근의 태그를 이용하기 때문에 label과 input의 순서를 바꾸면 작동이 되지 않는다. 이 때는 다음의 id를 사용하는 `#ids.next()`를 사용하면 된다.

> 셀렉트 박스와 라디오 버튼도 같은 방법이므로 생략
