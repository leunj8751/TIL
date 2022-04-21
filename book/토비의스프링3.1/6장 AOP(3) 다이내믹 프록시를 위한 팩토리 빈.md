p.449 ~ 461

### 팩토리 빈

이전에 만들었던 다이내믹 프록시를 빈으로 만들려고 한다.
그런데 스프링의 빈으로 등록하기 위해서는 명확한 클래스 이름이 들어가야 하는데, 프록시는 인터페이스 타입으로 만드니 이전처럼 빈으로 등록해서 DI할 수 없다.
그래서 스프링이 제공하는 `FactoryBean` 인터페이스를 통해 빈을 생성해줘야 한다.
 * 팩토리 빈 : 스프링을 대신해서 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈


##### 팩토리빈의 동작원리 이해하기
```java

public class Message {

	String text;
	
	private Message(String text) {
		this.text = text;
	}
	
	public String getText() {
		return text;
	}

	public static Message newMessage(String text) {
		return new Message(text);
	}
	
}


public class MessageFactoryBean implements FactoryBean<Message>{

	String text;
	
	public void setText(String text) {
		this.text = text;
	}

	@Override
	public Message getObject() throws Exception {
		
		return Message.newMessage(this.text);
	}

	@Override
	public Class<Message> getObjectType() {
		
		return Message.class;
	}

	@Override
	public boolean isSingleton() {
		
		return false;
	}
}

<bean id="message"
  class="springbook.learningtest.spring.factorybean.MessageFactoryBean">
  <property name="text" value="Factory Bean" />
</bean>


```
`Message` 클래스는 private 타입으로 생성자가 만들어져서 빈으로 등록할 수 없고, 스태틱 메소드를 이용해서 오브젝트를 생성해야 한다.
빈으로 등록해서 오브젝트를 생성할 수 있도록 Message 클래스의 팩토리빈 클래스를 만들어야 한다.
`setText()`는 오브젝트가 생성될때 필요한 정보를 DI해주는 것이고, `getObject()`를 이용하여 오브젝트를 생성해서 가져오고, `getObjectType`을 이용해서 오브젝트의 타입을 확인할 수 있다.
인터페이스를 구현한 클래스가 빈으로 등록이 되고, `getObject()`를 통해서 실제 클래스의 오브젝트를 가져오는 빈 팩토리의 동작과정은 팩토리 메소드 와 흡사하다.


#### 다이내믹 프록시를 만들어주는 팩토리 빈

팩토리 빈을 사용하면, 빈으로 생성할 수 없는 다이내믹 프록시를 빈으로 등록해 줄 수 있다.

```java
public class TxProxyFactoryBean implements FactoryBean<Object>{
	
	private Object target;
	private PlatformTransactionManager transactionManager;
	private String pattern;
	private Class<?> serviceInterface; // 팩토리 빈이 생성하는 오브젝트 타입은 DI받은 인터페이스 타입에 따라 달라진다. 고로 설정만 잘 해주면 다른 타입의 오브젝트로도 자유롭게 생성이 가능하다
	
	
	public void setTarget(Object target) {
		this.target = target;
	}

	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}

	public void setPattern(String pattern) {
		this.pattern = pattern;
	}
	
	public void setServiceInterface(Class<?> serviceInterface) {
		this.serviceInterface = serviceInterface;
	}
	
	@Override
	public Object getObject() throws Exception {
		
		TransactionHandler txHandler = new TransactionHandler();
		txHandler.setTarget(target);
		txHandler.setTransactionManager(transactionManager);
		txHandler.setPattern(pattern);
		
		return Proxy.newProxyInstance(
				getClass().getClassLoader(), new Class[] {serviceInterface},
				txHandler);
	}
	
	@Override
	public Class<?> getObjectType() {
		return serviceInterface;
	}

	@Override
	public boolean isSingleton() {
		return false;
	}
	
}


<bean id="userService" class="springbook.user.service.TxProxyFactoryBean">
		<property name="target" ref="userServiceImpl" />
		<property name="transactionManager" ref="transactionManager" />
		<property name="pattern" value="upgradeLevels" />
		<property name="serviceInterface" value="springbook.user.service.UserService" />
	</bean>
	
	<bean id="userServiceImpl" class="springbook.user.service.UserServiceImpl">
		<property name="userDao" ref="userDao" />
		<property name="mailSender" ref="mailSender" />
	</bean>

```

팩토리 빈과 `userServiceImpl`를 빈으로 등록한다. 팩토리 빈에 타깃 등등 `TransactionHandler`에게 던져줄 프로퍼티들을 설정한다.
클라이언트 클래스(`UserServiceTest`)에서 `UserService`타입으로 생성한 다이내믹 프록시를 이용하여 `TransactionHandler`를 이용해 부가기능인 트랜잭션 경계설정을 하게하고
타깃 오브젝트인 `UserServiceImpl`에 접근하여 비즈니스 로직을 실행하게 한다.








