## Reflection
리플렉션을 이용하면 Runtime에 메서드를 부를 수 있다. 
- 메서드 얻어오기
- getMethod()
- `getXXX()` 는 상속받은 클래스와 인터페이스를 포함하여 모든 public 요소를 가져온다
```java
Method sumInstanceMethod = Operations.class.getMethod("publicSum", int.class, double.class);
```

- getDeclaredMethod()
- `getDeclaredXXX()` 는 상속받은 클래스와 인터페이스를 제외하고 해당 클래스에 직접 정의된 내용만 가져온다
```java
Method andPrivateMethod = Operations.class.getDeclaredMethod(
"privateAnd", boolean.class, boolean.class);
```

- 메서드 실행하기
	- 인스턴스 메서드 실행
	- obj: 해당 메서드를 갖고있는 인스턴스
```java
public Object invoke(Object obj, Object... args)  
    throws IllegalAccessException, InvocationTargetException

//
Method sumInstanceMethod = Operations.class.getMethod("publicSum", int.class, double.class); Operations operationsInstance = new Operations(); Double result = (Double) sumInstanceMethod.invoke(operationsInstance, 1, 3);
```

### ClassName
- getName: 전체 경로의 클래스명
- getCanonicalName:전체 경로명의 클래스명
- getSimpleName: 패키지명이 없는 클래스명

### Field
- setAccessible: 객체 필드에 직접 접근 가능하게 한다.
```java
final Class<?> studentClass = Student.class;  
Constructor<?> constructor = studentClass.getConstructor();  
final Student student = (Student) constructor.newInstance();  
final Field field = studentClass.getDeclaredField("age");  
field.setAccessible(true);   
  
field.set(student, 99);  
  
assertThat(field.getInt(student)).isEqualTo(99);  
assertThat(student.getAge()).isEqualTo(99);
```