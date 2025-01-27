HTTP/1.1 스팩은 클라이언트와 서버는 응답에 Connection: Keep-alive 헤더가 존재할 경우 TCP 커넥션을 재사용해야 한다. HTTP/1.1은 Connection 헤더의 디폴트 값이 Keep-alive이고, 이는 Connection 헤더를 따로 설정하지 않는 이상 항상 TCP 커넥션을 재사용해야 하는 것이다. 

Connection: close
- TCP 연결을 맺고나서 작업 처리 작업내용을 처리후 재활용없이 바로 종료.
- 클라이언트는 매번 요청을 보낼때 `Connection: close` 를 명시해서 매번 요청을 수행한 후에는 즉시 커넥션을 재활용하지 않고 종료
→ 매 요청마다 TCP 커넥션을 맺어야하니, 오버헤드 발생
→ 클라이언트와 서버 사이에 Connection:close 이용을 하겠다는 규약을 맺어야 한다. 