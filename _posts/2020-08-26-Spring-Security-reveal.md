---
layout: post
title: Spring Security 뜯어보기
date: 2021-01-26
summary: 프레임워크가 원하는대로 커스텀하기
categories: study java
---

[comment]: <> (Spring Security를 제대로 사용하기 위해서는 Spring Security가 기본적으로 제공하는 Authentication logic에 대해 이해할 필요가 있습니다.<br>)

[comment]: <> (디버깅을 통해 제어권의 흐름을 따라가며 이를 알아보겠습니다.)

# SpringSecurity의 역할

---

&nbsp; WAS<sup>[1](#footnote_1)</sup>내부의 웹서버와 SpringApplicationContext<sup>[2](#footnote_2)</sup>가 시작되는
DispatcherServlet<sup>[3](#footnote_3)</sup> 사이에는 ServletContainer가 관리하는 **ServletFilter**가 있습니다. **ServletFilter**는
여러가지 필터체인으로 구성되어 있는데, SpringSecurity 는 ServletFilter에 **SecurityFilterChain**을 자동으로 구성합니다.

&nbsp; **SecurityFilterChain**은 이름처럼 Request가 DispatcherServlet으로 가기 전에 연속된 필터들을 순서대로 통과하면서 여과되고 SpringApplication
*(우리가 작성하는 서버)*의 보안과 관련된 로직을 수행하도록 하는 역할을 합니다.

> **Filter와 Interceptor의 차이는 실행시점**<br> > **ServletFilter**는 SpringApplicationContext이전에 위치한다는 점에서, DispatcherServlet과 Controller사이의 처리를 담당하는 Interceptor와 차이가 있습니다.<br>
> 현재까지 개발을 하면서 느낀 가장 큰 차이점은 SpringApplicationContext 전에 일어나는 Exception에 대해서는 Spring이 알지 못하기 때문에,<br>
> Exception Handling을 Spring의 @ControllerAdvice나 @ExceptionHandling이 아닌 다른 인터페이스로 처리해야 한다는 점입니다.

<sub> <a name="footnote_1">[1]</a>: Web Application Server, 대표적으로 Tomcat 같은 프로덕트를 말함 </sub><br>
<sub> <a name="footnote_2">[2]</a>: 우리가 작성하는 서버가 시작되는 ApplicationContext</sub><br>
<sub> <a name="footnote_3">[3]</a>: SpringApplicationContext의 가장 앞에서 모든 Request를 받아서 라우팅을 담당하는 FrontController</sub>

<br>

# SpringSecurity FilterChain

---

**SecurityFilterChain**의 구성에 대해 알아보겠습니다.

하지만 이 글은 우리가 작성하는 서버의 요청에대한 인증/인가를 어떻게 진행할 것인가에 대한 글임으로, CSRF등의 방어를 위한 필터 등은 다음 기회에 다루고, 사용자인증을 구현하는 것과 관련된 필터를 중점적으로
보겠습니다.

## 1. SecurityContextPersistenceFilter

**SecurityContext**는 SpringSecurity에 필요한 것들을 포함하는 전역 객체입니다. 말그대로 Spring의 보안 문백, 바탕 따위를 의미합니다.

**SpringContextPersistenceFilter**는 새로운 요청에 대해 <u>새로운 SecurityContext를 가져와 초기화</u>하고 싱글톤 클래스인 SecurityContextHolder에 끼우는
역할을 합니다. 즉, <u>여러 필터들을 지나가기에 앞서 보안과 관련된 여러가지 멤버변수들을 관리할 도화지를 만드는 셈입니다.</u>

우리가 관심있는 사용자인증은 SecurityContext의 Authentication 객체로 표현되고, 이 객체의 흐름이 Spring Security의 요청에 대한 인증의 핵심입니다.

## 2. (UsernamePassword)AuthenticationFilter

> **AuthenticationFilter는 실질적인 인증과 관련된 로직이 위치한 필터입니다.**<br> > &nbsp; 따라서, 우리가 인증 기능을 우리가 개발하는 서비스에 맞게 커스텀하여 사용하기 위해서는 이 필터의 동작을 이해하고 있는 것이 좋기에, 조금 상세히 살펴보도록 하겠습니다.<br> > &nbsp; 어떤 인증 방식을 사용하냐에 따라 다른 필터를 구현하여 이 단계에 추가할 수 있으므로, 좀더 일반적인 의미를 위해서 AuthenticationFilter라고 지칭하겠습니다.

**2.1 AuthenticationFilter:: 로그인 요청 파라미터로부터 미인증 상태의 Authentication Token 생성**

자, 요청이 **UsernamePasswordAuthenticationFilter**에 도착하였습니다.<br>
**UsernamePasswordAuthenticationFilter**는 추상 클래스 **AbstractAuthenticationProcessingFilter**의 실제 인증을 진행하는 과정(attemptAuthentication)을 Override합니다.

SpringSecurity의 기본 인증 전략은 세션을 이용한 **FormLogin**입니다. **HttpSecurity**옵션에서 FormLogin()을 키는 것만으로(Override 하지 않을 시 기본값) 아이디, 비밀번호를
통한 로그인 기능과 페이지, 모든 리소스에 대한 인증요구를 제공합니다. FormLogin().loginPage() 메소드를 활용해 직접만든 로그인 페이지를 사용하도록 설정할 수 도있습니다.
UsernamePasswordAuthentication필터는 이 FormLogin에 대한 사용자 입력을 검증하는 역할을 합니다. 이 필터의 requestMatcher에 해당하는 url(기본값: "POST /login")로 요청이 들어오면 인증 로직을 실행합니다.

UsernamePasswordAuthenticationFilter가 상속하는 AbstractAuthenticationProcessingFilter는 위와 같이, 특정한 패턴의 request url에 대해서, 검증이 필요할 때 사용합니다.

- <u>만약 커스텀 로그인을 위해 Spring Security 필터를 사용할 경우, UsernamePasswordAuthenticationFilter 처럼 AbstractAuthenticationProcessingFilter를 상속하여
  로그인을 진행할 request matcher를 설정해 주는 생성자와 인증을 위한 attemptAuthentication 메소드를 구현하여야 합니다.</u><br>
- <u>만약 로그인을 위해서 별도의 API를 사용하고, 요청검증에 Spring Security를 사용하는 경우,
  formLogin.disabled()로 UsernamePasswordAuthenticationFilter 를 사용하지 않도록 설정한 후, 로그인된 유저를 판별하기 위해(JWT 토큰 검증 등)
  AbstractAuthenticationProcessingFilter의 역할을 하는 별도의 필터를 추가해야합니다(이 때 추가되는 필터는 AbstractAuthenticationProcessingFilter를 상속하지 않음)</u>

![Untitled](/images/posting/0001.png)_<sup>SpringSecurity에 대한 설정시 상속하는 WebSecurityConfigurerAdapter.configure, 상속하여
Override하지 않으면 위 내용이 기본값으로 설정된다.</sup>_

SpringSecurity의 기본 인증전략이 FormLogin이기 때문에, 기본으로 SecurityFilterChain에는 **AbstractAuthenticationProcessingFilter**를 상속한
**UsernamePasswordAuthenticationFilter**가 포함되어있습니다. (FormLogin.disabled()시에는 UsernamePasswordAuthenticationFilter가 Security Filter Chain에서 제외 됨)

**UsernamePasswordAuthenticationFilter**는 요청 객체에서 필요한 검증 대상을 추출합니다. UsernamePasswordAuthenticationFilter에서는 설정된 파라미터이름으로 username과 password를 추출합니다.

![Untitled](/images/posting/0002.png)_<sup>UsernamePasswordAuthenticationFilter class의 속성과 기본값들,
SPRING_SECURITY_FORM_USERNAME_KEY와 SPRING_SECURITY_FORM_PASSWORD_KEY는 formLogin().usernameParameter(), formLogin()
.passwordParameter()로 설정가능</sup>_

**추출한 파라미터들로 인증되지 않은 상태의 UsernamePasswordAuthenticationToken을 생성합니다.**<br> AuthenticationFilter의 역할은 여기까지입니다. AuthenticationFilter는
인증과 관련된 실질적인 프로세스는 모두 AuthenticationManager에 **미인증 상태의 토큰을 던져주며 위임합니다.**

![Untitled](/images/posting/0003.png)_<sup>request로부터 username과 password 추출, 미인증 상태의 UsernamePasswordAuthenticationToken을 객체를
생성해서 AuthenticationManager에게 위임한다</sup>_

여기서 UsernameAuthenticationToken은 AbstractAuthenticationToken을 상속합니다. 또한 AbstractAuthenticationToken은 Authentication을
구현합니다. 즉, **앞서 말한 SecurityContext가 다루는 핵심 멤버가 바로 이 AbstractAuthenticationToken(이하 AuthToken)입니다.**

![Untitled](/images/posting/0004.png)_<sup>UsernamePasswordAuthenticationToken의 생성자, 객체 생성 시 authorities를 포함하면 인증성공 상태의 토큰을
생성, authorities 가 포함되지 않으면 미 인증 상태의 토큰을 생성</sup>_

**2.2 AuthenticationManager(ProviderManger):: 전달받은 AuthToken을 처리할 수 있는 provider를 찾음**

ProviderManager는 AuthenticationManager의 구현체입니다. 앞서 말햇듯, AuthToken을 이용한 인증을 위임받습니다. ProviderManager는
AuthenticationProvider의 리스트를 가지고 있습니다.

![Untitled](/images/posting/0005.png)_<sup>AuthenticationManager 인터페이스의 구현체 ProviderManager. provider의 리스트를 가지고 있음</sup>_

AuthenticationProvider 인터페이스는 Authentication 객체(AuthToken)을 받아서 자신이 처리할 수 있는 인증인지를 리턴하는 'boolean supports(Class<?>
authentication)' 메소드를 가지고 있습니다. 이를 이용하여 **ProviderManager는 providers 리스트를 순회하며** **전달받은 AuthToken을 처리할 수 있는 provider를
찾아서 인증을 수행합니다.**

![Untitled](/images/posting/0006.png)_<sup>providers를 for 문으로 순회하며 전달받은 AuthToken의 클래스를 supports하는 provider를 찾아서 인증을 수행한다</sup>_

**2.3 AuthenticationProvider :: AuthToken에 포함된 인증대상들에게 인증 진행. 검증 성공시 인증 완료된 AuthToken 리턴해줌**

![Untitled](/images/posting/0007.png)_<sup>AuthenticationProvider Interface. 실제로 인증을 수행하는 authenticate 메소드, 처리가능한 AuthToken을
구별하는 supports 메소드가 있다</sup>_

**AuthenticationProvider.authenticate(Authentication authToken)**은 입력으로 들어온 AuthToken을 자신만의 방식으로(FormLogin - DB비교, JWT -
parsing signiture) 검증하여 성공할 시 인증성공된(Authentication.isAuthenticated == true) 토큰을 리턴해야합니다. 실패할 시 AuthenticationException을
던집니다.

아무런 추가 구현을 해주지 않고 Spring Security가 활성화만 된 상태에서는 기본적으로 DaoAuthenticationProvider라는 구현체가 Provider로 사용됩니다. 이 Provider는
UserDetails와 UserDetailsService라는 인터페이스를 이용해 AuthToken 안의 검증 대상(username, password)과 DB의 데이터를 비교합니다. 우리가 임의의 인증 전략을
사용한다면, AuthenticationProvider의 구현체를 작성하여 요청에서 뽑아온 정보가 들어있는 AuthToken을 우리만의 방식으로 검증하도록 authenticate 메소드와 support 메소드를 Override하여야 합니다.
(\* SpringSecurity 는 Security context를 초기화 하는 과정에서 bean 에 등록된 AuthenticationProvider를 모두 ProviderManager(AuthenticationManager)에 등록합니다. 따라서, AuthenticationProvider를 Bean에 등록해주는 것만으로 커스텀 인증 로직을 사용할 수 있습니다.)

SpringSecurity를 만든사람은 이 프레임워크가 어떤 비즈니스에 쓰일지 알 방법이 없었기에 UserDetails과 UserDetailsService라는 일반적인 유저 인터페이스을 만들었을 것입니다.
FormLogin 방식의 AuthenticationProvider의 구현체인 DaoAuthenticationProvider는 이 UserDetails와 UserDetailsService 인터페이스들을 이용하여
DB에서 유저데이터를 추출하여 UsernamePasswordAuthenticationToken의 내용과 비교합니다. 따라서, Spring Security의 FormLogin방식을 사용할 시 UserDetails와
UserDetailsService의 구현체를 우리가 직접 구현해야 합니다.

\*\* 2.4 AbstractAuthenticationProcessingFilter.unsuccessfulAuthentication & successfulAuthentication

이후의 로직은 AbstractAuthenticationProcessingFilter에서 이루어집니다. 이 메소드에서는 앞서 인증을 위임받은 AuthenticationManager 가 인증을 성공 했을 경우
SecurityContext의 authentication 필드에 인증 완료 상태의 토큰을 세팅합니다. 만약 인증에 실패했다면, SecurityContext.authentication 은 null인 상태로 다음 필터로 향하게 됩니다.

## 3. **AnonymousAuthenticationFilter**

요청이 이 필터에 도착할 때 까지,SecurityContext의 authentication (authToken) 이 null 이라면 AnonymousAuthenticationToken을 넣습니다.

## 4. ExceptionTranslationFilter

앞선 필터들에서 throwing 된 AthenticationException, AccessDeniedException 을 핸들링 합니다.

## 5. FilterSecurityInterceptor

인가를 결정하는 AccessDecisionManager 에게 접근 권한이 있는지 확인하고 처리하는 필터.

-

Spring Security는 configure 하면서 빈으로 등록된 AuthenticationProvider 와 GenericFilterBean 들을 자동으로 로드한다.<br>
읽어온 AuthenticationProvider 는 Default AuthenticationManager 인 ProviderManager를 초기화 하는데 사용 되고, <br>
읽어온 GenericFilterBean 은 SecurityFilterChain이 아닌, ServletFilter의 가장 마지막에 추가한다.
