#### Creating and Customizing Filter Chains
Spring Boot 어플리케이션에서 기본 fallback filter chain은 순서가 SecurityProperties.BASIC_AUTH_ORDER로 사전 정의되어 있다.
security.basic.enabled=false를 통해 이것을 완전히 끄거나
fallback으로 사용할수 있고 더 우선순위 위에 다른 규칙을 정의할수 있다.
이렇게 하기 위해 WebSecurityConfigurerAdapter 또는 WebSecurityConfigurer타입의 @Bean을 추가하고  클래스에 @Order를 붙인다.

```java
@Configuration
@Order(SecurityProperties.BASIC_AUTH_ORDER - 10)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.antMatcher("/match1/**")
     ...;
  }
}
```
이 bean은 Spring Security로 하여금 새로운 filter chain을 추가하고 이것을 fallback이전 순서로 지정하게 한다.

많은 어플리케이션은 다른 어플리케이션과 비교해 리소스 집합에 대해
완전히 다른 access rule을 가진다.
예를 들어, UI와 backing API을 호스팅 하는 어플리케이션은 UI부분에 대해서는 로그인 페이지로 리다이렉트하는 쿠키 기반 인증을,
그리고 API부분에 대해서는 인증되지 않은 요청에 대해 401응답과 함께
토큰 기반 인증을 지원할수 있다.
각 리소스 세트에는 고유한 순서와 request matcher가 있는 WebSecurityConfigurerAdapte을 가진다.
만약 매칭 룰이 중복 되는 경우, 순서가 더 빠른 filter chain이 우선한다.

#### Request Matching for Dispatch and Authorization
security filter chain(또는 WebSecurityConfigurerAdapter)는 HTTP 요청에 적용할지를 결정하는데 사용되는 request matcher를 가진다.
특정 filter chain을 적용하기로 결정하면 다른 filter chain은 적용되지 않는다.
하지만 다음처럼 HttpSecurity 설정에 추가적인 matcher를 설정함으로써 인증을 보다 세부적으로 제어할수 있다.
```java
@Configuration
@Order(SecurityProperties.BASIC_AUTH_ORDER - 10)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.antMatcher("/match1/**")
      .authorizeRequests()
        .antMatchers("/match1/user").hasRole("USER")
        .antMatchers("/match1/spam").hasRole("SPAM")
        .anyRequest().isAuthenticated();
  }
}
```
Spring Security 설정에서 하기 쉬운 실수가 이러한 matcher가 다른 프로세스들에도 적용되는것을 잊는다는것이다.
하나는 전체 filter chain에 대한 request matcher이고 다른 하나는
적용될 access rule을 고르는것이다.

#### Combining Application Security Rules with Actuator Rules
Spring Boot Actuator는 기본적으로 안전하다.
어플리케이션에 Actuator을 추가하게 되면, Actuator endpoints에만 적용되는 추가적인 filter chain이 추가된다.
오직 Actuator endpoints에만 적용되는 request matcher로 정의되며
기본 SecurityProperties  fallback filter보다 5낮은 ManagementServerProperties.BASIC_AUTH_ORDER순서를 가지기 때문에 이것은 fallback전에 참조된다.
application security rule을 Actuator endpoints에 적용하길 원한다면, actuator이전 순서로 모든 actuator endpoints를 포함하는 request matcher를 가지는 filter chain을 추가할수 있다.
actuator endpoints를 위한 기본 security 설정을 선호 한다면,
가장 쉬운 것은 내가 만든 필터를 actuator이후,  fallback이전에 추가하는것이다.
```java
@Configuration
@Order(ManagementServerProperties.BASIC_AUTH_ORDER + 1)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.antMatcher("/foo/**")
     ...;
  }
}
```

#### Method Security
Spring Security는 웹 어플리케이션 뿐 아니라 java method 실행에도 access rule을 지원한다.
Spring Security에서  이것은 단지 "protected resource" 의 다른 타입일뿐이다.
이것은 access rule의 위치만 다를뿐 ConfigAttribute 문자열과 같은 포맷을 사용하여 선언된다는것을 의미한다.
우선 첫번째 단계는 method security을 활성화 하는것이다.

```java
@SpringBootApplication
@EnableGlobalMethodSecurity(securedEnabled = true)
public class SampleSecureApplication {
}
```
```java
@Service
public class MyService {

  @Secured("ROLE_USER")
  public String secure() {
    return "Hello Security";
  }

}
```
이 예시는 secure method를 가지는 서비스이다.
만약 위 타입의 @Bean을 만들면 프록시가 적용되고
메소드가 실제 실행되기전에 security interceptor을 통하게 된다.
만약 접근이 거부되면 호출자에게 실제 메서드 결과 반환 대신 AccessDeniedException이 발생한다.
security constraint를 강제하기 위해 메소드에 붙일수 있는 다른 애노테이션이 존재하는데,
대표적으로 @PreAuthorize, @PostAuthorize이 있다.
이것은  메서드 매개변수에 대한 참조가 포함된 표현식을 작성하고 각각 값을 반환할수 있다.

#### Working with Threads
기본 빌딩 블럭은 Authentication을 포함하는 SecurityContext이다.
SecurityContextHolder에 있는 static 메서드를 통해 SecurityContext 에 접근하고 조작할수 있는데, 이는 결국
ThreadLocal을 조작하는것이다.

```java
SecurityContext context = SecurityContextHolder.getContext();
Authentication authentication = context.getAuthentication();
assert(authentication.isAuthenticated);
```
유저 어플리케이션 코드에서 흔한것은 아니지만,
custom authentication filter를 작성할 필요가 있다면 유용하다.

web endpoint에서 현재 인증된 사용자에 접근할 필요가 있다면
다음처럼 메소드 파라미터를 사용할수 있다.

```java
@RequestMapping("/foo")
public String foo(@AuthenticationPrincipal User user) {
  ... // do stuff with user
}
```
메서드 파라미터 생성을 위해 SecurityContext에서 current Authentication을 꺼내고 getPrincipal() 메소드를 호출한다.
Authentication에서 Principal 타입은 인증 유효성 검사에 사용되는 AuthenticationManager에 따라 달라진다.
따라서 이는 사용자 데이터에 대한 type-safe한 참조를 얻는데 유용한 방식이다.

만약 Spring Security가 사용중일 경우 HttpServletRequest의 Principal은 Authentication타입이므로 직접 사용할수도 있다.

```java
@RequestMapping("/foo")
public String foo(Principal principal) {
  Authentication authentication = (Authentication) principal;
  User = (User) authentication.getPrincipal();
  ... // do stuff with user
}
```
이것은 Spring Security를 사용하지 않을때 코드를 작성할 필요가 있다면 유용하다.

#### Processing Secure Methods Asynchronously
SecurityContext는 thread-bound이기 때문에,
secure method(예를 들어 @Async가 붙은) secure method를 호출하는 background processing을 원한다면 컨텍스트가 전파된다는것을 보장해야한다.
즉, background에서 실행되는 task(Runnable, Callable 등)로 SecurityContext를 래핑해야한다.
Spring Security는 Runnable 그리고 Callable용 wrappers같은 몇가지 도움을 제공한다.
SecurityContext을 @Async 메소드로 전파 하기 위해,
AsyncConfigurer을 설정하고 Executor 가 옳은지 확인해야 한다.
```java
@Configuration
public class ApplicationConfiguration extends AsyncConfigurerSupport {

  @Override
  public Executor getAsyncExecutor() {
    return new DelegatingSecurityContextExecutorService(Executors.newFixedThreadPool(5));
  }

}
```
