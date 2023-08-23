# AOP

- 관점 지향 프로그래밍 (Aspect-Oriented Programming)
- 로깅, 보안, 트랜잭션 등 기술적인 관심사들과 같이 공통적으로 필요한 횡단 관심사를 분리하여 모듈화하는 방식
- IoC/DI, 서비스 추상화와 더불어 스프링의 3대 기반기술중 하나

# 6.1 트랜잭션 코드의 분리

- `UserService`의 코드를 많이 개선했으나, 여전히 찜찜한 구석이 있다.
- `UserService`라는 비즈니스 규칙의 코드에 기술적인 코드인 트랜잭션 경계설정이 거슬린다.
- 그러나 이를 제거할 수는 없는데, 비즈니스 전후에 반드시 설정돼야 하는 코드이기 때문이다.

![](list-6-1.jpeg)

- 코드를 분석해보면 목적이 다른 코드가 뚜렷하게 구분된다.
- 이를 분리할 수 있을 것이다.

```java
    public void upgradeLevels() {
        TransactionStatus status =
                transactionManager.getTransaction(new DefaultTransactionDefinition());

        try {
            // 비즈니스 로직 분리
            upgradeLevelInternal();
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }

    private void upgradeLevelInternal() {
        List<User> users = userDao.getAll();
        for (User user: users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
    }
```

## 6.1.2 DI를 이용한 클래스의 분리

- 비즈니스 로직을 따로 분리하긴 했지만, 여전히 트랜잭션을 담당하는 기술적인 코드가 버젓이 `UserService` 안에 자리 잡고 있다.
- 서로는 완전 별개인데, 트랜잭션 코드를 최소한 `UserService`에서 보이지 않게 할 수는 없을까?

### DI 적용을 이용한 트랜잭션 분리

- `UserService`는 결국 어디선가 생성되어 호출될 것이다.
- 따라서 트랜잭션 코드를  `UserService`  밖으로 빼내고 트랜잭션 경계설정 코드가 없는 `UserService`를 사용하게 한다면 일단 `UserService` 내에서는 트랜잭션 경계설정 코드를 빼낼 수는 있을 것이다.
- 해당 클라이언트는 `UserService`를 DI 받게 한다면, 어디선가 `UserService`의 트랜잭션 경계설정이 된 다른 구현체를 정의하여 사용할 수 있는 방법을 생각할 수 있다.

![](picture-6-3.jpeg)

### `UserService` 인터페이스 도입

- 기존의 `UserService` 클래스를 인터페이스로 변경해보자.
- 이를 위해 `UserService`의 이름을 `UserServiceImpl`로 변경하고. 인터페이스를 추출한다.

```java
public interface UserService {
    void add(User user);
    void upgradeLevels();
}
```

- 우리의 목적은 트랜잭션 경계설정 코드를 제거하는 것이므로, `UserServiceImpl`에서 해당 코드를 모두 제거한다.

```java
@Service
public class UserServiceImpl implements UserService {

    public static final int MIN_LOGCOUNT_FOR_SILVER = 50;
    public static final int MIN_RECOMMEND_FOR_GOLD = 30;

    private UserDao userDao;
    private MailSender mailSender;

    // ...

    // 트랜잭션 경계설정 코드를 제거
    public void upgradeLevels() {
        List<User> users = userDao.getAll();
        for (User user: users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
    }

    // ...
}
```

### 분리된 트랜잭션 기능

- 트랜잭션 경계설정 코드를 가진 `UserServiceTx` 클래스를 만들고, `UserService` 인터페이스를 구현하게 하자.
- 그리고 비즈니스 로직을 담은 `UserSerivce`를 DI받아 위임하게 한다.

```java
package com.example.chapter3;

@Service
public class UserServiceTx implements UserService {
    UserService userService;

    public UserServiceTx(UserService userService) {
        this.userService = userService;
    }

    @Override
    public void add(User user) {
        userService.add(user);
    }

    @Override
    public void upgradeLevels() {
        userService.upgradeLevels();
    }
}
```

- 이제 트랜잭션 경계설정 코드를 추가하자.

```java
@Service
public class UserServiceTx implements UserService {
    UserService userService;
    PlatformTransactionManager transactionManager;

    public UserServiceTx(UserService userService, PlatformTransactionManager transactionManager) {
        this.userService = userService;
        this.transactionManager = transactionManager;
    }
    
    // ...
    
    @Override
    public void upgradeLevels() {
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        
        try {
            userService.upgradeLevels();
            
            this.transactionManager.commit(status);
        } catch (RuntimeException e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```

- 이전에 `UserService`에 트랜잭션 경계설정 코드를 추가한 것과 크게 다르지 않다.

### 트랜잭션 적용을 위한 DI 설정

- 현재의 의존관계는 다음 그림과 같다.

![](picture-6-4.jpeg)

- 이제 `UserService`를 사용하는 Client는 `UserService` 인터페이스의 구현체를 DI 받을 때, `UserServiceTx`를 받도록 하면 된다.

```java
@Service
@Qualifier("userServiceImpl")
public class UserServiceImpl implements UserService {
    // ...
}

@Service
@Primary
public class UserServiceTx implements UserService {
    UserService userService;
    PlatformTransactionManager transactionManager;

    public UserServiceTx(@Qualifier("userServiceImpl") UserService userService, PlatformTransactionManager transactionManager) {
        this.userService = userService;
        this.transactionManager = transactionManager;
    }

    // ...
}
```

### 트랜잭션 분리에 따른 테스트 수정

- 위와 같이 `UserService` 인터페이스에 구현체가 2개 이상인 경우, 스프링은 어떤 구현체를 선택해야 할지 알 방도가 없다.
- 따라서 스프링 컨테이너가 선택할 수 있도록 정해줘야 한다.
- 테스트에서는 대상이 명확하므로, 인터페이스를 통해 DI 받을 필요가 없어서 구현체를 직접 DI 받아도 된다.
    - 또는 굳이 DI받지 말고 인스턴스화 해도 된다.

```java
@SpringBootTest
public class UserServiceImplTest {
    // ...

    // @Autowired를 제거
    UserServiceImpl userService;

    private List<User> users;

    @BeforeEach
    public void setUp() {
        users = Arrays.asList(
                new User("bumjin", "박범진", "p1", email, Level.BASIC, MIN_LOGCOUNT_FOR_SILVER - 1, 0),
                new User("joytouch", "강명성", "p2", email, Level.BASIC, MIN_LOGCOUNT_FOR_SILVER, 0),
                new User("erwins", "신승한", "p3", email, Level.SILVER, 60, MIN_RECOMMEND_FOR_GOLD - 1),
                new User("madnite1", "이상호", "p4", email, Level.SILVER, 60, MIN_RECOMMEND_FOR_GOLD),
                new User("green", "오민규", "p5", email, Level.GOLD, 100, Integer.MAX_VALUE)
        );

        // 직접 인스턴스화
        userService = new UserServiceImpl(userDao, mailSender);
    }

    // ...
}
```

- 그 외 테스트 코드도 수정하자.

![](list-6-9.jpeg)

### 트랜잭션 경계설정 코드 분리의 장점

- 비즈니스 로직과 기술적인 코드가 분리되어 코드의 가독성 및 유지보수성이 높아졌다.
    - 비즈니스가 변경되면 `UserServiceImpl`을, 기술적 코드가 변경되면 `UserServiceTx`를 보면 된다.
- 비즈니스 로직에 대한 테스트가 좀 더 수월해졌다.
    - 트랜잭션 경계설정에 대한 기술적 관심사를 배제하고, 순수하게 비즈니스에만 집중해서 테스트할 수 있게 됐다.

# 6.2 고립된 단위 테스트

- 작은 단위의 테스트 코드는 테스트가 실패했을 때 원인을 찾기 쉽다.
    - 의존성이 많을수록, 코드가 길어질 수록 영향범위가 넓어지므로 실패했을 때 확인해야 될 부분도 넓다.

## 6.2.1 복잡한 의존관계 속의 테스트

- `UserService`는 간단한 기능만을 갖고 있음에도, 협력하는 오브젝트들이 많다.

![](picture-6-5.jpeg)

- 따라서 테스트 대상인 `UserService`가 실패했을 경우, `UserService`의 문제로 실패했다고 보장할 수 없다.

## 6.2.2 테스트 대상 오브젝트 고립시키기

- 따라서 테스트의 대상 코드는 외부 환경이나 다른 클래스의 코드에 영향 받지 않도록 고립시킬 필요가 있다.
    - 이를 위해 Test Double에 대해 알아봤었다.

### 테스트를 위한 `UserServiceImpl` 고립

- `UserServiceImpl`에서 트랜잭션 코드를 독립시켰기 때문에 현재는 다음과 같은 관계를 갖고 있다.

![](picture-6-6.jpeg)

- DB에 반영됐다는 것을 확인하기 위해 `UserDao`를 Mock 오브젝트로 만들고, `update()` 메서드의 호출 여부를 검증하는 방법으로 간접적으로 테스트할 수 있다.

### 고립된 단위 테스트 활용

- 고립된 단위 테스트를 적용해보기 위해, 기존 테스트 코드를 살펴보자.

![](list-6-10.jpeg)

1. 테스트용 정보를 DB에 넣는다.
2. 메일 발송 여부 확인을 위해 `MailSender` 목 오브젝트를 DI 해준다.
3. `userService` 테스트를 진행한다.
4. DB에 저장된 결과를 확인한다.
5. 목 오브젝트를 통해 메일 발송 요청이 있었는지 확인한다.

### `UserDao` 목 오브젝트

- `UserDao` 목 오브젝트를 만들어보자.

```java
public class MockUserDao implements UserDao {
    private List<User> users;
    private List<User> updated = new ArrayList<>();

    public MockUserDao(List<User> users) {
        this.users = users;
    }

    public List<User> getUsers() {
        return users;
    }

    public List<User> getUpdated() {
        return updated;
    }

    @Override
    public List<User> getAll() {
        return this.users;
    }

    @Override
    public void update(User user) {
        updated.add(user);
    }
    
    @Override
    public void add(User user) {
        throw new UnsupportedOperationException();
    }

    @Override
    public User get(String id) {
        throw new UnsupportedOperationException();
    }

    @Override
    public void deleteAll() {
        throw new UnsupportedOperationException();
    }

    @Override
    public int getCount() {
        throw new UnsupportedOperationException();
    }
}
```

- 사용하지 않는 기능들은 `UnsupportedOperationException()`을 던지도록 해서, 테스트 대상이 아님에도 사용되는 경우를 방지했다.

```java
@SpringBootTest
public class UserServiceImplTest {
    // ...

    @Test
    public void upgradeLevels() {
        MockUserDao userDao = new MockUserDao(this.users);
        UserServiceImpl userServiceImpl = new UserServiceImpl(
                userDao,
                this.mailSender
        );
        
        userServiceImpl.upgradeLevels();
        
        List<User> updated = userDao.getUpdated();
        Assertions.assertThat(updated.size()).isEqualTo(2);
        checkUserAndLevel(updated.get(0), "joytouch", Level.SILVER);
        checkUserAndLevel(updated.get(1), "madnite1", Level.GOLD);
        
        // ...
    }

    private void checkUserAndLevel(User updated, String expectedId, Level expectedLevel) {
        Assertions.assertThat(updated.getId()).isEqualTo(expectedId);
        Assertions.assertThat(updated.getLevel()).isEqualTo(expectedLevel);
    }

    // ...
}

```

- 고립된 테스트기 때문에, DI를 받을 필요가 없이 인스턴스화해서 테스트하면 된다.
    - 그래야 해당 테스트에 대한 모든 통제를 메서드 내에서 할 수 있다.
    - DI를 받는다면 스프링 컨테이너의 영향까지 받게 되므로, 고립된 테스트라 할 수 없다.

### 테스트 수행 성능의 향상

- 고립된 테스트로 만들면서, 테스트 대상의 의존관계가 거의 다 사라졌으므로 테스트 시간도 획기적으로 빨라졌다.
- 테스트의 갯수가 많아질수록, 이 효과가 더 체감될 것이다.
    - 테스트를 더 많이, 더 자주 실행할 수 있다는 뜻이다.
- 테스트 시간이 길어지면 테스트 코드를 작성하지 않게 되거나 무시하게 된다.
    - 한 번 테스트 코드를 작성하지 않게 된다면, 결국 나중에 가서 다시 테스트코드를 작성하긴 매우 어렵다.

## 6.2.3 단위 테스트와 통합 테스트

- 단위 테스트에서 단위는 정하기 나름이다.
- 이 책에서는 **의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜서 테스트하는 것**을 단위 테스트라고 부르겠다.
- 통합 테스트는 2개 이상의 단위가 결합해서 동작하는 테스트라고 하겠다.

---
### 구글에서 테스트 나누기

#### 작은 테스트

- 단 하나의 프로세스에서 실행
    - 언어에 따라 프로세스가 아닌 스레드 레벨까지도 제한함
- 따라서 DB 등 외부자원에 의존하지 않아야 함
- `sleep`, `I/O` 오퍼레이션 등 블로킹 호출을 사용해서는 안 됨.
    - 파일시스템, 네트워크 자원 사용 불가능함
    - 이런게 필요할 때 테스트 대역을 사용함
- 비결정적인 테스트를 없애기 위함

#### 중간 크기 테스트

- 여러 프로세스, 스레드 사용 가능
- 블로킹 호출 가능
- 여전히 로컬호스트 내에서만 테스트 가능
#### 큰 크기 테스트

- 원격지 호출 가능
- end-to-end 테스트

---
### 테스트 가이드라인

1. 항상 단위 테스트를 먼저 고려한다.
2. 외부 리소스가 꼭 필요한 경우에만 통합 테스트로 만든다.
3. DAO와 같이 외부 DB와 연동되어야만 의미가 있는 경우 통합 테스트로 만든다.
4. 결국 여러 단위가 의존관계를 가지고 동작하는 통합 테스트가 필요하긴 하다.
    - 단위 테스트가 잘 작성되어 있다면 부담이 덜하다.
5. 단위 테스트로 만들기 너무 복잡한 경우에는 통합 테스트로 고려해본다.
6. 스프링 테스트 컨텍스트 프레임워크를 사용하는 테스트는 통합 테스트다.

## 6.2.4 목 프레임워크

- 스텁, 목 오브제트를 사용해야 하는 경우 이를 만드는 것이 번거로운것은 사실이다.
- 이를 위한 프레임워크를 사용하는 것도 한가지 방법이다.

> 하지만 계속 언급하지만, Mock은 최대한 자제하는 것이 좋은 것 같다. 결국 테스트 코드 자체가 구현에 강하게 결합되어 화이트박스 테스트가 되며, 이는 깨지기 쉬운 테스트가 될 수 있다.

### Mockito 프레임워크

- 가장 유명한 목 프레임워크
- 특정 메서드가 호출됐을 때 리턴되는 값을 지정할 수 있음

```java
UserDao mockUserDao = mock(UserDao.class);

// mockUserDao의 getAll() 메서드가 호출됐을 때
// this.users를 리턴하게 만든다.
when(mockUserDao.getAll())
    .thenReturn(this.users);

// mockUserDao의 update() 메서드가 2번 호출됐는지 검증한다.
verify(mockUserDao, times(2)).update(any(User.class));
```

![](list-6-14-1.jpeg)![](list-6-14-2.jpeg)

> 이런 상세 구현을 테스트 하는 코드가 정말 의미가 있을까? 결국 input에 대한 output이 중요하고, output이 올바른지 검사하면 되는 것이 아닌가 하는 생각을 한다.

# 6.3 다이내믹 프록시와 팩토리 빈

### 6.3.1 프록시와 프록시 패턴, 데코레이터 패턴

- 트랜잭션 경계설정 코드를 비즈니스 로직 코드에서 분리해낼 때 적용했던 전략패턴은 기능의 구현을 분리했을 뿐, 기능 자체는 코드에 그대로 남아 있다.

![](picture-6-7.jpeg)

- 트랜잭션 구현 코드를 아예 외부 클래스(`UserServiceTx`)로 추출하고, 부가기능 코드가 핵심기능 코드를 사용하도록 위임하는 방식으로 리팩토링하여 핵심기능에서는 부가기능 코드가 보이지 않도록 했다.

![](picture-6-8.jpeg)

- 핵심기능 코드는 부가기능의 존재를 모른다. 부가기능이 핵심기능의 클라이언트가 되는 것이다.
- 이렇게 하면 결국 핵심기능을 사용하기 위해서는 부가기능을 사용해야 하고, 이 부가기능을 사용하는 클라이언트는 마치 부가기능이 핵심기능인 것처럼 알아야 된다.
- 이를 프록시라고 한다.

![](picture-6-10.jpeg)

- 프록시는 타깃과 같은 인터페이스를 구현하며, 타깃과 클라이언트 사이에서 타깃을 제어할 수 있게 된다.
- 프록시를 사용하는 이유는 다음과 같을 수 있다.
    1. 클라이언트가 타깃에 접근하는 방법을 제어하기 위해서
    2. 타깃에 부가적인 기능을 부여해주기 위해서

### 데코레이터 패턴

- 데코레이터 패턴은 타깃에 부가적인 기능을 런타임에 다이나믹하게 부여해주기 위해 프록시를 사용하는 패턴을 말한다.
- 다이나믹하게 기능을 부가한다는 의미는 컴파일 시점에 코드상에서 어떤 방법과 순서로 타깃이 연결되어 사용되는지 정해져 있지 않다는 뜻이다.
    - 사실 컴파일 시점에 정해져 있을 수도 있다.
    - 이와 더불어서 런타임에 데코레이터를 동적으로 연결할 수도 있다.

![](picture-6-11.jpeg)

- 런타임에 데코레이터들은 서로의 존재를 모른다. 자신이 위임/호출하는 대상이 다른 데코레이터인지, 타깃인지는 런타임에 결정된다.
- `InputStream`, `OutputStream` 구현클래스는 데코레이터 패턴이 사용된 대표적인 예다.
- `UserService` 인터페이스를 정의하여 `UserServiceImpl`, `UserServiceTx` 를 구현하여 사용한 것도 데코레이터 패턴이라 볼 수 있다.
    - DI를 통해 런타임에 동적으로 비즈니스 코드가 담긴 `UserServiceImpl`을 `UserServiceTx` 데코레이터를 적용했다고 볼 수 있다.

### 프록시 패턴

- 프록시는 타깃의 기능을 확장하거나 추가하지 않고, 클라이언트가 타깃에 접근하는 방식을 변경해준다.
- 타깃 오브젝트가 필요한 곳에 프록시를 대신 넘겨주고, 클라이언트는 프록시를 마치 타깃 오브젝트인듯 사용하게 된다.
    - lazy init, RMI(Remote Method Invocation) 등
    - 타깃의 접근권한을 제어하기 위해

![](picture-6-12.jpeg)

## 6.3.2 다이내믹 프록시

- 프록시는 기존 코드에 영향을 주지 않으면서 타깃의 기능을 확장하거나 접근 방법을 제어할 수 있는 유용한 방법이다.
- 하지만 프록시를 생성하는 일은 매우 번거롭다.
- 자바에는 프록시를 쉽게 구현할 수 있도록 해주는 표준 라이브러리인 `java.lang.reflect` 패키지가 존재한다.

### 프록시의 구성과 프록시 작성의 문제점

- 프록시는 다음의 두 가지 기능으로 구성된다.
    1. 타깃과 같은 메서드를 구현하고 있다가 메서드가 호출되면 타깃 오브젝트로 위임한다.
    2. 지정된 요청에 대해서는 부가기능을 수행한다.

![](list-6-16-1.jpeg)
![](list-6-16-2.jpeg)

- 프록시를 만드는데 번거로운 이유는 다음과 같다.
    1. 타깃의 인터페이스를 구현하고 위임하는 코드를 작성하기가 번거롭다.
    2. 부가기능을 추가하는 기능이 있다면, 코드가 중복될 가능성이 많다.
        - 예를 들어, 위의 예제의 경우 다른 모든 `UserService` 메서드에 트랜잭션 경계설정 코드가 추가되어야 할 수도 있다.
- 2번의 경우 공통코드를 메서드로 추출하면 쉽게 해결할 수 있을 것 같지만, 위임코드를 반복적으로 작성하는 부분은 해결하는게 쉬워보이지 않는다.

### 리플렉션

- 다이내믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어준다.

```java
public class ReflectionTest {

    @Test
    public void invokeMethod() throws Exception {
        String name = "Spring";

        // length()
        Assertions.assertThat(name.length()).isEqualTo(6);

        // Reflection을 이용해 호출
        Method lengthMethod = String.class.getMethod("length");
        Assertions.assertThat((Integer) lengthMethod.invoke(name)).isEqualTo(6);

        // charAt()
        Assertions.assertThat(name.charAt(0)).isEqualTo('S');

        // Reflection을 이용해 호출
        Method charAtMethod = String.class.getMethod("charAt", int.class);
        Assertions.assertThat((Character) charAtMethod.invoke(name, 0)).isEqualTo('S');
    }
}
```

### 프록시 클래스

- 다이내믹 프록시를 이용해 프록시를 만들어 보도록 하자.

```java
public interface Hello {
    String sayHello(String name);
    String sayHi(String name);
    String sayThankYou(String name);
}

public class HelloTarget implements Hello {
    @Override
    public String sayHello(String name) {
        return "Hello " + name;
    }

    @Override
    public String sayHi(String name) {
        return "Hi " + name;
    }

    @Override
    public String sayThankYou(String name) {
        return "Thank You " + name;
    }
}
```

- 이제 클라이언트 역할을 하는 테스트 코드를 작성해보자.

```java
public class HelloTest {
    @Test
    public void simpleProxy() {
        Hello hello = new HelloTarget();

        Assertions.assertEquals("Hello Toby", hello.sayHello("Toby"));
        Assertions.assertEquals("Hi Toby", hello.sayHi("Toby"));
        Assertions.assertEquals("Thank You Toby", hello.sayThankYou("Toby"));
    }
}
```

- 이제 `Hello` 인터페이스를 구현한 데코레이터 역할을 하는 `HelloUppercase`  프록시를 만들고, 테스트를 해보자.

```java
public class HelloUppercase implements Hello {
    private final Hello hello;

    public HelloUppercase(Hello hello) {
        this.hello = hello;
    }

    @Override
    public String sayHello(String name) {
        return hello.sayHello(name).toUpperCase();
    }

    @Override
    public String sayHi(String name) {
        return hello.sayHi(name).toUpperCase();
    }

    @Override
    public String sayThankYou(String name) {
        return hello.sayThankYou(name).toUpperCase();
    }
}
```

```java
public class HelloTest {

    // ...
    
    @Test
    public void uppercaseProxy() {
        Hello hello = new HelloUppercase(new HelloTarget());

        Assertions.assertEquals("HELLO TOBY", hello.sayHello("Toby"));
        Assertions.assertEquals("HI TOBY", hello.sayHi("Toby"));
        Assertions.assertEquals("THANK YOU TOBY", hello.sayThankYou("Toby"));
    }
}
```

### 다이내믹 프록시 적용

- 위에서 구현했던 `HelloUppercase`를 다이내믹 프록시를 이용해 만들어보자.

![](picture-6-13.jpeg)

- 다이내믹 프록시는 `ProxyFactory`에 의해 타겟의 인터페이스와 같은 타입으로 런타임에 만들어진다.
- 이 덕분에 인터페이스를 모두 구현해가면서 클래스를 정의하지 않아도 된다.
- 그러나 필요한 부가기능 제공 코드는 당연하지만 직접 작성해야 한다.
- 부가기능은 프록시 오브젝트와 독립적으로 `InvocationHandler`를 구현한 오브젝트에 담는다.

```java
public class UppercaseHandler implements InvocationHandler {
    private Hello target;

    public UppercaseHandler(Hello target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String ret = (String) method.invoke(target, args);
        return ret.toUpperCase();
    }
}
```

- 다이내믹 프록시는 모든 요청을 `InvocationHandler`의 `invoke()` 메서드로 위임한다.
- 이제 이 `InvocationHandler`를 사용하고 `Hello` 인터페이스를 구현하는 프록시를 만들어보자.

![](list-6-24.jpeg)

- 첫번째 매개변수에는 다이내믹 프록시가 정의되는 클래스 로더를 지정한다.
- 두번째 매개변수는 해당 프록시가 구현할 인터페이스를 전달한다.
    - 인터페이스는 여러개를 구현할 수 있으므로, 배열로 전달한다.
- 세번째 매개변수로는 메서드 호출을 위임할 `InvocationHandler` 클래스 인스턴스를 전달한다.

```java
public class HelloTest {
    // ...
    
    @Test
    public void dynamicProxy() {
        Hello proxiedHello = (Hello) Proxy.newProxyInstance(
                getClass().getClassLoader(),
                new Class[] { Hello.class },
                new UppercaseHandler(new HelloTarget())
        );

        Assertions.assertEquals("HELLO TOBY", proxiedHello.sayHello("Toby"));
        Assertions.assertEquals("HI TOBY", proxiedHello.sayHi("Toby"));
        Assertions.assertEquals("THANK YOU TOBY", proxiedHello.sayThankYou("Toby"));
    }
}
```

- 그다지 효율적으로 보이진 않는다. 도대체 무슨 장점이 있는 걸까?

### 다이내믹 프록시의 확장

- 만약 `Hello` 인터페이스의 메서드가 수십개라면 어떨까?
    - 이 모든 메서드의 위임코드를 작성하는게 쉽진 않을 것이다.
- 현재의 구현체는 모든 메서드의 리턴 타입이 `String`이라고 가정한다.
    - 만약 다른 리턴 타입을 갖는 메서드가 추가되면 어떨까?
- 이런 문제점을 해결하도록 코드를 수정해보자.

```java
public class UppercaseHandler implements InvocationHandler {
    // 모든 타입을 받을 수 있도록 `Object` 타입으로 수정
    private Object target;

    public UppercaseHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object ret =  method.invoke(target, args);

        // `String`인 경우만 toUpperCase() 적용
        if (ret instanceof String) {
            return ((String)ret).toUpperCase();
        }

        return ret;
    }
}
```

- `InvocationHandler`는 `invoke()` 단일 메서드로 모든 요청을 처리하기 때문에 선택 과정이 필요할 수 있다.
- 이를 위한 메타 정보를 `invoke()` 매개변수로 받을 수 있다.

```java
public class UppercaseHandler implements InvocationHandler {
    // ...

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object ret =  method.invoke(target, args);

        // say로 시작하는 메서드임을 판별하여 적용
        if (ret instanceof String && method.getName().startsWith("say")) {
            return ((String)ret).toUpperCase();
        }

        return ret;
    }
}
```

## 6.3.3 다이내믹 프록시를 이용한 트랜잭션 부가기능

- `UserServiceTx`를 다이내믹 프록시 방식으로 변경해보자.
    - 모든 메서드에 대해 트랜잭션 처리코드를 적용해보자.

### 트랜잭션 `InvocationHandler`

```java
public class TransactionHandler implements InvocationHandler {
    private Object target;
    private PlatformTransactionManager transactionManager;
    private String pattern;

    public TransactionHandler(Object target, PlatformTransactionManager transactionManager, String pattern) {
        this.target = target;
        this.transactionManager = transactionManager;
        this.pattern = pattern;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        // 트랜잭션 적용 대상 메서드를 선별하여 트랜잭션 경계설정 기능을 부여한다.
        if (method.getName().startsWith(pattern)) {
            return invokeInTransaction(method, args);
        } else {
            return method.invoke(target, args);
        }
    }

    private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
        TransactionStatus status =
                this.transactionManager.getTransaction(new DefaultTransactionDefinition());

        // 트랜잭션 경계설정 코드
        try {
            Object ret = method.invoke(target, args);
            this.transactionManager.commit(status);
            return ret;
        } catch (InvocationTargetException e) {
            // 롤백을 적용하기 위한 예외는 RuntimeException 대신에
            // InvocationTargetException을 잡도록 한다.
            //
            // 리플렉션 메서드는 타겟 오브젝트에서 발생하는 예외를
            // InvocationTargetException 으로 래핑하여 던지기 때문이다.
            this.transactionManager.rollback(status);
            throw e.getTargetException();
        }
    }
}
```

- 타겟을 `Object`로 받기 때문에, 트랜잭션 경계설정이 필요한 모든 오브젝트를 대상으로 적용할 수 있다.

### `TransactionHandler`와 다이내믹 프록시를 이용하는 테스트

- 위에서 구현한 다이내믹 프록시인 `TransactionHandler`를 `UserServiceTx`를 대신 할 수 있는지 확인해보자.

![](list-6-28.jpeg)

## 6.3.4 다이내믹 프록시를 위한 팩토리 빈

- 이제 `TransactionHandler`와 다이내믹 프록시를 스프링의 DI를 통해 사용할 수 있도록 만들면 된다.
- 그런데 문제는 DI의 대상이 되는 다이내믹 프록시 오브젝트는 일반적인 스프링의 빈으로 등록할 방법이 없다는 점이다.
- 스프링은 해당 클래스의 이름을 가지고 리플렉션을 통해 오브젝트를 생성한다.
- 문제는 생성되는 다이내믹 프록시 오브젝트는 어떤 클래스인지 알 수가 없다.
- 다이내믹 프록시는 `Proxy.newProxyInstance()` 정적 메서드로만 생성될 수 있다.

### 팩토리 빈

- 사실 스프링은 클래스 정보를 가지고 디폴트 생성자를 통해 오브젝트를 만드는 방법 외에도 빈을 만들 수 있는 여러 가지 방법을 제공한다.
- 대표적으로 팩토리 빈을 이용한 방법이 있다.
- 팩토리 빈은 스프링을 대신해서 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈을 말한다.
- 팩토리 빈을 만드는 방법에는 여러 가지가 있는데, 가장 간단한 방법은 스프링의 `FactoryBean`이라는 인터페이스를 구현하는 것이다.

![](list-6-29.jpeg)

- `FactoryBean` 인터페이스를 구현한 클래스를 스프링의 빈으로 등록하면 팩토리 빈으로 동작한다.
- 이를 위한 학습 테스트를 하나 만들어보자.
- 먼저 스프링에서 빈 오브젝트로 만들어 사용하고 싶은 클래스를 하나 정의해보자.

![](list-6-30.jpeg)

- 위 클래스는 생성자가 `private`이므로 원래라면 생성될 수 없다.
- 하지만 스프링은 리플렉션을 이용하므로, `private` 접근 규약을 위반할 수 있다.
- 그러나 가능하다 하더라도 `private` 생성자를 가진 오브젝트를 빈으로 등록하는 것은 피해야 한다.
- 이제 `Message` 오브젝트를 팩토리 메서드 호출하여 빈으로 등록할 수 있도록 팩토리 빈 클래스를 만들어보자.

![](list-6-31.jpeg)

### 팩토리 빈의 설정 방법

![](list-6-32.jpeg)

- 일반적인 빈 설정과 다르지 않지만, `FactoryBean`을 빈으로 등록하는 경우 `MessageFactoryBean` 타입이 아닌 `Message` 타입이 된다는 점이다.
    - 이는 `getObjectType()` 메서드가 리턴하는 타입으로 결정된다.
- 또한 `getObject()` 메서드가 생성해주는 오브젝트가 `message` 빈의 오브젝트가 된다.
- 만약 팩토리 빈 자체를 가져오고 싶다면 `&` prefix를 붙이면 된다.

![](list-6-34.jpeg)

### 다이내믹 프록시를 만들어주는 팩토리 빈

- 팩토리 빈의 `getObject()`를 이용하면 다이내믹 프록시를 인스턴스화 하여 빈으로 주입할 수 있을 것이다.

![](picture-6-15.jpeg)

### 트랜잭션 프록시 팩토리 빈

- `TransactionHandler`를 이용하는 다이내믹 프록시를 생성하는 팩토리 빈 클래스를 정의해보자.

![](list-6-35-1.jpeg)
![](list-6-35-2.jpeg)

- 이제 `UserService` 빈은 `TxProxyFactoryBean` 팩토리 빈으로 설정하면 된다.

![](list-6-36.jpeg)

- XML 대신에 자바 설정파일을 이용하려면 `@Configuration` 어노테이션으로 빈 팩토리 메서드를 구현하면 된다.

```java
@Configuration
public class AppConfig {

    @Bean
    public UserService userService(PlatformTransactionManager transactionManager) {
        TxProxyFactoryBean txProxyFactoryBean = new TxProxyFactoryBean();
        txProxyFactoryBean.setTarget(userServiceImpl());
        txProxyFactoryBean.setTransactionManager(transactionManager);
        txProxyFactoryBean.setPattern("upgradeLevels");
        txProxyFactoryBean.setServiceInterface(UserService.class);
        return (UserService) txProxyFactoryBean.getObject();
    }

    @Bean
    public UserServiceImpl userServiceImpl() {
        return new UserServiceImpl();
    }
}
```

### 트랜잭션 프록시 팩토리 빈 테스트

- `UserServiceTest`에서 `add()` 테스트의 경우 `@Autowired`로 가져온 `UserService`빈을 사용하기 때문에 `TxProxyFactoryBean` 팩토리 빈이 생성하는 다이내믹 프록시를 통해 `UserService`를 사용할 것이다.
- 반면에 `upgradeLevels()`와 `mockUpgradeLevels()`는 목 오브젝트를 이용해 비즈니스 로직에 대한 단위 테스트로 만들었으니 트랜잭션과는 무관하다.
    - 가장 중요한 트랜잭션 적용 기능을 확인해야 하는데, `upgradeAllOrNothing()`의 경우는 수동 DI를 통해 직접 다이내믹 프록시를 만들어서 사용하니 팩토리 빈이 적용되지 않는다.

```java
// ...

    @Test
    public void upgradeAllOrNothing() {
        // 수동 DI를 이용
        UserServiceImpl testUserService = new TestUserService(userDao, mailSender, users.get(3).getId());

        userDao.deleteAll();
        for (User user: users) {
            userDao.add(user);
        }

        Assertions.assertThatThrownBy(testUserService::upgradeLevels)
                .isInstanceOf(TestUserServiceException.class);

        checkLevel(users.get(1), false);
    }
}
```

- `add()`는 트랜잭션이 적용되지 않는 메서드이므로 다이내믹 프록시를 거쳐도 단순 위임 방식으로만 동작한다.
    - `TxProxyFactoryBean`이 다이내믹 프록시를 기대한대로 완벽하게 구성해서 만들어주는지를 확인하려면 트랜잭션 기능을 테스트해봐야 한다.
- 기존의 `upgradeAllOrNothing()` 테스트의 경우, 예외 발생시 트랜잭션이 롤백됨을 확인하려면 비즈니스 로직 코드를 수정한 `TestUserService`를 타깃 오브젝트로 대신 사용해야 한다.
- 그러나 현재 스프링 빈 설정에는 `UserServiceImpl`이 설정되어 있으며, 이를 `TestUserService`로 변경하는 일이 쉽지 않다.
    - 가장 큰 문제는 타깃 오브젝트에 대한 레퍼런스는 `TransactionHandler` 오브젝트가 갖고 있는데, `TransactionHandler`는 `TxProxyFactoryBean` 내부에서 만들어져 다이내믹 프록시 생성에 사용될 뿐 별도로 참조할 방법이 없다는 점이다.

```java
public class TransactionHandler implements InvocationHandler {
    private Object target;
    private PlatformTransactionManager transactionManager;
    private String pattern;

    // TransactionHandler는 생성자로 타겟을 받아야 한다.
    public TransactionHandler(Object target, PlatformTransactionManager transactionManager, String pattern) {
        this.target = target;
        this.transactionManager = transactionManager;
        this.pattern = pattern;
    }

    // ...
}
```

![](list-6-35-1.jpeg)
![](list-6-35-2.jpeg)
- 문제는 `TransactionHandler`의 생성은 `TxProxyFactoryBean`의 `getObject()` 내부에서 생성된다는 점이다. 현재는 이를 테스트코드에서 제어할 방법이 없다.
- 이를 해결할 수 있는 방법은 `TestUserServic`를 사용하는 테스트용 설정을 별도로 만들거나 프록시 팩토리 빈 코드를 확장한다거나 할 수 있다.
    - `TransactionHandler`를 DI 받도록 수정하면 된다.
- 가장 단순한 방법은 어차피 `TxProxyFactoryBean`의 트랜잭션을 지원하는 프록시를 바르게 만들어주는지를 확인하는게 목적이므로 빈으로 등록된 `TxProxyFactoryBean`을 직접 가져와서 프록시를 만들어보면 된다.

![](list-6-37-1.jpeg)
![](list-6-37-2.jpeg)

## 6.3.5 프록시 팩토리 빈 방식의 장점과 한계

- `TransactionHandler`를 이용하는 다이내믹 프록시를 생성하는 `TxProxyFactoryBean`은 코드의 수정 없이도 다양한 클래스에 적용할 수 있다.
- 타깃 오브젝트에 맞는 프로퍼티 정보를 설정해서 빈으로 등록해주기만 하면 된다.
- 하나 이상의 `TxProxyFactoryBean`을 동시에 빈으로 등록해도 상관 없다.
    - 각 빈의 타입에 맞는 팩토리 빈을 이용해 스프링이 생성해준다.

![](list-6-38.jpeg)
![](list-6-39.jpeg)
![](list-6-40.jpeg)
- 모든 메서드에 적용하려면 `pattern`을 빈 문자열로 등록하면 된다.

![](picture-6-16.jpeg)

- 단순히 빈 설정을 변경한 것만으로도, 해당 클래스의 모든 메서드에 트랜잭션 기능이 설정됐다.

### 프록시 팩토리 빈 방식의 장점

- 인터페이스를 이용해 데코레이터를 직접 구현하는 경우, 모든 메서드를 직접 구현했어야 했고, 부가기능 코드가 각 위임코드마다 중복돼서 나타나게 된다.
- 프록시 팩토리 빈은 이러한 문제를 해결해준다.
    - 메서드 구현코드를 작성할 필요가 없다.
    - 부가기능은 핸들러를 이용해 한 번만 정의하면 된다.

### 프록시 팩토리 빈의 한계

- 타겟을 하나밖에 지정하지 못한다.
    - 수백개의 서비스 객체에 트랜잭션을 적용하려면? 빈 설정을 다 만들어줘야 하며, 모두 중복코드가 될 것이다.
- 하나의 타겟에 부가기능을 여러개 적용하기도 힘들다.
    - 마찬가지로 설정 파일에 중복이 발생할 것이다.
- `TransactionHandler` 오브젝트가 프록시 팩토리 빈 갯수만큼 생성된다.
    - 코드는 같지만 타겟 오브젝트가 다르므로 계속 생성된다.
    - 이것 역시 필요 없는 중복이 될 수 있다.

# 6.4 스프링의 프록시 팩토리 빈

## 6.4.1 `ProxyFactoryBean`

- 자바에는 다이내믹 프록시 외에도 편리한 방법으로 프록시를 만들 수 있도록 지원해주는 다양한 기술이 존재한다.
- 스프링은 이러한 프록시를 일관된 방법으로 만들 수 있게 도와주는 추상 레이어를 제공한다.
- `ProxyFactoryBean`은 `TxProxyFactoryBean`과 달리 순수하게 프록시를 생성하는 작업만을 담당하고, 프록시를 통해 제공해줄 부가기능은 별도의 빈에 둘 수 있다.
- `ProxyFactoryBean`이 생성하는 프록시에 사용할 부가기능은 `MethodInterceptor` 인터페이스를 구현해서 만든다.
- `MethodInterceptor`는 `InvocationHandler`와는 다르게 `invoke()` 메서드로 타겟 오브젝트에 대한 정보까지 전달 받는다.

```java
public class UppercaseHandler implements InvocationHandler {
    // ...

    // InvocationHandler의 `invoke`에는 어디에도 타겟 오브젝트 정보를 받을 수 있는 부분이 없다.
    // 따라서 타겟 오브젝트를 구현체가 알고 있어야만 한다.
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // ...
}
```

![](list-6-41-1.jpeg)
![](list-6-41-2.jpeg)

### Acvice: 타겟이 필요 없는 순수한 부가기능

- `MethodInterceptor`는 타겟 오브젝트에 대한 정보가 담긴 `MethodInvocation` 오브젝트를 전달 받기 때문에, 부가기능 구현에만 집중할 수 있다.
    - 즉, `InvocationHandler`와는 다르게 타겟에 대한 의존성이 없다.
- 또한 `MethodInterceptor`를 여러개 적용할 수 있는 `addAdvice()` 메서드가 있어, 기존의 문제점이었던 여러 개의 부가기능을 적용할 수 있는 것 또한 쉽게 해결 가능하다.

![](list-6-41-1.jpeg)

- `addAdvice()` 메서드 이름에서 확인할 수 있듯, `MethodInterceptor`는 부가기능을 추가할 수 있는 하나의 종류일 뿐, 다른 종류들 역시 존재함을 눈치챌 수 있다.
- 이처럼 스프링에서 부가기능을 담은 오브젝트를 `Advice`라고 부른다.
- `ProxyFactoryBean`에는 프록시가 구현해야 하는 타겟 오브젝트에 대한 정보가 없는데, 이는 스프링에서 Reflection을 통해 해당 오브젝트의 정보를 동적으로 알아낼 수 있기 때문이다.
    - 물론 이를 직접 지정하는 방법 또한 존재한다.

### Pointcut: 부가기능 적용 대상 메서드 선정 방법

- 기존 `InvocationHandler`를 직접 구현했을 때를 생각해보면, 메서드 이름을 비교하여 부가기능을 적용 대상 메서드를 선정하는 코드를 구현했었다.

```java
// ...

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        // 트랜잭션 적용 대상 메서드를 선별하여 트랜잭션 경계설정 기능을 부여한다.
        if (method.getName().startsWith(pattern)) {
            return invokeInTransaction(method, args);
        } else {
            return method.invoke(target, args);
        }
    }
// ...
```

- `MethodInterceptor`는 부가기능 적용 대상을 선정하는 책임을 다른 객체로 위임하여 처리한다.
    - 이렇게 적용 대상을 선정하는 책임을 가진 객체를 스프링에서는 `Pointcut`이라고 한다.

![](picture-6-17.jpeg)
![](picture-6-18.jpeg)

- `ProxyFactoryBean`은 `Advice`와 `Pointcut`을 DI받아 생성된 다이내믹 프록시들이 사용할 수 있도록 한다.
    - 이를 통해 기존 `InvocationHandler`와 달리 하나의 객체로 여러개의 다이내믹 프록시가 공유하여 사용할 수 있다.
- `MethodInterceptor`는 프록시로부터 전달받은 `MethodInvocation` 타입의 콜백 오브젝트를 이용하여 타겟 오브젝트의 메서드 호출 전/후에 부가기능을 적용할 수 있도록 설계되었다.
    - 즉, 전형적인 템플릿/콜백 패턴이다.
- 프록시로부터 `Advice`와 `Pointcut`을 독립시키고 DI받는 것은 전형적인 전략 패턴 구조다.
    - 덕분에 상황에 따라 기능확장을 유연하게 할 수 있다.

![](list-6-42.jpeg)

- 포인트컷이 필요 없을 때에는 `addAdvice()` 메서드를 호출해서 `Advice`만 등록했지만, `Pointcut`을 함께 등록할 때는 `addAdvisor()` 메서드를 호출해서 `Pointcut`과 `Advice`를 묶어서 추가해야 한다.
- 이는 `Pointcut`의 역할을 생각해보면 알 수 있는데, 대상을 선정하는 알고리즘이 추가됐다면 선정 대상에게 어떤 부가기능을 적용해야 하는지 결정해야 되는게 자연스럽기 때문이다.
    - 즉, `Pointcut`이 없다는 뜻은 모든 대상에 대해 `Advice`를 적용할 것이기 때문이고, `Pointcut`을 사용했다는 것은 특정 대상에게만 `Advice`를 적용하려 했기 때문이다.
- `Pointcut` + `Advice`를 `Advisor`라고 한다.

## 6.4.2 ProxyFactoryBean 적용

- `TxProxyFactoryBean`을 `ProxyFactoryBean`을 이용해서 수정해보자.

### TransactionAdvice

![](list-6-43.jpeg)

### 스프링 XML 설정파일

- 테스트에서 직접 DI 해서 사용했던 코드를 변경해주면 된다.
- 먼저 `Advice`를 등록하고, 트랜잭션 기능 적용을 위해 `transactionManager`만 DI 해주면 된다.

![](list-6-44.jpeg)
![](list-6-45.jpeg)
![](list-6-46.jpeg)
![](list-6-47-1.jpeg)

### 어드바이스와 포인트컷의 재사용

- `ProxyFactoryBean`의 설계구조로 인해 우리는 `Advice`, `Pointcut` 으로 관심사를 분리할 수 있었다.
- 또한 스프링의 DI, 템플릿/콜백 패턴, 서비스 추상화 등의 기법이 적용돼 변경에 유연하도록 만들 수 있었다.

![](picture-6-19.jpeg)

---
# 6.5 스프링 AOP

- 지금까지 해왔던 작업의 목표는 비즈니스 로직에 반복적으로 등장해야만 했던 트랜잭션 코드를 깔끔하고 효과적으로 분리해내는 것이다.
- 이렇게 분리해낸 트랜잭션 코드는 투명한 부가기능 형태로 제공돼야 한다.
- 투명하다는 건 부가기능을 적용한 후에도 기존 설계와 코드에는 영향을 주지 않는다는 뜻이다.
- 투명하기 때문에 언제든지 자유롭게 추가하거나 제거할 수도 있고, 기존 코드는 항상 원래의 상태를 유지할 수 있다.

## 6.5.1 자동 프록시 생성

- 투명한 부가기능을 적용하는 과정에서 발견됐던 문제는 거의 대부분 제거했다.
- 하지만 아직 남은 문제가 있다.
- 부가기능의 적용이 필요한 타겟 오브젝트마다 거의 비슷한 내용의 `ProxyFactory` 빈 설정정보를 추가해주는 부분이다.
- 새로운 타겟이 등장했다고 해서 코드를 손댈 필요는 없어졌지만, 설정은 매번 복사해서 붙이고 target 프로퍼티의 내용을 수정해줘야 한다.
- 이를 제거할 방법은 없을까?

### 중복 문제의 접근 방법

- 앞에서 JDBC를 리팩토링 할 때에는 바뀌는 부분과 바뀌지 않는 부분을 구분해서 분리하고, 템플릿/콜백 패턴, 클라이언트 나누기 등을 통해 해결했다.
- 전략 패턴과 DI를 적용한 덕분이다.
- 프록시의 경우, 다이내믹 프록시라는 런타임 코드 자동생성 기법을 이용했다.
- 변하지 않는 타겟으로의 위임과 부가기능 적용 여부 판단이라는 부분은 코드 생성 기법을 이용하는 다이내믹 프록시 기술에 맡기고, 변하는 부가기능 코드는 별도로 만들어서 다이내믹 프록시 생성 팩토리에 DI로 제공하는 방법을 사용했다.
- 반복적인 프록시의 메서드 구현을 코드 자동생성 기법을 이용해 해결했다면, 반복적인 `ProxyFactoryBean` 설정 문제는 설정 자동등록 기법으로 해결할 수 없을까?
- 일정한 타겟 빈의 목록을 제공하면 자동으로 각 타깃 빈에 대한 프록시를 만들어주는 방법이 있다면 `ProxyFactoryBean` 타입 빈 설정을 매번 추가해서 프록시를 만들어내는 수고를 덜 수 있을 것 같다.
### 빈 후처리기를 이용한 자동 프록시 생성기

- 스프링은 OCP의 가장 중요한 요소인 유연한 확장이라는 개념을 스프링 컨테이너 자신에게도 다양한 방법으로 적용하고 있다.
- 스프링은 컨테이너로서 제공하는 기능 중에서 변하지 않는 핵심적인 부분 외에는 대부분 확장할 수 있도록 확장 포인트를 제공해준다.
- 그중에서 관심을 가질 만한 확장 포인트는 바로 `BeanPostProcessor `인터페이스를 구현해서 만드는 빈 후처리기다.
- 빈 후처리기는 이름 그대로 스프링 빈 오브젝트로 만들어지고 난 후에, 빈 오브젝트를 다시 가공할 수 있게 해준다.

![](picture-6-20.jpeg)

### `DefaultAdvisorAutoProxyCreator`

- 어드바이저를 이용한 자동 프록시 생성기다.
- 빈 후처리기는 `FeactoryBean`처럼 스프링 컨테이너에 빈으로 등록하면 자동으로 적용된다.
- 스프링은 빈 후처리기가 빈으로 등록되어 있으면 빈 오브젝트가 생성될 때마다 빈 후처리기에 보내서 후처리 작업을 요청한다.
- 이를 이용하면 스프링이 생성하는 빈 오브젝트의 일부를 프록시로 포장하고, 프록시를 빈으로 대신 등록할 수도 있다.
- `DefaultAdvisorAutoProxyCreator` 빈 후처리기가 등록되어 있으면 스프링은 빈 오브젝트를 만들 때마다 후처리기에게 빈을 보낸다.
- 그 후, 빈으로 등록된 모든 어드바이저 내의 포인트컷을 이용해 전달받은 빈이 프록시 적용 대상인지 확인한다.
- 프록시 적용 대상이면 그때는 내장된 프록시 생성기에게 현재 빈에 대한 프록시를 만들게 하고, 만들어진 프록시에 어드바이저를 연결해준다.
- 그 뒤, 컨테이너에 해당 프록시를 돌려준다.
- 적용할 빈을 선정하는 로직이 추가된 포인트컷이 담긴 어드바이저를 등록하고 빈 후처리기를 사용하면 일일이 `ProxyFactoryBean` 빈을 등록하지 않아도 타겟 오브젝트에 자동으로 프록시가 적용되게 할 수 있다.
- 이는 마지막 남은 번거로운 `ProxyFactoryBean` 설정 문제를 말끔하게 해결해주는 놀라운 방법이다.

![](list-6-49.jpeg)

## **6.5.2** `DefaultAdvisorAutoProxyCreator`의 적용

### 클래스 필터를 적용한 포인트컷 작성

- 메서드 이름만 비교하던 포인트컷인 `NameMatchMethodPointcut`을 상속해서 프로퍼티로 주어진 이름 패턴을 가지고 클래스 이름을 비교하는 `ClassFilter`를 추가하도록 만들 것이다.

![](list-6-51.jpeg)

### 어드바이저를 이용하는 자동 프록시 생성기 등록

- `DefaultAdvisorAutoProxyCreator`는 등록된 빈 중에서 `Advisor` 인터페이스를 구현한 것을 모두 찾는다.
- 그리고 생성되는 모든 빈에 대해 어드바이저의 포인트컷을 적용해보면서 프록시 적용 대상을 선정한다.
- 빈 클래스가 프록시 선정 대상이라면 프록시를 만들어 원래 빈 오브젝트와 바꿔치기 한다.
- 따라서 타겟 빈에 의존한다고 정의한 다른 빈들은 프록시 오브젝트를 대신 DI 받게 될 것이다.
- `DefaultAdvisorAutoProxyCreator` 등록은 XML에 id 없이 등록하면 된다.
- 다른 빈에서 참조되거나 코드에서 빈 이름으로 조회될 필요가 없는 빈이라면 아이디를 등록하지 않아도 무방하다.

### 포인트컷 등록

![](list-6-52.jpeg)

- `ServiceImpl`로 이름이 끝나고, `upgrade`로 이름이 시작하는 메서드를 대상으로 판별하는 포인트컷을 빈으로 등록했다.

### 어드바이스와 어드바이저

- 어드바이스인 `transactionAdvice` 빈의 설정은 수정할 게 없다.
- 어드바이저인 `transactionAdvisor` 빈도 수정할 필요는 없다.
- 하지만 어드바이저로서 사용되는 방법이 바뀌었다는 사실은 기억해두자.
- 이제는 `ProxyFactoryBean`으로 등록한 빈에서처럼 `transactionAdvisor`를 명시적으로 DI 하는 빈은 존재하지 않는다.
- 대신 어드바이저를 이용하는 자동 프록시 생성기인 `DefaultAdvisorAutoProxyCreator`에 의해 자동수집되고, 프록시 대상 선정 과정에 참여하며, 자동생성된 프록시에 다이내믹하게 DI 돼서 동작하는 어드바이저가 된다.
### `ProxyFactoryBean` 제거와 서비스 빈의 원상복구

- 프록시를 도입했던 때부터 아이디를 바꾸고 프록시에 DI 돼서 간접적으로 사용돼야 했던 `userServiceImpl` 빈의 아이디를 이제는 당당하게 `userService`로 되돌려놓을 수 있다.
- 더 이상 명시적인 프록시 팩토리 빈을 등록하지 않기 때문이다.
- 마지막으로 남았던 `ProxyFactoryBean` 타입의 빈은 삭제해버려도 좋다.
- `UserService`와 관련된 빈 설정은 이제 `userService` 빈 하나로 충분하다.

![](list-6-53.jpeg)

### 자동 프록시 생성기를 사용하는 테스트

- 이제 `UserService` 타입 오브젝트는 `UserServiceImpl` 오브젝트가 아니라 트랜잭션이 적용된 프록시여야 한다.
- 이를 검증하려면 `upgradeAllOrNothing()` 테스트가 필요한데, 기존의 테스트 코드에서 사용한 방법으로는 이제 한계가 있다.
- 지금까지는 `ProxyFactoryBean`이 빈으로 등록되어 있었으므로 이를 가져와 타겟을 테스트용 클래스로 바꿔치기하는 방법을 사용했다.
- 하지만 자동 프록시 생성기를 적용한 후에는 더 이상 가져올 `ProxyFactoryBean` 같은 팩토리 빈이 존재하지 않는다.
- 자동 프록시 생성기라는 스프링 컨테이너에 종속적인 기법을 사용했기 때문에 예외상황을 위한 테스트 대상도 빈으로 등록해 줄 필요가 있다.
- 기존에 만들어서 사용하던 강제 예외 발생용 `TestUserService` 클래스를 직접 빈으로 등록하자.
- 대신 클래스 이름은 포인트컷이 선정해줄 수 있도록 `ServiceImpl`로 끝나야 한다.

![](list-6-54.jpeg)
![](list-6-55.jpeg)
![](list-6-56.jpeg)

### 자동생성 프록시 확인

- 지금까지 트랜잭션 어드바이스를 적용한 프록시 자동생성기를 빈 후처리기 메커니즘을 통해 적용했다.
- 최소한 두 가지는 확인해야 한다.
    1. 트랜잭션이 필요한 빈에 트랜잭션 부가기능이 적용됐는가이다. 트랜잭션이 정상적으로 커밋되는 경우에는 트랜잭션 적용 여부를 확인하기 힘들기 때문에 예외상황에서 트랜잭션이 롤백되게 함으로써 트랜잭션 적용 여부를 테스트해야 한다.
    2. 아무 빈에나 트랜잭션 부가기능이 적용된 것은 아닌지 확인해야 한다. 프록시 자동생성기가 어드바이저 빈에 연결해둔 포인트컷의 클래스 필터를 이용해서 정확히 원하는 빈에만 프록시를 생성했는지 확인이 필요하다.
- 방법은 간단하다. 포인트컷 빈의 클래스 이름 패턴을 변경해서 이번엔 `testUserService` 빈에 트랜잭션이 적용되지 않게 해보자.
- 이 테스트도 제대로 하려면 전용 설정파일을 만들어 포인트컷을 재구성하는 등의 복잡한 과정이 필요할 텐데, 여기서는 간단히 현재 테스트 설정파일을 수정해서 확인하고 다시 원상복귀시키는 것으로 하자.

![](list-6-58.jpeg)

## 6.5.3 포인트컷 표현식을 이용한 포인트컷

- 리플렉션 API는 클래스 이름, 메서드 이름, 정의된 패키지, 파라미터, 리턴값, 어노테이션, 인터페이스, 상속정보 등을 런타임에 알아낼 수 있다.
- 포인트컷 표현식을 이용하면 이들 정보를 이용해 포인트컷을 작성할 수 있다.

### 포인트컷 표현식

- `AspectJExpressionPointcut` 클래스를 사용하면 포인트컷 표현식을 이용할 수 있다.
- 스프링은 `AspectJ`라는 유명한 프레임워크에서 제공하는 것을 가져와 일부 문법을 확장해서 사용한다.
- 그래서 이를 `AspectJ` 포인트컷 표현식이라고도 한다.

### 포인트컷 표현식 문법

- `AspectJ` 포인트컷 표현식은 포인트컷 지시자를 이용한다.
- 포인트컷 지시자중 가장 대표적인 것은 `execution()` 지시자다.
- 리플렉션을 통한 시그니처를 생각하면 된다.

![](point-cut-expression.jpeg)

```java
System.out.println(Target.class.getMethod("minus", int.class, int.class));
// public int springbook.learningtest.spring.pointcut.Target.minus(int,int) throws java.lang.RuntimeException
```

![](table-6-1.jpeg)

## 6.5.4 AOP란 무엇인가?

- 비즈니스 로직을 담은 UserService에 트랜잭션을 적용해온 과정을 정리해보자.

### 트랜잭션 서비스 추상화

- 비즈니스 로직을 담은 코드에 트랜잭션 코드가 들어가면서 특정 트랜잭션 기술에 종속되는 코드가 됐다.
- JDBC의 로컬 트랜잭션 방식을 적용한 코드를 JTA를 이용한 글로벌/분산 트랜잭션 방식으로 바꾸려면 모든 트랜잭션 적용 코드를 수정해야 한다는 심각한 문제점이 발견됐다.
- 그래서 트랜잭션 적용이라는 추상적인 작업 내용은 유지한 채로 구체적인 구현 방법을 자유롭게 바꿀 수 있도록 서비스 추상화 기법을 적용했다.
- 트랜잭션 추상화란 결국 인터페이스와 DI를 통해 무엇을 하는지는 남기고, 그것을 어떻게 하는지를 분리한 것이다.

### 프록시와 데코레이터 패턴

- 트랜잭션을 어떻게 다룰 것인가는 추상화를 통해 코드에서 제거했지만, 여전히 비즈니스 로직 코드에는 트랜잭션을 적용하고 있다는 사실은 드러나 있다.
- 트랜잭션이라는 부가적인 기능을 어디에 적용할 것인가는 여전히 코드에 노출시켜야 했다.
- 문제는 트랜잭션은 거의 대부분의 비즈니스 로직을 담은 메서드에 필요하다는 점이다.
- 그래서 도입한 것이 바로 DI를 이용해 데코레이터 패턴을 적용하는 방법이었다.
- 같은 인터페이스를 구현하고, DI를 통해 트랜잭션 인스턴스가 비즈니스 인스턴스를 위임하는 방식을 통해 사용했다.

### 다이내믹 프록시와 프록시 팩토리 빈

- 프록시를 이용해서 비즈니스 로직 코드에서 트랜잭션 코드는 모두 제거할 수 있었지만, 비즈니스 로직 인터페이스의 모든 메서드마다 트랜잭션 기능을 부여하는 코드를 넣어 프록시 클래스를 만드는 작업이 오히려 큰 짐이 됐다.
- 그래서 프록시 클래스 없이도 프록시 오브젝트를 런타임시에 만들어주는 JDK 다이내믹 프록시 기술을 적용했다.
- 하지만 동일한 기능의 프록시를 여러 오브젝트에 적용할 경우 오브젝트 단위로는 중복이 일어나는 문제는 해결하지 못했다.
- JDK 다이내믹 프록시와 같은 프록시 기술을 추상화한 스프링의 프록시 팩토리 빈을 이용해서 다이내믹 프록시 생성 방법에 DI를 도입했다.

### 자동 프록시 생성 방법과 포인트컷

- 트랜잭션 적용 대상이 되는 빈마다 일일이 프록시 팩토리 빈을 설정해줘야 한다는 부담이 남아 있었다.
- 이를 해결하기 위해서 스프링 컨테이너의 빈 생성 후처리 기법을 활용해 컨테이너 초기화 시점에서 자동으로 프록시를 만들어주는 방법을 도입했다.
- 대상을 선정하는 방법은 포인트컷을 이용했다.

### 부가기능의 모듈화

- 관심사가 같은 코드를 분리해 한데 모으는 것은 소프트웨어 개발의 가장 기본이 되는 원칙이다.
- 하지만 트랜잭션 적용 코드는 이러한 방법으로는 해결할 수 없었다.
- 왜냐하면 트랜잭션 경계설정 기능은 다른 모듈의 코드에 부가적으로 부여되는 기능이라는 특징이 있기 때문이다.
- 그래서 트랜잭션 코드는 한데 모을 수 없고, 애플리케이션 전반에 여기저기 흩어져 있다.
- 트랜잭션 같은 부가기능은 핵심기능과 같은 방식으로는 모듈화하기가 매우 힘들다.
- 이름 그대로 부가기능이기 떄문에 스스로는 독립적인 방식으로 존재해서는 적용되기 어렵기 때문이다.
- 트랜잭션 부가기능이란 트랜잭션 기능을 추가해줄 다른 대상, 즉 타겟이 존재해야만 의미가 있다.
- 따라서 각 기능을 부가할 대상인 각 타겟의 코드 안에 침투하거나 긴밀하게 연결되어 있지 않으면 안 된다.
- 결국 지금까지 해온 모든 작업은 핵심기능에 부여되는 부가기능을 효과적으로 모듈화하는 방법을 찾는 것이었고, 어드바이스와 포인트컷을 결합한 어드바이저가 단순하지만 이런 특성을 가진 모듈의 원시적인 형태로 만들어지게 됐다.

### AOP: 애스펙트 지향 프로그래밍

- 애플리케이션의 핵심기능을 담고 있지는 않지만, 핵심기능에 부가되어 의미를 갖는 특별한 모듈을 에스팩트라고 한다.
- 에스팩트는 부가될 기능을 정의할 코드인 어드바이스와, 어드바이스를 어디에 적용할지를 결정하는 포인트컷을 함께 갖고 있다.
- 지금까지 사용해온 어드바이저는 아주 단순한 형태의 에스팩트라고 볼 수 있다.
- 즉, 애플리케이션의 핵심적인 기능에서 부가적인 기능을 분리해서 에스팩트라는 독특한 모듈로 만들어서 설계하고 개발하는 방법을 에스팩트 지향 프로그래밍 또는 AOP라고 부른다.

![](picture-6-21.jpeg)

> AOP는 OOP를 돕는 보조적인 기술이지 OOP를 완전히 대체하는 새로운 개념은 아니다.

## 6.5.5 AOP 적용기술

프록시를 이용한 AOP

- 스프링 AOP는 자바의 기본 JDK에 포함된 프록시를 이용하기 때문에, 특별한 기술이나 환경을 요구하지 않는다.  

바이트코드 생성과 조작을 통한 AOP

- AspectJ는 스프링처럼 다이내믹 프록시 방식을 사용하지 않는다.
- 타겟 오브젝트를 뜯어고쳐서 부가기능을 직접 넣어주는 직접적인 방법을 사용한다.
- 부가기능을 넣는다고 타겟 오브젝트의 소스코드를 수정할 수는 없으니, 컴파일된 타겟의 클래스 파일 자체를 수정하거나 클래스가 JVM에서 로딩되는 시점을 가로채서 바이트코드를 조작하는 복잡한 방법을 사용한다.
- 물론 소스코드를 수정하지 않으므로 개발자는 계속해서 비즈니스로직에 충실한 코드를 만들 수 있다.
- 왜 이렇게 복잡하고 어려운 방법을 사용할까?

1. 바이트코드를 조작해서 타겟 오브젝트를 직접 수정해버리면 스프링과 같은 DI 컨테이너의 도움을 받아서 자동 프록시 생성 방식을 사용하지 않아도 AOP를 적용할 수 있기 때문이다.
1. 프록시 방식보다 훨씬 강력하고 유연한 AOP가 가능하다.

- 프록시를 AOP의 핵심 매커니즘으로 사용하면 부가기능을 부여할 대상은 클라이언트가 호출할 때 사용하는 메서드로 제한된다.
- 하지만 바이트코드를 직접 조작하는 방식은 오브젝트의 생성, 필드 값의 조회와 조작, static 초기화 등 다양한 작업에 부가기능을 부여해줄 수 있다.
- 물론 대부분의 부가기능은 프록시 방식을 사용해 메서드의 호출 시점에 부여하는 것으로도 충분하다.
- 게다가 AspectJ 같은 고급 AOP 기술은 바이트코드 조작을 위해 JVM의 실행 옵션을 변경하거나, 별도의 바이트코드 컴파일러를 사용하거나, 특별한 클래스 로더를 사용하게 하는 등의 번거로운 작업이 필요하다.

## 6.5.6 AOP의 용어

- AOP에서 많이 사용되는 몇 가지 용어를 살펴보자.
### 타겟

- 부가기능을 부여할 대상이다.
- 클래스일 수도 있지만 프록시 오브젝트일 수도 있다.

### 어드바이스

- 타겟에게 제공할 부가기능을 담은 모듈이다.
- 오브젝트로 정의하기도 하지만 메서드 레벨에서 정의할 수도 있다.
### 조인 포인트

- 어드바이스가 적용될 수 있는 위치를 말한다.
- 스프링의 프록시 AOP에서 조인 포인트는 메서드의 실행 단계 뿐이다.
- 타겟 오브젝트가 구현한 인터페이스의 모든 메서드는 조인 포인트가 된다.

### 포인트컷

- 어드바이스를 적용할 조인 포인트를 선별하는 작업 또는 그 기능을 정의한 모듈을 말한다.
- 스프링 AOP의 조인 포인트는 메서드의 실행이므로 스프링의 포인트컷은 메서드를 선정하는 기능을 갖고 있다.
- 그래서 포인트컷 표현식은 메서드의 실행이라는 의미인 `execution`으로 시작하고 메서드 시그니처를 비교하는 방법을 주로 이용한다.

### 프록시

- 클라이언트와 타겟 사이에 투명하게 존재하면서 부가기능을 제공하는 오브젝트다.

### 어드바이저

- 포인트컷과 어드바이스를 하나씩 갖고 있는 오브젝트다.
- 어드바이저는 어떤 부가기능(어드바이스)를 어디에(포인트컷) 전달할 것인가를 알고 있는 AOP의 가장 기본이 되는 모듈이다.
- 어드바이저는 스프링 AOP에서만 사용되는 특별한 용어이고, 일반적인 AOP에서는 사용되지 않는다.

### 에스펙트

- AOP의 기본 모듈이다.
- 한 개 또는 그 이상의 포인트컷과 어드바이스의 조합으로 만들어지며 보통 Singleton 형태의 오브젝트로 존재한다.

## 6.5.7 AOP 네임스페이스

- 스프링의 AOP를 적용하기 위해 추가했던 어드바이저, 포인트컷, 자동 프록시 생성기 같은 빈들은 애플리케이션의 로직을 담은 `UserDao`나 `UserService` 빈과는 성격이 다르다.
- 이런 빈들은 스프링 컨테이너에 의해 자동으로 인식돼서 특별한 작업을 위해 사용된다.
- 스프링의 프록시 방식 AOP를 적용하려면 최소한 네 가지의 빈을 등록해야 한다.

### 자동 프록시 생성기

- `DefaultAdvisorAutoProxyCreator` 클래스를 빈으로 등록한다.
- 다른 빈을 DI 하지도 않고 자신도 DI 되지 않으며 독립적으로 존재한다.
- 빈 후처리기로 참여한다.

### 어드바이스

- 부가기능을 구현한 클래스를 빈으로 등록한다.
- `TransactionAdvice`는 AOP 관련 빈중에서 유일하게 직접 구현한 클래스를 사용한다.

### 포인트컷

- 스프링의 `AspectJExpressionPointcut`을 빈으로 등록하고 `expression` 프로퍼티에 포인트컷 표현식을 넣어주면 된다.

### 어드바이저

- 스프링의 `DefaultPointcutAdvisor` 클래스를 빈으로 등록해서 사용한다.
- 자동 프록시 생성기에 의해 자동 검색되어 사용된다.

어드바이스를 제외한 나머지는 모두 스프링이 직접 제공하는 클래스를 빈으로 등록하고 프로퍼티 설정만 해준다.

### AOP 네임스페이스

- 스프링에서는 이렇게 AOP를 위해 기계적으로 적용하는 빈들을 간편한 방법으로 등록할 수 있다.
- 스프링은 AOP와 관련된 태그를 정의해둔 aop 스키마를 제공한다.

![](list-6-66.jpeg)

# 6.6 트랜잭션 속성

- `PlatformTransactionManager`을 사용하면서 `DefaultTransactionDefinition` 오브젝트를 사용해왔는데, 이 오브젝트에 대해 자세히 설명하진 않았었다. 이제 용도를 알아보자.

## 6.6.1 트랜잭션 정의

- 트랜잭션이라고 모두 같은 방식으로 동작하는 것은 아니다.
- 물론 트랜잭션의 기본 개념인 더 이상 쪼갤 수 없는 최소 단위의 작업이라는 개념은 항상 유효하다.
- `DefaultTransactionDefinition`이 구현하고 있는 `TransactionDefinition` 인터페이스는 트랜잭션의 동작방식에 영향을 줄 수 있는 네 가지 속성을 정의하고 있다.

### 트랜잭션 전파

- 트랜잭션 전파란 트랜잭션의 경계에서 이미 진행 중인 트랜잭션이 있을 때 또는 없을 때 어떻게 동작할 것인가를 결정하는 방식을 말한다.

![](picture-6-22.jpeg)

### PROPAGATION_REQUIRED

- 가장 많이 사용되는 트랜잭션 전파 속성이다.
- 진행 중인 트랜잭션이 없으면 새로 시작하고, 이미 시작된 트랜잭션이 있으면 이에 참여한다.
### PROPAGATION_REQUIRES_NEW

- 항상 새로운 트랜잭션을 시작한다.  
### PROPAGATION_NOT_SUPPORTED

- 트랜잭션 없이 동작하도록 만들 수 있다.
- 진행 중인 트랜잭션이 있어도 무시한다.
- 트랜잭션 없이 동작하게 할 거라면 뭐하러 트랜잭션 경계를 설정해두는 것일까?
- 트랜잭션을 무시하는 속성을 두는 데는 이유가 있다.
- 트랜잭션 경계설정은 보통 AOP를 이용해 많은 메서드에 동시에 적용하는 방법을 사용한다.
- 그런데 그 중에서 특별한 메서드만 트랜잭션을 제외하려면?
- 이를 쉽게 하는 방법은 그냥 모든 메서드에 트랜잭션 AOP가 적용되게 하고, 특정 메서드의 트랜잭션만 적용 안되도록 동작하는게 낫기 때문에 이런 속성이 존재한다.

### 격리수준

- 모든 DB 트랜잭션은 격리수준(Isolation Level)을 갖고 있어야 한다.
    - `READ UNCOMMITTED`
    - `READ COMMITTED`
    - `REPEATABLE READ`
    - `SERIALIZABLE`
- 격리수준은 기본적으로 DB에 설정되어 있지만 JDBC 드라이버나 DataSource 등에서 재설정할 수 있고, 필요하다면 트랜잭션 단위로 격리 수준을 조정할 수 있다.

### 제한시간

- 트랜잭션을 수행하는 제한시간(timeout)을 설정할 수 있다.
- 제한시간은 트랜잭션을 직접 시작할 수 있는 `PROPAGATION_REQUIRED`나 `PROPAGATION_REQUIRES_NEW`와 함께 사용해야만 의미가 있다.

### 읽기전용

- 읽기전용으로 설정해두면 트랜잭션 내에서 데이터를 조작하는 시도를 막아줄 수 있다.
- 또한 데이터 엑세스 기술에 따라서 성능이 향상될 수도 있다.

### 트랜잭션 동작방식 정의

- `TransactionDefinition` 타입 오브젝트를 사용하면 네 가지 속성을 이용해 트랜잭션의 동작방식을 제어할 수 있다.
- 트랜잭션 정의를 수정하려면 어떻게 해야 할까?
- `TransactionDefinition` 오브젝트를 생성하고 사용하는 코드는 트랜잭션 경계설정 기능을 가진 `TransactionAdvice`다.
- 트랜잭션 정의를 바꾸고 싶다면 디폴트 속성을 갖고 있는 `DefaultTransactionDefinition`을 사용하는 대신 외부에서 정의된 `TransactionDefinition` 오브젝트를 DI 받아서 사용하도록 만들면 된다.
- 하지만 이 방법으로 트랜잭션 속성을 변경하면 `TransactionAdvice`를 사용하는 모든 트랜잭션 속성이 한꺼번에 바뀐다는 문제가 있다.
- 원하는 메서드만 선택해서 독자적인 트랜잭션 정의를 적용할 수 있는 방법은 없을까?

## 6.6.2 트랜잭션 인터셉터와 트랜잭션 속성

- 메서드별로 다른 트랜잭션 정의를 적용하려면 어드바이스의 기능을 확장해야 한다.
- 메서드 이름 패턴에 따라 트랜잭션 정의가 적용되도록 만드는 것이다.
### `TransactionInterceptor`

- 스프링에는 트랜잭션 경계설정 어드바이스로 사용할 수 있도록 만들어진 `TransactionInterceptor`가 존재한다.
- 이는 우리가 만든 `TransactionAdvice`와 크게 다르지 않으며, 트랜잭션 정의를 메서드 이름 패턴을 이용해서 다르게 지정할 수 있는 방법을 추가로 제공해준다.
- `PlatformTransactionManager`와 Properties 타입의 두 가지 프로퍼티를 갖고 있다.
- Properties 타입의 프로퍼티는 트랜잭션 속성을 정의한 `transactionAttribute` 프로퍼티를 갖는다.
- 트랜잭션 속성은 `TransactionDefinition`의 네 가지 기본 항목에 `rollbackOn()`이라는 메서드를 하나 더 갖고 있는 `TransactionAttribute` 인터페이스로 정의되는데, 이는 어떤 예외가 발생하면 롤백을 할 것인가를 결정하는 메서드다.
- 스프링이 제공하는 `TransactionInterceptor`는 기본적으로 두 가지 종류의 예외처리 방식이 있다.
    1. 런타임 예외가 발생하면 트랜잭션은 롤백된다.
    2. 체크 예외가 발생하면 예외상황으로 해석하지 않고 일종의 비즈니스 로직에 따른, 의미가 있는 리턴 방식의 한 가지로 인식해서 트랜잭션을 커밋해버린다.
- 그런데 `TransactionInterceptor`의 이러한 예외처리 기본 원칙을 따르지 않는 경우가 있을 수 있다.
- 그래서 `TransactionInterceptor`는 `rollbackOn()`이라는 속성을 둬서 기본 원칙과 다른 예외처리가 가능하게 해준다.

![](list-6-70.jpeg)

### 메서드 이름 패턴을 이용한 트랜잭션 속성 지정

- `Properties` 타입의 `transactionAttributes` 프로퍼티는 메서드 패턴과 트랜잭션 속성을 key, value로 갖는 컬렉션이다.
- 트랜잭션 속성은 다음과 같은 문자열로 정의할 수 있다.
- PROPAGATION_: 트랜잭션 전파방식 prefix (필수값)
- ISOLATION_: 격리수준 prefix
- readOnly: 읽기전용 항목. 생략 가능. 디폴트는 읽기전용이 아니다.
- timeout_: 제한시간(초단위)
- -Exception: 롤백 대상으로 추가할 예외
- +Exception: 롤백 시키지 않을 런타임 예외
- 이렇게 속성을 하나의 문자열로 표현하게 만든 이유는 트랜잭션 속성을 메서드 패턴에 따라 여러 개를 지정해줘야 하는데, 일일이 중첩된 태그와 프로퍼티로 설정하게 만들면 번거롭기 때문이다.

![](transsaction-expression.jpeg)
![](list-6-71.jpeg)

### tx 네임스페이스를 이용한 설정 방법

- `TransactionInterceptor` 타입의 어드바이스 빈과 `TransactionAttribute` 타입의 속성정보도 tx 스키마의 전용 태그를 이용해 정의할 수 있다.

![](list-6-72-1.jpeg)
![](list-6-72-2.jpeg)
## 6.6.3 포인트컷과 트랜잭션 속성의 적용 전략

- 트랜잭션 부가기능을 적용할 후보 메서드를 선정하는 작업은 포인트컷에 의해 진행된다.
- 그리고 어드바이스의 트랜잭션 전파 속성에 따라서 메서드별로 트랜잭션의 적용방식이 결정된다.
- 포인트컷 표현식과 트랜잭션 속성을 정의할 때 따르면 좋은 몇 가지 전략을 생각해보자.

### 트랜잭션 포인트컷 표현식은 타입 패턴이나 빈 이름을 이용한다

- 일반적으로 트랜잭션을 적용할 타겟 클래스의 메서드는 모두 트랜잭션 적용 후보가 되는 것이 바람직하다.
- 쓰기 작업이 없는 단순한 조회 작업을 하는 메서드에도 모두 트랜잭션을 적용하는게 좋다.
- 조회의 경우 읽기전용으로 트랜잭션 속성을 설정해두면 그만큼 성능의 향상을 가져올 수 있다.
- 또 복잡한 조회의 경우는 제한시간을 지정해줄 수도 있고, 격리수준에 따라 조회도 반드시 트랜잭션 안에서 진행해야 할 필요가 발생하기도 한다.
- 따라서 트랜잭션용 포인트컷 표현식에는 메서드나 파라미터, 예외에 대한 패턴을 정의하지 않는게 바람직하다.

### 공통된 메서드 이름 규칙을 통해 최소한의 트랜잭션 어드바이스와 속성을 정의한다

- 실제로 하나의 애플리케이션에서 사용할 트랜잭션 속성의 종류는 그다지 다양하지 않다.
- 너무 다양하게 트랜잭션 속성을 부여하면 관리만 힘들어질 뿐이다.
- 따라서 기준이 되는 몇 가지 트랜잭션 속성을 정의하고 그에 따라 적합한 메서드 명명 규칙을 만들어두면 하나의 어드바이스만으로 애플리케이션의 모든 서비스 빈에 트랜잭션 속성을 지정할 수 있다.
- 예외적인 패턴이 발생하는 경우, 트랜잭션 어드바이스와 포인트컷을 새롭게 추가해줄 필요가 있다.
- 우선 디폴트 속성(*)부터 시작해서 점차 추가해나가면 된다.
- 일반화하기에는 적당하지 않은 특별한 트랜잭션 속성이 필요한 타겟 오브젝트에 대해서는 별도의 어드바이스와 포인트컷 표현식을 사용하는 편이 좋다.

![](list-6-73.jpeg)
![](list-6-74.jpeg)
![](list-6-75.jpeg)

### 프록시 방식 AOP는 같은 타겟 오브젝트 내의 메서드를 호출할 때는 적용되지 않는다

- 프록시 방식의 AOP에서는 프록시를 통한 부가기능의 적용은 클라이언트로부터 호출이 일어날 때만 가능하다.
- 자기 자신의 메서드를 호출할 때는 프록시를 통한 부가기능의 적용이 일어나지 않는다.
- 이는 AspectJ와 같은 타겟의 바이트코드를 직접 조작하는 방식의 AOP 기술을 적용하면 해결 가능하다.

![](picture-6-23.jpeg)
## 6.6.4 트랜잭션 속성 적용

- 이제 `UserService`에 적용해보자.
### 트랜잭션 경계설정의 일원화

- 트랜잭션 경계설정의 부가기능을 여러 계층에서 중구난방으로 적용하는 건 좋지 않다.
- 일반적으로 특정 계층의 경계를 트랜잭션 경계와 일치시키는 것이 바람직하다.
- 비즈니스 로직을 담고 있는 서비스 계층 오브젝트의 메서드가 트랜잭션 경계를 부여하기에 가장 적절한 대상이다.
- 트랜잭션은 보통 서비스 계층의 메서드 조합을 통해 만들어지기 때문에 DAO가 제공하는 주요 기능은 서비스 계층에 위임 메서드를 만들어둘 필요가 있다.
- 아키텍처를 단순하게 가져가면 서비스 계층과 DAO가 통합될 수도 있다.
- 하지만 비즈니스 로직을 독자적으로 두고 테스트하려면 서비스 계층을 만들어 사용해야 한다.
- `UserService` 인터페이스에 DAO 메서드를 위임하는 메서드들을 추가하자.

![](list-6-76.jpeg)![](list-6-77.jpeg)

### 서비스 빈에 적용되는 포인트컷 표현식 등록

- `upgradeLevels()`에만 트랜잭션이 적용되게 했던 기존 포인트컷 표현식을 모든 비즈니스 로직의 서비스 빈에 적용되도록 수정한다.

![](list-6-78.jpeg)

### 트랜잭션 속성을 가진 트랜잭션 어드바이스 등록

- `TransactionAdvice` 클래스로 정의했던 어드바이스 빈을 스프링의 `TransactionInterceptor`를 이용하도록 변경한다.

![](list-6-79.jpeg)

- 이미 aop 스키마의 태그를 적용했으니 어드바이스도 이왕이면 tx 스키마에 정의된 태그를 이용하도록 만드는게 낫다.

![](list-6-80.jpeg)

### 트랜잭션 속성 테스트

- `get`으로 시작하는 메서드는 쓰기 작업이 허용되지 않는다. 과연 그럴까? 한 번 테스트해보자.
- `TestUserService`를 활용해 새로 추가한 `getAll()` 메서드를 오버라이드해서 강제로 DB에 쓰기 작업을 추가해보자.

![](list-6-81.jpeg)
![](list-6-82.jpeg)
![](list-6-83.jpeg)

---

# 6.7 애노테이션 트랜잭션 속성과 포인트컷

- 포인트컷 표현식과 트랜잭션 속성을 이용해 트랜잭션을 일괄적으로 적용하는 방식은 복잡한 트랜잭션 속성이 요구되지 않는 한 대부분의 상황에 잘 들어맞는다.
- 그런데 가끔은 클래스나 메서드에 따라 제각각 속성이 다른, 세밀하게 튜닝된 트랜잭션 속성을 적용해야 하는 경우도 있다.
- 이런 경우 메서드 이름 패턴을 이용해서 일괄적으로 트랜잭션 속성을 부여하는 방식은 적합하지 않다.
- 경우에 따라 계속 패턴을 새로 추가해줘야 하기 떄문이다.
- 스프링에서는 애노테이션을 지정하는 방법을 사용한다.

## 6.7.1 트랜잭션 애노테이션

### `@Transactional`

- 타겟은 메서드와 타입이다.
- 이 애너테이션을 트랜잭션 속성정보로 사용하도록 지정하면 스프링은 `@Transactional`이 부여된 모든 오브젝트를 자동으로 타겟 오브젝트로 인식한다.
- 이 때 사용되는 포인트컷은 `TransactionAttributeSourcePointcut`이다.
- 즉, `@Transactional` 애너테이션은 기본적으로 트랜잭션 속성을 정의하는 것이지만, 동시에 포인트컷의 자동등록에도 사용된다.

![](list-6-84-1.jpeg)
![](list-6-84-2.jpeg)

### 트랜잭션 속성을 이용하는 포인트컷

- `@Transactional` 애노테이션이 사용됐을 때 어드바이저는 다음과 같이 동작한다.
- `TransactionInterceptor`는 메서드 이름 패턴 대신 `@Transactional` 애노테이션의 엘리먼트에서 트랜잭션 속성을 가져오는 `AnnotationTransactionAttributeSource`를 사용한다.
- `@Transactional`은 메서드마다 다르게 설정할 수도 있으므로 매우 유연한 트랜잭션 속성 설정이 가능해진다.
- 동시에 포인트컷도 `@Transacitonal`을 통한 트랜잭션 속성정보를 참조하도록 만든다.
- 이 방식을 이용하면 포인트컷과 트랜잭션 속성을 애노테이션 하나로 지정할 수 있다.
- 트랜잭션 부가기능 적용 단위는 메서드다.
- 이러면 유연한 속성 제어는 가능하겠지만 코드는 지저분해지고, 동일한 속성 정보를 가진 애노테이션을 반복적으로 메서드마다 부여해주는 바람직하지 못한 결과를 가져올 수 있다.

![](picture-6-24.jpeg)

### 대체 정책

- 그래서 스프링은 4단계의 fallback 정책을 이용하게 해준다.
- 메서드의 속성을 확인할 때 타겟 메서드, 타겟 클래스, 선언 메서드, 선언 타입의 순서에 따라 `@Transactional`이 적용됐는지 차례로 확인하고, 가장 먼저 발견되는 속성정보를 사용하게 하는 방법이다.

1. 타겟 메서드를 확인한다.
2. 타겟 타입을 확인한다.
3. 선언 메서드를 확인한다.
4. 선언 타입을 확인한다.

> 1, 2는 클래스 레벨이고 3, 4는 인터페이스 레벨이다.

- 기본적으로 `@Transactional` 적용 대상은 클라이언트가 사용하는 인터페이스가 정의한 메서드이므로 타겟 클래스보다는 인터페이스에 두는 게 바람직하다.
- 하지만 인터페이스를 사용하는 프록시 방식의 AOP가 아닌 방식으로 트랜잭션으로 적용하면 인터페이스에 정의한 애노테이션은 무시되기 때문에 안전하게 타겟 클래스에 애노테이션을 두는 방법을 권장한다.
- 즉, `@Transactional`의 적용 대상을 적절하게 변경해줄 확신이 있거나, 반드시 인터페이스를 사용하는 타깃에만 트랜잭션을 적용하겠다는 확신이 있다면 인터페이스에 적용하고, 아니면 타겟 클래스와 타겟 메서드에 적용하는 편이 낫다.
- 인터페이스에 적용하면 구현 클래스가 바뀌더라도 트랜잭션 속성을 유지할 수 있는 장점도 있다.

![](list-6-85.jpeg)

## 6.7.2 트랜잭션 애노테이션 적용

- `UserService`에 `@Transactional` 애노테이션을 적용해보자.

![](list-6-87.jpeg)

- 만약 구체클래스인 `UserServiceImpl`에 `@Transactional` 애노테이션을 적용하면, `readOnly` 속성은 무시된다.
    - 트랜잭션 애노테이션의 적용 우선순위가 클래스가 먼저이기 때문이다.

![](list-6-88.jpeg)

# 6.8 트랜잭션 지원 테스트

## 6.8.1 선언적 트랜잭션과 트랜잭션 전파 속성

- [트랜잭션 전파](Toby의%20스프링/6장/README.md#트랜잭션%20전파)
- AOP를 이용해 코드 외부에서 트랜잭션의 기능을 부여해주고 속성을 지정할 수 있게 하는 방법을 선언적 트랜잭션(declarative transaction)이라고 한다.
- 반대로 TransactionTemplate이나 개별 데이터 기술의 트랜잭션 API를 사용해 직접 코드 안에서 사용하는 방법은 프로그램에 의한 트랜잭션(programmatic transaction)이라고 한다.

![](picture-6-25.jpeg)

## 6.8.2 트랜잭션 동기화와 테스트

- 트랜잭션의 자유로운 전파와 그로 인한 유연한 개발이 가능할 수 있었던 기술적인 배경에는 AOP가 있었다.
- 또한, 스프링의 추상화 덕분에 데이터 엑세스 기술에 상관없이 트랜잭션을 추상 레벨로 관리할 수 있었다.

### 트랜잭션 매니저와 트랜잭션 동기화

- 트랜잭션 추상화 기술의 핵심은 트랜잭션 매니저와 트랜잭션 동기화다.
- `PlatformTransactionManager` 인터페이스를 구현한 트랜잭션 매니저를 통해 구체적인 트랜잭션 기술의 종류와 상관없이 일관된 트랜잭션 제어가 가능했다.
- 또한, 트랜잭션 동기화 기술이 있었기에 시작된 트랜잭션 정보를 저장소에 보관해뒀다가 DAO에서 공유할 수 있었다.
- 그런데 특별한 이유가 있다면 트랜잭션 매니저를 이용해 트랜잭션에 참여하거나 트랜잭션을 제어하는 방법을 사용할 수도 있다.
- `transactionManager`라는 빈 id로 `DataSourceTransactionManager`가 스프링에서 빈으로 선언되어있으므로, 간단하게` @Autowired`를 통해 가져와서 사용할 수 있다.

![](list-6-90.jpeg)
![](list-6-91.jpeg)

### 트랜잭션 매니저를 이용한 테스트용 트랜잭션 제어

![](list-6-92.jpeg)

- 위의 경우, `userService`는 모두 같은 트랜잭션에 참여하게 된다.

### 트랜잭션 동기화 검증

- 실제로 3개의 트랜잭션이 하나의 트랜잭션 단위에서 일어남을 어떻게 알 수 있을까?
- 트랜잭션을 시작할 때, `readOnly` 속성을 통해 확인해보면 된다.
- `deleteAll`과 `add`는 `readOnly` 속성이 아니므로, 만약 `readOnly` 트랜잭션 단위에서 실행됐다면 `TransientDataAccessResourceException` 예외가 발생할 것이다.

![](list-6-93.jpeg)

- `JdbcTemplate`과 같이 스프링이 제공하는 데이터 엑세스 추상화를 적용한 DAO에도 동일한 ㅕㅇ향을 미친다.
- `JdbcTemplate`은 트랜잭션이 시작된 것이 있으면 그 트랜잭션에 자동으로 참여하고, 없으면 트랜잭션 없이 자동커밋 모드로 `JDBC` 작업을 수행한다.

![](list-6-94.jpeg)

- 트랜잭션이라면 당연히 롤백도 가능해야 한다.

![](list-6-95.jpeg)

### 롤백 테스트

- 롤백 테스트는 테스트를 진행하는 동안에 조작한 데이터를 모두 롤백하고 테스트를 시작하기 전 상태로 만들어주기 떄문에 유용하다.
- 롤백 테스트는 심지어 여러 개발자가 하나의 공용 테스트용 DB를 사용할 수 있게도 해준다.
- 적절한 격리수준만 보장해주면 동시에 여러 개의 테스트가 진행돼도 상관없다.
- 이처럼 테스트에서 트랜잭션을 제어할 수 있기 때문에 얻을 수 있는 가장 큰 유익한 점이 있다면 바로 이 롤백 테스트다.
- DB에 따라서 성공적인 작업이라도 트랜잭션을 롤백하면 커밋할 때보다 성능이 더 향상되기도 한다.
- 예를 들어 MySQL에서는 동일한 작업을 수행한 뒤에 롤백하는 게 커밋하는 것보다 더 빠르다.
- 하지만 DB의 트랜잭션 처리 방법에 따라 롤백이 커밋보다 더 많은 부하를 주는 경우도 있으니 단지 성능 때문에 롤백 테스트가 낫다고는 볼 수 없다.

## 6.8.3 테스트를 위한 트랜잭션 애노테이션

- `@Transactional` 애노테이션을 타겟 클래스 또는 인터페이스에 부여하는 것만으로 트랜잭션을 적용해주는 건 매우 편리한 기술이다.
- 이 편리한 방법은 테스트 클래스와 메서드에도 적용할 수 있다.
- 스프링의 컨텍스트 테스트 프레임워크는 애노테이션을 이용해 테스트를 편리하게 만들 수 있는 여러 가지 기능을 추가하게 해준다.
- `@ContextConfiguration`을 클래스에 부여하면 테스트를 실행하기 전에 스프링 컨테이너를 초기화하고, `@Autowired` 애노테이션이 붙은 필드를 통해 테스트에 필요한 빈에 자유롭게 접근할 수 있다.
- 그 외에도 스프링 컨텍스트 테스트에서 쓸 수 있는 유용한 애노테이션이 여러 개 있다.

### `@Rollback`

- 테스트에 적용된 `@Transactional`은 테스트가 끝나면 자동으로 롤백된다.
- 만약 롤백을 원하지 않는다면 `@Rollback` 애노테이션을 적용하면 된다.
- 기본값은 `true`로 롤백이 적용되는 것이며, 원치 않는 경우 `@Rollback(false)`로 지정해줘야 한다.

![](list-6-99.jpeg)

- 위의 경우, 테스트 전체에 걸쳐 하나의 트랜잭션이 만들어지고 예외가 발생하지 않는 한 트랜잭션은 커밋된다.

### `@TransactionConfiguration`

- `@Transactional`은 테스트 클래스에 넣어서 모든 테스트 메서드에 일괄 적용할 수 있지만, `@Rollback` 애노테이션은 메서드 레벨에만 적용할 수 있다.
- 테스트 클래스의 모든 메서드에 트랜잭션을 적용하면서 모든 트랜잭션이 롤백되지 않고 커밋되게 하려면 어떻게 해야 할까?
- 모든 메서드마다 `@Rollback`을 적용하는 것은 너무 무식하다.
- 이 때, `@TransactionConfiguration` 애노테이션을 사용하면 된다.

![](list-6-100.jpeg)

### NotTransactional과 Propagation.NEVER

- 테스트 클래스 안에서 일부 메서드에만 트랜잭션이 필요하다면 메서드 레벨의 `@Transactional`을 적용하면 된다.
- 반면에 대부분의 메서드에서 트랜잭션이 필요하다면 테스트 클래스에 @Transactional을 지정하는 것이 편리하다.
- 일부 메서드에만 트랜잭션을 적용하고 싶지 않으면 `@Transactional(propagation = Propagation.NEVER)`를 사용하면 된다.
- `@NotTransactional`은 스프링 3.0에서 deprecated 됐다.

### 효과적인 DB 테스트

- 일반적으로 의존, 협력 오브젝트를 사용하지 않고 고립된 상태에서 테스트를 진행하는 단위 테스트와, DB 같은 외부의 리소스나 여러 계층의 클래스가 참여하는 통합 테스트는 아예 클래스를 구분해서 따로 만드는게 좋다.
- DB가 사용되는 통합 테스트를 별도의 클래스로 만들어둔다면 기본적으로 클래스 레벨에 `@Transactional`을 부여해준다.
- DB가 사용되는 통합테스트는 가능한 한 롤백 테스트로 만드는 게 좋다.
- 애플리케이션의 모든 테스트를 한꺼번에 실행하는 빌드 스크립트 등에서 테스트에서 공통적으로 이용할 수 있는 테스트 DB를 셋업해주고, 각 테스트는 자신이 필요한 테스트 데이터를 보충해서 테스트를 진행하게 만든다.
- 테스트가 기본적으로 롤백 테스트로 되어 있다면 테스트 사이에 서로 영향을 주지 않으므로 독립적이고 자동화된 테스트로 만들기가 매우 편리하다.
- 테스트는 어떤 경우에도 서로 의존하면 안 된다.
- 테스트가 진행되는 순서나 앞의 테스트의 성공 여부에 따라서 다음 테스트의 결과가 달라지는 테스트를 만들면 안 된다.
- 코드가 바뀌지 않는 한 어떤 순서로 진행되더라도 테스트는 일정한 결과를 내야 한다.
- 즉, 결정적이어야 한다.