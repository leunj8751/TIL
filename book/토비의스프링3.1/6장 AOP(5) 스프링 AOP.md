p.475 ~ p.

프록시 팩토리 빈을 적용해 하나의 타깃에 여러가지 부가기능을 적용할 수 있도록 설정해줬다.
그럼 이제 부가기능이 필요한 타깃마다 매번 똑같은 빈 설정을 해줘야 하는 문제를 해결해보자.

이전에 매번 클래스마다 프록시를 구현해줘야하는 중복문제를 해결하기 위해 다이내믹 프록시를 도입하여 특정 인터페이스 타입으로 프록시의 역할을 해주는 클래스를 만들어줬었다.
이처럼 반복되는 ProxyFactoryBean 설정문제도 <u>프록시를 자동으로 만들어 주는 방법</u>으로 해결할 수 있다.

#### 확장된 포인트컷

이전에는 포인트 컷으로 타깃 오브젝트에 부가기능을 적용할 메소드를 찾았다. 그런데 포인트 컷은 또 다른 기능을 제공하는데,
스프링에 등록된 빈중에 어떤 빈에 프록시를 적용할지를 알려주는 기능도 있다.

```java
public interface Pointcut {

	ClassFilter getClassFilter(); // 프록시를 적용할 클래스인지 확인
	MethodMatcher getMethodMatcher(); // 어드바이스를 적용할 메소드 확인

}
```

#### 빈 후처리기를 이용한 자동 프록시 생성기

빈 후처리란, 빈을 생성한 후 빈 오브젝트를 다시 가공할 수 있게 해주는 것이다.
이 빈 후처리를 이용하면 생성한 오브젝트를 다른 오브젝트로 바꿔치기를 할 수도 있는데, 
이를 이용해서 생성된 빈 오브젝트의 일부를 프록시로 포장하고 프록시를 빈 대신 등록해서 사용할 수 있다.
스프링에서 빈 후처리기를 이용하려면 빈 후처리 자체를 빈으로 등록하면 된다.

`DefaultAdvisorAutoProxyCreator`는 빈 후처리중의 하나로 어드바이저를 이용한 자동 프록시 생성기이다.
`DefaultAdvisorAutoProxyCreator`를 빈 후처리기로 등록하면, 빈 오브젝트가 생성될 때 빈 후처리기가 받아서
포인트컷을 이용해 프록시를 적용할 오브젝트인지 확인해 주고, 프록시를 적용할 오브젝트 이면 `DefaultAdvisorAutoProxyCreator`의 내장된 
프록시 생성기가 해당 빈에 대한 프록시를 생성하고 어드바이저를 연결해준 후 다시 컨테이너에게 돌려주게 된다.

```java

@Test
	public void classNamePointcutAdvisor() {
		
		NameMatchMethodPointcut classMethodPointcut = new NameMatchMethodPointcut() {
			
			public ClassFilter getClassFilter() {
				return new ClassFilter() {
					public boolean matches(Class<?> clazz) {
						return clazz.getSimpleName().startsWith("HelloT"); // HelloT로 시작하는 클래스만 필터로 걸러 준다.
					}
				};
			}
		};

		
		classMethodPointcut.setMappedName("sayH*");  // sayH로 시작하는 메소드에만 어드바이스 적용
    
		checkAdviced(new HelloTarget(), classMethodPointcut, true);  //적용 클래스

		class HelloWorld extends HelloTarget {};
		checkAdviced(new HelloWorld(), classMethodPointcut, false);  //적용 클래스 아님
		
		class HelloToby extends HelloTarget {};
		checkAdviced(new HelloToby(), classMethodPointcut, true); // 적용 클래스
	}
	
	private void checkAdviced(Object target, Pointcut pointcut, boolean adviced) { 
		ProxyFactoryBean pfBean = new ProxyFactoryBean();
		pfBean.setTarget(target);
		pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice()));
		Hello proxiedHello = (Hello) pfBean.getObject();
		
		if (adviced) {
			assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
			assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
			assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
		}
		else {
			assertThat(proxiedHello.sayHello("Toby"), is("Hello Toby"));
			assertThat(proxiedHello.sayHi("Toby"), is("Hi Toby"));
			assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
		}
	}

```
클래스 필터에서 걸러진 클래스의 메소드에는 포인트컷이 적용이 되지 않는다.








