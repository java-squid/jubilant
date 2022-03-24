# 커스텀필터 공부하려던 배경

## 시나리오

![](https://user-images.githubusercontent.com/59721293/156748328-cd41da03-45bb-4ac1-bd91-fca8d6fb05e4.gif)

- 위 gif 이미지 처럼 oauth로 로그인을 하면 바로 메인서비스로 들어가는 게 아니라 추가정보를 입력하는 폼이 나오도록하고 제출하도록 하려고 함

- 추가정보 폼 페이지에서 나가면 세션이 제거됨, 즉 이 상태에서 메인서비스로 접근하려고하면 접근이 안된다

- 추가정보 폼을 제출하면 메인서비스로 이동됨, 이때 디비에 유저정보가 쌓인다, 다음에 로그인할땐 해당 유저를 찾아서 ROLE_USER를 부여해준다

## 시나리오 디테일

![](https://user-images.githubusercontent.com/59721293/148496542-13738ce0-ca70-4e8a-be55-263c32f0ad83.png)
사진: https://blog.naver.com/mds_datasecurity/222182943542

- 위 OAuth 흐름 이미지로 설명하자면 마지막에 Access Token을 전달받고 그걸로 자원을 요청하면 유저에게 응답하지 않고, 그 자원을 세션에 저장시키고 추가정보 폼 페이지로 이동시킬려고함, 즉 리소스오너로부터 자원을 얻는 모든 프로세스는 끝났음

- 코드에서 어떻게 해야할까? 먼저 시큐리티 없이한다면 어느부분에서 내가 커스텀을 해야하는지 확인하고 싶었음

```java
    // 시큐리티 없이 OAuth 구현
    @GetMapping("/api/auth/callback")
    public String processOAuth(@RequestParam String code, HttpSession session) {
        RestTemplate restTemplate = new RestTemplate();
        accessTokenUri = oauthUtil.getUriForAccesToken(code);
        clientId = oauthUtil.getClientId();
        secretId = oauthUtil.getClientSecret();
        redirectUri = oauthUtil.getRedirectUri();

        // 1. AccessToken을 얻기 위한 RequestEntity를 생성한다.
        RequestEntity<GithubAccessTokenRequestDto> requestDto = RequestEntity
                .post(accessTokenUri)
                .header("Accept", "application/json")
                .body(new GithubAccessTokenRequestDto(
                        clientId, secretId, code, redirectUri
                ));


        // 2. 생성된 RequestEntity를 AccessToken을 요청한다.
        ResponseEntity<GithubAccessTokenResponseDto> responseDto = restTemplate.exchange(requestDto, GithubAccessTokenResponseDto.class);

        // 3. 보호된 자원 (User) 를 얻기 위한 RequestEntity 생성.
        RequestEntity<Void> request = RequestEntity
                .get(oauthUtil.getUserinfoUri())
                .header("Accept", "application/json")
                .header("Authorization", "token " + responseDto.getBody().getAccessToken())
                .build();

        // 4. User를 얻는다.
        ResponseEntity<User> user = restTemplate.exchange(request, User.class);

        // 5. (!!) 여기서 원하는 정보를 얻었으므로 세션에 user를 저장하고 추가정보 폼으로 이동한다
        user.setRole("ROLE_메인서비스접근impossible유저");
        session.setAttribute("user", user);
        
        // 6. 폼 페이지로 이동
        return "/additionalInfo"; 
    }
    
    // 추가정보 폼에서 요청하는 API
    @PostMapping("/api/user/save")
    public String saveUser(@RequestBody SignupInfoDto dto, HttpSession session) {
        
        // 7. 유저가 입력한 폼과 세션에 저장한 유저정보를 불러와 합친다.
        User user = (User) session.getAttribute("user");
        user.setAdditionalinfo(dto);
        user.setRole("ROLE_메인서비스접근possible유저");
    
        // 8. 메인서비스로 바로 접근하기위해 세션에 다시 User를 저장해준다.
        user.setAttribute("user", user);
            
        // 9. 저장한다.
        userRepository.save(user);
        
        
        return "/main";
    }
```


- 커스텀하고 싶었던 부분은 6번부터 9번까지의 과정이었다. 아래처럼 SecurityConfig를 설정해주는 과정에서 중간에서 제어가 가능하다고 생각했다

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // ...
        // ...
        // ...
            .and()
            .oauth2Login().loginPage("/login")
            .defaultSuccessUrl("/main")
            .userInfoEndpoint()           
            .userService(customOAuth2UserService);
    }
```

- 그런데 만약에 위 시나리오대로 한다면 `http.addFilterAter(커스텀필터, OAuth마지막필터.class);` 이런식으로 해야할거같은데, 해당 부분에서 어려움을 느껴 아래와 같이 했다.

- 이 방법이 더 단순하고 좋아보였음, 애초에 커스텀이 필요없던거였던거 같기도 하다.
    - 그냥 유저에게 로그인 성공후 페이지 (추가정보 입력페이지)를 뿌려준다.
    - 대신에 메인서비스로 접근할 수 없는 상태를 가진다.
    - 해당 유저는 추가정보 입력폼을 제출해야만이 메인서비스로 접근이 가능하다.
- 스프링 시큐리티는 여러 필터를 거치고 세션의 SPRING_SECURITY_CONTEXT (Authentication 객체) 의 유저의 ROLE을 보고 메인서비스로의 접근 인가를 한다
- 그렇기 때문에 메인서비스로의 접근은 기본적으로 `ROLE_USER`로 permit하도록 해놓고 처음 로그인만 한 유저는 그외 다른 ROLE (여기서는 ROLE_GUEST) 로 세션에 설정을 해놓는다.

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception { // 5
        http
                // ...
                // ...
                .authorizeRequests() // 
                .antMatchers("/login/**", "/signup/**").permitAll()
                .antMatchers("/mainService/**", "/").hasRole("USER") // ROLE_USER만 접근가능
                // ...
                // ...
    }
```

- 유저는 추가정보폼 제출후 디비에 저장이 되면서 회원가입이 완료된다. 이때 ROLE_GUEST였던 세션에 있는 SPRING_SECURITY_CONTEXT를 ROLE_USER 로 변경한다.

```java
@RequiredArgsConstructor
@Service
public class CustomOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {
    private final MemberDao memberDao;
    private final HttpSession httpSession;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2UserService delegate = new DefaultOAuth2UserService();
        OAuth2User oAuth2User = delegate.loadUser(userRequest);

        String registrationId = userRequest.getClientRegistration().getRegistrationId();
        String userNameAttributeName = userRequest.getClientRegistration()
                .getProviderDetails().getUserInfoEndpoint().getUserNameAttributeName();

        // OAuthAttributes에서 여러 Resource Owner들로부터 들어온 유저정보를 일률적으로 담는다
        OAuthAttributes attributes = OAuthAttributes. 
                of(registrationId, userNameAttributeName, oAuth2User.getAttributes());

        
        // 해당 정보로 유저가 있으면 꺼내고 없으면, 기본적인 정보만 가지고있는 member 생성
        // 여기서 권한이 필터링된다
        Member member = findMember(attributes);
        httpSession.setAttribute("member", member);

        // 아직 이부분을 자세히 못봐서 정확하진 않지만, 경험적으로 결국에 Authentication을 리턴하는 부분인것같다.
        // 즉 member.getRole()이 Authentication으로 들어간다.
        return new DefaultOAuth2User(
                Collections.singleton(new SimpleGrantedAuthority(member.getRole())
                ), attributes.getAttributes(), attributes.getNameAttributeKey());
    }

    private Member findMember(OAuthAttributes attributes) {
        Member member = memberDao.findByEmail(attributes.getEmail())
                .filter(m -> memberDao.checkIfMemberStatusIsNormal(m.getIdx()))
                // 로그인할때 멤버가있으면(즉 회원가입상태이면) ROLE_USER로 업데이트
                .map(m -> m.updateRole("ROLE_USER"))
                // 멤버가 없으면 기본정보만 담은 Member를 리턴해주고 나중에 폼제출하는 서비스에서 회원가입이 됨 (디비)
                .orElse(attributes.toEntity()); 

        return member;
    }
}
```
- ROLE_USER를 가진 유저는 그 다음부터 로그인할때 세션에 저장되어있는(필요하면 디비에 저장하고) ROLE_USER를 가지고 오기때문에 컨트롤러에서 해당유저에 대하여 메인서비스로 forward 해준다

```java
    // OAuth 로그인시에 유저가 회원가입이 된지 안된지
    @GetMapping("/signup")
    public String divergingPoint(@RequestParam String inPath, Model model) {
        Member member = (Member) httpSession.getAttribute("member");
        // 이미 회원가입된 회원이면 ROLE_USER이면 메인서비스로
        if (member != null && member.getRole().equals("ROLE_USER")) {
            return "forward:/메인서비스";
        }

        // 일반 회원가입인지 oauth유저의 회원가입인지 구분하기 위한
        if (inPath != null) {
            model.addAttribute("inPath", inPath);
        } else {
            model.addAttribute("inPath", "oauth");
        }

        return "/signup/form";
    }
```
# 커스텀 필터 구현하기

## 커스텀 필터 신경써야 하는 부분 세가지

1. AuthenticationFilter
2. SecurityContextHolder.getContext() -> Authentication
3. AuthenticationProviders 

## 필터 위치 정하기

```java
@EnableWebSecurity
@RequiredArgsConstructor
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

	// 내가 구현하고 싶은 필터
    private final HeaderTestKeyCheckFilter headerTestKeyCheckFilter;

    // 필터에서 setAuthentication()에 넣었던 Authentication을 인증한다.
    private final TokenAuthenticationProvider tokenAuthenticationProvider;

    private final HeaderTestKeyCheckFilter headerTestKeyCheckFilter;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.
        authorizeRequests()
        .antMatchers("/login/**").permitAll()
        .anyRequest().authenticated();

		// BasicAuthenticationFilter 앞에 headCheckFilter를 실행한다는 의미
        http.addFilterBefore(headerTestKeyCheckFilter, BasicAuthenticationFilter.class);

                // Authentication 을 authenticate한다....?
        http.authenticationProvider(tokenAuthenticationProvider);
    }
}
```

```java
@Component
public class HeaderTestKeyCheckFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        String value = getValueFromTestKey(request).orElseThrow(NoAccessRightException::new); // extends AuthenticationException
        CustomAuthToken authenticatedToken = new CustomAuthToken(value);
        SecurityContextHolder.getContext().setAuthentication(authenticatedToken);

        filterChain.doFilter(request, response);
    }

    private Optional<String> getValueFromTestKey(HttpServletRequest request) {
        return Optional.ofNullable(request.getHeader("testKey"));
    }
}
```

- NoAccessRightException 에서 인증이 안되도록 한다
- 인증실패했을 경우의 Exception을 extend 할것이다
- `SecurityContextHolder.getContext().setAuthentication(authenticatedToken);`에서 Authentication을 설정해준다.

```java
public class CustomAuthToken extends AbstractAuthenticationToken {

    private final String principal;
    private Object credentials;

    public CustomAuthToken(String principal) {
        super(null);
        this.principal = principal;
        this.credentials = null;
    }

    @Override
    public Object getCredentials() {
        return credentials;
    }

    @Override
    public Object getPrincipal() {
        return principal;
    }
}
```
- AbstractAuthenticationToken 을 extend 하면 Authentication 을 아래와 같이 구현할수있다.
- credentials 는 principal 자체를 credentials로 사용하고 있기때문에 null로 준다.

![image](https://user-images.githubusercontent.com/59721293/157627874-4cae20b1-6afe-4ca6-aa9b-de3eaf759331.png)


```java
@Component
public class TokenAuthenticationProvider implements AuthenticationProvider {

    @Override
    public Authentication authenticate(Authentication authentication)
            throws AuthenticationException {

        CustomAuthToken authenticatedToken = (CustomAuthToken) authentication;
        String authNeeded = String.valueOf(authenticatedToken.getPrincipal());
        // 이 principal이 유효한 토큰인지 검증하는게 오겠지
        // 예를 들어 아이디,비번 로그인 유저라면 실제 인증하는 로직을 아래에 구현하고
        // 각종 처리를 구현
        // 비번이 일치하는지
        // 아이디로 회원을 조회 했을 때 존재하는 회원인지
        // 기타 등등과 적절한 예외 처리
        // 해당 객체를 true 라고 하면 인증이 된다.
        authenticatedToken.setAuthenticated(true);

        return authenticatedToken;
    }


    @Override
    public boolean supports(Class<?> authentication) {
        // 인증객체에서 선언한 객체는 reflection 을 통해서 해당 객체인지 파악하는데 이용된다.
        return CustomAuthToken.class.isAssignableFrom(authentication);
    }
}

```

- (아마도) setAuthentication 할때 그게 인증된 것이 아니라, 즉 마지막단계인 SPRING_SECURITY_CONTEXT 라는 이름으로 세션을 남기는게 아니라 일단 setAuthentication하고 나중에 이 authentication이 authenticated 되어야 인증이 끝나는것 같다

## 테스트 해보기

- BasicAuthenticationFilter 전에 커스텀필터를 해놓았고, DefaultLoginPageAuthenticationFilter 다음이기 때문에 해당 필터에 걸리면 `"/"`에 접속시 default login page가 나타나는 걸 볼수있음
![2022-03-11_00-00-34](https://user-images.githubusercontent.com/59721293/157689574-cc07822f-72c4-4adc-850f-494b7b4530a7.jpg)
- 헤더에 testKey 를 넣어보면 정상 접속됨

![2022-03-10_23-56-46](https://user-images.githubusercontent.com/59721293/157689655-6ca9c5dc-dfef-4579-86da-8e4bf6f2b885.jpg)

