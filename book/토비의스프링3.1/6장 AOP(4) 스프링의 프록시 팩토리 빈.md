p.462 ~ p.474

#### 스프링의 프록시 팩토리 빈

이전에 다이나믹 프록시를 스프링 빈으로 등록하여 오브젝트를 생성하기 위해서 `FactoryBean` 인터페이스를 구현한 팩토리 빈을 만들어주었다.
그래서 `userService` 인터페이스 타입의 모든 클래스는 해당 팩토리 빈을 이용하여 프록시에 접근하고, 부가기능을 실행하며 타깃에 접근 할 수 있게 됐다.

그런데 만약 `userService`가 아닌 다른 인터페이스를 구현한 클래스에 똑같은 부가기능을 적용하고 싶다면, 혹은 타깃인 `userServiceImpl`에 다른 부가기능을 적용하고 싶다면,
xml 빈 설정파일에서 일일이 클래스를 적용시켜 줘야 하니 비슷한 팩토리 빈의 설정이 반복될 것이다. 이 문제를 해결하기 위해 스프링은 <b>ProxyFactoryBean</b>을 제공한다.

* <b>ProxyFactoryBean</b>이란, 프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈이다. 이전에 만들었던 FactoryBean과는 달리 순수하게 프록시를 생성하는 작업만 담당하고,
프록시를 통해 제공해줄 부가기능은 별도의 빈에 둘 수 있다.
* 프록시에 사용할 부가기능은 `MethodInterceptor` 인터페이스를 구현해서 만드는데 이전에 사용했던 `InvocationHandler`와 다른점은 `invoke()`를 이용해서 타깃 오브젝트를 제공하지 않는다는 것이다.
그래서 `MethodIntercepter` 오브젝트는 타깃이 다른 여러 프록시에서 함께 사용할 수 있고, 싱글톤 빈으로 등록이 가능하다.

```java
@Test
	public void proxyFactoryBean() {
		ProxyFactoryBean pfBean = new ProxyFactoryBean();
		pfBean.setTarget(new HelloTarget());
		pfBean.addAdvice(new UppercaseAdvice());
		
		Hello proxieHello = (Hello)pfBean.getObject();
		
		assertThat( proxieHello.sayHello("Toby"),is("HELLO TOBY"));
		assertThat( proxieHello.sayHi("Toby"),is("HI TOBY"));
		assertThat( proxieHello.sayThankYou("Toby"),is("THANK YOU TOBY"));
		
	}
  
  static class UppercaseAdvice implements MethodInterceptor{

		@Override
		public Object invoke(MethodInvocation invocation) throws Throwable {
			String ret = (String)invocation.proceed();
			return ret.toUpperCase();
		}
		
	}
  
```

부가기능을 적용할 `UppercaseAdvice` 클래스는 `MethodInterceptor` 인터페이스를 구현하고, `invoke`메소드 안에서 `proceed()`메소드를 호출하여 타깃의 오브젝트에 접근했다.
(`MethodIntercepter`로는 메소드 정보와 함께 `MethodInvocation`에 타깃 오브젝트의 정보가 같이 넘어간다.)
그리고 프록시 팩토리 빈을 생성할때 `addAdvice`라는 프록시팩토리빈의 메소드를 이용해 부가기능 클래스를 적용해 주었다.<br>
 * 타깃 오브젝트에 적용할 부가기능을 담은 오브젝트를 <b>어드바이스</b>라고 한다. (타깃 오브젝트에 종속 X)
 
 #### 포인트 컷
 
 이전에는 부가기능을 적용할 메소드를 pattern이라는 프로퍼티에 메소드 이름을 지정해서 넣어줬었다. 
 그런데 이제 프록시 팩토리 빈을 이용해서 모든 인터페이스가 해당 빈을 이용할 수 있는데,
 특정 클래스의 메소드 이름을 프로퍼티에 넣는것은 그리 좋은 방법 같지 않다.
 이 문제는 <b>포인트 컷</b> 을 이용해 해결할 수 있다.
 
 ```java
 @Test
	public void pointCutAdvisor() {
		ProxyFactoryBean pfBean = new ProxyFactoryBean();
		pfBean.setTarget(new HelloTarget());
		
		NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
		pointcut.setMappedName("sayH*");
		
		pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut,new UppercaseAdvice()));
		
		Hello proxieHello = (Hello)pfBean.getObject();
		
		assertThat( proxieHello.sayHello("Toby"),is("HELLO TOBY"));
		assertThat( proxieHello.sayHi("Toby"),is("HI TOBY"));
		assertThat( proxieHello.sayThankYou("Toby"),is("THANK YOU TOBY"));
		
	}
 
 ```
 `NameMatchMethodPointcut` 클래스를 생성해서 부가기능을 적용할 메서드 이름은 set 해주면, 메소드 대상을 선정하는 포인트컷의 알고리즘이 해당 이름으로 시작하는 모든 메소드를 선택하게 해준다.
 그리고 `ProxyFactoryBean`에 `addAdvisor`메소드를 이용해서 포인트컷과, 어드바이스를 하나로 묶어서 추가해주면 된다.
 <b>(어드바이저 = 포인트컷 + 어드바이스)</b>
 
 `TxProxyFatoryBean`을 proxyFactoryBean 을 이용하도록 수정해보자
 ```java
 
 public class TransactionAdvice implements MethodInterceptor {
	PlatformTransactionManager transactionManager;

	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}

	public Object invoke(MethodInvocation invocation) throws Throwable {
		TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
		try {
			Object ret = invocation.proceed();
			this.transactionManager.commit(status);
			return ret;
		} catch (RuntimeException e) {
			this.transactionManager.rollback(status);
			throw e;
		}
	}
}
 
 
 @Test
	@DirtiesContext
	public void updateAllOrNothing() throws Exception{
		
		TestUserService testUserService = new TestUserService(users.get(3).getId());
		testUserService.setUserDao(this.userDao);
		testUserService.setMailSender(mailSender);
		
		ProxyFactoryBean txProxyFactoryBean = context.getBean("&userService", ProxyFactoryBean.class);
		txProxyFactoryBean.setTarget(testUserService);
		
		UserService txUserService = (UserService)txProxyFactoryBean.getObject();
		
		userDao.deleteAll();
		for(User user : users) userDao.add(user);
		
		try{
			txUserService.upgradeLevels();
			fail("TestUserServieExeption expected");
		}catch(TestUserServiceException e) {
		}
		
		checkLevelUpgraded(users.get(1), false);
		
	}
 
 
 <bean id="transactionAdvice" class="springbook.user.service.TransactionAdvice">
		<property name="transactionManager" ref="transactionManager" />
	</bean>
	
	<bean id="transactionPointcut" class="org.springframework.aop.support.NameMatchMethodPointcut">
		<property name="mappedName" value="upgrade*" />
	</bean>
	
	<bean id="transactionAdvisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
		<property name="advice" ref="transactionAdvice" />
		<property name="pointcut" ref="transactionPointcut" />
	</bean>
	
	<bean id="userService" class="org.springframework.aop.framework.ProxyFactoryBean">
		<property name="target" ref="userServiceImpl" />
		<property name="interceptorNames">
			<list>
				<value>transactionAdvisor</value>
			</list>
		</property>
	</bean>
 
 ```
 
 `MethodInterceptor`를 구현한 어드바이스 클래스를 만들고, 빈에다가 어드바이스, 포인트컷 을 설정하고 두개를 합쳐 어드바이저로 사용할 수 있도록 설정한 후
 `userService`빈에다가 어드바이저를 등록해서 사용하면 된다.
 
 
 
 
 
 
 
 
 
 
 ----
 * 템플릿/콜백 구조 
 
 
 
 
 
 
 
 
 
 
 
 
