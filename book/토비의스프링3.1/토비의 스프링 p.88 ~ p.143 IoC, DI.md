
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
위처럼 관심사를 분리시키기 위해 UserDAO의 생성을 책임질 팩토리 클래스를 만들고 이것을 사용한다면 
애플리케이션의 컴포넌트 역할을 하는 오브젝트와, 구조를 담당하는 오브젝트를 분리시켜 사용할 수 있다.

```java
public MessageDao messageDao(){
  return new MessageDao(connectionMaker()); 
}


public ConnectionMaker connectionMaker(){
  return new DConnectionMaker();
}
```
또한 해당 팩토리 클래스(DaoFactory) 안에서 여러개의 Dao가 만들어져도 한번만 connection을 생성하도록 connection을 생성해서 리턴해주는 메소드를 하나 만들고, dao에 메소드에서 그것을 사용하면,
UserDAO와 ConnectionMaker사이에 DaoFactory가 권한을 가지로 오브젝트를 관리하게 되는데 이것이 바로 제어의 역전(IoC)라고 할 수 있다.</br>
제어의 역전에는 오브젝트가 사용할 오브젝트를 직접 선택하지도 못하고, 생성하지도 못하는 것이다.



스프링에서는 애플리케이션 컨텍스트가 자신이 제어권을 가지고 직접 만들고 관계를 부여하는 빈을 생성하고, 관계를 설정한다.
애플리케이션 컨텍스트를 사용했을 때의 장점은, </br>
 1. 매번 오브젝트가 필요할때 생성해서 가져오지 않아도 되고, 
 2. 오브젝트가 만들어지는 방식, 시점 드을 다르게 가져가 오브젝트를 효과적으로 사용할 수 있다.

또한 스프링에서는 싱글톤 레지스트리 기능이 있어 직접 싱글톤의 형태로 오브젝트를 만들고 관리하는데, 이 덕분에 빈에서 public 생성자의 싱글톤 오브젝트를 가질 수 있게 된다.

---
스프링에서 오브젝트간의 의존관계가 필요하다면 3가지 조건을 충족해야 한다.
1. 코드에는 런타임 시점의 의존관계가 드러나지 않아야 한다. 그러기 위해선 인터페이스로 의존관계를 설정해줘야 한다.
2. 런타임 시점의 의존관계는 컨테이너 같은 제 3의 존재가 결정해야 한다.
3. 의존관계를 사용할 오브젝트에 대한 래퍼런스는 외부에서 제공해줘야 한다.

```java
public UserDao(){
	connectionMaker = new DConnectionMaker();
}

```
위의 UserDao의 생성자는 런타임 시점에 어떤 클래스를 구현할지를 알고 있는, 의존관계가 코드에 결정되어 있다.

```java

public class UserDao {
	private ConnectionMaker connectionMaker;
	
	public UserDao(ConnectionMaker connectionMaker) {
		this.connectionMaker = connectionMaker;
	}

```
아래와 같이 생성자의 코드를 수정한다면, 런타임 시점의 의존관계는 드러나지 않게 되고, 
생성자를 통해 외부에서 파라미터로 오브젝트를 주입시킬 수 있다.
여기서 의존관계를 맺어줄 클래스의 오브젝트를 만들고, 생성자의 파라미터로 오브젝트를 전달해주는것이 DI 컨테이너다.</br>
DI는 자신이 사용할 오브젝트에 대한 선택과 생성 제어권을 외부로 넘기고 자신은 수동적으로 주입받은 오브젝트를 사용한다는 점에서 Ioc개념에 들어맞는다.









