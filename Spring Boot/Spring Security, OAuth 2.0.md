Spring Security, OAuth 2.0
======

* `Spring Security`는 막강한 인증(Authentication)과 인가(Authorization)의 기능을 가진 프레임워크이다.   
  이는 사실상 Spring 기반의 Application에서는 보안을 위한 표준과 동일하게 취급된다.   
  Spring은 Interceptor, Filter 등을 기반으로 보안 기능을 구현하는 것 보다 Spring-Security를 활용할 것을 권장한다.
<hr/>

<h2>Spring Security & OAuth2 Client</h2>

* 많은 서비스에서 로그인 기능을 id/password 방식보다는 구글, 페이스북 등의 소셜 로그인 기능을 활용한다.   
  이를 활용하는 이유는 로그인 구현을 소셜 서비스에 맡기고, 서비스 개발에 집중할 수 있기 때문이다.

<hr/>

<h2>Google Service 등록</h2>

* 먼저 Google Service에 신규 서비스를 생성한다. 여기서 발급한 인증 정보(clientId, clientServer)를 통해   
  로그인 기능과 소셜 서비스 기능을 사용할 수 있으니 무조건 발급받고 시작해야 한다.
* http://console.cloud.google.com
* 위 사이트에서 OAuth Client ID 생성까지 마친 후 client-id와 client-secret 코드를 아래와 같이 `src/main/resources`의 하위에   
  `application-oauth.properties` 파일을 생성하고, 입력하자.
```properties
spring.security.oauth2.client.registration.google.client-id=clientid값
spring.security.oauth2.client.registration.google.client-secret=clientsecret값
spring.security.oauth2.client.registeration.google.scope=profile,email
```
* Spring-boot에서는 properties 파일의 이름을 `application-xxx.properties`로 만들면 xxx라는 이름의 `profile`이   
  생성되어 이를 통해 관리할 수 있다. 즉 `profile=xxx`라는 식으로 호출하면 해당 properties의 설정을 가져올 수 있다.   
  호출하는 방식은 여러 방식이 있지만 여기서는 Spring-boot의 기본 설정 파일인 `application.properties`에서   
  `application-oauth.properties`를 포함하도록 구성하자. `application.properties`파일에 아래 코드를 추가하자.
```properties
spring.profiles.include=oauth
```
<hr/>

<h2>구글 로그인 연동하기</h2>

* Google의 로그인 인증정보를 발급 받았으니 프로젝트의 구현을 진행해보자.   
  먼저 사용자 정보를 담당할 도메인인 `User`클래스를 생성하자. 위치 : `domain` 하위의 패키지
```java
package com.sangwoo.board.domain.user;

import com.sangwoo.board.domain.BaseTimeEntity;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

import javax.persistence.*;

@Getter
@NoArgsConstructor
@Entity
public class User extends BaseTimeEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable=false)
    private String name;
    
    @Column(nullable = false)
    private String email;
    
    @Column
    private String picture;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Role role;
    
    @Builder
    public User(String name, String email, String picture, Role role) {
        this.name = name;
        this.email = email;
        this.picture = picture;
        this.role = role;
    }
    
    public User update(String name, String picture) {
        this.name = name;
        this.picture = picture;
        return this;
    }
    
    public String getRoleKey() {
        return this.role.getKey();
    }
}
```
* `@Enumerated(EnumType.STRING)`은 JPA로 DB로 저장할 때 Enum값을 어떤 형태로 저장할지를 결정한다.   
  기본적으로는 int로 된 숫자가 저장되는데, 숫자로 저장되면 db 확인 시 그 값이 어떤 의미를 가지는지 알기 힘들다.   
  따라서 문자열(EnumType.STRING)로 저장될 수 있도록 선언했다.

* 다음으로는 각 사용자의 권한을 관리할 `Enum` 클래스 `Role` 을 생성하자.
```java
package com.sangwoo.board.domain.user;

import lombok.Getter;
import lombok.RequiredArgsConstructor;

@Getter
@RequiredArgsConstructor
public enum Role {
    
    GUEST("ROLE_GUEST", "손님"),
    USER("ROLE_USER", "일반 사용자");
    
    private final String key;
    private final String title;
}
```
* Spring Security에서는 권한 코드에 항상 __ROLE_ 이 앞에 있어야만__ 한다.   
  따라서 코드별 key값을 ROLE_GUEST, ROLE_USER로 지정했다.

* 마지막으로 `User`의 CRUD를 책임질 `UserRepository`를 생성하자.
```java
package com.sangwoo.board.domain.user;

import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}
```
* 위 인터페이스의 `findByEmail()` 메소드는 소셜 로그인으로 반환되는 값 중 email을 통해 이미 생성된 사용자인지,   
  처음 가입하는 사용자인지를 판단하기 위한 메소드이다.
<hr/>

<h2>Spring Security 설정</h2>

* 먼저 `build.gradle` 파일에 Spring-Security 관련 의존성을 하나 추가하자.
```gradle
compile('org.springframework.boot:spring-boot-starter-oauth2-client')
```
* 위 의존성은 소셜 로그인 등 클라이언트 입장에서 소셜 기능 구현 시 필요한 의존성이다.   
  또한 `spring-security-oauth2-client`와 `spring-security-oauth2-jose`를 기본으로 관리해준다.

* 다음으로는 OAuth 라이브러리를 이용한 소셜 로그인 설정 코드를 작성하자.   
  `config.auth` 패키지를 생성하고, 앞으로 __security 관련 모든 클래스는 이 곳에 보관__ 한다.

* 위에서 생성한 패키지에 `SecurityConfig` 클래스를 생성하고, 아래와 같이 작성한다.
```java
@RequiredArgsConstructor
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    private final CustomOAuth2UserService customOAuth2UserService;
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable().headers().frameOptions().disable().and()
                .authorizeRequests()
                .antMatchers("/", "/css/**", "/images/**", "/js/**", "/h2-console/**")
                .permitAll().antMatchers("/api/v1/**").hasRole(Role.USER.name())
                .anyRequest().authenticated().and()
                .logout().logoutSuccessUrl("/")
                .and().oauth2Login().userInfoEndpoint().userService(customOAuth2UserService);
    }
}
```
* __@EnableWebSecurity__ : Spring Security 설정들을 활성화시킨다.
* `csrf().disable().headers().frameOptions().disable()` : h2-console화면을 사용하기 위해 해당 option들을 disable 한다.
* `authorizeRequests()`: URL별 권한 관리를 설정하는 option의 시작점이다. 이 메소드가 선언되어야만 뒤에   
  `antMatchers()` 옵션을 사용할 수 있다.
* `antMatchers()` : 권한 관리 대상을 지정하는 option이다. URL, HTTP Method별로 관리가 가능하다.   
  "/" 등 지정된 URL들은 `permitAll()` 옵션을 통해 전체 열람 권한을 부여했다.   
  "/api/v1/**" 주소를 가진 API는 USER권한을 가진 사람만 가능하도록 설정했다.
* `anyRequest()` : 설정된 값들 이외 나머지의 URL들을 나타낸다. 위에서는 바로 다음에 `authenticated()` option을 추가하여   
  나머지 URL들은 모두 인증된 사용자들에게만 허용하게 했다. 인증된 사용자는 로그인한 사용자들을 의미한다.
* `logout().logoutSuccessUrl("/")` : 로그아웃 기능에 대한 여러 설정의 진입점으로, 로그아웃 성공 시 "/"의 주소로 이동한다.
* `oauth2Login()` : OAuth2 로그인 기능에 대한 여러 설정의 진입점이다.
* `userInfoEndpoint()` : OAuth2 로그인 성공 이후 사용자 정보를 가져올 때의 설정들을 담당한다.
* `userService()` : 소셜 로그인 성공 시 후속 조치를 진행할 `UserService`인터페이스의 구현체를 등록한다.   
  리소스 서버, 즉 소셜 서비스에서 사용자 정보를 가져온 상태에서 추가로 진행하고자 하는 기능을 명시할 수 있다.

* 이제 `CustomOAuth2UserService` 클래스를 작성하자. 이 클래스는 구글 로그인 이후 가져온 사용자의 정보(email, name, phone)   
  들을 기반으로 가입 및 정보 수정, 세션 저장 등의 기능을 지원한다.
```java
@RequiredArgsConstructor
@Service
public class CustomOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {
    
    private final UserRepository userRepository;
    private final HttpSession httpSession;
    
    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2UserService<OAuth2UserRequest, OAuth2User> delegate = new DefaultOAuth2UserService();
        OAuth2User oAuth2User = delegate.loadUser(userRequest);
        
        String registrationId = userRequest.getClientRegistration().getRegistrationId();
        String userNameAttributeName = userRequest.getClientRegistration().getProviderDetails()
                .getUserInfoEndpoint().getUserNameAttributeName();
        
        OAuthAttributes attributes = OAuthAttributes.of(registrationId, userNameAttributeName, oAuth2User.getAttributes());
        
        User user = saveOrUpdate(attributes);
        
        httpSession.setAttribute("user", new SessionUser(user));
        
        return new DefaultOAuth2User(Collections.singleton(new SimpleGrantedAuthority(user.getRoleKey())),
                attributes.getAttributes(), attributes.getNameAttributeKey());
    }
    
    private User saveOrUpdate(OAuthAttributes attributes) {
        User user = userRepository.findByEmail(attributes.getEmail())
                .map(entity -> entity.update(attributes.getName(), attributes.getPicture()))
                .orElse(attributes.toEntity());
        return userRepository.save(user);
    }
}
```
* `registrationId` : 현재 로그인 진행중인 서비스를 구분하는 코드이다. 현재는 Google만 사용하므로 불필요하지만,   
  이후 네이버 등 다른 로그인 연동 시에 어떤 서비스에 로그인하는지를 구분하기 위해 사용한다.
* `userNameAttributeName` : OAuth2 로그인 진행 시 key가 되는 필드값을 의미한다. PK와 같은 의미이다.   
  Google의 경우 기본적으로 코드를 지원하지만, 네이버, 카카오 등은 지원하지 않는다. Google의 기본 코드는 "sub" 이다.   
  이 필드는 이후 네이버 로그인과 구글 로그인을 동시에 지원할 때 사용된다.
* `OAuthAttributes` : `OAuth2UserService`를 통해 가져온 `OAuth2User`의 attribute를 담을 클래스이다.
* `SessionUser` : 세션에 사용자 정보를 저장하기 위한 Dto 클래스이다.
* `saveOrUpdate()` 메소드에는 구글 사용자 정보가 업데이트 되었을 때를 대비하여 update 기능도 구현했다.   
  사용자의 이름이나 picture가 변경되면 User Entity에도 반영된다.

* 다음으로는 `OAuthAttributes` 클래스를 작성하자.
```java
@Getter
public class OAuthAttributes {
    private Map<String, Object> attributes;
    private String nameAttributeKey;
    private String name;
    private String email;
    private String picture;
    
    @Builder
    public OAuthAttributes(Map<String, Object> attributes, String nameAttributeKey, String name, String email, String picture) {
        this.attributes = attributes;
        this.nameAttributeKey = nameAttributeKey;
        this.name = name;
        this.email = email;
        this.picture = picture;
    }
    
    public static OAuthAttributes of(String registrationId, String userNameAttributeName, Map<String, Object> attributes) {
        return ofGoogle(userNameAttributeName, attributes);
    }
    
    public static OAuthAttributes ofGoogle(String userNameAttributeName, Map<String, Object> attributes) {
        return OAuthAttributes.builder().name((String)attributes.get("name"))
                .email((String)attributes.get("email")).picture((String)attributes.get("picture"))
                .attributes(attributes).nameAttributeKey(userNameAttributeName).build();
    }
    
    public User toEntity() {
        return User.builder().name(name).email(email).picture(picture).role(Role.GUEST).build();
    }
}
```
* `of()` : `OAuth2User`에서 반환하는 사용자 정보는 `Map`형식이기 때문에 값 하나하나를 반환해야 한다.
* `toEntity()` : `User` Entity를 생성한다. `OAuthAttributes`에서 Entity를 생성하는 시점은 처음 가입할 때 이다.   
  가입할 때의 기본 권한을 GUEST로 주기 위해서 `role()` 빌더값에는 Role.GUEST를 지정했다.

* 다음으로는 `SessionUser` 클래스를 생성하자.
```java
@Getter
public class SessionUser implements Serializable {
    private String name;
    private String email;
    private String picture;
    
    public SessionUser(User user) {
        this.name = user.getName();
        this.email = user.getEmail();
        this.picture = user.getPicture();
    }
}
```
* `SessionUser` 클래스는 `HttpSession` 객체에 저장할 정보이므로 __인증된 사용자 정보만 필요__ 하기 때문에 name, email,   
  picture만 저장하도록 했다.
* `User`클래스를 `Serializable`을 구현하게 하지 않고, 따로 `SessionUser` 클래스를 작성하여 세션에 저장한 이유는,   
  `User`클래스는 Entity이기 때문이다. Entity class에는 언제 다른 Entity와 관계가 형성될지 모른다. 만약 직렬화 대상에   
  자식들까지 포함되면 성능상의 이슈 및 부수 효과가 발생할 확률이 높다. 따라서 직렬화 기능을 가진 Session Dto를 만든 것이다.   
  이는 이후 운영 및 유지보수 때 많은 도움이 된다.
<hr/>

<h3>Login Test</h3>

* 이제 Spring-Security가 잘 적용됐는지를 확인하기 위해 화면에 로그인 버튼을 추가해보자.
* 아래는 수정된 `index.mustache` 의 코드이다.
```mustache
{{>layout/header}}

<h1>Web Service using Spring-Boot.</h1>
<div class="col-md-12">
    <!-- 로그인 기능 영역-->
    <div class="row">
        <div class="col-md-6">
            <a href="/posts/save" role="button" class="btn btn-primary">글 등록</a>
            {{#userName}}
                Logged in as : <span id="user">{{userName}}</span>
                <a href="/logout" class="btn btn-info active" role="button">Logout</a>
            {{/userName}}
            {{^userName}}
                <a href="/oauth2/authorization/google" class="btn btn-success active" role="button">Google Login</a>
            {{/userName}}
        </div>
    </div>
    <br/>

    <!-- 목록 출력 영역 -->
    <table class="table table-horizontal table-bordered">
        <thead class="thead-strong">
            <tr>
                <th>게시글 번호</th>
                <th>제목</th>
                <th>작성자</th>
                <th>최종 수정일</th>
            </tr>
        </thead>
        <tbody id="tbody">
            {{#posts}}
                <tr>
                    <td>{{id}}</td>
                    <td><a href="/posts/update/{{id}}">{{title}}</a></td>
                    <td>{{author}}</td>
                    <td>{{modifiedDate}}</td>
                </tr>
            {{/posts}}
        </tbody>
    </table>
</div>

{{>layout/footer}}
```
* `{{#userName}}` : Mustache는 다른 언어들에 있는 if문을 제공하지 않는다. 오로지 true/false만 판별할 뿐이다.   
  따라서 mustache에는 항상 최종값을 넘겨줘야한다. 위 코드에서는 userName이 있다면 userName을 노출시키도록 한 것이다.
* `a href="/logout"` : 이 URL은 Spring Security에서 기본적으로 제공하는 로그아웃 URL 이다. 즉, 개발자가 별도로   
  위 URL에 해당하는 컨트롤러를 만들 필요가 없다.
* `{{^userName}}` : mustache에서 해당 값이 존재하지 않는 경우에는 `^` 기호를 사용한다. 위 코드에서는 userName값이   
  없다면 로그인 버튼을 노출시켰다.
* `a href="/oauth2/authorization/google` : Spring Security에서 기본적으로 제공하는 로그인 URL 이다.   
  로그아웃 URL과 마찬가지로 개발자가 별도의 컨트롤러를 생성할 필요가 없다.

* 위의 `index.mustache`에서 userName을 session에서 가져와 사용하므로, `IndexController`에 userName을 `Model`객체에   
  추가하는 코드를 작성해보자.
```java
@RequiredArgsConstructor
@Controller
public class IndexController {

    private final PostsService postsService;
    private final HttpSession httpSession;

    @GetMapping("/")
    public String index(Model model) {
        model.addAttribute("posts", postsService.findAllDesc());

        SessionUser user = (SessionUser)httpSession.getAttribute("user");
        if(user != null) {
            model.addAttribute("userName", user.getName());
        }
        return "index";
    }

    // 생략
}
```
<hr/>

<h2>어노테이션 기반으로 개선</h2>

* 위 코드는 문제점이 있는데, 바로 코드가 반복된다는 점이다. 아래 코드가 반복된다.
```java
SessionUser user = (SessionUser)httpSession.getAttribute("user");
```

* 따라서 위 부분을 __메소드의 인자로 세션값을 받을 수 있도록 변경__ 해보자.
```java
package com.sangwoo.issuemanager.config.auth;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface LoginUser {
}
```
* `@Target(ElementType.PARAMETER)`는 이 어노테이션이 생성될 수 있는 위치를 지정한다. 위 코드에서는 `PARAMETER`로   
  지정했으니 메소드의 파라미터로 선언된 객체에서만 사용할 수 있게 된다. 이 외에도 클래스 선언문에 사용할 수 있는 `TYPE` 등이 있다.
* `@interface` : 이 파일을 어노테이션 클래스로 지정한다. 즉, 위 코드에서는 `@LoginUser` 어노테이션이 생성된 것이다.
  
* 다음으로는 `HandlerMethodAgrumentResolver` 인터페이스를 구현하는 `LoginUserAgrumentResolver`클래스를 작성하자.   
  `HandlerMethodAgrumentResolver`는 한가지 기능을 지원하는데, 바로 조건에 맞는 경우에 메소드가 있다면   
  이 인터페이스의 구현체가 지정한 값으로 해당 메소드의 파라미터로 넘길 수 있다는 것이다.
```java
import com.sangwoo.issuemanager.web.dto.SessionUser;
import lombok.RequiredArgsConstructor;
import org.springframework.core.MethodParameter;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.ModelAndViewContainer;

import javax.servlet.http.HttpSession;

@RequiredArgsConstructor
@Component
public class LoginUserArgumentResolver implements HandlerMethodArgumentResolver {
    
    private final HttpSession httpSession;
    
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        boolean isLoginUserAnnotation = parameter.getParameterAnnotation(LoginUser.class) != null;
        boolean isUserClass = SessionUser.class.equals(parameter.getParameterType());
        return isLoginUserAnnotation && isUserClass;
    }
    
    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest,
                                  WebDataBinderFactory binderFactory) throws Exception {
        return httpSession.getAttribute("memberInfo");
    }
}
```

* `supportsParameter()` : 컨트롤러 메소드의 특정 파라미터를 지원하는지 판단한다. 위 코드에서는 파라미터에   
  `@LoginUser` 어노테이션이 붙어 있고, 파라미터 클래스 타입이 `SessionUser.class`인 경우에 true를 반환한다.
* `resolveArgument()` : 파라미터에 전달된 객체를 생성한다. 위 코드에서는 session에서 객체를 가져온다.

* 다음으로는 위에서 생성한 `@LoginUser` 어노테이션과 `LoginUserArgumentResolver`클래스가 스프링에서 인식될 수 있도록   
  `WebMvcConfigurer`에 추가하자.
```java
@RequiredArgsConstructor
@Configuration
public class WebConfig implements WebMvcConfigurer {

    private final LoginUserArgumentResolver loginUserArgumentResolver;

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(loginUserArgumentResolver);
    }
}
```
<hr/>

<h2>Naver Login</h2>

* 네이버도 마찬가지로 clientID와 clientSecret를 발급받은 후, 아래와 같이 `application-oauth.properties`에 등록한다.
```properties
#registration

spring.security.oauth2.client.registration.naver.client-id=clientID값
spring.security.oauth2.client.registration.naver.client-secret=clientSecret값
spring.security.oauth2.client.registration.naver.redirect-uri={baseUrl}/{action}/oauth2/code/{registrationId}
spring.security.oauth2.client.registration.naver.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.naver.scope=name,email,profile_image
spring.security.oauth2.client.registration.naver.client-name=Naver

#provider
spring.security.oauth2.client.provider.naver.authorization-uri=https://nid.naver.com/oauth2.0/authorize
spring.security.oauth2.client.provider.naver.token-uri=https://nid.naver.com/oauth2.0/token
spring.security.oauth2.client.provider.naver.user-info-uri=https://openapi.naver.com/v1/nid/me
spring.security.oauth2.client.provider.naver.user-name-attribute=response
```

* 위의 `#provider`와 `#registration` 아래의 코드들은 네이버가 SpringSecurity를 공식 지원하지 않기 때문에   
  그동안 `CommonOAuth2Provider`가 처리해주던 값들을 수동으로 입력한 것이다.

* Spring-Security에서는 __하위 필드를 명시할 수 없다__. 최상위 필드들만 `user_name`으로 지정 가능하지만, 네이버의   
  응답값 최상위 필드는 `resultCode`, `message`, `response`이다. 이러한 이유로 Spring-security에서 인식 가능한 필드는   
  3개중에 골라야 하는데, 위 코드는 본문에서 담고 있는 `response`를 `user_name`으로 지정하고, 이후 Java 코드에서   
  `response`의 id를 `user_name`으로 지정한다.

* 다음으로는 `OAuthAttributes` 클래스에 로그인하는 OAuth2가 네이버인지를 판별하는 코드를 추가한다.
```java
// OAuthAttributes.java

@Getter
public class OAuthAttributes {

  //..

  public static OAuthAttributes of(String registrationId, String userNameAttributeName, Map<String, Object> attributes) {
        if("naver".equals(registrationId)) {
            return ofNaver("id", attributes);
        }
        return ofGoogle(userNameAttributeName, attributes);
    }
  
  private static OAuthAttributes ofNaver(String userNameAttributeName, Map<String, Object> attributes) {
        Map<String, Object> response = (Map<String, Object>)attributes.get("response");
        return OAuthAttributes.builder()
                .name((String)response.get("name"))
                .email((String)response.get("email"))
                .picture((String)response.get("profile_image"))
                .attributes(response)
                .nameAttributeKey(userNameAttributeName)
                .build();
    }
}
```
<hr/>

<h2>기존 테스트 코드에 Spring-security 적용하기</h2>

* `src/main` 환경과 `src/test`의 환경은 각자의 환경 구성을 가진다. 다만, 테스트 시 `src/main/resources/application.properties`   
  가 적용되는 이유는 `test` 폴더 하위에 `application.properties`가 없기에 `main` 하위의 파일을 가져오기 때문이다.   
  단, 자동으로 가져오는 옵션의 범위는 `application.properties` 파일 까지이다. 즉, `application-oauth.properties`는   
  __test에 파일이 없다고 main에서 가져오지 않는다__. 이를 해결하기 위해 테스트 환경을 위한 `application.properties`를 만들자.   
  실제로 google 연동까지 진행할것이 아니므로 __가짜 설정값__ 을 등록한다.

```properties
server.address=localhost
server.port=8080
spring.jpa.show-sql=true

spring.jpa.database=mysql
spring.jpa.database-platform=org.hibernate.dialect.MySql5InnoDBDialect
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/DBName?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC
spring.datasource.username=MySQLAccount
spring.datasource.password=MySQLPassword
spring.jpa.hibernate.ddl-auto=none
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL5InnoDBDialect

# Test OAuth
spring.security.oauth2.client.registration.google.client-id=test
spring.security.oauth2.client.registration.google.client-secret=test
spring.security.oauth2.client.registration.google.scope=profile,email
```

* 다음으로는 Spring-security 설정 때문에 __인증되지 않은 사용자의 요청은 이동__ 시키는 문제를 해결하자.   
  이러한 API 요청은 임의로 인증된 사용자를 추가하여 API만 테스트해볼 수 있게 한다.

* `build.gradle`에 `Spring-Security test`를 위한 여러 도구를 지원하는 모듈을 추가하자.
```gradle
testCompile('org.springframework.security:spring-security-test')
```

* 다음으로는 `IssuesApiControllerTest`의 테스트 코드에 아래와 같이 __임의 사용자 인증__ 을 추가하자.
```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class IssuesApiControllerTest {

  //..

  @Test
  @WithMockUser(roles = "USER")
  public void issue_is_uploaded() throws Exception {

    //..

  }

  @Test
  @WithMockUser(roles = "USER")
  public void issue_is_edited() throws Exception {

    //..
  }
}
```

* `@WithMockUser(roles="USER")` : 인증된 모의 사용자를 만들어 사용한다. roles에 권한을 설정할 수 있으며, 이 어노테이션으로   
  인해 `ROLE_USER` 권한을 가진 사용자가 API를 요청하는 것과 동일한 효과를 가지게 된다.

* __@WithMockUser__ 는 `MockMvc`상에서만 작동하기 때문에 위 테스트 코드는 수정이 필요하다. 현재 위 코드는 __@SpringBootTest__ 로만   
  되어 있으며, `MockMvc`를 사용하지 않는다. 따라서 __@SpringBootTest__ 에서 다음과 같이 하여 `MockMvc`를 사용하도록 하자.
```java
package com.sangwoo.issuemanager.web;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sangwoo.issuemanager.domain.issues.Issues;
import com.sangwoo.issuemanager.domain.issues.IssuesRepository;
import com.sangwoo.issuemanager.domain.issues.Status;
import com.sangwoo.issuemanager.web.dto.IssuesSaveRequestDto;
import com.sangwoo.issuemanager.web.dto.IssuesUpdateRequestDto;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.web.server.LocalServerPort;
import org.springframework.http.*;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers.springSecurity;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.put;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class IssuesApiControllerTest {

    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private IssuesRepository issuesRepository;

    @Autowired
    private WebApplicationContext context;

    private MockMvc mvc;

    @Before
    public void setup() {
        mvc = MockMvcBuilders.webAppContextSetup(context).apply(springSecurity()).build();
    }

    @After
    public void tearDown() throws Exception {
        issuesRepository.deleteAll();
    }

    @Test
    @WithMockUser(roles = "USER")
    public void issue_is_uploaded() throws Exception {

        // given
        String email = "test@test.com";
        String title = "test title";
        String content = "test content";
        Status status = Status.OPEN;

        IssuesSaveRequestDto requestDto = IssuesSaveRequestDto.builder().email(email).title(title).content(content).status(status).build();

        String url = "http://localhost:" + port + "/issues";

        // when
        mvc.perform(post(url).contentType(MediaType.APPLICATION_JSON_UTF8).content(new ObjectMapper().writeValueAsString(requestDto)))
                .andExpect(status().isOk());

        // then
        List<Issues> all = issuesRepository.findAll();
        assertThat(all.get(0).getEmail()).isEqualTo(email);
        assertThat(all.get(0).getTitle()).isEqualTo(title);
        assertThat(all.get(0).getContent()).isEqualTo(content);
        assertThat(all.get(0).getStatus()).isEqualTo(Status.OPEN);
    }

    @Test
    @WithMockUser(roles = "USER")
    public void issues_is_edited() throws Exception {

        // given
        Issues savedIssues = issuesRepository.save(Issues.builder().title("Test title").email("test@test.com").content("test content.").status(Status.CLOSED).build());

        Long updateId = savedIssues.getId();
        String expectedTitle = "Test title";
        String expectedContent = "test content.";
        Status expectedStatus = Status.CLOSED;

        IssuesUpdateRequestDto requestDto = IssuesUpdateRequestDto.builder().title(expectedTitle).content(expectedContent).status(expectedStatus).build();

        String url = "http://localhost:" + port + "/issues/" + updateId;
        HttpEntity<IssuesUpdateRequestDto> requestEntity = new HttpEntity<>(requestDto);

        // when
        mvc.perform(put(url).contentType(MediaType.APPLICATION_JSON_UTF8).content(new ObjectMapper().writeValueAsString(requestDto)))
                .andExpect(status().isOk());

        // then
        List<Issues> all = issuesRepository.findAll();
        assertThat(all.get(0).getTitle()).isEqualTo(expectedTitle);
        assertThat(all.get(0).getContent()).isEqualTo(expectedContent);
        assertThat(all.get(0).getStatus()).isEqualTo(expectedStatus);
    }
}
```
* __@Before__ : 매번 테스트가 시작되기 전에 `MockMvc` 인스턴스를 생성한다.
* `mvc.perform()` : 생성된 `MockMvc`를 통해 API를 테스트한다. 본문(Body) 영역은 문자열로 표현하기 위해   
  `ObjectMapper`를 통해 JSON문자열로 변환한다.
