<a href="./">목록 보기</a>

<h1 style="text-align: center; ">Authentication & Authorization</h1>
<p style="text-align: right;"> updated 2020.08 </p>

# JWT 토큰 방식 인증, JS/TS로는 해봤었는데...

---

&nbsp;Express.js에서 인증 미들웨어를 직접 구현하여 **JWT를 이용한 AccessToken & RefreshToken 인증 전략** 을 사용한 적이 있어서, 우선 같은 방식을 Java Spring
환경에서 시도해 보고싶었다.<br>

&nbsp;하지만, JS진영에서는 인증용 미들웨어의 적용이 자유롭고 정해진 틀이 없어서 유연하게 구현할 수 있었던 반면에, 경험이 적은 Java Spring에는 쉽게 인증 코드를 추가할 수가 없었다.

JWT, AccessToken, RefreshToken 등에 대한 개념은 이미 잘 알고있었지만, Spring 프레임워크의 방법으로 인증 전략을 적용하기 위해서는 **Spring Security에 대한 이해를 모르고는
접근하기가 힘들었다**.

# 내가 하고있었던 오해들, 겪은 문제들

--- 

&nbsp;Nest.js를 운영해본 당시 Spring의 **Filter**와 **Interceptor**에 대해서 얕게나마 조사해본 적이 있었기에, 요청이 Application Context에 들어오기전에
Filter를 통과하는 것을 알고있었다.<br>
&nbsp;**'미들웨어 추가하는것 처럼 JWT토큰을 파싱해서 유저ID, RefreshToken에 대한 해쉬 값 등의 정보를 추출해내는 필터를 하나 추가하면 되겠지'** 라고 쉽게 생각했다.

하지만, Spring Security 는 자체적인 인증/인가 솔루션인 **AuthenticationFilter를** 가지고 있었고, **기본 전략으로 토큰이 아닌 Username & Password, Session
을 사용하고 있었다.** 나는 아래 두 가지를 수행해야 했는데 두 가지 모두가 잘 되지 않았다.

- UsernamePasswordAuthenticationFilter를 대체할 JwtAutheticationFilter를 추가해야함<br>
  &nbsp;&nbsp;→ 추가에는 성공했으나, JwtAuthFilter 이후의 필터들에서 AuthenticationException 이 throwing 됨.
  UsernamepasswordAuthenticationFilter도 꺼졌는지 모르겠음.

- 특정한 URL(회원가입, 중복확인, 로그인 등)에 대해서는 JwtAuthenticationFilter 를 통과하지 않도록 해야함<br>
  &nbsp;&nbsp;→ 특정한 필터에 대해서만 pass하도록 하는 설정 인터페이스를 찾을 수 없었음, HttpSecurity에 permitAll을 해줬는데도 불구하고 필터에 자꾸 걸림.

결국엔 기존에 있던 지식과 동치되는 것들을 Spring 에서 찾는 것을 포기하고 Spring Security의 로직을 이해하기 위해서 레퍼런스를 찾고 디버깅을 하며 점차 감을 잡아갔다.

# Spring Security에 대해 오해하고 있었던 것

---

기존 JS진영에서 생각하던 틀을 버리고 SpringSecurity가 포함하고 있는 FilterChain 가지고 여러 실험을 하며 공부하다보니 Spring Security의 철학, 접근방법, 기본원리 같은 것들이 좀
보이기 시작했다.

1. **Spring Security의 FilterChain 에 속한 필터들은 마음대로 빼거나 대체하거나 부분적으로 적용할 수 있는 것이 아니었다.**

   &nbsp;Spring Security는 이미 그 자체로 하나의 인증서비스를 제공하기 때문에, 임의로 필터를 제거하거나 빼는 인터페이스는 제공되지 않고, 추가할 수만 있다. 사용자는
   WebSecurityConfigurerAdapter 상속으로 그 필터들과 상세내용들의 적용여부와 옵션을 컨트롤 한다. 만약, 어떤 Filter가 실행되지 않기를 원한다면, 그 Filter가 하는 역할을 잘
   파악하여서 Pass되도록 해주는 추가 커스텀 필터를 추가하는 방식으로 우회가 가능할 것이다. 따라서 나는 **JWT 인증로직을 Spring Security의 의도를 해치지 않고 통합하기 위해 최대한 Spring
   Security를 이해하고 파악해야했다.**

   나는 커스텀으로 JwtAuthenticationFilter를 만들어서 UsernamePasswordAuthenticationFilter 대신에 사용하고 싶었는데.
   UsernamePasswordAuthenticationFilter는 formLogin()을 diabled 시키면 필터체인에 등록되지 않는 것을 확인하였다.

2. **JWT인증을 꼭 Spring Security에 통합 시킬 필요는 없는 것이었다.**

   &nbsp;Spring Security 에 모든 것을 의존하여 JWT 인증 방식을 적용하려고 하니, 이미 Spring Security에 정해져 있는 인터페이스 등이 마음에 들지 않았다.

   &nbsp;예를 들어, Persistance layer에서 사용하고 있는 Entity의 인터페이스와는 별도의 유저 정보 클래스 UserDetails, UserDetailService 를 구현 해야 한다는 것과
   loadByUsername, UsernamePasswordAuthenticationToken 등 JWT 특성상 Username과 Password를 인증여부 검증에 사용하지도 않는데 그러한 이름을 사용하는 것들이
   그랬다. (심지어 이 서비스에는 email과 password 로 로그인을 기획햇다.)

   &nbsp;물론 Spring Security를 설계할 당시에 모든 비즈니스 로직을 고려할 수는 없으니 하나의 인터페이스 기준이 필요했던 것은 알 수 있지만, 시간이나 자원에 쫒기고 있지 않은 지금 같은
   상황에서는 읽기 쉽고 말이 되는 코드가 더 중요했다.

   &nbsp;Spring Security 가 구현하고 있는 기능은 로그인(사용자 입력검증), 로그인 세션2관리, 요청 인증여부 검증, 요청의 인가검증이다. 이 모든 기능들을 전부 Spring Security에
   의존할 수 있겠지만, 지금의 나와 같이 기본으로 제공하는 formLogin 외에 다른 방식의 인증을 구현하고 싶은 경우 **요청의 인증/인가 검증 등의 로직은 Spring Security의 FilterChain
   을 커스텀하여 위임하고, 로그인 인증과 토큰 갱신 따위의 것들은 별도의 컨트롤러를 만드는 것도 좋다고 생각한다.**

3. **HttpSecurity의 permitAll은 권한의 인가이다.**

   &nbsp;Spring Security Configuration에는 WebSecurity와 HttpSecurity가 있다. 이 설정에서 resource에 대한 접근에 따라 Security Filter의
   적용여부를 설정 할 수 있다.<br>
   &nbsp;**WebSecurity의 ignoring은 request요청이 왔을 때 필터들을 SecurityFilterChain에 등록 시킬지 말지를 결정한다.** 만약 ignoring된 경로로 요청이
   들어온다면, SecurityFilterChain은 빈 필터가 되어 요청에 대한 어떤 검증도 수행되지 않는다.<br>
   &nbsp;HttpSecurity의 permitAll은 SecurityFilterChain은 SecurityFilterChain을 정상적으로 등록하지만, 권한을 검사하는 필터에서 적용되어

   &nbsp;JWT 커스텀필터를 구현하여 UsernamePasswordAuthenticationFilter앞에 위치시켰지만, 해당 위치는 권한을 검사하는 필터 이전이기 때문에, 어떤 요청에든 실행되었다.<br>
   &nbsp;실제로 FilterSecurityInterceptor의 AccessDecisionManager에서 권한을 검사할때 permitAll이라면 그냥 통과 시키는 것을 확인 할 수 있다.

4. **Spring은 Bean으로 등록된 필터들을 자동으로 Servlet 필터에 추가한다.**

   &nbsp;하지만 그렇게 로그인, 회원가입 등 인증이 필요하지 않은 URL로 들어오는 요청을 WebSecurityConfigurer에서 ignoring 하였지만, 여전히 요청은 내가 작성한 Custom
   Filter를 작동시켰다. WebSecurity 설정과 HttpSecurity설정을 바꿔보고 디버깅 모드로 SecurityFilterChain자체가 비어 있는 것을 확인 했음에도 요청은 내 Custom
   Filter를 작동시키는 것이 이해가 가지 않았다.

   &nbsp;아무리 WebSecurityConfig에서 ignoring 해도 필터가 동작하는 이유가 여기에 있었다. **Spring은 개발자 편의를 위해서 Bean으로 등록된 모든 Filter를
   ServletFilterChain에 등록시킨다.**<br>

   &nbsp;Spring Web Security 디버깅에서는은 SecurityFilterChain만을 보이기 때문에 SecurityFilterChain 이후의 ServletFilter에 등록된 커스텀 필터가
   잡히지 않는 것이었다.

# 실제로 해보기

---

&nbsp;SpringSecurity에 JWT Authentication 방식을 결합하며 [공부한 내용들을 정리하였다](/posting/Spring_Security의_Authentication에_대한_이해). 그리고
이를 진행중인 프로젝트에 적용하여 사용해 보기로 했다.

<sub><a href="https://github.com/jinkyuhan/fakebook/blob/main/fakebook-server/src/main/java/com/jkhan/fakebookserver/auth/JwtAuthenticationFilter.java">
Github에서 코드 참고 </a></sub>

### 1. 로그인

&nbsp;로그인을 구현하기 위해 Spring Security의 AuthenticationManager를 사용 할 수도 있겠지만, Spring Security에서 미리 정해놓은 UserDetails,
loadByUsername 등의 이름이 내가 구현하려는 JWT 토큰 방식의 인증과 동 떨어져 있었다.<br>
&nbsp;따라서, 사용자의 입력을 받아 JWT 토큰과 RefreshToken을 담당해줄 컨트롤러를 Spring Security를 벗어나서 직접 구현하기로 했다.<br>
&nbsp;요청의 사용자 입력과 DB의 유저정보(email, password)를 비교한뒤, 통과하면 JWT을 생성하여 응답한다. 총 아래와 같이 2개의 토큰을 생산하도록 구현했는데, 첫번째는 사용자의 접근 권한을
인증할 AccessToken 역할로써의 토큰 한개와, 두번째는 AccessToken의 만료기한이 다했을 때, 새로운 AccessToken을 갱신할 수 있는 RefreshToken이다.<br>

```json
// Payload of AccessToken
{
  "iat": UnixTimestamp,
  "exp": UnixTimestamp,
  "userId": String
}

// Payload of RefreshToken
{
  "iat": UnixTimestamp,
  "exp": UnixTimestamp,
  "userId": String,
  "loginSessionId": String
}

```

&nbsp;AccesToken의 payload에는 userId를 넣어 요청의 주인을 구별할 수 있도록 했고, RefreshToken에는 userId와 unique한 uuid한개를
loginSessionId<sup>[1](#footnote_1)</sup>라는 이름으로 넣어 생성했다.<br>

&nbsp;Session ID는 DB나 Redis에 영속적으로 저장하고 있다가, RefreshToken을 요구하는 요청의 검증단계에서 사용하거나 특정 유저의 로그인을 강제로 만료시키는 수단으로 사용하기로 했다.

<sub><a name="footnote_1">1</a>: 로그인 상태의 연결을 의미하는 개념적인 워딩으로 사용, 서버리소스 Session 아님, JWT에서는 서버리소스를 사용하는 세션인증 방식을 사용하지
않는다.</sub><br>

### 2. 인증 검증

&nbsp;앞서 공부한 바를 바탕으로 인증을 검증하는 로직을 구현하기 위해서는 **AbstractAuthenticationProcessingFilter**를 상속하여
**UsernamePasswordAuthenticationFilter**를 대체할 필터를 작성해야햇다.<br>

&nbsp;하지만 Formlogin 방식처럼 로그인을 특정 라우터로 설정하는 내용의 구현이 강제되있어서 그냥 같은 역할을 하는 **JwtAuthenticationFilter를** 구현하여 대체 하기로 했다.<br>

&nbsp;**JwtAuthenticationFilter**에서는 들어오는 요청의 'Authorization' 헤더의 Bearer 토큰을 Credential로 삼아, 미인증 상태의 **
JwtAuthorizationToken**을 생성하여 **JwtProvider**에게 인증을 위임한다.<br>

&nbsp;**JwtProvider**에서는 Bearer토큰을 parsing 하여 유효한 JWT 토큰인지를 검사한다. 그리고 파싱된 payload안의 userId를 Principal로 삼고 새로운 인증성공
상태의 **JwtAuthorizationToken**을 생성한다. **<u>이 때, payload 안에 loginSessionId가 있다면 JwtAuthorizationToken의 detail로 삼는다.</u>**

### 3. 인증 갱신

&nbsp;사용자가 로그인한 상태를 오랫동안 유지하면서도 JWT토큰을 짧은 시간으로 설정하여 안정성을 확보하기 위해서 RefreshToken방식을 사용한다. 이를 위해, &nbsp;RefreshToken으로 새로운
JWT토큰을 재발급받는 컨트롤러를 추가 했다.<br>

&nbsp;**클라이언트가 Authorization에 만료된 AccessToken 대신에 RefreshToken을 넣어 해당 라우터로 요청을 보낸다면, <u>SecurityFilter를 통과한 Authentication
객체의 detail에 loginSessionId가 있을 것이다.</u>**<br>

&nbsp;해당 loginSessionId로 DB에서 요청 유저의 로그인 만료시간을 연장하도록 업데이트 해주고, 로그인과 마찬가지로 만료시간이 갱신된 새로운 AccessToken과 RefreshToken을 생성하여
응답한다.

### 4.로그아웃

&nbsp;AccessToken만을 사용한다면 클라이언트 측에서 AccessToken을 폐기하는 것만으로 로그아웃이 될테지만, 나는 RefreshToken과 대응되는 loginSessionId를 관리하고 있으므로,
RefreshToken으로 요청을 받아서 DB에서 해당 loginSessionId를 지워줘야만 서버에서도 사용자의 로그인이 종료된 것으로 처리 할 수 있다.

<br>


# 마치며

---

&nbsp;이번 프로젝트에 JWT인증을 적용하면서 얻게된 바는 Spring Security가 어떤식으로 동작하는지 깊게 공부할 기회가 됬다는 것이다. 가장 익숙하고 많이 해본 인증 방식이라서 일단은 JWT로 구현했지만, 이후에 Oauth 2.0을 Spring Seucurity에 연동하는 것도 도전해볼 생각이다.


   
    
