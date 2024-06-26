p.88 ~ p.143

이전에는 UserDAO에서 실제 커넥션 구현을 막기 위해 UserDaoTest 에다가 커넥션을 구현해서 테스트를 진행했었다.
한 코드에 (테스트),(커넥션 생성) 두가지의 관심사가 함께 있으니 이를 분리시키기 위해 UserDao 생성을 책임질 UserDaoFactory 클래스를 만든다. 

```java
public class UserDaoFactory {
	public UserDao userDao() {
		UserDao dao = new UserDao(connectionMaker());
		return dao;
	}

	public ConnectionMaker connectionMaker() {
		ConnectionMaker connectionMaker = new DConnectionMaker();
		return connectionMaker;
	}
}


public class UserDaoTest {
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		UserDao dao = new UserDaoFactory().userDao();
    //......
  }
}

```
//1111
 ** **제어의 역전** **<br>
 
위 코드에서 보면 UserDaoFactory가 오브젝트 생성과 구현을 담당하니, UserDao는 수동적으로 UserDaoFactory의 오브젝트를 공급받아 사용한다.
원래는 UserDao가 갖고 있던 제어의 권한이 UserDaoFactory로 넘어간 것이다. 이것이 바로 제어의 역전(IOC)이다.
제어의 역전이 적용되면, 직접 사용할 오브젝트를 선택하지도, 생성하지도 못하게 되는것이다.<br>
스프링에서는 애플리케이션 컨텍스트가 제어권을 가지고 빈을 생성하고, 관계를 설정한다.

*싱글톤 레지스트리*<br>
스프링에서는 직접 싱글톤의 형태로 오브젝트를 만들고 관리하는 기능을 제공하는데, 이 덕분에 빈에서 public 생성자의 싱글톤 오브젝트를 가질 수 있게 된다.
이덕분에 싱글톤 패턴과 다르게 스프링이 지지하는 객체지향적인 설계가 가능하게 되는 것이다.


** **의존관계** **<br>
스프링에서 오브젝트간의 의존관계가 필요하다면 3가지 조건을 충족해야 한다.
1. 코드에는 런타임 시점의 의존관계가 드러나지 않아야 한다. 그러기 위해선 인터페이스로 의존관계를 설정해줘야 한다.
2. 런타임 시점의 의존관계는 컨테이너 같은 제 3의 존재가 결정해야 한다.
3. 의존관계를 사용할 오브젝트에 대한 래퍼런스는 외부에서 제공해줘야 한다.

```java
public UserDao(){
	connectionMaker = new DConnectionMaker();
}

```
위의 UserDao의 생성자는 런타임 시점에 어떤 클래스를 구현할지를 알고있다. 즉 의존관계가 코드에 결정되어 있는 것 

```java

public class UserDao {
	private ConnectionMaker connectionMaker;
	
	public UserDao(ConnectionMaker connectionMaker) {
		this.connectionMaker = connectionMaker;
	}

```
위와 같이 생성자의 코드를 수정한다면, 런타임 시점의 의존관계는 드러나지 않게 되고, 
생성자를 통해 외부에서 파라미터로 오브젝트를 주입시킬 수 있다.
여기서 의존관계를 맺어줄 클래스의 오브젝트를 만들고, 생성자의 파라미터로 오브젝트를 전달해주는것이 DI 컨테이너다.

위 예시에서는 UserDaoFactory가 오브젝트를 생성해서 UserDao에게 주입시켜 주니 UserDaoFactory가 DI컨테이너라고 할 수 있다.
즉, UserDaoFactory는 Ioc/DI 컨테이너인 것이다.



---
**의문**

```java
1.
public class UserDao {

	private ConnectionMaker connectionMaker;
	
	public UserDao(ConnectionMaker simpleConnectionMaker) {
		this.connectionMaker = simpleConnectionMaker;
	}
}

2.
public class UserDao {

	private ConnectionMaker connectionMaker;
	private Connection c;
	private User user;
	
}

```
1번코드를 2번코드와 같이 오브젝트가 생성될 변수를 전역변수로 선언해버리면, 멀티스레드 환경에서 싱글톤으로 실행될때 서로 값을 덮어쓰기 때문에 위험하다고 한다고 하는데 이해가 잘....





