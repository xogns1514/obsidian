### 프록시
- 클라이언트와 사용 대상 사이에 대리 역할을 맡은 오브젝트를 두는 방법
- 목적
	- 클라이언트가 타깃에 접근하는 방법 제어
	- 타깃에 부가기능을 부여하기 위해

프록시 패턴
- 타깃에 접근하는 방법을 제어하는 패턴
- 프록시와 다름
데코레이터 패턴
- 타깃에 부가기능을 부여하는 패턴

### JDK Proxy로 프록시 적용하기
```java
final var userService = (UserService) Proxy.newProxyInstance(
	getClass().getClassLoader(),
	new Class[] {UserService.class},
	new TransactionHandler(plaformTransactionManager, new  AppUserService(userDao, userHistory))
);
```
```java
public class TransactionHandler implements InvocationHandler {
	@Override
	public Object invoke(final Object proxy, final Method method, final Object[] args) throws Throwble {
	// todo 트랜잭션 경계 설정
	final var ret = method.invoke(target, args); // 타겟 객체의 메서드 실행
	// todo 트랜잭션 경계 설정
	return ret;
	}
}
```

문제점
- Proxy로 생성한 객체의 타입은 UserService다. AppUserService가 아니므로 스프링 빈 등록이 불가하다. 
- 매번 인터페이스를 만들고, 프록시를 적용할 InvocationHandler를 구현해야 한다. 
- 메서드만 프록시를 적용할 수 있다. 

### ProxyFactoryBean 도입하기
- JDK Proxy를 사용할 때 문제점을 해결하기 이해 스프링이 제공하는 클래스
- Proxy로 생성한 객체를 스프링 빈에 등록할 수 있다.
- 매번 인터페이스를 만들지 않아도 된다. 
- 메서드 외에 클래스도 지정할 수 있다. 
- 프록시 생성 방법을 정할 수 있다. 
	- JDK Proxy 또는 CGLib
![image](https://github.com/user-attachments/assets/677762c3-ae7b-4eba-a083-ff293855daba)

### AOP의 핵심 개념

| 용어        | 개념                        | 인터페이스       |
| --------- | ------------------------- | ----------- |
| Aspect    | Pointcut + Advice         | Advisor     |
| Pointcut  | Advice를 적용할 Joinpoint를 선별 | Pointcut    |
| Advice    | 특정 Joinpont에 실행되는 코드      | Interceptor |
| Joinpoint | Advice를 적용할 위치            | Invocation  |
| Target    | 부가기능(advice)을 적용할 대상      |             |
AOP란?
- 부가 기능의 모듈화
	- OOP 개념으로 코드 중복을 제거할 수 없을때 사용 ex) 트랜잭션
	- 미들웨어, 인프라와 관련된 코드를 모듈화
- OOP와 대척되는 개념이 아니다.
	- AOP는 OOP를 보완하는 개념이다. 
	- AOP만으로 애플리케션을 개발할 수 없다. 
- 스프링은 프록시 기술로 AOP 개념을 구현했다. 
- 위빙(Weaving)이란 코드의 적절한 위치에 aspect를 추가하는 과정
- `런타임`에 aspect를 추가할때
	- 동적(Dynamic) AOP
	- 스프링 AOP
		- JDK Dynamic Proxy(interface based)
		- CGLiB Proxy(subclass based)
- `컴파일 시점` 바이트 코드에 aspect를 추가할때 
	- 정적(static) AOP
	- AspectJ

### CGLIB(Code Generator Library)
- 서드파티 코드나 수정할 수 없는 레거시 코드로 작업할 때, 인터페이스를 사용할 수 없는 경우가 있다. 이때는 CGLIB를 사용해야 한다. 
- 동적으로 새 클래스에 대한 바이트코드를 생성하고, 이미 생성된 클래스를 사용할 수 있다면 재사용한다. 
- CGLIB 프록시는 타깃 메서드에 적절한 바이트코드를 생성하여 프록시로 인한 성능 오버헤드를 줄인다. 
	- 매번 invoke()를 호출하는 JDK 프록시와 비교하면 CGLIB는 한 번만 수행되기에 성능이 더 좋다. 
- 주의점
	- 상속을 사용하여 프록시를 생성하므로, final 클래스, final 메서드, private 메서드는 프록시할 수 없다. 

### 스프링 AOP
- 스프링 AOP는 순수 자바로 구현된다. 특별한 컴파일 프로세스가 필요하지 않습니다. 
- 스프링 AOP는 클래스 로더 계층 구조를 제어할 필요가 없으므로 서블릿 컨테이너 또는 애플리케이션 서버에서 사용하기에 적합하다. → 원본 클래스 자체를 수정하거나, 클래스 로딩 시점에 직접적인 개입을 하지 않기 때문이다. 
- 목표
	- 엔터프라이즈 애플리케이션의 일반적인 문제를 해결하는 데 도움이 되도록 AOP 구현과 스프링 IoC간의 긴밀한 통합을 제공하는 것이 목표이다.
	- 스프링 프레임워크의 AOP 기능은 일반적으로 스프링 IoC 컨테이너와 함께 사용된다. 
- 스프링 프레임워크의 핵심 철학 중 하나는 비침투성이다. → 비즈니스 또는 도메인 모델에 프레임워크 클래스 or 인터페이스 강제 도입 X
