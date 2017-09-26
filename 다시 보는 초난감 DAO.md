**개방 폐쇄 원칙**  
 -> 각각 다른 목적과 다른 이유에 의해 다른 시점에 독립적으로 변경될 수 있는 효율적인 구조를 만드는 것 
 
**템플릿** 
 -> 다른 코드 중에서 변경이 거의 일어나지 않으며 **일정한 패턴으로 유지되는 특성을 가진 부분**을 **자유롭게 변경되는 성질을 가진 부분으로부터 독립**시켜서 효과적으로 활용할 수 있도록 하는 방법
  
  
## 다시보는 초난감 DAO

### 예외처리 기능을 갖춘 DAO 
`UserDao`의 `deleteAll()`메소드를 보면 메소드를 마치기 전에 각각 `close()`를 호출해 리소스를 반환한다. 
하지만 `Connection`과 `PreparedStatement`의 `close()`메소드가 실행되지 않아서 제대로 리소스가 반환되지 않을 수 있다. 
그래서 어떤 상황에서도 리소스를 반환할 수 있도록 JDBC코드에서는 `try/catch/finally`구문을 권장하고 있다. 
`finally`구문 안에서 `close()`메소드를 호출하면 무조건 리소스를 반환하고 메소드를 마치게 되니 문제가 해결된다. 
  하지만 `close()`메소드 자체에서도 `SQLException`이 발생할 수 있는 메소드이므로, 역시나 `try/catch`문으로 처리해 주어야 한다. 
  
```java

public void deleteAll() throws SQLException {
    Connection c = null;
    PreparedStatement ps = null;

    try {
        c = dataSource.getConnection();
        ps = c.prepareStatement("delete from users");
        ps.excuteUpdate();
            
    } catch (SQLException e) {
        throw e;
    } finally {
        if (ps != null) {
            try {
                ps.close();
            } catch (SQLException e) {
            }
        }
        if (c != null) {
            try {
                c.close();
            } catch (SQLException e) {
            }
        }
    }
}
    

```

어느 시점에서 예외가 발생했는지에 따라서 `close()`를 사용할 수 있는 변수가 달라질 수 있기 때문에 `finally`에서는 반드시 `c`와 `ps`가 `null`이 아닌지 먼저 확인한 후에 `close()`메소드를 호출해야 한다.
 
 
### JDBC 조회 기능의 예외처리 
조회를 위한 JDBC 코드에서는 `Connection`, `PrepareStatement` 외에 `ResultSet` 이 추가되기 때문에 코드가 좀 더 복잡하다.
하지만 구성읕 마찬가지 이므로, `ResultSet`의 `close()`메소드가 반드시 호출되도록 만들면 된다. 

















