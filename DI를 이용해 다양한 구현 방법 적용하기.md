어느 정도 안전한 업데이트가 가능한 SQL레지스트리를 구현해보자.  




## `ConcurrentHashMap`을 이용한 수정 가능 SQL 레지스트리 
멀티스레드 환경에서 안전하게 `HashMap`을 조작하려면 `Collections.synchronizedMap()`등을 이용해 외부에서 동기화해줘야 한다. 
하지만 이렇게 `HashMap`에 대한 전 작업을 동기화하면 `SqlService`처럼 `DAO`의 요청이 많은 고성능 서비스에서는 성능에 문제가 생긴다. 
그래서 동기화된 해시 데이터 조작에 최적화되도록 만들어진 `ConcurrentHashMap`을 사용하는 방법이 권장된다.  
`ConcurrentHashMap`은 데이터 조작 시 전체 데이터에 대해 락을 걸지 않고 조회는 락을 아예 사용하지 않는다. 



### 수정 가능 SQL 레지스트리 테스트 

SQL 등록한 것이 잘 조회되는지 확인해야 하고, 
이를 수정한 후에 수정된 SQL이 바르게 적용됐는지 확인해봐야 한다. 
또 존재하지 않는 SQL에 대해 수정을 시도했을 때 적절한 예외가 발생하는지도 검증해보자. 


## 내장형 데이터베이스를 이용한 SQL 레지스트리 만들기 
`ConcurrentHashMap` 대신 내장형 DB를 이용해 SQL을 저장하고 수정하도록 만들어보자. 
내장형 DB는 애플리케이션에 내장돼서 어플리케이션과 함께 시작되고 종료되는 DB를 말한다. 

### 스프링의 내장형 DB 지원 기능 
스프링은 내장형 DB를 손쉽게 이용할 수 있도록 내장형 DB 지원 기능을 제공하고 있다. 일종의 내장형 DB를 위한 서비스 추상화 기능인데, 다른 서비스 추상화처럼 별도의 레이어와 인터페이스를 제공하지는 않는다. 
 대신 내장형 DB를 초기화하는 작업을 지원하는 편리한 내장형 DB 빌더를 제공한다. 
 
## 트랜잭션 적용

내장형 DB를 사용하는 경우에는 트랜잭션 적용이 상대적으로 쉽다. 
 
용 