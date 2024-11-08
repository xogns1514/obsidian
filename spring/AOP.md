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
<img width="373" alt="image" src="https://github.com/user-attachments/assets/8e207ff5-3259-4f96-884a-bdcdd7116ae8">

```java
final var userService = (UserService) Proxy.newProxyInstance(
	getClass().getClassLoader(), // 프록시를 만들 ClassLoader
	new Class[] {UserService.class}, // 프록시를 만들 인터페이스
	new TransactionHandler(plaformTransactionManager, new  AppUserService(userDao, userHistory)) // 프록시의 내용, 직접 작성한 구현체
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
- JDK 프록시는 인터페이스에 적혀있는 모든 메서드에 프록시를 적용시킨다. 

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
<img width="373" alt="image" src="https://github.com/user-attachments/assets/c05d4ae0-2a70-4c90-96ef-8090651a16e8">

```java
public class SecurityMethodInterceptor implements MethodInterceptor {

	private final MethodMatcher methodMatcher;

	@Override
	public Object intercept(final Object obj, final Method method, final Object[] args, final MethodProxy methodProxy) throws Throwable {
		if (methodMatcher.matches(method)) {
			// 특정 메서드에 프록시 적용
		}
	}
}
```
```java
.
.
@Override
public Object getProxy() {
	final Enhancer enhancer = new Enhancer();
	enhancer.setSuperclass(targetClass);
	enhancer.setCallback(methodInterceptor); //handler
	return enhancer.create(); // proxy 생성
}
```
- 서드파티 코드나 수정할 수 없는 레거시 코드로 작업할 때, 인터페이스를 사용할 수 없는 경우가 있다. 이때는 CGLIB를 사용해야 한다. 
- 동적으로 새 클래스에 대한 바이트코드를 생성하고, 이미 생성된 클래스를 사용할 수 있다면 재사용한다. 
- CGLIB 프록시는 타깃 메서드에 적절한 바이트코드를 생성하여 프록시로 인한 성능 오버헤드를 줄인다. 
	- 매번 invoke()를 호출하는 JDK 프록시와 비교하면 CGLIB는 한 번만 수행되기에 성능이 더 좋다. 
- MethodMatcher 객체를 사용하여 특정 메서드만 프록시화 하고, 나머지는 프록시를 거치지 않고 실제 객체 메서드를 호출하도록 만들 수 있다.
- 주의점
	- 상속을 사용하여 프록시를 생성하므로, final 클래스, final 메서드, private 메서드는 프록시할 수 없다. 

### 스프링 AOP
- 스프링 AOP는 `순수 자바`로 구현된다. 특별한 컴파일 프로세스가 필요하지 않습니다. 
- 스프링 AOP는 클래스 로더 계층 구조를 제어할 필요가 없으므로 서블릿 컨테이너 또는 애플리케이션 서버에서 사용하기에 적합하다. → 원본 클래스 자체를 수정하거나, 클래스 로딩 시점에 직접적인 개입을 하지 않기 때문이다. 
- 목표
	- 엔터프라이즈 애플리케이션의 일반적인 문제를 해결하는 데 도움이 되도록 AOP 구현과 스프링 IoC간의 긴밀한 통합을 제공하는 것이 목표이다.
	- 스프링 프레임워크의 AOP 기능은 일반적으로 스프링 IoC 컨테이너와 함께 사용된다. 
- 스프링 프레임워크의 핵심 철학 중 하나는 비침투성이다. → 비즈니스 또는 도메인 모델에 프레임워크 클래스 or 인터페이스 강제 도입 X
	→ 하지만 필요에 따라 코드베이스에 스프링 프레임워크의 특징 종속성을 도입할 수 있는 옵션을 제공한다. 왜냐하면 특정 상황에서는 특정 기능을 읽거나 코딩하는 데 더 쉬울 수 있기 때문이다. 
- Runtime Weaving: Aspect가 대상 객체의 Proxy를 실행시 Weaving된다. 
	- 사용자의 특정 호출 시점에 IoC 컨테이너에 의해 AOP를 할 수 있는 Proxy Bean을 생성해준다. 

- 어드바이스
	- 타깃에 적용할 부가기능을 정의한다. 
	- 스프링의 어드바이스는 타깃의 메서드로 제한하고 있다. 
	- 어드바이스를 적용하려면 위치를 지정해야 한다. 
	- 스프링은 주로 메서드 호출 시점에 부가기능을 적용한다. 
		- MethodInterceptor: 메서드 호출할 때 부가 기능을 적용한다. 
		- MethodBeforeAdvice: 메서드 호출 전에 실행할 부가기능을 정의한 메서드
		- AfterReturnAdvice: 메서드 호출 후에 실행할 부가기능을 정의한 메서드
		- ThrowsAdvice: 애플리케이션에서 예외 처리와 로깅을 한 곳으로 모을 수 있다.

- 포인트 컷
	- 어드바이스 적용을 모든 메서드가 아닌 특정 메서드로 제한하려면 포인트컷을 사용해야 한다. 
	- 스프링 AOP가 제공하는 Pointcut 인터페이스를 살펴보면 MethodMatcher를 지원한다. MethodMatcher는 메서드 이름, 메서드 시그니처, 메서드 인수 등을 사용하여 메서드를 선택한다. 
	- StaticMethodMatcherPointcut(정적 포인트컷)
		- 대상 메서드에 대해 한 번만 MethodMatcher의 matches() 메서드를 호출한다.
		- 반환값은 캐싱하여 메서드를 호출하는 데 사용된다.
		- 메서드를 호출할 때 추가적인 검사가 필요하지 않아 동적 포인컷보다 성능이 훨씬 좋다.
	- DynamicMethodMatcherPointcut(동적 포인트컷)
		- 메서드를 호출하면 matches(Method, Class<T>) 메서드를 사용해 정적 검사를 한다.
		- 위에서 true를 반환하면, matches(Method, Class<T>, Object[]) 메서드를 사용해 추가로 검사를 수행한다.
	- AspectJ 포인트컷 표현식
		- 스프링 AOP보다 많은 기능을 가진 AOP 구현체인 AspectJ를 사용하면 표현식으로 포인트컷을 정의할 수 있다. AspectJ를 사용하려면 build.gradle에 aspectjweaver, aspectjrt 의존성을 추가해야 한다.
		- ex) “execution(*getMessage*(…))”
	- Annotation 매칭 포인트 컷
		- 커스텀 어노테이션이 적용된 메서드나 타입에 어드바이스를 적용하고 싶을 때 사용한다. 
		- ex) AnnotationMatchingPointcut.forMethodAnnotation(CustomAnnotation::class.java)

### AspectJ
<img width="725" alt="image" src="https://github.com/user-attachments/assets/47f94a84-3585-4d81-8796-08d4ab972ef6">

- 스프링 AOP는 완전한 솔루션을 제공하지 않는다. 
- 스프링 컨테이너에서 관리하는 Bean에만 적용할 수 있다.
- AspectJ는 완전한 AOP 솔루션 제공을 목표로 한다.
- 스프링 AOP보다 더 다양한 기능을 제공한다.
- AspectJ와 스프링 AOP의 가장 큰 차이점은 위빙(weaving) 시점이다.
- `위빙`이란 애플리케이션 코드의 적절한 위치에 애스펙트(Advisor)를 적용하는 과정을 말한다.
- 스프링 AOP는 프록시 메커니즘을 사용해 런타임 시점에 위빙을 수행한다. 
- 3가지 유형의 바이트코드 weaving을 사용한다.
	- Compile-time weaving: AspectJ 컴파일러는 aspect와 애플리케이션의 소스 코드를 입력으로 취하고 출력으로 엮인 클래스 파일을 생성한다. 
	- Post-compile weaving: 기존 클래스 파일와 JAR파일을 위빙하는데 사용한다. 
	- Load-time weaving: 조작되지 않은 바이트코드가 JVM에 로드될 때 ClassLoader를 이용하여 바이트코드를 조작하는 위빙 방식이다. 
	→ `런타임 시점에 영향을 끼치지 않는다.`
	→ 밴치마킹상 AspectJ가 Spring AOP보다 최소 8배, 최대 35배정도 빠르다.
- AnnotatedAdvice 클래스에 @Aspect를 사용하여 AspectJ AOP를 적용한다.
- @Component를 붙여야 스프링 빈으로 등록된다.
