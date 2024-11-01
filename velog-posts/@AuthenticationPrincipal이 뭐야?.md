<p><br /><br /></p>
<h2 id="배경">배경</h2>
<p>프로젝트를 진행할때 큰 지각없이 사용하던 @AuthenticationPrincipal 이라는 어노테이션이 있다</p>
<pre><code> @PostMapping(&quot;/&quot;)
    public ApiResponse&lt;MemoDTO.MemoCreateResp&gt; save(@RequestBody @Valid MemoDTO.MemoCreateDTO requestDTO,
                                                    @AuthenticationPrincipal UserDetails userDetails) {
        MemoDTO.MemoCreateResp response = memoService.create(requestDTO, userDetails.getUsername());

        return ApiResponse.onSuccess(response);
    }</code></pre><p>이런식으로 회원에 대한 정보를 매핑하기 위해 member에 대한 특정 정보를 파라미터로 받는 것이 아닌 <strong>다른 무언가를 통해서 회원을 식별하는 방법</strong> 으로 이해하고 있었다
<br />
하지만 이번 단풍톤을 진행하면서 부터는 프론트 분들에게 내가 만든 토큰을 어떻게 사용할지 사전에 공지를 해야할 것 같아, 내가 먼저 확실하게 이해를 해야겠다는 생각으로 이 글을 정리하게 되었다.
<br /><br /><br /></p>
<h2 id="스프링-시큐리티의-이해">스프링 시큐리티의 이해</h2>
<p>해당 코드는 로그인을 통과한 이용자에 대해 토큰을 생성하고 그 회원의 정보를 <strong>SecurityContextHolder</strong>
에 저장한다</p>
<pre><code>@RequiredArgsConstructor
@Slf4j
public class JWTFilter extends OncePerRequestFilter{

    private final JWTUtil jwtUtil;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

        // 헤더에서 Authorization 토큰을 꺼냄
        String authorizationHeader = request.getHeader(&quot;Authorization&quot;);

        // Authorization 헤더가 없거나 Bearer 스킴이 없으면 다음 필터로 이동
        if (authorizationHeader == null || !authorizationHeader.startsWith(&quot;Bearer &quot;)) {
            filterChain.doFilter(request, response);
            return;
        }

        // Bearer 뒤의 토큰을 추출
        String accessToken = authorizationHeader.substring(7).trim();

        // 토큰 만료 여부 확인
        try {
            log.info(&quot;토큰 있어서 검증 시작&quot;);
            jwtUtil.isExpired(accessToken);
        } catch (ExpiredJwtException e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().print(&quot;access token expired&quot;);
            return;
        }

        // 토큰이 access인지 확인
        String category = jwtUtil.getCategory(accessToken);
        if (!category.equals(&quot;access&quot;)) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().print(&quot;invalid access token&quot;);
            return;
        }

        //토큰에서 username과 role 획득
        String username = jwtUtil.getUsername(accessToken);
        String roleString = jwtUtil.getRole(accessToken);

        // String role을 Role enum으로 변환
        Role role = Role.valueOf(roleString);


        Member member = Member.builder()
                .username(username)
                .role(role)
                .build();

        //UserDetails에 회원 정보 객체 담기
        CustomUserDetails customUserDetails = new CustomUserDetails(member);

        //스프링 시큐리티 인증 토큰 생성
        Authentication authToken = new UsernamePasswordAuthenticationToken(customUserDetails, null, customUserDetails.getAuthorities());

        SecurityContextHolder.getContext().setAuthentication(authToken);

        filterChain.doFilter(request, response);
    }
}</code></pre><p>여기서 <strong>SecurityContextHolder</strong> 에 대해서 약간의 이해가 필요한데,
회원의 정보를 저장한다는 개념에서
로그인 한 회원의 정보를 서버에서 관리한다
<br />
-&gt; 세션을 유지한다
-&gt; Stateless 하지 못한 설계
-&gt; 틀린 설계!
<br /></p>
<p>라고 생각 할 수 있지만 이는 사실 잘못된 해석이다
SecurityContextHolder 는 엄밀히 말하자면 <strong>임시세션</strong>이다
여기에 저장된 회원정보는 한 번의 요청-응답이 완료되면 사라진다</p>
<p>즉, 한 번의 요청을 해결하기 위해서 필요한 회원정보만 잠시 담아두는 형태이기 때문에 세션이라고 볼 수 없다. 저장되는 위치도 LocalThread이고 이렇게 로그인을 구현할 시, 토큰을 매 요청 받아야한다는 점에서도 Stateful 하다고 볼 수 없다.
<br /><br /><br /></p>
<h2 id="authenticationprincipal">@AuthenticationPrincipal</h2>
<p>그럼 이제 다시 어노테이션으로 돌아와보자</p>
<pre><code> @PostMapping(&quot;/&quot;)
    public ApiResponse&lt;MemoDTO.MemoCreateResp&gt; save(@RequestBody @Valid MemoDTO.MemoCreateDTO requestDTO,
                                                    @AuthenticationPrincipal UserDetails userDetails) {
        MemoDTO.MemoCreateResp response = memoService.create(requestDTO, userDetails.getUsername());

        return ApiResponse.onSuccess(response);
    }</code></pre><p>어차피 검증은 회원 id같은 것으로 하는게 아닌, 토큰의 유효성으로 회원을 검증하기 때문에 서버 입장에서는 헤더를 통해 토큰만 받을 수 있다면 굳이 회원의 id를 파라미터로 받을 필요가 없다</p>
<p>앞서 설정한 것처럼 SecurityContextHolder에 데이터를 저장할때 UserDetails의 구현체인 CustomUserDetails를 이용하여 데이터를 저장하므로 불러올때도 똑같이 UserDetails를 통해서 데이터를 불러온다</p>
<p><br /><br /><br /></p>
<h2 id="develop-idea">DEVELOP IDEA</h2>
<br />

<h4 id="왜-토큰에-memberid를-넣어야할까">왜 토큰에 memberId를 넣어야할까</h4>
<ul>
<li>아직도 이해할 수 없다</li>
<li>토큰 안에 회원을 식별 할 수 있는 몇가지의 값은 필요하지만 memberId와 같이 특정되는 값은 넣지 않는게 좋지 않을까 하는 생각<br />
#### 프론트 분들은 어떻게 이 토큰을 관리하게 될까</li>
<li>아마 서버에서 반환되는 토큰을 로컬 스토리지에 저장하여 사용 할 것 같은데</li>
<li>이 로직도 따로 이해 할 수 있다면 개인 공부에 도움이 될 듯 하다</li>
</ul>