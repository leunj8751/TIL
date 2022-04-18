
##### 트랜잭션 코드의 분리

트랜잭션 흐름을 UserService로 갖고오느라 부득이하게 비즈니스코드와 트랜잭션 경계설정 코드와 비즈니스 로직 코드가 섞이게 되었으니, 이것을 또 분리해보자.

1. 트랜잭션 경계설정 로직과 비즈니스 로직을 구분하기 위해 메소드를 두개로 나눈다. (트랜잭션 경계설정 메소드 명 : upgradeLevels / 비즈니스 로직 메소드 명 : upgradeLevelsInternal)
2. 다른 클래스에서 userService 클래스를 호출해서 사용하면, 그 클래스는 트랜잭션 경계설정이 빠진 UserService를 사용하게 된다. 
그러니 userService를 인터페이스로 만들어주고 (클라이언트 클래스와 결합도 낮추기), userService에 있는 로직은 userServiceImpl이라는 클래스를 만들어서 올며주자
3. 트랜잭션 경계설정 기능을 제대로 사용하기 위해, 트랜잭션 경계설정 로직을 userService 인터페이스를 구현한 새로운 클래스(클래스명 : UserServiceTx)를 만들어 그 안에 넣어준다. 
4. 즉 UserService 인터페이스를 구현한 클래스가 두개 만들어지는데, 트랜잭션 경계설정을 담당할 클래스(UserServiceTx)에 비즈니스 로직을 담당할 클래스(UserServiceImpl)를 주입하면,
클라이언트가될 클래스는 UserServiceTx의 오브젝트만 구현하여 사용하면 되는것이다.

```java
public interface UserService {
	
	void add(User user);
	void upgradeLevels();
	
}

public class UserServiceTx implements UserService{

	UserService userService;
	PlatformTransactionManager transactionManager;
	
		
	public void setUserService(UserService userService) {
		this.userService = userService;
	}
	
	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}
	
	@Override
	public void add(User user) {
		userService.add(user);
		
	}

	@Override
	public void upgradeLevels() {
		
		TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());

		try {			
			
			this.userService.upgradeLevels();
			
			this.transactionManager.commit(status);
		} catch (Exception e) {    
			this.transactionManager.rollback(status);
		} finally {
		}

	}

}


```

--------

##### 고립된 단위 테스트

의존관계에 있는 다른 클래스의 영향을 받지 않고 작은 단위의 하나의 클래스만의 로직을 빠르고 정확하게 테스트 하기 위해 단위테스트를 만드는데,
테스트 대상 클래스가 환경이나 서버, 다른 클래스의 코드에 종속되는것을 방지하기 위해 목오브젝트와 같은 대역 클래스를 만들어주어 테스트할 수 있다.

지금 유저의 Level을 업그레이드하고 메일을 보내는 UserService 클래스의 기능을 테스트 하기 위해서는 UserDAO 오브젝트를 사용해야 되는데, DB와 연결하여 데이터를 가져오는 행위는 
userService 테스트에 방해가 될 수 있으니, 고립된 단위 테스트를 만들기 위해 UserDao 역할을 하며 UserService 사이에서 주고받은 정보를 저장해둘 수 있는 목 오브젝트를 만들어보자.







