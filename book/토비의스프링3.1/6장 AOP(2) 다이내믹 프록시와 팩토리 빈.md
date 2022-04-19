p.429 ~ p.

#### 프록시 / 데코레이션 패턴

이전에 `UserService`에서 트랜잭션 경계설정 로직과 비즈니스 로직을 분리시켜 주기 위해 `UserService`를 인터페이스로 만들고, 인터페이스를 구현해서
트랜잭션 경계설정을 담당할 `UserServiceTx`, 비즈니스 로직을 담당할 `UserServiceImpl` 클래스 두개를 만들고,
클라이언트 클래스에서 `userServiceTx`에만 접근할 수 있도록 한 후  `UserServiceTx`에 `UserServiceImpl` 오브젝트를 주입하여 비즈니스 로직에는 간접적으로 접근하도록 만들어 줬었다.

여기서 `userServiceTx`처럼 부가기능이 마치 자신이 핵심기능을 가진 클래스인 것처럼 꾸며서 클라이언트에게 호출되게 하는 것을 <b>프록시</b>라고 하고,
`UserServiceImpl` 같이 실제 핵심 기능을 담당하는 것을 <b>타깃</b>이라고 한다.

또한 `UserServiceTx`에 `UserServiceImpl` 오브젝트를 주입하기 위해 DI를 설정해준 것처럼 
인터페이스를 통해 오브젝트를 위임하여 여러가지의 부가적인 기능을 구현하는 것을 <b>데코레이터 패턴</b>이라고 한다.
데코레이터 패턴의 흔한 예로는 BufferedInputStream이 있다. `ex) InputStream is = new BufferedInputStream(new FileInputStream("a.txt"));`

----
#### 프록시 패턴

프록시 패턴은, 프록시가 타깃의 기능을 확장하거나 추가하지 않고, 타깃에 접근방식을 제어하는 목적을 갖고있을때 사용하는 패턴이다.
 * 타깃의 실제 오브젝트는 필요하지 않지만, 타깃 오브젝트의 레퍼런스가 필요할 때
 * 다른 서버에 존재하는 오브젝트를 프록시로 만들어 로컬에서 로컬에 실제 존재하는 오브젝트처럼 사용하고 싶을때
 * 타깃에 대한 접근권한을 제어해야 할 때
 
 #### 다이내믹 프록시
 
 매번 인터페이스를 구현한 프록시를 만들어내는것은 코드가 중복될 수 있고 복잡해지기 때문에 비효율적이다.
 그래서 java가 지원하는 API(java.lang.reflect)를 이용해 일일이 인터페이스를 구현하지 않고 손쉽게 프록시를 만들 수 있다.
 
 ###### 리플렉션 
  다이내믹 프록시를 구현하기 위해서는 리플렉션 기능을 사용해야 하는데, 리플렉션은 특정 클래스의 하나의 메소드를 갖고와서 클래스로 사용할 수 있게 해주는 것이다.
  (자바의 코드 자체를 추상화해서 접근하게 하는 것)
  
  
 ```java
 //리플렉션 예시
 public class ReflectionTest {

	@Test
	public void invokeMethod() throws Exception{
		
		String name = "Spring";
		
		//length()
		assertThat(name.length(), is(6));
		
		Method lengthMethod = String.class.getMethod("length");
		assertThat((Integer)lengthMethod.invoke(name), is(6));
		
		//charAt()
		assertThat(name.charAt(0), is('S'));
		
		Method charAtMethod = String.class.getMethod("charAt",int.class);
		assertThat((Character)charAtMethod.invoke(name,0),is('S'));
		
		
	}
	
}
 ```
 
 다이내믹 프록시는 인터페이스와 같은 타입으로 만들어져서 일일이 클래스를 구현하지 않아도 된다.
 프록시 팩토리에게 인터페이스 정보만 제공해주면, 자동으로 오브젝트를 만들어준다.
 부가기능에 대한 코드는 직접 작성해야 되는데, `InvocationHandler`를 구현한 클래스를 만들어 부가기능 코드를 넣어주고, `invoke()`메소드를 이용하여 전달받은 요청을 다시 타깃에 위임하게끔
 코드를 작성해주면 된다.
 
 
 ```java
 
 public class UppercaseHandler implements InvocationHandler{

	Hello target;
	
	public UppercaseHandler(Hello target) {
		this.target = target;
	}
	
	
	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		
		String ret = (String)method.invoke(target, args);
		
		return ret.toUpperCase();
	}
	
}

//프록시 생성코드

Hello proxyHello = (Hello)Proxy.newProxyInstance(
			getClass().getClassLoader(),
			new Class[] {Hello.class}, // 구현할 인터페이스
			new UppercaseHandler(new HelloTarget())); 
 ```
 
 `userServiceTx`를 다이내믹 프록시를 이용하여 수정하면 아래와 같이 만들 수 있다.
 ```java
 
 public class TransactionHandler implements InvocationHandler{

	private Object target;
	private PlatformTransactionManager transactionManager;
	private String pattern;
	
	public void setTarget(Object target) {
		this.target = target;
	}
	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}
	public void setPattern(String pattern) {
		this.pattern = pattern;
	}
	
	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		
		if(method.getName().startsWith(pattern)) {
			return invokeInTransaction(method,args);
		} else {
			return method.invoke(target, args);
		}
		
	}
	

	private Object invokeInTransaction(Method method, Object[] args)
			throws Throwable {
		TransactionStatus status = this.transactionManager
				.getTransaction(new DefaultTransactionDefinition());
		try {
			Object ret = method.invoke(target, args);
			this.transactionManager.commit(status);
			return ret;
		} catch (InvocationTargetException e) {
			this.transactionManager.rollback(status);
			throw e.getTargetException();
		}
	}
	
	
	
}



//테스트 코드
@Test
	public void updateAllOrNothing() throws Exception{
		
		TestUserService testUserService = new TestUserService(users.get(3).getId());
		testUserService.setUserDao(this.userDao);
		testUserService.setTransactionManager(this.transactionManager);
		testUserService.setMailSender(mailSender);
		
		TransactionHandler txHandler = new TransactionHandler();
		txHandler.setTarget(testUserService);
		txHandler.setTransactionManager(this.transactionManager);
		txHandler.setPattern("upgradeLevels");
		
		
		UserService txUserService = (UserService)Proxy.newProxyInstance(
				getClass().getClassLoader(),
				new Class[] {UserService.class}, // 구현할 인터페이스
				txHandler); 
		
		
		userDao.deleteAll();
		for(User user : users) userDao.add(user);
		
		try{
			txUserService.upgradeLevels();
			fail("TestUserServieExeption expected");
		}catch(TestUserServiceException e) {
			
		}
		
		checkLevelUpgraded(users.get(1), false);
		
	}
	
	static class TestUserService extends UserServiceImpl {
		
		private String id;
		
		private TestUserService(String id){
			this.id = id;
		}
		
		@Override
		protected void upgradeLevel(User user) {

			if(user.getId().equals(this.id)) {
				throw new TestUserServiceException();
			}
			
			super.upgradeLevel(user);
		}
		
		
		
		
	}




 ```
 
