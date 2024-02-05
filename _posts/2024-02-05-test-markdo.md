---
layout: post
title: Spring - MVC 프레임워크 만들기
tags: [Spring, 스프링 MVC]
comments: true
---

## JSP의 한계점을 극복해보자

저번 시간에 JSP로 MVC패턴을 구현해보고 공통적인 코드가 나온다는 한계점에 대해서 알아보았다. 이번 시간에는 `FrontController`라는 패턴을 통해 공통 로직을 처리해보겠다.

![frontcontroller](/assets/img/frontcontroller.PNG)

원래는 각 요청마다 다른 서블릿 객체에서 요청을 받았다. 하지만 `FrontController`패턴은 프론트 컨트롤러가 모든 요청을 다 받고 요청에 받는 각각의 컨트롤러를 호출하여 처리해준다. 따라서 프론트 컨트롤러만 서블릿(HttpServlet)을 써야하고 나머지 컨트롤러는 서블릿을 쓰지 않아도 된다.

실제 스프링 웹의 `DispatcherServlet`이 이 패턴으로 이루어져 있다.

## 프론트 컨트롤러 도입

![v1](/assets/img/v1.PNG)

매핑 정보에 원하는 컨트롤러를 모두 넣어놓고 요청이 들어오면 프론트 컨트롤러가 맞는 컨트롤러를 골라서 호출을 해주는 구조이다.

```java
public interface ControllerV1 {
    public void process(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException;
}
```

```java
public class MemberFormControllerV1 implements ControllerV1{
    @Override 
    public void process(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException {
        String viewPath = "/WEB-INF/views/new-form.jsp";
        RequestDispatcher dispatcher = req.getRequestDispatcher(viewPath);
        dispatcher.forward(req, res);
    }
}
```

컨트롤러는 공통 인터페이스를 만들어서 구현하도록 한다. 그리고 process로직에서 viewPath를 설정해주고 만약 해당 뷰에 필요한 데이터가 있으면 `attribute`에 넣어주고 forward를 통해서 뷰를 넘겨준다.

```java
@WebServlet(name = "frontControllerServletV1", urlPatterns = "/front-controller/v1/*")
public class FrontControllerServletV1 extends HttpServlet {
    
    Map<String, ControllerV1> controllerMap = new HashMap<>();
    
    public FrontControllerServletV1(){
        System.out.println("constructor");
        controllerMap.put("/front-controller/v1/members/new-form", new MemberFormControllerV1());
        controllerMap.put("/front-controller/v1/members/save", new MemberSaveControllerV1());
        controllerMap.put("/front-controller/v1/members", new MemberListControllerV1());
    }
    
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException{
    	  String path = req.getRequestURI();
        System.out.println(path);
        ControllerV1 controller = controllerMap.get(path);
        System.out.println(controller);
        if(controller == null){
            res.setStatus(HttpServletResponse.SC_BAD_REQUEST);
        }
        
        controller.process(req, res);
    }
}
```

프론트 컨트롤러에서는 생성자에서 controllerMap에 컨트롤러 정보를 모두 넣어준다. 그리고 요청받은 정보에서 URI를 통해서 컨트롤러 정보를 빼온다. 그 후, 해당 컨트롤러의 process 로직을 실행하면 된다. 

> urlPatterns 에서 `*`을 사용했다. /front-controller/v1 하위의 모든 url은 이 컨트롤러가 실행된다.

## View 분리

이전에 했던 작업의 컨트롤러들을 봐보자

```java
public void process(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException {
      String viewPath = "/WEB-INF/views/new-form.jsp";
      RequestDispatcher dispatcher = req.getRequestDispatcher(viewPath);
      dispatcher.forward(req, res);
}
```

```java
public void process(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException {
       List<Member> members = memberRepository.findAll();
       req.setAttribute("members", members);
       String viewPath = "/WEB-INF/views/members.jsp";
       RequestDispatcher dispatcher = req.getRequestDispatcher(viewPath);
       dispatcher.forward(req, res);
}
```

```java
public void process(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException {
      String name = req.getParameter("username");
      int age = Integer.parseInt(req.getParameter("age"));
      
      Member member = new Member(name, age);
      memberRepository.save(member);
      
      req.setAttribute("member", member);
      
      String viewPath = "/WEB-INF/views/save-result.jsp";
      RequestDispatcher dispatcher = req.getRequestDispatcher(viewPath);
      dispatcher.forward(req, res);
}
```

모든 컨트롤러에 viewPath를 지정하고 dispatcher를 생성하고 forward를 실행하는 공통로직이 있다. 이렇게 view를 처리해주는 부분을 분리 해줄것이다.

![v2](/assets/img/v2.PNG)

```java
public class MyView {
    private String viewPath;
    
    public MyView(String viewPath){
        this.viewPath = viewPath;
    }
    
    public void render(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException{
        RequestDispatcher dispatcher = req.getRequestDispatcher(viewPath);
        dispatcher.forward(req, res);
    }
}
```

먼저 view를 처리해주는 `MyView`클래스를 만들어준다. 이 클래스의 `render`메서드를 실행하면 view를 넘겨줄 수 있다.

```java
@WebServlet(name = "frontControllerServletV2", urlPatterns = "/front-controller/v2/*")
public class FrontControllerServletV2 extends HttpServlet {
    
    Map<String, ControllerV2> controllerMap = new HashMap<>();
    
    public FrontControllerServletV2(){
        controllerMap.put("/front-controller/v2/members/new-form", new MemberFormControllerV2());
        controllerMap.put("/front-controller/v2/members/save", new MemberSaveControllerV2());
        controllerMap.put("/front-controller/v2/members", new MemberListControllerV2());
    }
    
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException{
    	String path = req.getRequestURI();
        ControllerV2 controller = controllerMap.get(path);
        System.out.println(controller);
        if(controller == null){
            res.setStatus(HttpServletResponse.SC_BAD_REQUEST);
        }
        
        MyView myView = controller.process(req, res);
        myView.render(req, res);
    }
}
```

프론트 컨트롤러에서는 MyView를 컨트롤러에서부터 가져오고 MyView의 render를 호출해주면 된다.

```java
public MyView process(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException {
    List<Member> members = memberRepository.findAll();
    req.setAttribute("members", members);
    return new MyView("/WEB-INF/views/members.jsp");
}
```

그럼 process메서드가 이렇게 깔끔해진다! 

## 서블릿 종속성 제거

다시한번 이전에 구현했던 컨트롤러를 봐보자

```java
public class MemberListControllerV2 implements ControllerV2 {
    private MemberRepository memberRepository = MemberRepository.getInstance();
    
    @Override 
    public MyView process(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException {
        List<Member> members = memberRepository.findAll();
        req.setAttribute("members", members);
        return new MyView("/WEB-INF/views/members.jsp");
    }
}
```

```java
public class MemberSaveControllerV2 implements ControllerV2{
    private MemberRepository memberRepository = MemberRepository.getInstance();
    
    @Override 
    public MyView process(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException {
        String name = req.getParameter("username");
        int age = Integer.parseInt(req.getParameter("age"));
        
        Member member = new Member(name, age);
        memberRepository.save(member);
        
        req.setAttribute("member", member);
        return new MyView("/WEB-INF/views/save-result.jsp");
    }
}
```
process메서드 입장에서 HttpServletRequest, HttpServletResponse를 알필요가 있을까? 해당 컨트롤러는 프론트 컨트롤러를 거쳐서 오기 때문에 Map 객체로 필요한 정보들을 넘겨주면 컨트롤러가 서블릿 기술을 몰라도 실행이 된다. 서블릿에 종속되지 않으면 코드가 단순해지고, 테스트코드 작성이 쉬워진다.

![v3](/assets/img/v3.PNG)

원래는 프론트 컨트롤러에서 request, response 를 넘겨줬다면 이번에는 프론트 컨트롤러에서 데이터를 추출하여 객체로 미리 만들어 컨트롤러에 전달해준다. 그리고 `viewPath`의 앞부분 url이 중복되는 문제를 `viewResolver`에서 처리해준다.

```java
@Getter @Setter
public class ModelView{
    private String viewName;
    private Map<String, Object> model = new HashMap<>();
    
    public ModelView(String viewName){
        this.viewName = viewName;
    }
}
```

컨트롤러에서 request, response를 받지 않을거기 때문에 컨트롤러에서 attribute에 데이터를 넣을 수 없다. 따라서 return 받을 객체를 만들어준다. 

```java
@WebServlet(name = "frontControllerServletV3", urlPatterns = "/front-controller/v3/*")
public class FrontControllerServletV3 extends HttpServlet {
    
    Map<String, ControllerV3> controllerMap = new HashMap<>();
    
    public FrontControllerServletV3(){
        controllerMap.put("/front-controller/v3/members/new-form", new MemberFormControllerV3());
        controllerMap.put("/front-controller/v3/members/save", new MemberSaveControllerV3());
        controllerMap.put("/front-controller/v3/members", new MemberListControllerV3());
    }
    
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException{
    	  String path = req.getRequestURI();
        ControllerV3 controller = controllerMap.get(path);
        if(controller == null){
            res.setStatus(HttpServletResponse.SC_BAD_REQUEST);
        }
        
        Map<String, String> paramMap = new HashMap<>();
        Enumeration e = req.getParameterNames();
    		while ( e.hasMoreElements() ){
    			String name = (String) e.nextElement();
    			paramMap.put(name, req.getParameter(name));
    		}
        
        ModelView mv = controller.process(paramMap);
        String viewName = mv.getViewName();
        
        MyView myView = viewResolver(viewName);
        myView.render(mv.getModel(), req, res);
    }
    
    public MyView viewResolver(String viewName){
        return new MyView("/WEB-INF/views/" + viewName + ".jsp");
    }
}
```

paramMap을 생성하여 모든 파라미터 정보를 담아준다. 그리고 process의 인자로 paramMap을 넘겨주면 된다. 이 때 `viewResolver`를 이용해서 뷰의 이름만 넘겨주면 뷰를 넘겨받아서 사용할 수 있다.

```java
public class MemberSaveControllerV3 implements ControllerV3{
    private MemberRepository memberRepository = MemberRepository.getInstance();
    
    @Override
    public ModelView process(Map<String, String> paramMap){
        String name = paramMap.get("username");
        int age = Integer.parseInt(paramMap.get("age"));

        Member member = new Member(name, age);
        memberRepository.save(member);
        
        ModelView mv = new ModelView("save-result");
        mv.getModel().put("member", member);
        return mv;
    }
}
```

이렇게 되면 컨트롤러에서 Servlet 기술을 사용할 필요가 없게 된다!

## 사용하기 쉬운 프레임워크가 좋은 프레임워크다.

다시 한번 이전의 컨트롤러를 보자

public class MemberFormControllerV3 implements ControllerV3{
    @Override
    public ModelView process(Map<String, String> paramMap){
        return new ModelView("new-form");
    }
}

사실 이것도 엄청 잘 설계된 코드이지만 조금은 쓰기 불편해보인다. 사용자가 직접 ModelView 생성해서 넘겨주어야 한다. ModelView를 프론트 컨트롤러에서 만들어서 넘겨주면 컨트롤러가 더 깔끔해질거 같다.

![v4](/assets/img/v4.PNG)

구조는 거의 똑같고 ModelView 대신 viewName만 반환한다.

```java
@WebServlet(name = "frontControllerServletV4", urlPatterns = "/front-controller/v4/*")
public class FrontControllerServletV4 extends HttpServlet {
    
    Map<String, ControllerV4> controllerMap = new HashMap<>();
    
    public FrontControllerServletV4(){
        System.out.println("constructor");
        controllerMap.put("/front-controller/v4/members/new-form", new MemberFormControllerV4());
        controllerMap.put("/front-controller/v4/members/save", new MemberSaveControllerV4());
        controllerMap.put("/front-controller/v4/members", new MemberListControllerV4());
    }
    
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException{
    	  String path = req.getRequestURI();
        ControllerV4 controller = controllerMap.get(path);
        if(controller == null){
            res.setStatus(HttpServletResponse.SC_BAD_REQUEST);
        }
        
        Map<String, String> paramMap = new HashMap<>();
        Enumeration e = req.getParameterNames();
    		while ( e.hasMoreElements() ){
    			String name = (String) e.nextElement();
    			paramMap.put(name, req.getParameter(name));
    		}
        
        Map<String, Object> model = new HashMap<>();
        String viewName = controller.process(paramMap, model);
      
        MyView myView = viewResolver(viewName);
        myView.render(model, req, res);
    }
    
    public MyView viewResolver(String viewName){
        return new MyView("/WEB-INF/views/" + viewName + ".jsp");
    }
}
```

코드를 보면 `Map<String, Object> model = new HashMap<>();` 라는 것이 추가되었다. process의 인자로 model을 추가로 넘겨준다. 또한 render의 인자로도 model을 넘겨준다.

```java
public void render(Map<String, Object> model,  HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException{
    model.forEach((key, value) -> req.setAttribute(key, value));
    RequestDispatcher dispatcher = req.getRequestDispatcher(viewPath);
    dispatcher.forward(req, res);
}
```

Myview클래스에 해당 메서드를 추가해준다. model을 for문으로 돌려서 attribute에 모두 넣어주고 forward 해준다.

```java
public class MemberFormControllerV4 implements ControllerV4{
    @Override
    public String process(Map<String, String> paramMap, Map<String, Object> model){
        return "new-form";
    }
}
```

이렇게 하면 컨트롤러는 `new-form`이라는 문자열만 반환하면 된다!

프론트 컨트롤러는 더 복잡해졌지만 우리가 실제로 구현하는 컨트롤러 부부은 훨씬 간단해졌다

> 프레임워크나 공통 기능이 수고로워야 사용하는 개발자가 편리해진다.

## 유연한 컨트롤러

지금까지 프론트 컨트롤러를 V1부터 V4까지 만들어왔다. V4만 사용해야할 거 같지만 어떤 개발자는 자기는 V3를 사용하고 싶다고 한다.

```java
@WebServlet(name = "frontControllerServletV4", urlPatterns = "/front-controller/v4/*")
public class FrontControllerServletV4 extends HttpServlet {
    
    Map<String, ControllerV4> controllerMap = new HashMap<>();
    
    public FrontControllerServletV4(){
        System.out.println("constructor");
        controllerMap.put("/front-controller/v4/members/new-form", new MemberFormControllerV4());
        controllerMap.put("/front-controller/v4/members/save", new MemberSaveControllerV4());
        controllerMap.put("/front-controller/v4/members", new MemberListControllerV4());
    }
}
```

`controllerMap.put("/front-controller/v4/members/new-form", new MemberFormControllerV4());` 이 부분을 `new MemberFormControllerV3()` 이렇게 바꾼다고 해결될 문제일까? 아니다. `Map<String, ControllerV4> controllerMap = new HashMap<>();` 부분도 V3로 바꿔줘야한다. 즉, 지금의 프론트 컨트롤러는 한개 버전의 컨트롤러만을 쓸 수 있다.

![v5](/assets/img/v5.PNG)

이럴 때 사용하는 것이 어댑터 패턴이다. 핸들러(컨트롤러)가 저장되어 있는 곳에서 요청에 맞는 핸들러를 꺼내오고 핸들러에 맞는 핸들러 어댑터를 찾아준다. 그리고 컨트롤러를 직접 실행하는 것이 아니라 찾은 핸들러 어댑터를 통해서 실행시켜주면 된다.

```java
public class ControllerV3HandlerAdapter implements MyHandlerAdapter{
    @Override
    public boolean supports(Object handler){
        return (handler instanceof ControllerV3);
    }
    
    @Override
    public ModelView handle(HttpServletRequest req, HttpServletResponse res, Object handler) throws ServletException, IOException{
        ControllerV3 controller = (ControllerV3) handler;
        
        Map<String, String> paramMap = new HashMap<>();
        Enumeration e = req.getParameterNames();
    		while ( e.hasMoreElements() ){
    			String name = (String) e.nextElement();
    			paramMap.put(name, req.getParameter(name));
    		}
        
        ModelView mv = controller.process(paramMap);
        return mv;
    }
}
```

먼저 어댑터를 만들어준다. `support` 메서드는 알맞은 어댑터를 찾기 위해서 쓰인다. 이 어댑터는 V3 컨트롤러만을 위한 어댑터이므로 핸들러가 ControllerV3의 인스턴스인지 확인해준다. handle 메서드는 실제로 컨트롤러를 실행해주는 메서드이다. 핸들러를 ControllerV3로 형변환해준다.(handler가 ControllerV3라는 확실한 정보때문에 쓸 수 있다.) 그리고 이전에 했던것처럼 ModelView를 만들어서 반환하면 된다.

```java
@WebServlet(name = "frontControllerServletV5", urlPatterns = "/front-controller/v5/*")
public class FrontControllerServletV5 extends HttpServlet{
    private Map<String, Object> handlerMappingMap = new HashMap<>();
    private List<MyHandlerAdapter> handlerAdapters = new ArrayList<>();
    
    public FrontControllerServletV5(){
    	initHandlerMappingMap();
        initHandlerApaters();
    }
    
    public void	initHandlerMappingMap(){
        handlerMappingMap.put("/front-controller/v5/v3/members/new-form", new MemberFormControllerV3());
        handlerMappingMap.put("/front-controller/v5/v3/members/save", new MemberSaveControllerV3());
        handlerMappingMap.put("/front-controller/v5/v3/members", new MemberListControllerV3());
        
        handlerMappingMap.put("/front-controller/v5/v4/members/new-form", new MemberFormControllerV4());
        handlerMappingMap.put("/front-controller/v5/v4/members/save", new MemberSaveControllerV4());
        handlerMappingMap.put("/front-controller/v5/v4/members", new MemberListControllerV4());
    }
    
    public void initHandlerApaters(){
        handlerAdapters.add(new ControllerV3HandlerAdapter());
        handlerAdapters.add(new ControllerV4HandlerAdapter());
    }
    
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException{
        Object handler = getHandler(req);
        
        if(handler == null){
            res.setStatus(HttpServletResponse.SC_BAD_REQUEST);
        }
        
        MyHandlerAdapter adapter = getAdapter(handler);
        
        ModelView mv = adapter.handle(req, res, handler);
    
        String viewName = mv.getViewName();
        
        MyView myView = viewResolver(viewName);
        myView.render(mv.getModel(), req, res);
    }
    
    public Object getHandler(HttpServletRequest req){
        String path = req.getRequestURI();
        return handlerMappingMap.get(path);
    }
    
    public MyHandlerAdapter getAdapter(Object handler){
        for(MyHandlerAdapter adapter : handlerAdapters){
            if(adapter.supports(handler)){
                return adapter;
            }
        }
        return null;
    }
    
    public MyView viewResolver(String viewName){
        return new MyView("/WEB-INF/views/" + viewName + ".jsp");
    }
}
```

코드가 너무 기니 쪼개서 보겠다.

```java
public void	initHandlerMappingMap(){
    handlerMappingMap.put("/front-controller/v5/v3/members/new-form", new MemberFormControllerV3());
    handlerMappingMap.put("/front-controller/v5/v3/members/save", new MemberSaveControllerV3());
    handlerMappingMap.put("/front-controller/v5/v3/members", new MemberListControllerV3());
    
    handlerMappingMap.put("/front-controller/v5/v4/members/new-form", new MemberFormControllerV4());
    handlerMappingMap.put("/front-controller/v5/v4/members/save", new MemberSaveControllerV4());
    handlerMappingMap.put("/front-controller/v5/v4/members", new MemberListControllerV4());
}

public void initHandlerApaters(){
    handlerAdapters.add(new ControllerV3HandlerAdapter());
    handlerAdapters.add(new ControllerV4HandlerAdapter());
}
    
```

먼저 핸들러와 어댑터를 모두 등록해준다.(실제에서는 new-form은 v3를 쓰고 save는 v4를 쓰고 이렇게 한다.)

```java
public Object getHandler(HttpServletRequest req){
    String path = req.getRequestURI();
    return handlerMappingMap.get(path);
}
```

그리고 이전과 같이 URI로 맞는 핸들러(컨트롤러)를 찾아준다.

```java
public MyHandlerAdapter getAdapter(Object handler){
    for(MyHandlerAdapter adapter : handlerAdapters){
        if(adapter.supports(handler)){
            return adapter;
        }
    }
    return null;
}
```

그리고 어댑터 리스트에서 반복문을 돌며 타입이 맞는 어댑터를 찾아준다.

```java
ModelView mv = adapter.handle(req, res, handler);

String viewName = mv.getViewName();

MyView myView = viewResolver(viewName);
myView.render(mv.getModel(), req, res);
```

handle메서드를 실행하고 결과로 얻은 ModelView를 이용해서 뷰를 넘겨주면 된다.

근데 우리가 V4에서 코드를 단순화하는 과정에서 process의 결과를 ModelView가 아니라 문자열로 반환하게 설정해두었다. 이럴때 유용하게 어댑터 패턴이 쓰일 수 있다.

```java
@Override
public ModelView handle(HttpServletRequest req, HttpServletResponse res, Object handler) throws ServletException, IOException{
    ControllerV4 controller = (ControllerV4) handler;
    
    Map<String, String> paramMap = new HashMap<>();
    Enumeration e = req.getParameterNames();
    while ( e.hasMoreElements() ){
      String name = (String) e.nextElement();
      paramMap.put(name, req.getParameter(name));
    }
    
    Map<String, Object> model = new HashMap<>();
    String viewName = controller.process(paramMap, model);
    
    ModelView mv = new ModelView(viewName);
    mv.setModel(model);
    return mv;
}
```

process의 결과로 viewName을 받아서 어댑터에서 ModelView를 직접 생성해주면 된다. 이렇게 어댑터를 도입해서 유연한 설계를 할 수 있다.

