
스프링은 `DaoFactory`와 같은 자바 클래스를 이용하는 것 외에도 다양한 방법을 통해 DI 의존관계 설정정보를 만들 수 있는데 가장 대표적인 것이 바로 XML이다.
 
 
 ## XML 설정 
 스프링의 Application Context는 XML에 담긴 DI 정보를 활용할 수 있다.
  - DI 정보 : `<beans>`를 루트 엘리먼트로 사용
  - `@Configuration` : `<bean>`
  - `@Bean` : `<bean>`
  
 **하나의 `@Bean` 메소드를 통해 얻을 수 있는 빈의 DI 정보**
 1. 빈의 이름 : `genBean()`에서 사용 
 2. 빈의 클래스 : 빈 오브젝트를 어떤 클래스에서 이용해서 만들지 정의
 3. 빈의 의존 오브젝트 : 빈의 생성자 또는 수정자 메소드를 통해 의존 오브젝트를 넣어준다. 
 
 -> xml에서 `<bean>`을 사용햊도 이 세가지 정보를 정의할 수 있다. 
 
 ### `connectionMaker()` 전환
 
 `<bean>` 태그의 어트리뷰트
 - `class` : 자바 메소드에서 오브젝트를 만들 때 사용하는 클래스 이름. 패키지까지 모두 포함해야 함  
 - `id` : 메소드 이름
 
 ### `userDao()` 전환 
 userDao에는 DI 정보의 세 가지 요소가 모두 들어 있다. 특히 수정자 메소드를 사용해 의존관계를 주입해주는 부분이 들어가 있다.  
 XML에서는 `<property>` 태그를 사용해 의존 오브젝트와의 관계를 정의한다. 
 
 `<property>` 태그의 어트리뷰트 
 - `name` : 프로퍼티의 이름 
 - `ref` : 수정자 메소드를 통해 주입해 줄 오브젝트 빈 이름
  
 ### XML의 의존관계 주입 정보
 
 완성된 XML 설정 정보 
 ```xml
 <beans>
    <bean id="connectionMaker" class="springbook.user.dao.DConnectionMaker"/>
    <bean id="userDao" class="springbook.user.dao.UserDao">
        <property name="connectionMaker" ref="connectionMaker" />
    </bean>
 </beans>
```
 주의할 점은 `name` 어트리뷰트는 DI에 사용할 수정자 메소드의 프로퍼티 이름이며 `ref` 어트리뷰트는 주입할 오브젝트를 정의한 빈의 ID이다.  
 같은 인터페이스를 구현한 의존 오브젝트를 여러 개 정의해두고 그중에서 원하는 것을 골라서 DI 하는 경우도 있다. 이때는 각 빈의 이름을 독립적으로 만들어두고 `ref` 어트리뷰트를 이용해 DI 받을 빈을 지정해주면 된다.
 
 ## XML을 이용하는 어플리케이션 컨텍스트 
 
 Application Context의 역할을 `DaoFactory` 대신 XML 설정정보를 활용하도록 한다.  
 `UserDaoTest`의 어플리케이션 컨텍스트 생성 부분을 `GenericXmlApplicationContext()` 로 수정하고, 인자로 XML 설정파일 클래스패스를 넣는다.
  
  
 ## `DataSource` 인터페이스로 변환 
 `ConnectionMaker`는 DB 커넥션을 생성해주는 기능 하나만을 정의한 매우 단순한 인터페이스다. 사실 자바에서는 DB 커넥션을 가져오는 오브젝트의 기능을 추상화해서 비슷한 용도로 사용할 수 있게 만들어진 `DataSource`라는 인터페이스가 이미 존재한다. 따라서 이를 가지고 `UserDao`를 리펙토링한다. 
 이를 사용하기 위해서는 `DataSource` 구현 클래스가 필요한데, `SimpleDriverDataSource`라는 것이 있는데, 이 클래스를 사용하도록 DI를 재 구성 해볼 것이다. 
  
  ### 자바 코드 설정 방식
  ```java
  @Bean
  public DataSource dataSource(){
    SimpleDriverDataSource dataSource = new SimpleDriverDataSource();
    
    dataSource.setDriverClass(com.mysql.jdbc.Driver.class);
    dataSource.setUrl("jdbc:mysql://localhost/spring?useSSL=false");
    dataSource.setUsername("username");
    dataSource.setPassword("password");
    
    return dataSource;
  }

```
 
 ### XML 설정 방식 
 
 ```xml
 <bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource"/>
```
 문제는, 오브젝트를 만드는 것까지는 가능하지만, DB 연결정보가 나타나 있지 않다. 
 
 ## 프로퍼티 값의 주입
 ### 값 주입 
 수정자 메소드에는 다른 빈이나 오브젝트뿐만 아니라 스트링 같은 단순 값을 넣어줄 수도 있다. 
 다른 빈 오브젝트의 레퍼런스가 아닌 단순 정보도 오브젝트를 초기화하는 과정에서 수정자 메소드에 넣을 수 있는데, 이는 클래스 외부에서 DB연결정보와 같이 변경 가능한 정보를 설정해줄 수 있도록 만들기 위함이다.   
 
 이렇게 텍스트나 단순 오브젝트 등을 수정자 메소드에 넣어주는 것을 스프링에서는 '값을 주입한다'라고 말한다. 사용할 오브젝트 자체를 바꾸지는 않지만 오브젝트의 특성은 외부에서 변경할 수 있는 성격으로 일종의 DI라고 볼 수 있다. 
 `<property>`의 `ref` 어트리뷰트 대신 `value` 어트리뷰트를 사용하여 설정한다. 
 
 ```xml

    <property name="driverClass" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost/spring?useSSL=false"/>
    <property name="username" value="username"/>
    <property name="password" value="password"/>
```
 `value` 어트리뷰트에 들어가는 것은 다른 빈의 이름이 아니라 실제로 수정자 메소드의 파라미터로 전달되는 스트링 그 자체다. 
 
 ### `value`값의 자동 변환 
 
 스프링은 `value`에 지정한 텍스트 값을 적절한 자바 타입으로 변환해준다. 
 