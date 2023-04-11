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

로그인 처리를 하기위한 user setting yml <br>
loginId = user, password = 0000 으로 설정하였다.
```yml
spring:
  security:
    user:
      name: 'user'
      password: '0000'
```

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



