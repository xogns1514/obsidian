## 동시성 확장하기
### ExecutorService
병렬 작업 시 여러 개의 작업을 효율적으로 처리하기 위해 제공되는 JAVA 라이브러리
- Task만 지정해주면 ThreadPool을 이용해서 Task를 실행하고 관리한다.
- Task는 Queue로 관리된다. ThreadPool에 있는 Thread수보다 Task가 많으면, 미실행된 Task는 Queue에 저장되고, 실행을 마친 Tread로 할당되어 순차적으로 수행된다. 
### execute, submit
<img width="570" alt="image" src="https://github.com/user-attachments/assets/b7ef42ed-0ef3-4ff4-922c-0e4094773e49">
- ExecutorService는 Executor 인터페이스를 상속받고 있어, execute() 메서드를 실행할 수 있다. 
- execute() 메서드는 Runnable 인터페이스만 인자로 받을 수 있다. 
- submit() 메서드는 Runnable, Callable 을 모두 인자로 받을 수 있다. 
- execute() 메서드는 반환타입이 void이다. 
- submit() 메서드는 Future 객체를 반환한다.
	- Future 객체는 비동기 연산의 결과를 의미한다.
	- 실행결과가 필요할 때 submit() 메서드를 사용할 수 있다.
	- Future 객체의 get() 메서드를 호출해서 결과를 받아볼 수 있다.
	- 쓰레드 실행 중 예외가 발생했다면 Future 객체의 get() 메서드를 호출했을 때 해당 예외가 발생한다. 

### shutdown
쓰레드풀은 데몬 쓰레드가 아니기 때문에 main 쓰레드가 종료되더라도 계속 실행 상태로 남아있다. 따라서 main() 호출이 종료되어도 애플리케이션 프로세스는 종료되지 않는다. 

shutdownNowI()
- 쓰레드 풀의 모든 쓰레드에 thread.interrupt()를 실행시켜서 하던 작업을 모두 멈추게 하는 것이다. 
- ExecutorService에 실행중인 Runnable or Callable의 구현이 
	`InterruptedException` 예외처리를 하는 코드가 없거나 `interrupt flag` 를 검사하는 코드가 없으면 shutdown 하더라도 프로그램이 종료되지 않을 수 있다. 
shutdown()
- ThreadPool에 요청되는 submit을 더이상 받아주지 않는 대신, 기존에 실행 중이던 쓰레드 풀의 쓰레드들은 계속해서 실행한다. 
- shutdown 후 executorService.submit을 하면 `RejectedExecutionException` 예외가 발생한다.
- InterruptedException를 통해서만 멈추는 작업이 submit 되어있다면, shutdown 을 통해서 종료될 수 없다. 이때는 shutdown을 호출해야 한다. 
- shutdown은 non-blocking이다. 
awaitTermination
- shutdown을 호출한 이후, task가 모두 종료되거나, timeout이 발생하거나, 현재스레드가 종료될 때까지 block 한다. 