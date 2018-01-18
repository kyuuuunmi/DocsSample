
지금까지 기존 코드의 수정 없이 트랜잭션 부가기능을 추가해줄 수 있는 다양한 방법을 살펴봤다.  
스프링은 이에 대해서 어떤 해결책을 제시해 주는가



### ProxyFactoryBean 

스프링은 트랜잭션 기술과 메일 발송 기술에 적용했던 **서비스 추상화**를 **프록시** 기술에도 동일하게 적용하고 있다.  
즉, 프록시를 만들 수 있게 도와주는 추상 레이어를 제공하는데, 
생성된 프록시는 스프링의 빈으로 등록돼야 한다. 
스프링은 프록시 오브젝트를 생성해주는 기술을 추상화한 팩토리 빈을 제공해준다.
> `ProxyFactoryBean`은 프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈이다. 
순수하게 프록시를 생성하는 작업만을 담당하고 부가기능은 별도의 빈(`MethodInterceptor`)에 둘 수 있다.  

####`MethodInterceptor` VS `InvocationHandler`

**타깃 오브젝트에 대한 의존도**  
=> `invoke()`메소드가 타깃 오브젝트의 정보를 함께 받음 vs 받지 않음 


### 어드바이스: 타깃이 필요 없는 순수한 부가기능 

1 . `InvocationHandler`와 달리 `MethodInterceptor`를 구현한 `UppercaseAdvice`에는 타깃 오브젝트가 등장하지 않는다.  
-> 메소드 정보와 함꼐 타깃 오브젝트가 담긴 `MethodInvcoation`오브젝트가 전달되는데, 타깃 오브젝트의 메소드를 실행할 수 있는 기능이 있기 때문에 `MethodInterceptor`은 부가 기능을 제공하는 데만 집중할 수 있다.   

`MethodInvcoation`는 일종의 콜백 오브젝트. `proceed()`메소드를 실행하면 타깃 오브젝트의 메소드를 내부적으로 실행. 
일종의 공유 가능한 템플릿처럼 동작함.  

2 . 일반적인 DI가 아닌, `addAdvice()`라는 메소드 사용  
-> 여러개의 `MethodInterceptor`를 추가할 수 있음 

**Advice** 타깃 오브젝트에 적용하는 부가기능을 담은 오브젝트  

3 . 다이내믹 프록시 오브젝트의 타입을 결정할 수 있는 `Hello`인터페이스 제공 ?
`ProxyFactoryBean`에 인터페이스 자동검출 기능을 사용해 타깃 오브젝트가 구현하고 있는 인터페이스 정보를 알아 낼 수 있다. 
 그리고 알아낸 인터페이스를 모두 구현하는 프록시를 만들어준다.  
 
```java

 // InvocaionHandler를 이용한 다이내믹 프록시 생성
 Hello proxiedHello = (Hello) Proxy.newProxyInstance(
                getClass().getClassLoader(),
                new Class[]{Hello.class}, // 타깃 오브젝트의 인터페이스 정보를 넘겨준다. 
                new UppercaseHandler(new HelloTarget()));

 // ProxyFacotoryBean를 이용한 다이내믹 프록시 생성
  ProxyFactoryBean pfBean = new ProxyFactoryBean();
        pfBean.setTarget(new HelloTarget());
        pfBean.addAdvice(new UppercaseAdvice());

        Hello proxiedHello = (Hello)pfBean.getObject();
        
        
        
```

### 포인트컷 부가기능 적용 대상 메소드 선정 방법 

메소드의 이름을 가지고 부가기능을 적용 대상 메소드를 선정하는 절차도 있었다. 
이를 `ProxyFactoryBean` & `MethodInterceptor`에도 넣을 수 있을까? 

프록시에 부가기능 적용 메소드를 선택하는 기능을 넣자.  
프록시의 핵심 가치는 타깃을 대신해서 클라이언트의 요청을 받아 처리하는 오브젝트로서의 존재 자체이므로, 메소드를 선별하는 기능은 프록시로부터 다시 분리하는 편이 낫다. 

이러한 메소드 선정 알고리즘을 담은 오브젝트를 **포인트컷**이라고 부른다.  
**어드바이스**와 **포인트컷**은 모두 프록시에 DI로 주입돼서 사용된다. 

포인트컷이 필요 없을 때는 `ProxyFactoryBean`의 `addAdvice()`메소드를 호출해서 어드바이스만 등록하면 됐다. 그런데 포인트컷을 함께 등록할 때는 어드바이스와 포인트컷을 `Advisor`타입으로 묶어서 `addAdvisor()`메소드를 호출해야한다. 

이유는 `ProxyFactoryBean`에는 여러 개의 어드바이스와 포인트컷이 추가될 수 있기 때문이다. 

> 어드바이저 = 포인트컷(메소드 선정 알고리즘) + 어드바이스(부가기능)


## ProxyFacotoryBean 적용 

JDK 다이내믹 프록시의 구조를 그대로 이용해서 만들었던 `TxProxyFactoryBean`을 이제 스프링이 제공하는 `ProxyFactoryBean`을 이용하도록 수정해보자.  

### TransactionAdvice
 부가기능을 담당하는 어드바이스는 테스트에서 만들어본 것처럼 `MethodInterceptor`라는 `Advice`서브인터페이스를 구현해서 만든다.  
 
### 어드바이스와 포인트컷의 재사용 
`ProxyFactoryBean`은 스프링의 DI와 템플릿/콜백 패턴, 서비스 춯상화 등의 기법이 모두 적용된 것이다. 








ㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡ
ㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡ


####Issue 
> 여러개의 `MethodInterceptor`를 추가할 수 있음  

순서는 어떻게 ? 


  
  