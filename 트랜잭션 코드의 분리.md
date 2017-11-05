

## 트랜잭션 코드의 분리 

스프링이 제공하는 트랜잭션 인터페이스를 썼음에도 비즈니스 로직 안에 트랜잭션 코드가 자리잡고 있다. 이를 분리해보자. 

### 메소드 분리 

`UserService`의 `upgradeLevels()` 코드의 특징은 트랜잭션 경계설정의 코드와 비즈니스 로직 코드 간에 서로 주고받는 정보가 없다는 것이다. 
즉 트랜잭션 시작과 끝을 담당하는 부분과, 비즈니스 로직의 코드가 성격이 다를 뿐 아니라, 서로 주고받는 것도 없는 완벽하게 독립적인 코드 라는 것이다. 
다만, 이 비즈니스 로직을 담당하는 코드가 트랜잭션의 시작과 종료 작업 사이에서 수행돼야 한다는 사항만 지켜지면 된다. 
그러므로 두 개의 메소드로 분리를 시키자. 


## DI를 이용한 클래스의 분리

트랜잭션 코드를 클래스 밖으로 뽑아내어 `UserService`에서 보이지 않게 하자. 

### DI 적용을 이용한 트랜잭션 분리 

`UserService`를 인터페이스로 분리하고, 비즈니스 로직을 구현 클래스인 `UserServiceImpl`에 구현한다. 그리고 트랜잭션 처리를 또 다른 구현 클래스인 `UserServiceTx`에게 맡긴다. 
### `UserService` 인터페이스 도입

`UserServiceImpl`는 메일 발송 기능을 추가했던 것을 제외하면 트랜잭션을 고려하지 않고 단순하게 로직만을 구현했던 처음 모습으로 돌아왔다. 

```java

...
public class UserServiceImpl implements UserService {
    UserDao userDao;
    MailSender mailSender;
    
    public void upgradeLevels() {
            List<User> users = userDao.getAll();
            for (User user : users) {
                if (canUpgradeLevel(user)) {
                    upgradeLevel(user);
                }
            }
            
    }
     ...    
}

```
   
### 분리된 트랜잭션 기능 

비즈니스 트랜잭션 처리를 담은 `UserServiceTx`는 기본적으로 `UserService`를 구현하게 만들고, 같은 인터페이스를 구현한 다른 오브젝트에게 고스란히 작업을 위임하게 만들면 된다. 

```java

public class UserServiceTx implements UserService {

    UserService userService;
    PlatformTransactionManager transactionManager;

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setUserService(UserService userService) {
        this.userService = userService;
    }


    public void add(User user) {
        userService.add(user);
    }

    public void upgradeLevels() throws Exception {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {

            userService.upgradeLevels();
            this.transactionManager.commit(status);
        } catch (RuntimeException e ){
            this.transactionManager.rollback(status);
            throw e;
        }
    }
    
```

### 트랜잭션 적용을 위한 DI 설정 

클라이언트가 `UserService`라는 인터페이스를 통해 사용자 관리 로직을 이용하려고 할 때 먼저 트랜잭션을 담당하는 오브젝트가 사용돼서 트랜잭션에 관련된 작업을 진행해주고, 
실제 사용자 관리 로직을 담은 오브젝트가 이후에 호출돼서 비즈니스 로직에 관련된 작업을 수행하도록 만든다. 


### 트랜잭션 분리에 따른 테스트 수정

테스트를 돌려봐야 하는데, 손볼 곳이 있다. 

- `@Autowired`는 기본적으로 타입이 일치하는 빈을 찾아준다. 현재 수정한 스프링의 설정 파일에는 `UserService`라는 인터페이스 타입을 가진 두 개의 빈이 존재 하는데, 어떤 빈을 가져오게 될까?  
기본적으로는 타입을 이용해 빈을 찾지만, 하나의 빈을 결정할 수 없을 경우에는 필드 이름을 이용해 빈을 찾는다. 
그러므로 `userService`변수를 설정해두면 아이디가 `userService`인 빈(현재 `UserServiceTx`)을 주입할 것이다. 

- `UserServiceImpl`클래스로 정의된 빈 또한 주입해야 한다. 앞 장에서 만든 `MailSender`목 오브젝트를 이용한 테스트에서 `MailSender`가 직접 DI해줄 오브젝트를 알고 있어야 하기 때문에, `@Autowired`를 이용해 직접 주입받도록 한다. 
 
 
 ### 트랜잭션 경계설정 코드 분리의 장점 
 
 - 비즈니스 로직을 담당하고 있는 `UserServiceImpl`의 코드를 작성할 때에는 온전히 비즈니스 로직에만 집중 할 수 있고, 트랜잭션과 같은 기술적인 내용에는 전혀 신경 쓰지 않아도 된다.
 - 비즈니스 로직에 대한 테스트를 손쉽게 만들어 낼 수 있다. 
 
 
 

































