
JdbcTemplate을 적용한 코드에서는 `SQLException`이 모두 사라졌다. 어디로 갔을까?

## 초난감 예외처리

### try-catch 블록으로 감싸놓은 예외처리용 코드
  예외가 발생했을 때 그것을 `catch`로 잡는 것까지는 인정, 하지만 아무것도 하지 않고 넘어가 버리는 것은 위험한 일이다. 예외에 대해서 콘솔이나 로그에 예외 메시지를 출력해 주는 것 역시나 예외를 처리한 것이 아니다. 모든 예외는 _적절하게 복구 되든지 아니면 작업을 중단시키고 운영자 또는 개발자에게 분명하게 통보돼야 한다._ 굳이 잡아서 조치를 취할 방법이 없다면 잡지 말고, 메소드에 `throws SQLException`을 선언해서 메소드 밖으로 던지고 자신을 호출한 코드에 예외처리 책임을 전가하라. 
```java
try{
    ... 
} catch(SQLException e) {
    e.parintStackTrace();
    System.out.println(e);
}
``` 
### throws Exception 모든 예외를 던져버리는 선언 
  모든 예제를 무조건 던져버리는 선언을 기계적으로 넣는 것은 의미 있는 정보를 얻을 수 없다. 결국 적절한 처리를 통해 복구될 수 있는 예외상황도 제대로 다룰 수 있는 기회를 박탈 당한다.  
```java
public void method1() throws Exception{
    method2();
    ...
}

public void method2() throws Exception{
    method3();
    ... 
}
``` 

## 예외의 종류와 특징

1. Error
- 시스템에 `OutOfMemoryError`나 `ThreadDeath` 같은 비정상적인 상황이 발생했을 경우에 사용. 
- 시스템 레벨에서 하는 작업이 아니면 애플리케이션에서 이런 에러에 대한 처리는 신경 쓰지 않아도 된다. 
2. Exception - 체크예외
- `RuntimeException`을 상속하지 않는 `Exception`클래스. 
- 체크 예외가 발생할 수 있는 메소드를 사용할 경우 반드시 예외를 처리하는 코드를 함께 작성해야 한다. 
- `IOException`, `SQLException`등 
3. Exception - 언체크/런타임 예외
- `RuntimeException` 클래스를 상속하여 명시적인 예외처리를 강제하지 않는 `Exception`. 
- 프로그램의 오류가 있을 때 발생하도록 의도된 것. 

## 예외처리 방법

### 예외 복구 
예외상황을 파악하고 문제를 해결해서 정상 상태로 돌려 놓는 방법이다. 예외로 기본 작업 흐름이 불가능하면, 다른 작업 흐름으로 자연스럽게 유도해 주는 것이다. 예를들어 네트워크 접속이 원활하지 않아서 예외가 발생했다면, 일정 시간 대기 했다가 다시 접속을 시도해보는 방법을 사용하여 복구를 시도할 수 있다. 예외처리 코드를 강제하는 체크 예외들은 이렇게 예외를 어떤 식으로든 복구할 가능 성이 있는 경우에 사용한다. 

```java
int maxretry = MAX_RETRY;
while(maxretry -- > 0){
    try {
        ...
        return;
    } catch(SomeException e) {
        // 로그 출력. 정해진 시간만큼 대기
    } finally { 
        // 리소스 반납, 정리 작업
    }
} 
throw new RetryFailedException(); // 최대 재시도 횟수를 넘기면 직접 예외 발생 
```

### 예외처리 회피
예외처리를 자신이 담당하지 않고 `throw`나 `catch`안에서 rethrow로 자신을 호출한 쪽으로 던져버리는 것이다. 하지만 무분별한 책임회피는 `throws Exception`을 사용할 가능성이 높다. 회피하는 것은 복구하는 것처럼 의도가 분명해야 한다. 콜백/템플릿처럼 긴밀한 관계에 있는 오브젝트에게 책임을 분명히 지게 하거나, 자신을 사용하는 쪽에서 예외를 다루는 게 최선의 방법이라는 분명한 확신이 있어야 한다. 

### 예외 전환
예외를 메소드 밖으로 던지는 것이지만, 회피와 달리 발생한 예외를 그대로 넘기는 것이 아니라 적절한 예외로 전환해서 던진다는 특징이 있다. 
예외상황에 대해 의미를 분명하게 해줄 수 있는 예외로 바꿔주거나, 반대로 처리하기 쉽고 단순하게 만들기 위해 포장하기 위해서 예외를 전환한다. **전자**는 상황에 적합한 의미를 가진 예외로 변경하는 것이고, **후자**는 어차피 복구가 불가능한 예외라면 가능한 한 빨리 런타임 예외로 포장해 던지게 해서 다른 계층의 메소드를 작성할 때 불필요한 `throws` 선언이 들어가지 않도록 해주는 것이다. 


## 예외처리 전략 

### 런타임 예외의 보편화 
일반적으로 체크 예외가 일반적인 예외를 다루고, 언체크 예외는 시스템 장애나 프로그램상의 오류에 사용한다고 했지만, 자바 엔터프라이즈 서버 환경에서는 요청에 대한 해당 작업만 중단시키면 사용자와의 커뮤니케이션이 끊기기 때문에 애플리케이션 차원에서 예외상황을 미리 파악하고, 발생하지 않도록 차단하는게 좋다. 즉, 대응이 불가능한 체크 예외라면 빨리 런타임 예외로 전환해서 던지는 게 낫다. 

### add() 메소드의 예외처리 
add()메소드는 `DuplicatedUserIdException`과 `SQLException`, 두 가지의 체크 예외를 던지게 되어 있다. 
전자는 충분히 복구 가능한 예외이므로 add()에서 대응 하는 것이 낫고, 후자는 런타임 예외로 포장하여 다른 메소드들이 신경 쓰지 않게 해주는 편이 낫다. 

```java
public void add() throws DuplicateUserIdException{
    try {
        // JDBC를 이용해 user 정보를 DB에 추가하는 코드 또는 
        // 그런 기능이 있는 다른 SQLException을 던지는 메소드를 호출하는 코드
    } catch {
        if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY) 
            throw new DuplicatedUserIdException(e);   // 예외 전환
        else. 
            throw new RuntimeException(e);            // 예외 포장
    }
}
```

### 애플리케이션 예외 
시스템 또는 외부의 예외상황이 원인이 아니라 애플리케이션 자체의 로직에 의해 의도적으로 발생시키고, 반드시 `catch` 해서 무엇인가 조치를 취하도록 요구하는 예외를 애플리케이션 예외라고 한다. 이때 사용하는 예외는 의도적으로 체크 예외로 만들어 개발자에게 로직을 구현하도록 강제해주는 것이 좋다.  

```java
try {
    BigDecimal balance = account.withdraw(amount);
    ...
    // 정상적인 처리 결과를 출력하도록 진행
} catch(InsufficientBalanceException e) { // 체크예외
    // InsufficientBalanceException에 담긴 인출 가능한 잔고금액 정보를 가져옴
    BigDecimal availFunds = e.getAvailFunds();
    ... 
    // 잔고 부족 안내 메시지를 준비하고 이를 출력하도록 진행 
}
```

### SQLException은 어떻게 됐나?
`SQLException`은 과연 복구가 가능한 예외인가를 먼저 생각해보자. 99%의 `SQLException`은 코드 레벨에서는 복구할 방법이 없다.개발자가 빨리 인식할 수 있도록 발생한 예외를 빨리 전달하는 것 외에는 할 수 있는 게 없다. 스프링의 JdbcTemplate은 이 예외처리 전략을 따르고 있다. 이 템플릿과 콜백 안에서 발생하는 모든 SQLException을 런타임 예외인 `DataAccessException`으로 포장해서 던져준다. 따라서 JdbcTemplate을 사용하는 `UserDao`메소드에선 꼭 필요한 경우에만 런타임 예외인 `DataAccessException`을 잡아서 처리하면 되고, 그 외의 경우에는 무시해도 된다. 
