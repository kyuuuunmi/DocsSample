스프링의 핵심을 담당하는 건, 바로 Bean Factory 또는 Application Context라고 불리는 것이다. 이 두 가지는 우리가 만든 DaoFactory가 하는 일을 좀 더 일반화하한 것 이라고 설명할 수 있다. 

## Object Factor를 이용한 스프링 IoC 
### Application Context와 설정정보 
- Bean : 스프링이 제어권을 가지고 직접 만들고 관계를 부여하는 Object. Object 단위의 Application Component를 말한다. 
- Bean Factory : Bean의 생성과 관계설정같은 제어를 담당하는 IoC 오브젝트(IoC의 기본 기능에 초점) 
- Application Context : Bean Factory보다 좀 더 확장한 것(App 전반에 걸쳐 모든 구성요소의 제어 작업을 담당하는 IoC 엔진의 의미)

### DaoFactory를 사용하는 어플리케이션 컨텍스트  
`@Configuration` : 빈 팩토리를 위한 오프젝트 설정을 담당하는 클래스라는 것을 인식할 수 있도록 하는 어노테이션. 어플리케이션 컨텍스트 또는 빈 펙토리가 사용할 설정정보라는 표시. 
`@Bean` : 오브젝트를 만들어주는 메소드. 오브젝트를 생성하고 초기화 해서 돌려줌. 오브젝트 생성을 담당하는 IoC용 메소드라는 표시 

## 어플리케이션 컨텍스트의 동작방식
어플리케이션 컨텍스트는 DaoFactory 클래스를 설정정보로 등록해두고 `@Bean`이 붙은 메소드의 이름을 가져와 빈 목록을 만들어둔다. 클라이언트가 어플리케이션 컨텍스트의 getBean() 메소드를 호출하면 자신의 빈 목록에서 요청한 이름이 있는지 찾고, 있다면 빈을 생성하는 메소드를 호출해서 오브젝트를 생성시킨 후 클라이언트에게 돌려준다. 
**Application Context를 사용했을 때의 장점** 
- 클라이언트는 구체적인 팩토리 클래스를 알 필요가 없다. 
- 종합 IoC 서비스를 제공해준다. 
- 빈을 검색하는 다양한 방법을 제공한다. 

## 용어 정리 
- **Bean** 
Spring이 직접 그 생성과 제어를 담당하는 오브젝트 
- **Bean Factory** 
스프링의 IoC를 담당하는 핵심 컨테이너. 보통 이를 확장한 Application Context를 이용한다. 
- **Application Context** 
빈 팩토리를 확장한 IoC 컨테이너. 빈을 등록하고 관리하는 기본적인 기능에 스프링이 제공하는 각종 부가 서비스를 추가로 제공한다. ApplicationContexgt는 BeanFactory를 상속한다. 
- **설정정보 / 설정 메타정보** 
IoC를 적용하기 위해 사용하는 메타정보. 
- **Container or IoC Container** 
IoC 방식으로 빈을 관리한다는 의미에서 어플리케이션 컨텍스트나 빈 팩토리를 컨테이너 또는 IoC 컨테이너라고도 한다. 
- **Spring Framework** 
스프링 프레임워크는 IoC컨테이너, 어플리케이션 컨텍스트를 포함해서 스프링이 제공하는 모든 기능을 통틀어 말할 때 주로 사용한다. 





