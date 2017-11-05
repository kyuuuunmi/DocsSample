작은 단위의 테스트로 얻을 수 있는 장점은 많다. 하지만 테스트 대상이 다른 오브젝트와 환경에 의존하고 있다면 작은 단위의 테스트가 주는 장점을 얻기 힘들다. 
 
 
 ## 복잡한 의존관계 속의 테스트 
 
 `UserService`의 경우,  
 `UserDao` 타입의 오브젝트를 통해 DB와 데이터를 주고받아야 하고,  
 `MailSender`를 구현한 오브젝트를 이용해 메일을 밸송해야 한다. 
 마지막으로 트랜잭션 처리를 위해 `PlatformTransactionManger`와 커뮤니케이션이 필요하다.  
 
 따라서 `UserService`를 테스트 하는 것은, 오브젝트들과 환경, 서비스, 서버, 심지어 네트워크까지 함께 테스트하는 셈이 된다. 
 그 어느 것이라도 바르게 셋업되어 있지 않거나, 코드에 문제가 있다면 그 때문에 `UserService`에 대한 테스트가 실패해버리기 때문이다. 
 
 ## 테스트 대상 오브젝트 고립시키기 
 
 그래서 테스트의 대상이 환경이나, 외부 서버, 다른 클래스의 코드에 종속되고 영향을 받지 않도록 고립시킬 필요가 있다.  
 `mailSender`에 적용했던 테스트를 위한 대역을 `UserDao`에도 적용해 보자. 
 
 ### 테스트를 위한 `UserServiceImpl`고립 
 
 트랜잭션은 이미 비즈니스 로직으로부터 분리를 했으니, `upgradeLevels()`를 확인하기 위해 `UserDao`에 대한 목 오브젝트를 만들어보자. 
 테스트 대상인 `UserServiceImpl`와 그 협력 오브젝트인 `UserDao`에게 어떤 요청을 했는지를 확인하는 작업이 필요한 것인데, 
 `UserDao`와 같은 역할을 하면서 `UserServiceImpl`과의 사이에서 주고받은 정보를 저장해두었다가, 테스트 검증에 사용할 수 있게 목 오브젝트를 만들 필요가 있다. 
 
 ### 고립된 단위 테스트 활용 
 
 `upgradeLevels()` 테스트 
 1. 테스트 실행 중에 `UserDao`를 통해 가져올 테스트용 정보를 DB에 얻는다. `UserDao`는 결국 DB를 이용해 정보를 가져오기 때문에 최후의 의존 대상인 DB에 직접 정보를 넣어줘야 한다. 
 2. 메일 발송 여부를 확인하기 위해 `MailSender` 목 오브젝트를 DI 해준다. 
 3. 실제 테스트 대상인 `userService`의 메소드를 실행한다. 
 4. 결과가 DB에 반영됐는지 확인하기 위해서 `UserDao`를 이용해 DB에서 데이터를 가져와 결과를 확인한다. 
 5. 목 오브젝트를 통해 `UserService`에 의한 메일 발송이 있었는지 확인하면 된다. 
 
 
 ### `UserDao` 목 오브젝트 
 
 실제 `UserDao`와 DB에 직접 의존하고 있는 첫 번째와 네번째의 테스트 방식도 목 오브젝트를 만들어서 적용시켜보자. 
 
 `userDao.getAll()`은 레벨 업그레이드 후보가 될 사용자의 목록을 받아온다.  
 `userDao.update(user)`의 호출은 리턴값이 없다. 하지만 **변경**에 해당하는 부분을 검증할 수 있는 중요한 기능이기도 하기 때문에, 목 오브젝트로서 동작하는 테스트 대역이 필요하다. 
 
 
 완전히 고립돼서 테스트만을 위해 독립적으로 동작하는 테스트 대상을 사용할 것이기 때문에 스프링 컨테이너에서 빈을 가져올 필요가 없다. 
 그러므로 테스트 대상 오브젝트를 직접 생성하여 주입하도록 한다. 
 
 ### 테스트 수행 성능의 향상 
 
 `UserServiceImple`와 테스트를 도와주는 두 개의 목 오브젝트의 외에는 사용자 관리 로직을 검증하는 데 직접적으로 필요하지 않은 의존 오브젝트와 서비스를 모두 제거한 까닭에, 테스트 수행시간을 500배 이상 줄였다. 
 
 고립된 테스트를 하면 테스트가 다른 의존 대상에 영향을 받을 경우를 대비해 복잡하게 준비할 필요가 없을 뿐만 아니라, 테스트 수행 성능도 크게 향상된다. 
 
 
 
 ## 단위 테스트와 통합 테스트 
 
 **단위테스트**  
 테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜서 테스트 하는 것
 
 **통합테스트**  
 두 개 이상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나, 외부의 DB나 파일, 서비스 등의 리소스가 참여하는 테스트. 즉 두 개 이상의 단위가 결합해서 동작하면서 테스트가 수행되는 것.  
  
  
 ## 목 프레임워크  
 
 단위 테스트를 만들기 위해서는 스텁이나 목 오브젝트의 사용이 필수적이다. 
 목 오브젝트를 편리하게 작성하도록 도와주는 다양한 목 오브젝트 지원 프레임워크가 있다. 
 
 ### `Mockito`프레임워크 
 
 - 인터페이스를 이용해 목 오브젝트를 만든다. 
 - 목 오브젝트가 리턴할 값이 있으면 이를 지정해준다. 메소드가 호출되면 예외를 강제로 던지게 만들 수도 있다. 
 - 테스트 대상 오브젝트에 DI해서 목 오브젝트가 테스트 중에 사용되도록 만든다. 
 - 테스트 대상 오브젝트를 사용한 후에 목 오브젝트의 특정 메소드가 호출됐는지, 어떤 값을 가지고 몇 번 호출됐는지를 검증한다. 
 
 
 ```java
   @Test
    public void mockUpgradeLevels() throws Exception {
        UserServiceImpl userService = new UserServiceImpl();

        UserDao mockUserDao = mock(UserDao.class);
        when(mockUserDao.getAll()).thenReturn(this.users);
        userService.setUserDao(mockUserDao);
        
        MailSender mockMailSender = mock(MailSender.class);
        userService.setMailSender(mockMailSender);
        
        userService.upgradeLevels();
        
        verify(mockUserDao, times(2)).update(any(User.class));
        verify(mockUserDao, times(2)).update(any(User.class));
        verify(mockUserDao, times(2)).update(users.get(1));
        assertThat(users.get(1).getLevel(), is(Level.SILVER));
        verify(mockUserDao).update(users.get(3));
        assertThat(users.get(3).getLevel(), is(Level.GOLD));

        ArgumentCaptor<SimpleMailMessage> mailMessageArgumentCaptor = ArgumentCaptor.forClass(SimpleMailMessage.class);
        verify(mockMailSender, times(2)).send(mailMessageArgumentCaptor.capture());
        List<SimpleMailMessage> mailMessages = mailMessageArgumentCaptor.getAllValues();
        assertThat(mailMessages.get(0).getTo()[0], is(users.get(1).getEmail()));
        assertThat(mailMessages.get(1).getTo()[0], is(users.get(3).getEmail()));
        
    }


```
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 