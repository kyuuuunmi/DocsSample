## 자동 프록시 생성 

1. 부가기능이 타깃 오브젝트마다 새로 만들어지는 문제 
> `ProxyFactoryBean`의 어드바이스를 통해 해결 

2. 부가기능의 적용이 필요한 타깃 오브젝트마다 비슷한 내용의 `ProxyFactoryBean` 빈 설정 정보를 추가하는 문제   
 => 중복 제거의 방법은? 
 
### 중복 문제의 접근 방법 

 `DAO`같은 코드 중복 경우, 전략 패턴과 DI를 이용하여 해결했다.  
 
 반복적인 위임 코드가 필요한 프록시 클래스 코드는 **다이내믹 프록시**라는 런타임 코드 자동생성 기법을 이용했다.  
 (JDK의 다이내믹 프록시는 특정 인터페이스를 구현한 오브젝트에 대해서 프록시 역할을 해주는 클래스를 런타임 시 내부적으로 만들어준다.)  
 
 
 변하지 않는 타깃으로의 위임과 부가기능 적용 여부 판단이라는 부분은 코드 생성 기법을 이용하는 다이내믹 프록시 기술에 맡기고,  
 변하는 부가기능 코드는 별도로 만들어서 다이내믹 프록시 생성 팩토리에 DI로 제공하는 방법을 사용한 것이다.  
 
 이는 의미있는 부가기능 로직인 트랜잭션 경계설정은 코드로 만들게 하고, 기계적인 코드인 타깃 인터페이스 구현과 위임, 부가기능 연동 부분은 자동생성하게 한 것이다.  
 
 반복적인 프록시의 메소드 구현을 코드 자동생성 기법을 이용해 해결했다면, 반복적인 `ProxyFactoryBean` 설정 문제는 설정 자동등록 기법으로 해결할 수 없을까? 
 
 

### 빈 후처리기를 이용한 자동 프록시 생성기  

스프링은 OCP의 가장 중요한 요소인 유연한 확장이라는 개념을 스프링 컨테이너 자신에게도 다양한 방법으로 적용하고 있다.  
변하지 않는 핵심적인 부분 외에는 대부분 확장할 수 있도록 확장 포인트를 제공해 주는데, 관심을 가질 만한 확장 포인트는 바로 `BeanPostProcessor`인터페이스를 구현해서 만드는 빈 후처리기다.  

**`DefaultAdvisorAutoProxyCreator`**는 어드바이저를 이용한 자동 프록시 생성기다  

스프링은 빈 후처리기가 빈으로 등록되어 있으면 빈 오브젝트가 생성될 때마다 빈 후처리기에 보내서 후처리 작업을 요청한다.  
이를 잘 이용하면 스프링이 생성하는 빈 오브젝트의 일부를 프록시로 포장하고, 프록시를 빈으로 대신 등록할 수도 있다. 이것이 **자동 프록시 생성 빈 후처리기**다.  


### 확장된 포인트컷  

원래 알던 포인트컷 : 타깃 오브젝트의 메소드 중에서 어떤 메소드에 부가기능을 적용할지를 선정해주는 역할  
확장된 포인트컷? : 등록된 빈 중에서 어떤 빈에 프록시를 적용할 것인지 선택 

포인트컷은 두 가지 기능을 모두 가지고 있다. 

`ProxyFactoryBean`에서는 굳이 클래스 레벨의 필터는 필요 없었지만, 모든 빈에 대해 프록시 자동 적용 대상을 선별해야 하는 빈 후처리기인 `DefaultAdvisorAutoProxyCreator`는 클래스와 메소드 선정 알고리즘을 모두 갖고 있는 포인트 컷이 필요하다.  

### 포인트컷 테스트 

`NameMatchMethodPointcut`이 클래스도 고를 수 있게 확장하고, 이 포인트컷을 적용한 `ProxyFactoryBean`으로 프록시를 만들도록 해서 과연 어드바이스가 적용되는지 아닌지를 확인해보자.  

포인트컷이 클래스 필터까지 동작해서 클래스를 걸러버리면 아무리 프록시를 적용했다고 해도 부가기능은 전혀 제공되지 않는다는 점에 주의해야 한다.  

## DefaultAdvisorAutoProxyCreator의 적용 


### 클래스 필터를 적용한 포인트컷 작성 
 
 메소드 이름만 비교하던 포인트컷인 `NamedMatchMethodPointcut`을 상속해서 프로퍼티로 주어진 이름 패턴을 가지고 클래스 이름을 비교하는 `ClassFilter`를 추가하도록 만들어보자.  
 
### 어드바이저를 이용하는 자동 프록시 생성기 등록 

`DefaultAdvisorAutoProxyCreator`은 등록된 빈 중에서 `Advisor`인터페이스를 구현한 것을 모두 찾는다.  

```xml
    <bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>

```
특이하게도 id 애트리뷰트가 없고 class뿐이다. 다른 빈에서 참조되거나 코드에서 빈 이름으로 조회될 필요가 없는 빈이라면 아이디를 등록하지 않아도 무방하다.  

### 포인트컷 등록 
기존의 포인트컷 설정을 삭제하고 새로 만든 클래스 필터 지원 포인트컷을 빈으로 등록한다.  

### 어드바이스와 어드바이저 
어드바이스인 `transactionAdvice`빈의 설정은 수정할 게 없다. 어드바이저 `transactionAdvisor`빈도 수정할 필요는 없다. 하지만 어드바이저로서 사용되는 방법은 바뀌었다.  
어드바이저를 이용하는 자동 프록시 생성기인 `DefaultAdvisorAutoProxyCreator`에 의해 자동수집되고, 프록시 대상 선정 과정에 참여하며, 자동생성된 프록시에 다이내믹하게 DI돼서 동작하는 어드바이저가 된다.  

### ProxyFactoryBean 제거와 서비스 빈의 원상복구  
더 이상 명시적인 프록시 팩토리 빈을 등록하지 않기 때문에 다시 `userService`로 id를 돌려 놓는다.  
 

### 자동생성 프록시 확인 

지금까지 한 일 : 트랜잭션 어드바이스를 적용한 프록시 자동 생성기를 빈 후처리기 메커니즘을 통해 적용함. 

**확인점**

 **1. 트랜잭션이 필요한 빈에 트랜잭션 부가기능이 적용됐는가**  
   > `upgradeAllOrNothing()`을 통해 확인. 트랜잭션 도중에 에러를 일으켜서 롤백됐는지 확인.
 
 **2. 아무 빈에나 트랜잭션이 부가기능이 적용된 것은 아닌가**
  > 프록시 자동생성기가 어드바이저 빈에 연결해둔 포인트컷의 클래스 필터를 이용해서 정확히 원하는 빈에만 프록시를 생성했는지 확인 필요. 
  즉 클래스 필터가 제대로 동작해서 프록시 생성 대상을 선별하고 있는지. 
   


## 포인트컷 표현식을 이용한 포인트컷 

스프링은 간단하고 효과적인 방법으로 포인트컷의 클래스와 메소드를 선정하는 알고리즘을 작성할 수 있는 방법을 제공한다. 
표현식 언어를 사용해서 포인트 컷을 작성할 수 있도록 하는 방법인데, 이를 **포인트컷 표현식**이라고 한다.  

### 포인트컷 표현식 
`AspectJExpressionPointcut`클래스 
클래스와 메소드의 선정 알고리즘을 포인트컷 표현식을 이용해 한 번에 지정할 수 있게 해준다. 

### 포인트컷 표현식 문법 
AspectJ 포인트컷 표현식은 포인트컷 지시자를 이용해 작성한다.  
가장 대표적으로 사용되는 것은 excution()이다.  

### 포인트컷 표현식 테스트 

- 타입패턴에 패키지를 표현할 때 '..'을 사용하면 서브패키지의 모든 클래스까지 포함한다.  
- 인터페이스를 선정조건으로 하면 인터페이스를 구현한 **메소드**에게만 포인트컷이 적용된다.  

### 포인트컷 표현식을 이용하는 포인트컷 적용 

확장하여, 특정 어노테이션이 타입, 메소드, 파라미터에 적용되어 있는 것을 보고 메소드를 선정하게 하는 포인트컷도 만들 수 있다. 
아래와 같이 쓰면 `Transactional`이라는 어노테이션이 적용된 메소드를 선정하게 해준다.  

`@annotation(org.springframework.transaction.annotation.Transactional)`

### 타입 패턴과 클래스 이름 패턴 
포인트컷 표현식의 클래스 이름에 적용되는 패턴은 클래스 이름 패턴이 아니라 타입패턴이다.  
슈퍼클래스의 패턴을 지정하면 상속받은 서브클래스들에게도 적용이 된다는 사실을 잊지말자.  


## AOP란 무엇인가? 

비즈니스 로직을 담은 `UserService`에 트랜잭션을 적용해온 과정을 정리해보자.  

### 트랜잭션 서비스 추상화 
트랜잭션 경계설정 코드를 비즈니스 로직을 담은 코드에 넣으면서 특정 트랜잭션 기술에 종속되는 코드가 돼버렸다. 
이는 추상적인 작업은 유지한 채로 구체적인 구현 방법을 자유롭게 바꿀 수 있도록 서비스 추상화 기법을 적용했다.  
구체적인 구현 내용을 담은 의존 오브젝트는 런타임 시에 다이내믹하게 연결해 주는 DI의 접근방법을 사용했다.  


### 프록시와 데코레이터 패턴 
트랜잭션 코드를 제거 했지만, 여전히 비즈니스 로직 코드에는 트랜잭션을 적용하고 있다는 사실은 드러나 있다. 
그래서 DI를 통해 데코레이터 패턴을 적용했다. 투명한 부가기능 부여를 가능하게 하는 데코레이터 패턴의 적용으로 비즈니스 로직을 담당하는 클래스도 자신을 사용하는 클라이언트와 DI관계를 맺을 이유를 찾게 됐다.  

### 다이내믹 프록시와 프록시 팩토리 빈 
비즈니스 로직 코드에서 트랜잭션 코드를 모두 제거할 수 있었지만, 비즈니스 로직 인터페이스의 모든 메소드마다 트랜잭션 기능을 부여하는 코드를 넣어 프록시 클래스를 만드는 작업이 짐이 됐다.  
기능을 부여하지 않아도 되는 메소드조차 프록시로서 위임 기능이 필요하기 떄문에 일일이 다 구현을 해줘야 했다.  

그래서 프록시 오브젝트를 런타임 시에 만들어주는 JDK 다이내믹 프록시 기술을 적용했다.  
하지만 동일한 기능의 프록시를 여러 오브젝트에 적용할 경우 오브젝트 단위로는 중복의 문제를 해결하지 못했는데, 
프록시 팩토리 빈을 통해 다이내믹 프록시 생성 방법에 DI를 도입함으로 해결했다. 
부가기능을 담은 어드바이스와 부가기능 선정 알고리즘인 포인트컷은 프록시에서 분리될 수 있었고, 여러 프록시에서 공유해서 사용할 수 있게 됐다.  
 
 
### 자동 프록시 생성 방법과 포인트 컷
트랜잭션 적용 대상이 되는 빈마다 일일이 프록시 팩토리 빈을 설정해줘야 한다는 부담이 남았다. 
이를 해결하기 위해 스프링 컨테이너의 빈 생성 후처리 기법을 활용하여 컨테이너 초기화 시점에서 자동으로 프록시를 만들어주는 방법을 도입했다.  

### 부가기능의 모듈화
트랜잭션 경계설정 기능 : **다른 모듈의 코드에 부가적으로 부여되는 기능** 
핵심기능과 같은 방식으로 모듈화 하기 힘들다. 그러므로 DI, 데코레이터 패턴, 다이내믹 프록시, 오브젝트 생성 후처리, 자동 프록시 생성, 포인트컷과 같은 기법을 필요로 한다.  
이 방법이 곧 핵심기능에 부여되는 부가기능을 효과적으로 모듈화하는 방법을 찾는 것이었다. 
> Advice + Pointcut => Advisor 

### AOP: Aspect Oriented Programming 
**부가기능의 모듈화** 
 **Aspect** : 그 자체로 어플리케이션의 핵심 기능을 담고 있진 않지만, 어플리케이션을 구성하는 중요한 한 가지 요소이고, 핵심기능에 부가되어 의미를 갖는 특별한 모듈을 가르킨다. 
 
 어플리케이션의 핵심적인 기능에서 부가적인 기능을 분리해서 Aspect라는 독특한 모듈로 만들어서 설계하고 개발하는 방법을 **Aspect Oriented Programming**(**AOP**)라고 부른다. 
 
 
 ## AOP 적용 기술 
 ###프록시를 이용한 AOP
 독립적으로 개발한 부가기능 모듈을 다양한 타깃 오브젝트의 메소드에 다이나믹하게 적용해주기 위해 가장 중요한 역할을 맡고 있는 게 바로 프록시다. 
 그래서 스프링 AOP는 프록시 방식의 AOP라고 할 수 있다.  
 
 ### 바이트코드 생성과 조작을 통한 AOP 
 AspectJ는 프록시를 사용하지 않는 대표적인 AOP기술이다.  
 AspectJ는 프록시처럼 간접적인 방법이 아니라, 컴파일된 타깃의 클래스 파일 자체를 수정하거나 클래스가 JVM에 로딩되는 시점을 가로채서 바이트코드를 조작하는 복잡한 방법을 사용한다. 
 
 왜 그렇게 복잡한 방법을 사용할까? 
 
 1. DI 컨테이너의 도움을 받아서 자동 프록시 생성 방식을 사용하지 않아도 된다. 
 2. 프록시 방식보다 훨씬 더 유연하고 강력한 AOP가 가능하기 때문이다. 
 
 ## AOP의 용어 
 
 #### 타깃 
 부가기능을 부여할 대상. 핵심기능을 담은 클래스일 수도, 부가기능을 제공하는 프록시 오브젝트일 수도. 
 
 #### 어드바이스 
 타깃에게 제공할 부가기능을 담은 모듈 
 
 #### 조인 포인트 
 어드바이스가 적용될 수 있는 위치 
 
 #### 포인트컷 
 어드바이스를 적용할 조인포인트를 선별하는 작업 또는 그 기능을 정의한 모듈. 스프링의 포인트컷은 메소드를 선정하는 기능을 가지고 있다. 
 
 #### 프록시 
 클라이언트와 타깃 사이에 투명하게 존재하면서 부가기능을 제공하는 오브젝트 
 
 #### 어드바이저 
 포인트컷과 어드바이스를 하나씩 갖고 있는 오브젝트 
 
 #### 애스팩트 
 OOP의 클래스처럼 AOP의 기본모듈. 한 개 또는 그 이상의 포인트컷과 어드바이스의 조합으로 만들어지며 보통 싱글톤 형태의 오브젝트로 존재함. 
 
 ## AOP 네임스페이스  
 
 스프링의 프록시 방식 AOP를 적용하려면 최소한 네 가지 빈을 등록해야 한다. 
 
 #### 자동 프록시 생성기 
 스프링의 `DefaultAdvisorAutoProxyCreator` 클래스를 빈으로 등록한다. 
 
 #### 어드바이스 
 부가기능을 구현한 클래스를 빈으로 등록한다. 
 
 #### 포인트컷 
 스프링의 `AspectJExpressionPointcut`을 등록하고 expression 프로퍼티에 포인트컷 표현식을 넣어준다. 

 #### 어드바이저 
 스프링의 `DefaultPointcutAdvisor` 클래스를 빈으로 등록해서 사용한다. 
 
 
 ### AOP 네임스페이스  
 스프링은 간편한 방법으로 AOP를 등록할 수 있도록 aop스키마를 제공한다.  
 


















































