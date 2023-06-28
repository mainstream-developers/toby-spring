# 핵심은 오브젝트 -> 객체지향설계

- 오브젝트의 라이프사이클
- 오브젝트들간의 관계
- 오브젝트의 설계

> 스프링은 객체지향 설계 및 구현에 관해 특정한 모델과 기법을 억지로 강요하지는 않지만, 오브젝트의 설계, 구현, 사용, 개선에 관한 기준을 마련해준다. -> 프레임워크로 어느 정도 강제되어 있다.

# DAO (Data Access Object)

- DB에 엑세스하여 CRUD를 전담하는 책임을 가진 객체

## 예제: `UserDao`

- DB에 있는 사용자의 정보를 CRUD 하기 위한 클래스

```java
public void add(User user) throws ClassNotFoundException, SQLException {
  Class.forName("com.mysql.jdbc.Driver");

  Connection c = DriverManager.getConnection(
    "jdbc:mysql://localhost/springbook", "spring", "book"
  );

  PreparedStatement ps = c.prepareStatement(
    "insert into users(id, name, password) values(?, ?, ?)"
  );

  ps.setString(1, user.getId());
  ps.setString(2, user.getName());
  ps.setString(3, user.getPassword());

  ps.executeUpdate();

  ps.close();
  c.close();
}
```

```java
public User get(String id) throws ClassNotFoundException, SQLException {
  Class.forName("com.mysql.jdbc.Driver");

  Connection c = DriverManager.getConnection(
    "jdbc:mysql://localhost/springbook", "spring", "book"
  );

  PreparedStatement ps = c.prepareStatement(
    "select * from users where id = ?"
  );

  ps.setString(1, id);

  ResultSet rs = ps.executeQuery();

  rs.next();

  User user = new User();
  user.setId(rs.getString("id"));
  user.setName(rs.getString("name"));
  user.setPassword(rs.getString("password"));

  rs.close();
  ps.close();
  c.close();

  return user;
}
```

- 위와 같은 코드는 잘 동작하지만, 분명히 문제가 있는 코드다.
- 스프링을 공부한다는 것은 이런 코드에 대한 문제 제기와 의문에 대한 답을 찾아나가는 과정이다.

### SoC: Separation of Concerns

- 가장 중요한 것은 관심사의 분리
- 소프트(Soft)웨어 -> 부드럽다 -> 잘 변한다.
- 관심사를 분리함으로써 변화에 효과적으로 대응할 수 있다.
	- 변경은 특정 관심사항에 대해서 일어난다.
	- 특정 관심사항 외의 코드가 함께 있으면, 이를 파악하기도 힘들고 변경하기도 쉽지 않다.

## `UserDao`: 관심사를 각각 분리하기

- DB연결을 위한 관심
- 사용자 정보의 CRUD에 대한 관심
- 커넥션 리소스 정리에 대한 관심

### 커넥션 코드 추출

- 개발의 기본: 중복코드 제거
- 커넥션 코드의 중복을 제거함으로써, 커넥션에 대한 관심을 하나의 메서드로 모을 수 있게 됨
- 커넥션과 관련된 관심사항은 추출된 메서드에서 해결 가능

```java
private Connection getConnection() throws ClassNotFoundException, SQLException {
  Class.forName("com.mysql.jdbc.Driver");

  Connection c = DriverManager.getConnection(
    "jdbc:mysql://localhost/springbook", "spring", "book"
  );

  return c;
}
```

### 리팩토링

- 코드의 동작은 변화가 없지만, 내부 구조를 변경해 재구성하는 작업 또는 기술
- 구조 변경의 목적은 유지보수성 향상
	- 개발은 현실 -> 시간에 쫓기다 보면 중복된 코드를 생산해낼 수 있음
		- 당장의 Ctrl+C, Ctrl+V가 리팩토링보다 더 빠름
	- 당장 품질은 좋지 않지만 동작은 하는 코드는 나중에 변경이 발생하게 된다면 시간을 더 잡아먹게됨
	- 이를 개선하는 것이 목적

### 커넥션 생성 독립

- 기존 코드의 변화 없이 커넥션 생성 방식을 각자가 정할 수 있을까?

### 상속과 추상화

- 커넥션 생성에 대한 부분을 사용자가 구현하도록 변경한다.
- 나머지 부분 (예제의 경우 CRUD, 리소스 정리)은 이미 구현된 코드로 동작하게 한다.

```java
public abstract Connection getConnection() throws ClassNotFoundException, SQLException;
```

```java
public class NUserDao extends UserDao {
  public Connection getConnection() throws ClassNotFoundException, SQLException {
    // N사 DB connection 생성코드
  }
}

public class DUserDao extends UserDao {
  public Connection getConnection() throws ClassNotFoundException, SQLException {
    // D사 DB connection 생성코드
  }
}
```

### 템플릿 메서드 패턴

- 기존 코드를 수정하지 않고, 코드의 흐름 중 일부를 사용자가 알맞게 구현할 수 있도록 하는 방법

### 상속의 단점

- 자식클래스가 슈퍼클래스와 강하게 결합됨
	- 슈퍼클래스를 함부로 변경할 수 없음 -> 변경이 자식클래스 모두에게 전파됨
- 다중상속이 불가능 (언어적 특징)
	- 일부 기능을 확장하기 위해 상속구조로 만들어 버리면, 추후 다른 목적으로 상속을 적용하기 힘듬
		- 상속을 한 번 밖에 못하므로, 이미 상속구조를 가진 클래스는 추가적인 상속이 불가능

## 클래스의 분리

- 특정 관심사를 메서드 차원에서가 아니라 클래스 차원으로 분리하기
- 분리된 관심사를 해당 관심사를 필요로 하는 클래스가 사용하게 하기

```java
public class UserDao {
  private SimpleConnectionMaker simpleConnectionMaker;

  public UserDao() {
    simpleConnectionMaker = new SimpleConnectionMaker();
  }

  public void add(User user) throws ClassNotFoundException, SQLException {
    Connection c = simpleConnectionMaker.makeNewConnection();

    // ...
  }

  public User get(String id) throws ClassNotFoundException, SQLException {
    Connection c = simpleConnectionMaker.makeNewConnection();

    // ...
  }
}
```

```java
public class SimpleConnectionMaker {
  public Connection makeNewConnection() throws ClassNotFoundException, SQLException {
    Class.forName("com.mysql.jdbc.Driver");

    Connection c = DriverManager.getConnection(
      "jdbc:mysql://localhost/springbook", "spring", "book"
    );

    return c;
  }
}
```

### 구현과 종속

- `SimpleConnectionMaker`라는 구체적인 구현 클래스에 코드가 종속되어, 이를 기존의 상속의 형식으로 확장할 수가 없음
	- 기존의 코드(`UserDao`)를 수정해야만 커넥션 코드가 수정이 가능해짐
- 항상 구현에 의존하지 말고 인터페이스에 의존해야함
- 인터페이스는 다중 구현(`implements`)이 가능하다는 장점도 있음

```java
public interface ConnectionMaker {
  public Connection makeConnection() throws ClassNotFoundException, SQLException;
}
```

```java
public interface ConnectionMaker {
  public Connection makeConnection() throws ClassNotFoundException, SQLException;
}

public class UserDao {
  private ConnectionMaker connectionMaker;

  public UserDao() {
    // ❌ 다시 상세구현에 의존하게 됨
    // `UserDao`의 변경 없이 connectionMaker를 자유롭게 확장할 수 없음
    connectionMaker = new DConnectionMaker();
  }

  public void add(User user) throws ClassNotFoundException, SQLException {
    Connection c = connectionMaker.makeConnection();

    // ...
  }

  public User get(String id) throws ClassNotFoundException, SQLException {
    Connection c = connectionMaker.makeConnection();

    // ...
  }
}
```

# 오브젝트간의 관계

- 결국 오브젝트들간의 관계(의존성)에 관한 문제
- 컴파일 타임(코드 레벨)의 의존성(키워드에 의한 의존성)과 런타임의 의존성(실제 생성된 리소스)은 다른 문제
- 중요한 것은 컴파일 타임, 즉 코드의 변경 없이 런타임에 자유롭게 우리가 원하는 대로 관계를 맺도록 해야함
- 생성자 메서드로 관계에 대한 책임을 `UserDao` 사용자에게 떠넘기게 하자

```java
public UserDao(ConnectionMaker connectionMaker) {
  this.connectionMaker = connectionMaker;
}
```

## OCP: Open Closed Principle

- 확장에 대해 열려있어야 하고, 변경에는 닫혀있어야 한다.
	- 기능을 자유롭게 확장할 수 있다 -> 그 외 코드(관심사 외)의 수정 없이
	- 기능을 자유롭게 변경할 수 있다 -> 그 외 코드(관심사 외)의 수정 없이

## 응집도

- 변경이 같이 일어나는 부분의 정도 -> 많으면 응집도가 높다.
- 관심사의 분리가 잘 된 클래스는 응집도가 높을 것이다.
	- 같은 관심사를 갖고 있으므로, 기능이 변경된다면 변경될 여지가 높다.

## 결합도

- 객체간의 관계, 즉 의존성
- 결합도가 높다면 변경하기 힘들다 -> 변경의 영향이 모두 전파됨

## 전략패턴

- 기능 맥락을 통째로 추상화하여 필요에 따라 구현체를 바꿔서 사용할 수 있도록 하는 디자인 패턴
- 추상화되어 있으므로 결합도는 낮으며, 해당 기능 맥락 자체는 응집도가 높아 유지보수하기 좋다.

# IoC: Inversion of Control

- 결국은 관계, 의존성에 관한 이야기
- 관심사의 분리 -> 같은 관심사를 가진 클래스는 응집도를 높이고, 분리된 관심사로 인해 관계의 결합도는 낮아진다.

> 제어의 역전: 원래 모든 프로그램은 `main`함수인 entry point에서 시작하여 프로그램의 흐름을 구성한다. 즉, 오브젝트를 생성하고, 관계를 맺는 등의 "제어"를 개발자가 직접 구성하는데, 이런 흐름이 미리 정해져 있는 경우 제어권이 개발자에게 없으므로 제어의 역전이 발생했다고 표현한다.

## 팩토리

- 오브젝트의 생성과 이를 사용하는 것은 분명히 다른 관심이다.
- 코드는 인터페이스에 의존하여 결합도를 낮추고, 실제 런타임에서 실행되는 리소스는 팩토리를 통해 결정짓는다.
- 비즈니스 로직은 런타임에 실행되는 오브젝트에 의해 결정되지만, 이 오브젝트를 구성하고 오브젝트들간의 관계를 맺게하는 책임은 팩토리에 있다.
- 제어의 역전 관점에서 오브젝트의 생성 책임을 갖고 있는 것이다.

# 스프링의 IoC

## Application Context(Bean Factory)

- 스프링이 제어권을 가지고 직접 만들고 관계를 부여하는 오브젝트를 `Bean`이라고 칭함
- 이러한 빈의 생성과 관계설정의 책임을 가진 IoC 오브젝트가 바로 `Bean Factory`
- 이를 좀 더 확장한 것이 `Application Context`
	- 단순히 빈(객체)의 생성과 관계설정을 넘어서 애플리케이션 전반적인 제어를 담당
- 스프링 컨테이너는 `IoC 엔진`이다.

## `@Configuration`

- 빈 팩토리를 위한 오브젝트 설정을 담당하는 클래스를 표시하는 애노테이션

## `@Bean`

- 오브젝트 생성을 담당하는 메서드 애노테이션

```java
@Configuration
public class DaoFactory {
  @Bean
  public UserDao userDao() {
    return new UserDao(connectionMaker());
  }

  @Bean
  public ConnectionMaker connectionMaker() {
    return new DConnectionMaker();
  }
}
```

## Application Context의 동작방식

- 결국은 일종의 `Bean Factory`
- 객체의 생성과 관계설정을 담당한다.
- Application Context는 `@Configuration` 클래스로부터 설정정보를 얻어 오브젝트(`@Bean`)을 생성하고 관계설정을 한다.
- 이는 객체의 **생성** 부분이 분리되어 있다는 뜻
	- 일종의 팩토리이므로, 클라이언트는 요청한 빈의 구체적인 사항을 알 필요 없다.
	- 더 나아가, Application Context가 사용하는 `Bean Factory`에 대해서도 알 필요가 없다.
		- 팩토리가 어떤 구체 오브젝트를 내뱉는지 우리는 알 필요가 없다.
		- Application Context가 어떤 구체 팩토리를 선택하는지 우리는 알 필요가 없다.

## 용어 정리

### Bean

- 스프링이 IoC 방식으로 관리하는 오브젝트
- Managed object 라고도 부른다.

### Bean Factory

- 스프링의 IoC를 담당하는 핵심 컨테이너
- 빈을 등록, 생성, 조회, 리턴하고 그 외 여러가지 빈을 관리하는 기능을 담당한다.
- 보통은 이를 확장한 어플리케이션 컨텍스트를 사용한다.

### Application Context

- 빈 팩토리를 확장한 IoC 컨테이너
- 기본적인 빈 팩토리의 기능을 모두 갖고 있다.
- 스프링이 제공하는 각종 부가 서비스를 추가로 제공한다.
- 빈 팩토리라고 부를 때에는 빈의 생성과 제어의 관점에서 부르는 것이다.
- 어플리케이션 컨텍스트라고 할 때에는 스프링이 제공하는 어플리케이션 지원 기능을 모두 포함해서 얘기하는 것이다.

### Configuration Metadata

- 어플리케이션 컨텍스트 또는 빈 팩토리가 IoC를 적용하기 위해 사용하는 메타 정보를 뜻한다.
- IoC 컨테이너 자체 설정정보를 조정하는데 사용되기도 하지만, 주로 IoC 컨테이너에 의해 관리되는 어플리케이션 오브젝트를 생성하고 구성할 때 사용된다.

### IoC Container

- Inversion of Control Container
- 빈 팩토리, 어플리케이션 컨텍스트, 스프링 컨테이너 등 의미와 맥락 등을 고려해서 사람마다 부르는 이름이 다르다.

### Spring Framework

- IoC 컨테이너, 어플리케이션 컨텍스트 등을 포함해서 스프링이 제공하는 모든 기능을 통틀어 말할 때 주로 사용한다.
- 그냥 스프링이라고 줄여서 말하기도 한다.

## 싱글톤 레지스트리와 오브젝트 스코프

- Application Context에 의해 리턴되는 `Bean`은 기본적으로 싱글톤이다.
	- `identical` 하다.
- 스코프란 빈의 인스턴스화된 범위를 뜻한다.
	- Singleton: 물리적으로 단 하나의 빈으로 관리됨
	- Prototype: 매번 새로운 빈이 생성됨
	- Request: 웹 애플리케이션에서 요청마다 생성됨
	- Session: 웹 애플리케이션에서 사용자 세션마다 생성됨
	- Application: 웹 애플리케이션 내에서 단 하나의 인스턴스만 생성됨
		- 스프링은 여러 개의 웹애플리케이션을 실행시킬 수 있음: Singleton과 대비됨

### 동일성과 동등성

- 실제 메모리가 같은 경우 동일성(`identical`)
- 논리적으로 동일한 경우 동등성(`equivalent`)
- 스코프는 동일성 개념에서 봐야함

# DI: Dependency Injection

- IoC 컨테이너에 의해 오브젝트들간의 관계가 맺어진다고 했다.
- 관계는 인터페이스로 맺어진다.
- 하지만 런타임에는 결국 구체클래스가 필요하다.
- DI는 인터페이스간의 관계의 의존성을 구체클래스로 채워준다.
	- 의존관계의 주입
- 의존관계 주입을 통해 구체클래스간의 관계를 서술하는 코드가 사라져 좀 더 유연한 프로그램이 작성될 수 있다.

## 의존관계

- A가 B를 사용하고 있다면, A는 B에 의존한다는 뜻이다.
- 인터페이스에 의존하는 경우 결합도가 낮아진다는 의미는
	- 프로그램은 결국 구현되어야만 하며
	- 이러한 구현은 변경이 자주 발생하는 부분이므로
	- 자주 변경되는 부분에 의존하는 부분 역시 변경이 같이 발생하므로 결합도가 높아진다.
- 인터페이스에 의존한다는 것은 결국 구체적인 구현에 의존하는 것이 아니므로 변경에 대해서 좀 더 자유로워 진다.

## 메서드를 이용한 의존관계 주입

### 생성자 메서드 방식

```java
public UserDao(ConnectionMaker connectionMaker) {
  this.connectionMaker = connectionMaker;
}

@Configuration
public class DaoFactory {

  @Bean
  public UserDao userDao() {
    return new UserDao(connectionMaker());
  }

  @Bean
  public ConnectionMaker connectionMaker() {
    return new DConnectionMaker();
  }
}
```

### 수정자 메서드 방식

```java
public class UserDao {
  private ConnectionMaker connectionMaker;

  public void setConnectionMaker(ConnectionMaker connectionMaker) {
    this.connectionMaker = connectionMaker;
  }
}

@Bean
public USerDao userDao() {
  UserDao userDao = new UserDao();

  userDao.setConnectionMaker(connectionMaker());
  return userDao;
}
```

- 생성자 DI뿐만 아니라 메서드, Setter-based DI도 가능하다.
- 생성자 DI를 사용하면 순환의존성을 방지할 수 있다.
- 하지만 두 빈이 순환의존성을 가져야만 하고, 런타임엔 이에 대해 문제가 전혀 없다면 Setter-based DI를 사용하면 된다.
- Setter-based DI를 사용한다면 메서드의 이름을 잘 결정해야 한다.
- (자바의 경우)관례적으로 `setField()` 와 같이 사용하면 된다.
