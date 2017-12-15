
스프링의 모든 기술은 결국 개체지향적인 언어의 장점을 적극적으로 활용해서 코드를 작성하도록 도와주는 것이다. 
7장에서는 IoC/DI, 서비스 추상화, AOP 세 가지 기술을 애플리케이션 개발에 활용해서 새로운 기능을 만들어보고 
이를 통해 스프링의 개발철학과 추구하는 가치, 스프링 사용자에게 요구되는 게 무엇인지를 살펴보겠다.  


`UserDao`에서, SQL을 DAO에서 분리해 보자. 
데이터 엑세스 로직은 바뀌지 않더라도 DB의 테이블, 필드 이름과 SQL 문장이 바뀔 수 있기 때문에, SQL을 적절히 분리해 DAO 코드와 다른 파일이나 위치에 두고 관리할 수 있도록 만들어 보자.


## XML 설정을 이용한 분리
가장 쉬운 방법은 SQL을 스프링의 XML 설정파일로 빼내는 것이다. 
스프링은 설정을 이용해 빈에 값을 주입해줄 수 있다.
 
 ### 개별 SQL 프로퍼티 방식 
 
 `UserDaoJdbc`클래스의 SQL 6개를 프로퍼티로 만들고 이를 XML에서 지정하도록 해보자. 이렇게 하면 간단히 SQL을 DAO코드에서 분리할 수 있다.
  
  
 ### SQL 맵 프로퍼티 방식 
 하지만 SQL이 점점 많아지면 그때마다 DAO에 DI용 프로퍼티를 추가하기가 상당히 귀찮다. 
 그러므로 이번에는 SQL을 하나의 컬렉션으로 담아두는 방법을 시도해보자.  
 맵을 이용하면 키 값을 이용해 SQL 문장을 가져올 수 있다.
  
 `Map`은 하나 이상의 복잡한 정보를 담고 있기 때문에 `<property>` 태그의 `value` 애트리뷰트로는 정의해줄 수가 없다. 
 이때는 스프링이 제공하는 `<map>`태그를 사용해야 한다. 
 
 

## SQL 제공 서비스 

스프링 설정파일 안에 SQL을 두고 이를 DI 해서 DAO가 사용하게 하면 손쉽게 SQL을 코드에서 분리해낼 수 있긴 하지만 본격적으로 적용하기엔 몇 가지 문제점이 있다. 

1. SQL과 DI 설정정보가 섞여 있으면 보기에도 지저분하고 관리하기에도 좋지 않다. 
2. SQL을 꼭 스프링의 빈 설정 방법을 사용해 XML에 담아둘 이유도 없다. 
3. 스프링의 설정파일로부터 생성된 오브젝트와 정보는 애플리케이션을 다시 시작하기 전에는 변경이 매우 어렵다는 점도 문제다. 

문제점들을 해결하기 위해선 DAO가 사용할 SQL을 제공해주는 기능을 독립시킬 필요가 있다. 


### 서비스 인터페이스 
가장 먼저 할 일은 SQL 서비스의 인터페이스를 설계하는 것이다. DAO를 SQL 서비스의 구현에서 독립적으로 만들도록 인터페이스를 사용하고, DI로 구현 클래스의 오브젝트를 주입해주어야 한다. 


### 스프링 설정을 사용하는 단순 SQL 서비스 
`SqlService`인터페이스에는 어떤 기술적인 조건이나 제약사항도 담겨 있지 않다. 어떤 방법을 사용하든 상관없이 DAO가 요구하는 SQL을 돌려주기만 하면 된다. 

이제 `UserDao`를 포함한 모든 DAO는 SQL을 어디에 저장해두고 가져오는지에 대해서는 전혀 신경 쓰지 않아도 된다. 구체적인 구현 방법과 기술에 상관없이 `SqlService 인터페이스 타입의 빈을 DI 받앗 필요한 SQL을 가져다 쓰기만 하면 된다.

동시에 다양한 방법으로 구현된 `SqlService`타입 글래스를 적용할 수 있다. 이제 DAO의 수정 없이도 얼마나 편리하고 자유롭게 SQL 서비스 구현을 발전시ㅕ나갈 수 있는지 살펴보자. 


```java
public class SimpleSqlService implements SqlService {

    private Map<String, String> sqlMap;

    public void setSqlMap(Map<String, String> sqlMap) {
        this.sqlMap = sqlMap;
    }

    public String getSql(String key) throws SqlRetrievalFailureException {
        String sql = sqlMap.get(key);
        if (sql == null)
            throw new SqlRetrievalFailureException(key + "에 대한 SQL을 찾을 수 없습니다");
        else
            return sql;
    }
}

```
 
 
 