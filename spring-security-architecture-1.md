출처    
[https://spring.io/guides/topicals/spring-security-architecture/](https://spring.io/guides/topicals/spring-security-architecture/)    
[https://mangkyu.tistory.com/76](https://mangkyu.tistory.com/76)    
[https://mangkyu.tistory.com/77](https://mangkyu.tistory.com/77)    
[https://velog.io/@devsh/%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98-Spring-Security-Architecture-translation](https://velog.io/@devsh/%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98-Spring-Security-Architecture-translation)    
[https://happyer16.tistory.com/entry/Spring-Security-%EC%95%84%ED%82%A4%ED%85%8D%EC%B3%90](https://happyer16.tistory.com/entry/Spring-Security-%EC%95%84%ED%82%A4%ED%85%8D%EC%B3%90)

### Authentication and Access Control

```
출처
https://happyer16.tistory.com/entry/Spring-Security-%EC%95%84%ED%82%A4%ED%85%8D%EC%B3%90

Authentication 객체?
* 이름
* 권한
* 인증여부
```
Spring Security는 authentication과 authorization 을 구분하도록 설계된 아키텍처로, 두개를 위한 전략과 확장 지점을 가지고 있다.

#### Authentication
authentication을 위한 main strategy interface는 AuthenticationManager이고
이것은 오직 하나의 메소드만 가지고 있다.

```java
public interface AuthenticationManager {

  Authentication authenticate(Authentication authentication)
    throws AuthenticationException;
}
```
AuthenticationManager는 authenticate()메소드를 통해 다음 세가지중 한가지를 할수 있다.

* 유효한 principal일 경우 Authentication 반환

```
출처: https://velog.io/@devsh/%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98-Spring-Security-Architecture-translation
접근 주체 (Principal) :
보호된 리소스에 접근하는 대상
```
* 유효하지 않은 principal일 경우 AuthenticationException 던짐

* 결정하지 못하면 null 반환

AuthenticationException은 런타임 예외이다.
예외는 어플리케이션의 스타일이나 목적에 기반하여 일반적인 방법으로 어플리케이션에 의해 핸들링 된다.
예를 들어, 웹 UI는 인증에 실패했다고 말하는 페이지를 렌더링하고, 백엔드 HTTP service는 WWW-Authenticate 헤더와 함께 또는 없이 401응답을 보낼수도 있다.

가장 많이 사용되는 AuthenticationManager의 구현체는
ProviderManager로, ProviderManager는 AuthenticationProvider  chain에게 위임한다.
AuthenticationProvider은 AuthenticationManager와 비슷하지만, AuthenticationProvider는 호출자가 지정된 인증 유형을 지원하는 여부를 확인할수 있는 메소드를 제공한다.
```java
public interface AuthenticationProvider {

	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;

	boolean supports(Class<?> authentication);
}
```
supports() 메소드 안에 Class<?> 아규먼트는 Class<? extends Authentication>이다. (authenticate메소드에 전달된 인증 방법을 지원하는지 여부만 묻는다)
ProviderManager 는 일련의 AuthenticationProviders 에게 위임함으로써 동일 어플리케이션에서 여러 다른 인증 메커니즘을 지원한다.
만약 ProviderManager가 특정 Authentication 인스턴스 타입을 인증하지 않는다면 이것은 스킵된다.
ProviderManager 는 선택적 부모를 가지며, 모든 providers가 null을 반환하는 경우 참조할수 있다.
만약 부모를 참조할수 없다면,null Authentication 으로 인해 AuthenticationException이 발생한다.
때로, 어떤 어플리케이션은 보호되는 리소스의 논리 그룹을 가지고(예를 들어, /api/** 같이 path pattern과 매치되는 모든 웹 리소스들) 각각의 그룹은 각각의 전용 AuthenticationManager을 가질수 있다.
이들 각각은 ProviderManager이고,
그리고 그들은 부모를 공유한다.
부모는 global리소스의 일종으로, 모든 providers의 대체 리소스 역할을 한다.
![](https://github.com/spring-guides/top-spring-security-architecture/raw/main/images/authentication.png)

#### Customizing Authentication Managers
Spring Security는 어플리케이션에서 공통의 authentication manager 설정을 쉽게 해주는 configuration helper를 제공한다.
가장 많이 사용되는 helper는 AuthenticationManagerBuilder인데, in-memory, JDBC 또는 LDAP user details 설정 또는 커스텀 UserDetailsService를 추가하는데 유용하다.
다음은 AuthenticationManager을 설정하는 예제이다.

```java
@Configuration
public class ApplicationSecurity extends WebSecurityConfigurerAdapter {

   ... // web stuff here

  @Autowired
  public void initialize(AuthenticationManagerBuilder builder, DataSource dataSource) {
    builder.jdbcAuthentication().dataSource(dataSource).withUser("dave")
      .password("secret").roles("USER");
  }

}
```
이 예제는 웹 어플리케이션과 관련되지만, AuthenticationManagerBuilder는 더 광범위하게 적용가능하다.

AuthenticationManagerBuilder이 @Bean 메소드 안에서 @Autowired 되어 있다는것에 유념해야 한다. AuthenticationManagerBuilder가 global AuthenticationManager을 빌드 하는것이다.
다음은 반대의 예시이다.

```java
@Configuration
public class ApplicationSecurity extends WebSecurityConfigurerAdapter {

  @Autowired
  DataSource dataSource;

   ... // web stuff here

  @Override
  public void configure(AuthenticationManagerBuilder builder) {
    builder.jdbcAuthentication().dataSource(dataSource).withUser("dave")
      .password("secret").roles("USER");
  }

}
```
우리가 메소드에 @Override를 사용했다면, AuthenticationManagerBuilder 는 오직 local AuthenticationManager을 빌드하는데 사용된다. local AuthenticationManager는 global AuthenticationManager
의 자식이 된다.
Spring Boot 어플리케이션에서는 전역 Bean을 다른 Bean에 주입하기 위해서 @Autowired를 사용할수 있지만, 로컬 Bean은 명시적으로 선언하지 않는 이상 불가능하다.
Spring Boot는 AuthenticationManager빈을 제공하지 않는 한, 기본 global AuthenticationManager을 제공한다.
custom global AuthenticationManager이 필요하지 않다면 기본 global AuthenticationManager은 충분히 안전하기 때문에 크게 걱정할 필요가 없다. 필요시 지역적으로 보호가 필요한 자원에 대해서만 AuthenticationManager 을 추가할수 있다.

#### authorization or Access Control
authentication이 성공적으로 된다면, authorization로 이동한다.
authorization에서 core strategy는 AccessDecisionManager이다.
framework에 의해 제공되는 세개의 구현체가 있고 ProviderManager 가AuthenticationProviders에 위임하는것 처럼, 세개는 모두 일련의 AccessDecisionVoter 에게 위임한다.
AccessDecisionVoter 는 secure object와 Authentication을 고려한다.
```java
boolean supports(ConfigAttribute attribute);

boolean supports(Class<?> clazz);

int vote(Authentication authentication, S object,
        Collection<ConfigAttribute> attributes);
```
 Object 는 제네릭으로 사용자가 접근 하고자 하느 모든것(웹 리소스 또는 자바 클래스내의 메소드 등)을 나타낸다.
 ConfigAttributes또한 상당히 제네릭하며 액세스에 요구되는 허용 수준을 결정하는 메타 데이터이다.
 ConfigAttribute는 인터페이스이다.
 String을 리턴하는 오직 하나의 메서드만을 가지는데, String은 리소스 소유자가 의도하는 방식으로 인코딩하여 누가 액세스 할수 있는지에 대한 규칙을 표현한다.
 일반적인 ConfigAttribute은 user role의 이름이다.(ROLE_ADMIN 또는 ROLE_AUDIT같은)
때론 특수한 포맷(ROLE_prefix같은)또는 평가해야하는 expression을 가진다.
대부분 기본 AccessDecisionManager인, AffirmativeBased 을 사용한다.
 ConfigAttributes 는 Spring Expression Language (SpEL) expressions을 사용하는게 일반적이다.
 이것은 expressions을 처리하고 context를 만들수 있는 AccessDecisionVoter 에 의해 지원된다.
 처리할수 있는 expression의 범위를 확장 하기 위해서는 SecurityExpressionRoot 구현이 필요하고  때로는 SecurityExpressionHandler도 필요하다.

 ### Web Security
웹 티어(UI, HTTP 백엔드)의 스프링 시큐리티는 서블릿 필터 기반이기 때문에 필터의 역할을 먼저 살펴보는것이 좋다.
다음의 사진은 단일 HTTP request를 위한 핸들러의 일반적인 계층을 보여준다.
![](https://github.com/spring-guides/top-spring-security-architecture/raw/main/images/filters.png)

클라이언트가 어플리케이션에 요청을 보내면 컨테이너는 요청 URL 경로를 기반으로 적용될 필터와 서블릿을 결정한다.
대부분 하나의 서블릿은 하나의 요청을 처리할수 있지만 필터는 체인을 형성하며 순서가 지정된다.
요청 자체를 처리 하려는 경우 필터가 나머지 체인을 거부할수 있다.
필터는 다운스트림 필터와 서블릿을 사용해서 요청과 응답을 수정할수 있다.
filter chain의 순서는 매우 중요한데, Spring Boot는 두개의 메커니즘을 통해 이것을 관리한다.
Filter타입의 @Beans에 @Order를 붙이거나 Ordered를 구현하는것이고 다른 하나는 API의 일부로 순서를 가지는 FilterRegistrationBean 의 일부가 되는것이다.
일부 이미 만들어진 필터는 상수를 정의하여 상대적인 순서를 지정할수 있다. (예를 들어, Spring Session의 SessionRepositoryFilter는 Integer.MIN_VALUE + 50 의 값을 가지는 DEFAULT_ORDER를 가지며 이는 체인의 앞단에 있고자 하지만 그 앞에 다른 필터가 오는 것을 배제하지는 않는다.)
Spring Security는 체인에서 single Filter로 설치되고 이것의 구체적인 타입은 FilterChainProxy이다.
Spring Boot 어플리케이션에서 security filter는 ApplicationContext의 @Bean이고, 기본으로 설치되어 모든 요청에 적용된다.
이것은 SecurityProperties.DEFAULT_FILTER_ORDER에 의해 정의된 위치에 설치되며 차례로 FilterRegistrationBean.REQUEST_WRAPPER_FILTER_MAX_ORDER에 의해 초기화 된다.
하지만 그 이상의 것이 있는데, container의 관점에서 보면 Spring Security는 single filter이다. 하지만 그 내부를 보면 추가적인 필터들이 존재하고 각각은 특별한 역할을 하고 있다.
다음 사진은 이러한 relationship을 보여준다.
![https://github.com/spring-guides/top-spring-security-architecture/raw/main/images/security-filters.png](https://github.com/spring-guides/top-spring-security-architecture/raw/main/images/security-filters.png)    
사실 security filter에는 다음과 같은 간접 계층이 하나 더 있다.
이것은 일반적으로 DelegatingFilterProxy로 컨테이너 안에 설치되며 Spring @Bean일 필요는 없다.
이 Proxy는 항상 @Bean이고 springSecurityFilterChain 라는 고정된 이름을 사용하는  FilterChainProxy에게 위임한다.
FilterChainProxy는 필터 또는 체인의 체인으로 내부적으로 배열된 모든 security login을 포함한다.
모든 필터들은 같은 API를 가지며(그들은 모두 Filter 인터페이스를 구현한다.) 그들은 모두 체인의 나머지를 거절할수 있다.
Spring Security filter는 필터 체인들의 리스트를 포함하고 요청과 매치되는 첫번째 체인에 요청을 보낸다. 다음 사진은 요청 경로(/foo/**) 매칭을 기반으로 dispatch가 발생하는것을 보여준다.
이것은 매우 흔한 방법이지만 요청을 매칭하는 유일한 방법은 아니다.
이러한 dispatch process의 가장 중요한 특징은 요청을 처리하는 체인이 하나뿐이라는것이다.
![https://github.com/spring-guides/top-spring-security-architecture/raw/main/images/security-filters-dispatch.png](https://github.com/spring-guides/top-spring-security-architecture/raw/main/images/security-filters-dispatch.png)
custom security configuration이 없는 vanilla Spring Boot application은 여러개의 필터 체인을 가지는데, 일반적으로 6개이다.
첫번째 체인은 /css/**, /images/** 같은 정적 리소스 패턴, error view: /error을 무시한다.
마지막 체인은 catch-all path(/**)와 일치하고 권한 부여, 예외 처리, 세션처리, 헤더 쓰기 등에 대한 로직을 포함하여 더 활성화 된다.
기본적으로 이 체인에는 11개의 필터가 포함되어 있지만, 일반적으로  어떤 필터를 언제 사용하는지 우리가 고민 할 필요는 없다.
