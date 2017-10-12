
스프링이 제공하는 템플릿/콜백 기술을 살펴보자. 스프링은 JDBC를 이용하는 DAO에서 사용할 수 있도록 준비된 다양한 템플릿과 콜백을 제공한다. 

## update()


```java
    public void deleteAll() throws SQLException {
        this.jdbcTemplate.update(new PreparedStatementCreator() {
            public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
                return con.prepareStatement("delete from users");
            }
        });
    }
```

앞에서 만들었던 `executeSql()`은 SQL문장만 전달하면 미리 준비된 콜백을 만들어서 템플릿을 호출하는 것까지 한 번에 해주는 편리한 메소드였다. 
`JdbcTemplate`에도 기능이 비슷한 메소드가 존재한다.
 
 ```java
public void deleteAll() throws SQLException {
        this.jdbcTemplate.update("delete from users");
    }
```


또한 `PreparedStatement`를 만들고 함께 제공하는 파라미터를 순서대로 바인딩해주는 기능을 하는 `update()`메소드를 사용할 수 있다. 
```java
 this.jdbcTemplate.update("insert into users(id, name, password) values (?,?,?)", user.getId(), user.getName(), user.getPassword());
```



## queryForInt()

`getCount()`는 SQL 쿼리를 실행하고 `ResultSet`을 통해 결과 값을 가져오는 코드다. 
이런 작업 흐름을 가진 코드에서 사용할 수 있는 템플릿은 `PreparedStatementCreator`콜백과 `ResultSetExtractor` 콜백을 파라미터로 받는 `query()`메소드다.

```java
    public int getCount() throws SQLException {
        return this.jdbcTemplate.query(new PreparedStatementCreator() {
            public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
                return con.prepareStatement("select count(*) from users");
            }
        }, new ResultSetExtractor<Integer>() {
            public Integer extractData(ResultSet rs) throws SQLException, DataAccessException {
                rs.next();
                return rs.getInt(1);
            }
        });
    }
```

이 콜백 오브젝트 코드는 재사용하기 좋은 구조인데, 이 역시 JDBCTemplate에서 제공한다. SQL의 실행 결과가 하나의 정수 값이 되는 경우, `queryForInt()`라는 메소드를 쓰면 한줄로 바꿀 수 있다. 
(`queryForObejct()`로 변경됨)

## queryForObejct()

`get()`메소드는 지금까지 만들었던 것 중에서 가장 복잡하다. 
일단 SQL은 바인딩이 필요한 치환자를 갖고 있고,
`ResultSet`에서 `getCount()`처럼 단순한 값이 아니라 복잡한 `User`오브젝트로 만들어야 한다. 

 이를 위해, `Rowmapper`콜백을 사용한다. 
 `ResultSetExtractor`는 `ResultSet`을 한 번 전달받아 알아서 추출 작업을 모두 진행하고 최종 결과만 리턴해주면 되는 데 반해, `RowMapper`는 `ResultSet`의 로우 하나를 매핑하기 위해 사용되기 때문에 여러 번 호출될 수 있다. 
 

```java
public User get(String id) throws ClassNotFoundException, SQLException {
        return this.jdbcTemplate.queryForObject("select * from users where id = ? ",
                new Object[]{id},
                new RowMapper<User>() {
                    public User mapRow(ResultSet rs, int rowNum) throws SQLException {
                        User user = new User();
                        user.setId(rs.getString("id"));
                        user.setName(rs.getString("name"));
                        user.setPassword(rs.getString("password"));
                        return user;
                    }
                });
    }
```

첫 번째 파라미터는 `PreparedStatement`를 만들기 위한 SQL이고, 두 번째는 여기에 바인딩할 값들이다. 
`update)()`에서처럼 가변인자를 사용하면 좋겠지만 뒤에 다른 파라미터가 있기 때문에 이 경우엔 가변인자 대신 `Object`타입 배열을 사용해야 한다. 
배열 초기화 블록을 사용해서 SQL의 ?에 바인딩할 id값을 전달한다.
 
 
 
## query()
### 기능 정의와 테스트 작성 
`getAll()`메소드를 추가한다. 
테이블의 모든 로우를 가져오고 결과를 `User` 오브젝트의 컬렉션으로 만든다. 
그리고 가져올 때에는 기본키인 `id`순으로 정렬해서 가져오도록 만들자. 

###query() 템플릿을 이용하는 getAll()구현 

```java
public List<User> getAll(){
        return this.jdbcTemplate.query("select * from users order by id", new RowMapper<User>() {
            public User mapRow(ResultSet rs, int rowNum) throws SQLException {
                User user = new User();
                user.setId(rs.getString("id"));
                user.setName(rs.getString("name"));
                user.setPassword(rs.getString("password"));

                return user;
            }
        });
    }
```

### 테스트 보안 
`query()`는 결과가 없을 경우에 `queryForObject()`처럼 예외를 던지지는 않는다. 대신에 크기가 0인 `List<T>` 오브젝트를 돌려준다. 
그대로 돌려주는 결과를 리턴하도록 만들자. 
테스트에는 검증 코드를 추가해서, `getAll()`에 대한 테스트 코드인 동시에 `getAll()`의 기능을 설명해주는 코드로 만든다. 

```java
   public void getAll() throws ClassNotFoundException, SQLException {
        dao.deleteAll();

        List<User> users0 = dao.getAll();
        assertThat(users0.size(), is(0));
        ...


```


## 재사용 가능한 콜백의 분리 

### DI를 위한 코드정리 

필요 없어진 `DataSource` 인스턴스 변수는 제거하자. 

### 중복 제거 
`User`용 `RowMapper` 콜백을 메소드에서 분리해 중복을 없애고 재사용되게 만든다. 
``` java
  private RowMapper<User> userRowMapper =
            new RowMapper<User>() {
                public User mapRow(ResultSet rs, int rowNum) throws SQLException {
                    User user = new User();
                    user.setId(rs.getString("id"));
                    user.setName(rs.getString("name"));
                    user.setPassword(rs.getString("password"));
                    return user;
                }
            };
```


### 템플릿콜백 패턴과 UserDao

`UserDao`에는 `User`정보를 DB에 넣거나 가져오거나 조작하는 방법에 대한 핵심적인 로직만 담겨 있다. 
`User`라는 자바 오브젝트와 USER 테이블 사이에 어떻게 정보를 주고 받을지, DB와 커뮤니케이션하기 위한 SQL문장이 어떤 것인지에 대한 최적화된 코드를 갖고 있다. 
응집도가 높다고 볼 수 있다.  

반면 JDBC API를 사용하는 방식, 예외처리, 리소스의 반납, DB 연결을 어떻게 가져올지에 관한 책임과 관심은 모두 JdbcTemplate에게 있다.
그런 면에서 책임이 다른 코드와는 낮은 결합도를 유지하고 있다.  



여기서 더 개선할 수 있는 사항을 살펴보자. 
1. `userMapper`를 아예 `UserDao `빈의 DI용 프로퍼티로 만들어버리면 어떨까?
2. DAO메소드에서 사용하는 SQL문장을 `UserDao`코드가 아니라 외부 리소스에 담고 이를 읽어와 사용하게 한다면? 







