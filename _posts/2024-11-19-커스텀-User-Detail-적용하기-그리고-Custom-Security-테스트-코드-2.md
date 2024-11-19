---
title: "커스텀 User Detail 적용하기, 그리고 Custom Security 테스트 코드 #2"
date: 2024-11-19 23:10:00 +0900
categories: ["Spring Boot"]
tags: [Spring Boot, JWT, Spring Security]
---

[저번](/spring%20boot/JWT-토큰으로-Sprin-Security-적용하기-1/) 포스팅에서 JWT의 인증 처리와 Spring Security Context에 인증 정보를 저장하여 `@AuthenticationPrinciple`을 Parameter로 주입받아 사용하는 것 까지 해보았다. 이번에는 커스텀한 Principle을 사용해 나만의 UserDetail을 주입받아보자.

먼저, UserDetails를 구현하는 구현체를 정의해야한다. 나머지 UserDetails의 추상 메소드들은 default 메소드로 정의되어 있기 때문에 간략한 포스팅을 위해 생략하겠다.

```java
@Getter
@AllArgsConstructor
public class ShopMemberDetail implements UserDetails {
    private String loginId;     // 유저가 로그인 시에 사용하는 ID
    private String role;        // DB에서 가져온 해당 유저의 Authority
    private Long customerId;    // DB에서 특정하기 위한 Primary Key

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return Collections.singletonList(new SimpleGrantedAuthority(role));
    }

    @Override
    public String getPassword() {
        // jwt 토큰에는 없음.
        return null;
    }

    @Override
    public String getUsername() {
        return loginId;
    }
}
```

이제 Jwt 인증을 수행하는 Filter를 수정해보자.

```java
@RequiredArgsConstructor
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final MemberService memberService;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        try {
            // Authorization 헤더가 없으면 401을 리턴.
            if (request.getHeader(HttpHeaders.AUTHORIZATION) == null) {
                response.sendError(401, "Unauthorized Error");
                return;
            }
            String token = request.getHeader(HttpHeaders.AUTHORIZATION).substring(7);

            Map<String, Object> claims = JwtUtil.decodePayload(token);
            
            String loginId = (String) claims.get("loginId");
            String role = (String) claims.get("role");

            // loginId로 유저를 찾음.
            long customerId = memberService.findMember(loginId).getId();

            // UserDetail 대신 새로 만든 ShopMemberDetail
            ShopMemberDetail detail = new ShopMemberDetail(loginId, role, customerId);

            AbstractAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
                detail, null, Collections.singletonList(new SimpleGrantedAuthority(
                role))
            );
            authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            SecurityContextHolder.getContext().setAuthentication(authToken);
        } catch (AuthenticationException | MemberNotFoundException | MalformedJwtException e) {
            response.sendError(400, "Bad Reqeuest");
            return;
        }
        filterChain.doFilter(request, response);
    }
}
```

UserDetail이 없어지고, 그 자리에 새로 만든 ShopMemberDetail을 넣는다. AuthenticationToken에도 principle로 해당 detail을 넣는다.

이제 `@AuthenticationPrinciple`어노테이션으로 ShopMemberDetail을 주입받을 수 있다!

```java
    @DeleteMapping("/member")
    public ResponseEntity<MessageDto> deleteMember(@AuthenticationPrincipal ShopMemberDetail memberDetail) {
        memberService.deleteById(memberDetail.getCustomerId());
        customerService.deleteCustomer(memberDetail.getCustomerId());
        return ResponseEntity.ok().body(new MessageDto("탈퇴 처리가 완료됐습니다."));
    }
```

그러나 아직 할 일은 더 남았다. 이전부터 Spring Security에서 제공하는 `@WithMockUser`는 더이상 작동하지 않는다. 직접 만든 CustomUserDetail이기 때문이다..

이제 테스트에서도 커스터마이즈된 인증정보를 가져올 수 있도록 해보자.

먼저 테스트에서 사용할 Security Configuration을 정의하자. 이 설정에서 원래 사용했던 Jwt 인증 Filter를 제외한다.
```java
@TestConfiguration
public class TestSecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception{
        http.authorizeHttpRequests(authorize -> authorize.anyRequest().permitAll())
            .csrf(AbstractHttpConfigurer::disable)
            .cors(AbstractHttpConfigurer::disable)
            .formLogin(AbstractHttpConfigurer::disable)
            .httpBasic(AbstractHttpConfigurer::disable)
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            ;
        return http.build();
    }
}
```

그리고 `@WithMockUser`를 대체할 어노테이션을 정의한다. ShopMemberDetail에 필요한 사항들을 받는다.
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
// Security Context에 인증정보를 주입할 Context Factory 클래스. 곧 정의할 것이다.
@WithSecurityContext(factory = WithCustomMockUserSecurityContextFactory.class)
public @interface WithCustomMockUser {
    String loginId();
    String role();
    long customerId();
}
```
특이한 점으로 `@WithSecurityContext`라는 메타 어노테이션이 붙어있다. 해당 어노테이션의 정보를 가지고 인증정보가 저장된 Security Context를 생산해내는 팩토리 클래스를 지정할 수 있다!

그렇다면 해당 클래스를 정의해보자. WithSecurityContextFactory를 구현하면 되고, 이전에 했던 Security Context에 인증정보를 넣는 과정과 동일하기 때문에 만들기 매우 쉽다.
```java
public class WithCustomMockUserSecurityContextFactory implements WithSecurityContextFactory<WithCustomMockUser> {
    @Override
    public SecurityContext createSecurityContext(WithCustomMockUser annotation) {
        String loginId = annotation.loginId();
        String role = annotation.role();
        long customerId = annotation.customerId();
        // 인증정보 생산
        AbstractAuthenticationToken token = new UsernamePasswordAuthenticationToken(
            new ShopMemberDetail(loginId, role, customerId), null, Collections.singletonList(new SimpleGrantedAuthority(role))
        );
        SecurityContext context = SecurityContextHolder.getContext();
        // 인증정보 주입
        context.setAuthentication(token);

        return context;
    }
}
```

이제 다음과 같이 `@WithCustomMockUser`를 사용해 테스트 인증정보를 주입할 수 있다.
```java
@WebMvcTest(    // 뭔가 설정을 많이 하고있다
    value = TestController.class,
    excludeFilters = @ComponentScan.Filter(
        type= FilterType.ASSIGNABLE_TYPE,
        classes = {SecurityConfig.class, JwtAuthenticationFilter.class}
    )
)
@Import(TestSecurityConfig.class)
class SecurityTest {
    @Autowired
    MockMvc mvc;

    @Test
    @WithCustomMockUser(loginId = "123", role = "ROLE_MEMBER", customerId = 2L)
    void securityTest() throws Exception{
        mvc.perform(MockMvcRequestBuilders.get("/api/shop/test/auth"))
            .andDo(print());
    }
}
```

그런데 특이하게, 어노테이션의 여러 설정값이 보인다. `@WebMvcTest`의 excludeFilters 값에 앞서 정의했던 Jwt 인증필터와 Security Config*ㅡ테스트용이 아닌ㅡ*가 보인다. 

`@WebMvcTest`는 해당 컨트롤러 테스트에 필요한 Configuration도 가져오기 때문에, Security Config와 Filter도 같이 가져온다. 일일이 JWT 토큰을 테스트에서 헤더로 보낼 생각은 없으므로, 두 빈을 Component scan에서 제외시켜주고 앞서 정의한 Test Security Config를 `@Import` 어노테이션을 통해 주입해준다.

이로써 Custom User Detail과 테스트에서 커스텀된 인증정보를 주입받는 방법에 대해 알아보았다.