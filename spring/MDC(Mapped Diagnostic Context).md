- SLF4J와 Logback과 같은 로깅 프레임워크에서 사용되는 기능이다.
- 각 스레드에 대해 특정한 진단 정보를 제공하고, 이를 로깅할 때 사용되는 컨텍스트이다. 이를 통해, 여러 스레드가 동시에 실행되는 멀티스레드 환경에서도 로그의 추적을 쉽게 할 수 있다. 

- 스레드-로컬 저장소: MDC는 기본적으로 스레드-로컬(Tread-local) 저장소를 사용하여, 각 스레드가 자신의 고유한 데이터를 저장하고, 다른 스레드와 공유하지 않도록 한다. 따라서 멀티스레드 환경에서도 다른 스레드 간의 데이터 충돌 없이 진달 정보를 유지할 수 있다. 
- key-value로 구성된 컨텍스트: MDC는 Map<String, String> 구조로, 각 항목이 키-값 형태로 구성된다. 
- 로그 메시지에 컨텍스트 정보 추가: MDC에 저장된 값들은 로깅 포맷에서 logback.xml 등의 설정을 통해 쉽게 추가할 수 있어, 로그 메시지에 필요한 추가 정보를 일관되게 포함시킬 수 있다. 

- 주의 사항: 스레드 풀을 사용하는 경우, 스레드가 재사용되기 때문에 MDC의 값이 이전 요청의 데이터로 오염될 수 있다. 이를 방지하기 위해, MDC를 초기화해준다. 
