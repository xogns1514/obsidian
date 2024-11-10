## OSIV의 정의 및 개념
`OSIV(Open Session in View)`는 영속성 컨텍스트를 뷰 영역까지 열어두는 것입니다. 여기서 뷰는 스프링 MVC의 뷰(View)를 의미합니다. 해당 개념은 Hibernate와 JPA에서 데이테베이스 세션을 관리하는 전략 중 하나입니다.

### OSIV 등장 배경

OSIV 패턴 등장 이전에는 뷰를 렌더링하는 시점에 영속성 컨텍스트가 존재하지 않아 준영속 상태가 된 객체의 프록시를 초기화 할 수 없는 문제가 있었습니다. 예시 코드와 함께 문제를 살펴보겠습니다.

```java
// 도메인 계층

// -- Post
@Entity
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    private List<Comment> comments;

    // Getters and Setters
}

// -- Comment
@Entity
public class Comment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String content;

    // Getters and Setters
}

// 서비스 계층
@Service
public class PostService {
    private final PostRepository postRepository;

    public PostService(PostRepository postRepository) {
        this.postRepository = postRepository;
    }

    @Transactional
    public Post getPostById(Long id) {
        return postRepository.findById(id).orElse(null);
    }
}

// 컨트롤러 계층
@Controller
public class PostController {
    private final PostService postService;

    public PostController(PostService postService) {
        this.postService = postService;
    }

    @GetMapping("/post/{id}")
    public String getPost(@PathVariable Long id, Model model) {
        Post post = postService.getPostById(id);
        model.addAttribute("post", post);
        return "postView"; // postView.jsp 또는 postView.html
    }
}

// view - postView.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Post Details</title>
</head>
<body>
    <h1>Post: ${post.title}</h1>
    <h2>Comments:</h2>
    <ul>
        <c:forEach items="${post.comments}" var="comment">
            <li>${comment.content}</li>
        </c:forEach>
    </ul>
</body>
</html>
```

Post를 조회하는 api가 호출되었다고 가정하겠습니다. 서비스 계층의 Post를 조회하는 메서드에 `@Transactional` 이 붙어있습니다. 따라서 `getPostById` 메서드를 호출하면 세션을 열고, 트랜잭션을 시작합니다. 해당 메서드가 종료될 때 트랜잭션을 커밋하고, 세션을 닫습니다.

세션이 닫히면 영속성 컨텍스트가 함께 종료되면서 엔티티들은 준영속 상태가 됩니다. View를 렌더링 할때 `post` 엔티티의 `comments` 필드에 접근할 때 지연 로딩을 시도합니다. 하지만 `comments` 엔티티는 이미 준영속 상태가 되었기 때문에 지연 로딩을 수행할 수 없습니다. 결과적으로 `LazyInitializationException` 에러가 발생하게 됩니다.

Post 조회 요청 실행 경로와, 세션 및 트랜잭션의 유지를 그림으로 나타내면 다음과 같습니다.

![image](https://github.com/user-attachments/assets/cdd835c9-d65a-42b2-87ee-2b2fe9f69df4)


위 문제를 해결하기 위해 등장한 개념이 OSIV입니다. OSIV는의 목적은 뷰에서도 `Lazy Loading`을 지원하여, 필요한 데이터만 효율적으로 가져올 수 있도록 하는 것입니다. 사용자의 요청마다 필요한 데이터가 다를 수 있으므로, 모든 데이터를 한 번에 가져오는 대신에 사용자가 요청한 데이터만을 가져옴으로써 성능을 향상시키는데 기여합니다.

## 스프링 프레임워크 OSIV

스프링 프레임워크가 제공하는 OSIV는 비즈니스 계층에서 트랜잭션을 사용하는 OSIV입니다. 즉, OSIV를 사용하지만 트랜잭션은 비즈니스 계층에서만 사용한다는 뜻입니다.

### 스프링 프레임워크 OSIV 동작과정

![image](https://github.com/user-attachments/assets/8163be09-8a05-461e-b6f0-a55754ca8cee)


다음은 스프링 프레임워크에서 OSIV가 동작하는 과정입니다.

1. 클라이언트로부터 HTTP 요청이 들어오면 서블릿 필터 또는 스프링 인터셉터에서 요청을 가로챕니다. 요청을 가로챈 이후, 영속성 컨텍스트를 생성합니다. 이때, 트랜잭션은 시작하지 않습니다.
2. 서비스 계층에서 `@Transactional` 이 붙은 메서드가 실행되면, 위에서 미리 생성해둔 영속성 컨텍스트를 찾아와 트랜잭션을 시작합니다.
3. 서비스 계층의 비즈니스 로직 실행이 끝나면 트랜잭션을 커밋합니다. 트랜잭션이 커밋되면 영속성 컨텍스트를 플러시합니다. 이때 트랜잭션은 끝나지만, 영속성 컨텍스트는 끝나지 않습니다.
4. 컨틀롤러와 뷰까지 영속성 컨텍스트가 유지되기 때문에 조회하는 엔티티는 영속 상태를 유지합니다.
5. 서블릿 필터 또는 스프링 인터셉터로 요청이 돌아오면 영속성 컨텍스트를 종료합니다. 이때 플러시를 호출하지 않고 바로 종료합니다.

스프링에서는 다음 방식들로 OSIV를 제공합니다.

1. JPA OEIV 서블릿 필터 - OpenEntityManagerInViewFilter
2. JPA OEIV 스프링 인터셉터 - OpenEntityManagerInViewInterceptor
3. 하이버네이트 OSIV 서블릿 필터 - OpenSessionInViewFilter
4. 하이버네이트 OSIV 스프링 인터셉터 - OpenSessionInViewInterceptor

서블릿 필터는 디스패처 서블릿 이전에 존재하는 필터를 기반으로 동작하는 OSIV이고, 인터셉터는 디스패처 서블릿 이후에 존재하는 인터셉터를 기반으로 동작하는 OSIV입니다.

스프링 부트에서는 OSIV가 기본적으로 활성화 되어있습니다. application 설정을 통해 비활성화 할 수 있습니다.

```yaml
spring:
  jpa:
    open-in-view: false
```

## OSIV 장점과 단점

OSIV를 통해 얻을 수 있는 장점에 대해 알아보겠습니다.

장점

- LazyLoading 문제 해결

OSIV의 가장 큰 장점은 LazyLoading(지연로딩) 문제를 해결하는 것입니다. JPA를 사용하는 엔티티 매핑에서, 관계된 엔티티를 실제로 사용할 때까지 데이터베이스에서 로드하지 않는 LazyLoading은 자주 사용됩니다. 그러나 트랜잭션이 종료된 이후 뷰 단계에서 엔티티의 필드에 접근하려고 할 때, 지연 로딩된 필드에 접근하려 하면 `LazyInitializationException`이 발생합니다. OSIV는 HTTP 요청이 완료될 때까지 영속성 컨텍스트를 열어두므로, 뷰에서 지연 로딩된 엔티티를 안전하게 사용할 수 있습니다.

- 개발 편의성 증가

OSIV는 개발자가 트랜잭션 범위와 데이터 로딩 타이밍을 신경 쓰지 않아도 되는 장점을 제공합니다. 특히 복잡한 객체 연관 관계가 많을 때 LazyLoading이 적용된 엔티티를 뷰에서 자유롭게 사용할 수 있으므로 개발이 수월해집니다. 따라서 개발자는 비즈니스 로직에만 집중할 수 있게 됩니다.

단점

- 긴 DB 커넥션 유지

OSIV의 가장 큰 단점 중 하나는 **데이터베이스 커넥션을 장시간 유지**해야 한다는 점입니다. HTTP 요청이 시작되고 뷰 렌더링이 끝날 때까지 영속성 컨텍스트를 열어두기 때문에, 트랜잭션이 끝난 이후에도 데이터베이스 연결이 유지됩니다. 이는 특히 트래픽이 많은 애플리케이션에서 **DB 커넥션 자원 부족**을 초래할 수 있습니다.

- 트랜잭션 외부에서 데이터 조작

OSIV가 활성화된 상태에서는 **트랜잭션이 종료된 후에도 데이터베이스에 접근**할 수 있습니다. 컨트롤러 계층에서 엔티티를 수정한 직후에 트랜잭션을 시작하는 서비스 계층을 호출할 경우 문제가 생길 수 있습니다. 예시 코드와 함께 살펴보겠습니다.

```java
@Controller
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/user/{id}")
    public String getUser(@PathVariable Long id, Model model) {
        // DB에서 User 엔티티를 조회 (OSIV가 활성화된 상태에서 영속성 컨텍스트 유지)
        User user = userService.findUserById(id);

        // 컨트롤러에서 트랜잭션 밖에서 엔티티 수정
        user.setName("Updated Name");

        // 수정한 엔티티를 다시 서비스 계층에 전달 (트랜잭션을 시작)
        userService.updateUser(user);

        return "userView";

```

컨트롤러 계층에서 엔티티를 수정한뒤, `updateUser()` 호출했습니다. 이때 서비스 계층에서 새로운 트랜잭션이 시작됩니다. 트랜잭션이 커밋되면 변경 감지가 동작하면서 엔티티의 수정 사항을 데이터베이스에 반영합니다.

컨트롤러에서 엔티티를 수정하고 즉시 뷰를 호출한 것이 아니라 트랜잭션이 동작하는 비즈니스 로직을 호출했으므로 문제가 발생한 것이다. 스프링 OSIV는 같은 영속성 컨텍스트를 여러 트랜잭션이 공유할 수 있으므로 위와 같은 문제가 발생할 수 있다.

- 참고
- https://hudi.blog/multi-datasource-issue-with-osiv/