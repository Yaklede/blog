---
title: "Spring Security 기본 API 및 Filter"
description: spring boot 3.x 기반 기본 설정
date: 2023-04-10T23:00:15+09:00
image: 
math: 
license: 
hidden: false
comments: true
draft: true
---
# Spring Security 맛보기
> 이번 글에서는 간단하게 spring security가 무엇인지 살펴보고 간단하게 구현 해볼 것이다.
# Spring Security 사용자 정의 보안 기능

## 시작하기전에 Config 설정하기
### spring boot 2.7 부터 spring security 설정이 바뀌었습니다.
기존 spring boot 2.x 버전에서는 
```kotlin
@Configuration
@EnableWebSecurity
class SpringConfig : WebSecurityConfigurerAdapter() {
    
    override fun configure(http: HttpSecurity) {
        http
            .authorizeHttpRequests()
            .anyRequest().authenticated()
        http
            .formLogin()
    }
} 
```
이런식으로 config 설정을 했다면 바뀐 방식으로는
```kotlin
@Configuration
class SecurityConfig {
    @Bean
    fun configurer(http: HttpSecurity): SecurityFilterChain {
        http
            .authorizeHttpRequests()
            .anyRequest().authenticated()

        http
            .formLogin()
        return http.orBuild
    }
}
```
이런식으로 바뀌게 되었다.

## 자동 보안 기능
Spring boot 프로젝트에서 Spring-Security 의존성을 넣으면 자동으로
스프링 시큐리티가 적용되기 시작한다. 하지만 이 자동 보안 기능 같은경우에는 실무에서 사용하지 적합하지 않다.

## 사용자 정의 보안 기능
> **spring boot 2.x**
<br>
위에서 설정한 config가 사용자 정의 보안에 해당하는 기능이다. spring security를 사용 할 땐 개발자가 직접 보안 기능들을 정의하는 것이 일반적이다
<br>
개발자가 직접 보안을 정의 하기 위해서는 다음과 같다. <br>
WebSecurityConfigurerAdapter를 상속받고 자식 클래스인 Config에서 configurer를 override하면 된다.

> **spring boot 3.x**
<br>
Config class를 하나 정의하고 SecurityFilterChain을 반환 하는 bean을 등록해주면 된다.


여기서 중요한 포인트는 WebSecurityConfigurerAdapter()과 HttpSecurity이다.
### WebSecurityConfigurerAdapter 역할
- 스프링 애플리케이션이 시작 시 동작
- 웹 보안 기본 설정 초괴화 수행

### HttpSecurity
- 사용자의 세부적인 보안 기능 설정
- 인증 및 인가와 관련된 다양한 API 제공

### Form 인증 방식
spring security에서 기본으로 사용하는 방식이며 우리가 아는 form형식에서 데이터를 받아와서 검증한다

```kotlin
class SuccessDefaultHandler() : AuthenticationSuccessHandler {
            override fun onAuthenticationSuccess(
                request: HttpServletRequest,
                response: HttpServletResponse,
                authentication: Authentication
            ) {
                println("authentication ${authentication.name}")
                response.sendRedirect("/")
            }
        }
```
로그인 성공시 거치는 Default Handler
```kotlin
class FailDefaultHandler() : AuthenticationFailureHandler {
    override fun onAuthenticationFailure(
        request: HttpServletRequest,
        response: HttpServletResponse,
        exception: AuthenticationException
    ) {
        println("authentication Fail")
        response.sendRedirect("/login")
    }
}
```
로그인 실패시 거치는 Fail Handler

```kotlin
fun configurer(http: HttpSecurity): SecurityFilterChain {
        //인가 정책
        http
            .authorizeHttpRequests()
            .anyRequest().authenticated() // 모든 요청에 대해서 인증 정책

        //인증 정책 default form 인증
        http
            .formLogin() //form 로그인 인증 정책
            .defaultSuccessUrl("/") // 로그인 인증 성공 후 redirect Url
            .failureForwardUrl("/login") // 로그인 인증 실패 redirect Url
            .usernameParameter("username") // form LoginId parameter 지정
            .passwordParameter("password") // form Password parameter 지정
            .loginProcessingUrl("/login_proc") // form action url
            .successHandler(SuccessDefaultHandler()) // 로그인 인증 성공시 거치는 필터
            .failureHandler(FailDefaultHandler()) // 로그인 실패시 거치는 필터
            .permitAll() //인증이 되어 있지 않아도 접근이 가능함

        return http.orBuild
    }
```
위와 같이 설정 해줄 수 있다.

### 



# Spring Security 시작하기

먼저 spring security의 의존성을 주입받는다
```gradle.kts
implementation("org.springframework.boot:spring-boot-starter-security")
```

config 설정하기
```kotlin
    @Bean
    fun configurer(http: HttpSecurity): SecurityFilterChain {
        //모든 접근에 대해서 인증을 한다.
        http
            .authorizeHttpRequests()
            .anyRequest().authenticated()

    http
        .formLogin()
        .defaultSuccessUrl("/")
        .failureForwardUrl("/login")
        .usernameParameter("username")
        .passwordParameter("password")
        .loginProcessingUrl("/login_proc")

    //logout
    http
        .logout() // logout
        //.logoutUrl("/logout") //logout request url
        .logoutSuccessUrl("/login") // logout Success Redirect Url
        .addLogoutHandler(DefaultLogoutHandler()) // 로그아웃 요청시 동작하는 handler
        .logoutSuccessHandler(DefaultLogoutSuccessHandler()) //로그아웃 성공시 동작하는 handler

    //session Management Filter
    http
        .sessionManagement()
        .maximumSessions(1) // 최대 세션 1개
        .maxSessionsPreventsLogin(true) // 1개 이후에는 더이상 세션이 생기지 않도록 즉 2번째 요청부터 로그인을 막음 false일 땐 기존 사용자 로그아웃

    return http.orBuild 
    }
    
    //spring security에서 사용하는 passwordEncoder등록
    @Bean
    fun passwordEncoder(): PasswordEncoder {
        return BCryptPasswordEncoder(16)
    }
```
DefaultLogoutHandler 간단구현, 기본으로 사용할 logout handler 작성
```kotlin
class DefaultLogoutHandler : LogoutHandler {
    override fun logout(
        request: HttpServletRequest,
        response: HttpServletResponse,
        authentication: Authentication
    ) {
        //session 만료
        infoLog {
            "start logout handler"
        }
        request.session.let { it.invalidate() }
    }
}
```

DefaultLogoutSuccessHandler 간단구현, 로그아웃에 성공했을 때 동작하는 handler
```kotlin
class DefaultLogoutSuccessHandler: LogoutSuccessHandler {
    override fun onLogoutSuccess(
        request: HttpServletRequest,
        response: HttpServletResponse,
        authentication: Authentication
    ) {
        infoLog {
            "success log out"
        }
        response.sendRedirect("/login")
    }
}
```

이로써 기본 config 설정이 끝이났다 이제 UserDetails와 userDeatilsService를 구현해야한다

## domain 구성
먼저 UserDetails를 구현하기 위한 Member domain을 간단하게 만들어보자

entity member
```kotlin
@Entity
@Table(name = "member")
data class Member (
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var username: String? = null,
    var loginId: String? = null,
    var password: String? = null,
    var email: String? = null,
    var age: Int? = null
)
```
member repository
```kotlin
interface MemberRepository: JpaRepository<Member, Long> {
    @Query("select m from Member m where m.username =: username")
    fun findByUsername(@Param("username") username: String): Member?
}
```
## UserDetails
> UserDetilas란 Spring security에서 사용자의 정보를 담는 인터페이스이다.

```kotlin
class MemberDetails(
    private val username: String,
    private val password: String,
): UserDetails {
    
    companion object {
        fun from(member: Member): MemberDetails {
            return with(member) {
                MemberDetails(
                    username = username!!,
                    password = password!!
                )
            }
        }
    }
    
    //권한여부, 권한 여부는 사용하지 않을 것이기에 null을 반환한다.
    override fun getAuthorities(): MutableCollection<out GrantedAuthority> {
        return null
    }
    //사용자 패스워드
    override fun getPassword(): String {
        return password
    }
    //사용자 이름
    override fun getUsername(): String {
        return username
    }
    //계정의 만료여부, true(만료안됌)
    override fun isAccountNonExpired(): Boolean {
        return true
    }
    //계정의 잠금여부, true(잠금안됌)
    override fun isAccountNonLocked(): Boolean {
        return true
    }
    //비밀번호 만료여부, true(만료안됌)
    override fun isCredentialsNonExpired(): Boolean {
        return true
    }
    //계정의 활성화 여부, true(활성화됌)
    override fun isEnabled(): Boolean {
        return true
    }
}
```

## UserDetailsService
> userDetailsService란 Spring Security에서 유저의 정보를 가져오는 인터페이스이다.
> 해당 기능을 이용하기 위해서는 loadUserByUsername 메소드를 구현해야한다

```kotlin
@Service
class MemberDetailsService(
    private val memberRepository: MemberRepository
): UserDetailsService {
    override fun loadUserByUsername(username: String): UserDetails {
        return memberRepository.findByUsername(username)?.let {
            MemberDetails.from(it)
        } ?: throw UsernameNotFoundException("회원을 찾을 수 없습니다.")
    }
}
```

로그인 처리를 하기위한 user setting yml <br>
loginId = user, password = 0000 으로 설정하였다.
```yml
spring:
  security:
    user:
      name: 'user'
      password: '0000'
```

테스트용 controller
```kotlin
@RestController
class SecurityController() {
    @GetMapping("/")
    fun index(): String {
        return "home"
    }
}
```

## 마치며
이번 글에서는 spring security를 간단하게 사용해보았다. 다음 글에서는 spring security의 내부 동작방식을 하나씩 뜯어볼 것이다.








