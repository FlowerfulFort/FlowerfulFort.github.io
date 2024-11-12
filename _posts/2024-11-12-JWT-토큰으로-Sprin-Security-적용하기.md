---
title: "JWT 토큰으로 Spring Security 적용하기"
date: 2024-11-12 09:30:00 +0900
categories: ["Spring Boot"]
tags: [Spring Boot, JWT, Spring Security]
---

프로젝트를 하던 도중, 팀원이 인증시스템을 만들어 JWT 토큰을 발급하게 하였다. Servlet API의 Session을 사용하지 않고, 브라우저의 쿠키 만으로 로그인 세션을 가질 수 있게 되었다.

이전의 회원 ID를 참조하는 Rest API Controller는 다음과 비슷한 맥락이었다.

```java
    @PostMapping("/api/shop/order/{customerId}")
    public ResponseEntity<OrderCreateResponseDto> createNewOrder(
        @PathVariable long customerId,
        @Valid @RequestBody OrderCreateRequestDto requestDto,
        BindingResult result
    ) {
        if (result.hasErrors()) {
            throw new BadRequestException(
                "Order creation failed: bad request"
            );
        }
        return ResponseEntity.status(HttpStatus.CREATED).body(
            orderService.createOrder(customerId, requestDto)
        );
    }
```

이 컨트롤러에서는 PathVariable로 customer의 참조 ID를 직접 받아 사용한다. 이제 Front View 서버에서 JWT 토큰을 헤더에 담아 넘길 수 있게 되므로 다음과 같은 코드로 리팩토링 하고싶다!

```java
    @PostMapping("/api/shop/order")
    public ResponseEntity<OrderCreateResponseDto> createNewOrder(
        @AuthenticationPrincipal ShopMemberDetail memberDetail,
        @Valid @RequestBody OrderCreateRequestDto requestDto,
        BindingResult result
    ) {
        if (result.hasErrors()) {
            throw new BadRequestException("Order creation failed: bad request");
        }
        return ResponseEntity.status(HttpStatus.CREATED).body(
            orderService.createOrder(memberDetail.getCustomerId(), requestDto)
        );
    }
```
PathVariable이 빠지고 Spring Security의 SecurityContext에서 User를 가져와 해당 User에서 customerId를 가져온다.

PathVariable을 사용하지 않아도 되니 요청의 일관성이 높아지고 `@PreAuthorize` 같은 Spring Security의 어노테이션을 적용하여 role에 따른 authorization 체크에 대한 관점도 분리할 수 있는 장점이 있다.

Spring Security를 적용하기 위해 dependency를 추가하자.

```xml
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.security</groupId>
      <artifactId>spring-security-test</artifactId>
      <scope>test</scope>
    </dependency>
```

Rest API를 호출하는 Front View 서버는 JWT 토큰을 헤더에 담아 호출할 것이다. JWT 토큰을 해석하는 유틸리티 클래스를 만들자.

JWT 토큰은 method, payload, signature 3가지로 구성되어 있고. 각각 base64로 인코딩 되어 구분자 `.`으로 구분된다. 팀원이 JWT 시그니처 체크는 Spring Gateway에서 진행한다고 하였으니, 여기서는 base64 디코딩만 진행한다.

```java
    public static Map<String, Object> decodePayload(String jwt) throws IOException {
        String[] parts = jwt.split("\\.");
        if (parts.length < 2) {
            throw new IllegalArgumentException();
        }
        byte[] decode = Base64.getUrlDecoder().decode(parts[1]);

        ObjectMapper mapper = new ObjectMapper();
        return mapper.readValue(decode, Map.class);
    }
```

이제 JWT 토큰을 해석하여 Payload의 내용물을 열어볼 수 있게 되었다.

Filter를 정의하여 헤더를 해석하여 SecurityContext에 저장해보자.

```java
@RequiredArgsConstructor
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final MemberService memberService;
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
        FilterChain filterChain) throws ServletException, IOException {
        try {
            if (request.getHeader(HttpHeaders.AUTHORIZATION) == null) {
                filterChain.doFilter(request, response);
                return;
            }
            String token = request.getHeader(HttpHeaders.AUTHORIZATION).substring(7);

            if (token.equals("ANONYMOUS")) {
                filterChain.doFilter(request, response);
                return;
            }

            Map<String, Object> claims = JwtUtil.decodePayload(token);
            Member member = memberService.getMember(claims.get("loginId"));

            User detail = new User(member.getCustomerId(), null, Collections.singletonList(new SimpleGrantAuthority("ROLE_MEMBER")));

            AbstractAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
                detail, null, Collections.singletonList(new SimpleGrantedAuthority("ROLE_MEMBER"))
            );
            authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

            // Spring Security Context에 추가.
            SecurityContextHolder.getContext().setAuthentication(authToken);
        } catch (AuthenticationException | MemberNotFoundException | MalformedJwtException e) {
            // do nothing
        }
        filterChain.doFilter(request, response);
    }
}
```

1. `request.getHeader(HttpHeaders.AUTHORIZATION)`을 이용해 Authorization 헤더를 가져온다.
2. JWT 토큰 해석을 진행한다. 현재 JWT 토큰에는 `loginId`와 `role`필드가 존재한다
3. `MemberService`에서 loginId 에 대응하는 Member Entity를 가져온다.
4. User principle을 정의하여 member정보를 넣고, `ROLE_MEMBER` role을 부여한다.(토큰의 role 필드는 지금 사용하지 않는다.)
5. AuthToken을 만들고, SecurityContext에 추가한다.

Filter를 만들었으니 Filter를 적용시켜야 한다.

```java
@EnableWebSecurity
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http,
        JwtAuthenticationFilter jwtAuthenticationFilter) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .cors(AbstractHttpConfigurer::disable)
            .formLogin(AbstractHttpConfigurer::disable)
            .httpBasic(AbstractHttpConfigurer::disable)
            .sessionManagement(
                session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
        ;
        http.authorizeHttpRequests(
            authorizedRequests -> authorizedRequests.anyRequest().permitAll());
        return http.build();
    }
}
```
Rest API 서버에서 도움이 안되는 각종 Security 설정들을 꺼버리고, addFilterBefore()을 이용해 필터를 붙여주자.

이제 다음과 같이 리팩토링이 가능하다.

```java
    @PostMapping("/api/shop/order")
    public ResponseEntity<OrderCreateResponseDto> createNewOrder(
        @AuthenticationPrincipal UserDetails userDetail,
        @Valid @RequestBody OrderCreateRequestDto requestDto,
        BindingResult result
    ) {
        // ...
    }
```

그러나 위에서 원했던 Custom user detail을 적용하지는 못했다. 이에 대해서는 다음에 다룬다.