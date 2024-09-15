## HTTP 서버 구현하기
```
package org.apache.coyote.http11;  
  
import com.techcourse.db.InMemoryUserRepository;  
import com.techcourse.exception.UncheckedServletException;  
import com.techcourse.model.User;  
import java.io.BufferedReader;  
import java.io.IOException;  
import java.io.InputStream;  
import java.io.InputStreamReader;  
import java.io.OutputStream;  
import java.net.Socket;  
import java.net.URISyntaxException;  
import java.net.URL;  
import java.nio.file.Files;  
import java.nio.file.Path;  
import java.util.HashMap;  
import java.util.Map;  
import org.apache.coyote.Processor;  
import org.slf4j.Logger;  
import org.slf4j.LoggerFactory;  
  
public class Http11Processor implements Runnable, Processor {  
  
    private static final Logger log = LoggerFactory.getLogger(Http11Processor.class);  
    private static final String DEFAULT_PATH = "static";  
  
    private final Socket connection;  
  
    public Http11Processor(final Socket connection) {  
        this.connection = connection;  
    }  
  
    @Override  
    public void run() {  
        log.info("connect host: {}, port: {}", connection.getInetAddress(), connection.getPort());  
        process(connection);  
    }  
  
    @Override  
    public void process(final Socket connection) {  
        try (final InputStream inputStream = connection.getInputStream();  
             final OutputStream outputStream = connection.getOutputStream();  
             BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream))  
        ) {  
            StringBuilder builder = new StringBuilder();  
            String line;  
            while ((line = reader.readLine()) != null && !line.isEmpty()) {  
                builder.append(line).append("\n");  
            }  
  
            HttpRequest httpRequest = new HttpRequest(builder.toString());  
  
            String response = response(httpRequest.getPath());  
  
            outputStream.write(response.getBytes());  
            outputStream.flush();  
        } catch (IOException | UncheckedServletException e) {  
            log.error(e.getMessage(), e);  
        } catch (URISyntaxException e) {  
            throw new RuntimeException(e);  
        }  
    }  
  
    private String response(String uri) throws URISyntaxException, IOException {  
        String responseBody = "";  
        String path = uri;  
  
        if (uri.equals("/")) {  
            responseBody = "Hello world!";  
            return generateResponse("text/html;charset=utf-8 ", responseBody);  
        }  
        if (uri.contains("?")) {  
            int index = uri.indexOf("?");  
            String queryString = "";  
            path = uri.substring(0, index);  
            queryString = uri.substring(index + 1);  
  
            Map<String, String> map = new HashMap<>();  
  
            if (!queryString.isEmpty()) {  
                String[] strings = queryString.split("&");  
                for (String string : strings) {  
                    String[] keyValue = string.split("=");  
                    map.put(keyValue[0], keyValue[1]);  
                }  
                User user = InMemoryUserRepository.findByAccount(map.get("account"))  
                        .orElseThrow(() -> new IllegalArgumentException("존재하지 않는 회원입니다."));  
                if (!user.checkPassword(map.get("password"))) {  
                    path = "401";  
                }  
                System.out.println(user);  
            }  
        }  
  
        if (path.equals("/login")) {  
            path += "/login.html";  
        }  
  
        URL resource = getUrl(path);  
        responseBody = new String(Files.readAllBytes(Path.of(resource.toURI())));  
        if (resource.getPath().endsWith(".css")) {  
            return generateResponse("text/css;charset=utf-8 ", responseBody);  
        } else {  
            return generateResponse("text/html;charset=utf-8 ", responseBody);  
        }  
    }  
  
    private String generateResponse(String contentType, String responseBody) {  
        return String.join("\r\n",  
                "HTTP/1.1 200 OK ",  
                "Content-Type: " + contentType,  
                "Content-Length: " + responseBody.getBytes().length + " ",  
                "",  
                responseBody);  
    }  
  
    private URL getUrl(String path) {  
        path = DEFAULT_PATH + path;  
        return getClass().getClassLoader().getResource(path);  
    }  
}
```

```
package org.apache.coyote.http11;  
  
public class HttpBody {  
  
    private final String body;  
  
    public HttpBody(String body) {  
        this.body = body;  
    }  
  
    public String getBody() {  
        return body;  
    }  
}
```

```
package org.apache.coyote.http11;  
  
import java.util.HashMap;  
import java.util.Map;  
  
public class HttpHeader {  
  
    private static final int KEY_INDEX = 0;  
    private static final int VALUE_INDEX = 1;  
    private static final String HEADER_REGEX = ": ";  
    private static final String LINE_FEED = "\n";  
  
    private final Map<String, String> headers;  
  
    public HttpHeader(String headers) {  
        this.headers = parseHeaders(headers);  
    }  
  
    private Map<String, String> parseHeaders(String header) {  
        Map<String, String> map = new HashMap<>();  
  
        String[] headers = header.split(LINE_FEED);  
        for (String value : headers) {  
            String[] keyValue = value.split(HEADER_REGEX);  
            map.put(keyValue[KEY_INDEX], keyValue[VALUE_INDEX]);  
        }  
        return map;  
    }  
  
    public Map<String, String> getHeaders() {  
        return headers;  
    }  
}
```

```
package org.apache.coyote.http11;  
  
public enum HttpMethod {  
  
    GET,  
    POST,  
    PUT,  
    DELETE,  
    PATCH,  
    OPTIONS,  
    TRACE,  
    HEAD  
    ;  
}
```

```
package org.apache.coyote.http11;  
  
import java.util.Map;  
  
public class HttpRequest {  
  
    private static final String LINE_FEED = "\n";  
    private static final int STARTLINE_INDEX = 0;  
    private static final int HEADER_INDEX = 1;  
  
    private final HttpStartLine startLine;  
    private final HttpHeader headers;  
    private final HttpBody body;  
  
    public HttpRequest(String request) {  
        startLine = new HttpStartLine(parseStartLine(request));  
        headers = new HttpHeader(parseHeaders(request));  
        body = new HttpBody(parseBody(request));  
    }  
  
    private String parseStartLine(String request) {  
        String[] messages = parseMessage(request);  
        return messages[STARTLINE_INDEX];  
    }  
  
    private String parseHeaders(String request) {  
        String[] messages = parseMessage(request);  
        StringBuilder builder = new StringBuilder();  
  
        for (int i = HEADER_INDEX; i < messages.length; i++) {  
            if (messages[i].isEmpty()) {  
                break;  
            }  
            builder.append(messages[i]).append(LINE_FEED);  
        }  
        return builder.toString();  
    }  
  
    private String parseBody(String request) {  
        String[] messages = parseMessage(request);  
        StringBuilder builder = new StringBuilder();  
  
        for (int i = HEADER_INDEX; i < messages.length; i++) {  
            if (messages[i].isEmpty()) {  
                for (int j = i; j < messages.length; j++) {  
                    builder.append(messages[j]).append(LINE_FEED);  
                }  
                break;  
            }  
        }  
        return builder.toString();  
    }  
  
    private String[] parseMessage(String request) {  
        return request.split(LINE_FEED);  
    }  
  
    public Map<String, String> getHeaders() {  
        return headers.getHeaders();  
    }  
  
    public String getBody() {  
        return body.getBody();  
    }  
  
    public String getPath() {  
        return startLine.getPath();  
    }  
}
```

```
package org.apache.coyote.http11;  
  
public class HttpResponse {  
  
  
  
}
```

```
package org.apache.coyote.http11;  
  
public class HttpStartLine {  
  
    private static final int METHOD_INDEX = 0;  
    private static final int PATH_INDEX = 1;  
    private static final int HTTP_VERSION_INDEX = 2;  
    private static final String SPACE = " ";  
  
    private final HttpMethod method;  
    private final String path;  
    private final String httpVersion;  
  
  
    public HttpStartLine(String startLine) {  
        this.method = parseMethod(startLine);  
        this.path = parsePath(startLine);  
        this.httpVersion = parseHttpVersion(startLine);  
    }  
  
    private HttpMethod parseMethod(String startLine) {  
        String[] parts = parseStartLine(startLine);  
        return HttpMethod.valueOf(parts[METHOD_INDEX].toUpperCase());  
    }  
  
    private String parsePath(String startLine) {  
        String[] parts = parseStartLine(startLine);  
        return parts[PATH_INDEX];  
    }  
  
    private String parseHttpVersion(String startLine) {  
        String[] parts = parseStartLine(startLine);  
        return parts[HTTP_VERSION_INDEX];  
    }  
  
    private String[] parseStartLine(String startLine) {  
        return startLine.split(SPACE);  
    }  
  
    public HttpMethod getMethod() {  
        return method;  
    }  
  
    public String getPath() {  
        return path;  
    }  
  
    public String getHttpVersion() {  
        return httpVersion;  
    }  
}
```



<img width="359" alt="image" src="https://github.com/user-attachments/assets/0cfa2b98-7143-4a20-949d-921b94229eb8">
**tomcat-coyote.jar** - Tomcat에 TCP를 통한 프로토콜 지원
catalina - Java Servlet을 호스팅하는 환경