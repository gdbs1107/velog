<h2 id="배경">배경</h2>
<p>이번에 단풍톤 프로젝트를 진행하면서 단일 EC2에 도메인을 넣는 방식으로 DN를 구현하였다.</p>
<p>평소에 Load Balancer의 기능을 하는 AWS Elastic Beanstalk를 이용해서 서버를 구성하였기 때문에 Alias를 이용하여 DNS를 넣을 수 있었지만, 이번엔 단일 EC2를 이용하여 서버를 개설하였기 때문에 ALB를 직접 구축하여 거기에 ACM을 넣는 방식으로 HTTPS를 구현하여야 했다.</p>
<p>처음 해보는 방식이었기 때문에 신경쓸게 너무 많았던 나머지 다양한 에러가 발생했는데, 오늘은 프론트와의 협업과정에서 발생한 403 ERROR에 대해서 남겨보겠다.</p>
<p><br /><br /><br /></p>
<h2 id="trouble">TROUBLE</h2>
<p>프론트분에게 온 연락의 내용은 다음과 같았다</p>
<blockquote>
<p>http에선 잘 되는데 https에선 
ERR_SSL_PROTOCOL_ERROR
가 발생하네요. 아직 https가 설정되지 않은걸까요..??</p>
</blockquote>
<p>음음 적잖이 당황했다. 포스트맨으로 테스트 해봤을땐 분명 200이 뜨는 API들이었는데, 다시 스웨거를 통해서 테스트해보니 나도 403이 반환되었다.</p>
<p>분명 CORS에 대한 설정은 잘 해놓고 프로젝트를 시작했는데 무엇이 문제였을까 하고 돌아보니,</p>
<br />
```

<pre><code>@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception{

    http
            .cors(corsCustomizer -&gt; corsCustomizer.configurationSource(new CorsConfigurationSource() {

                @Override
                public CorsConfiguration getCorsConfiguration(HttpServletRequest request) {

                    CorsConfiguration configuration = new CorsConfiguration();

                    configuration.setAllowedOrigins(Arrays.asList(
                            &quot;http://localhost:3000&quot;,
                            &quot;http://[IP주소]:8080/&quot;
                    ));
                    configuration.setAllowedMethods(Arrays.asList(&quot;GET&quot;, &quot;POST&quot;, &quot;PUT&quot;, &quot;DELETE&quot;, &quot;OPTIONS&quot;));
                    configuration.setAllowCredentials(true);
                    configuration.setAllowedHeaders(Collections.singletonList(&quot;*&quot;));
                    configuration.setMaxAge(3600L);

                    // exposedHeaders에 중복 설정 제거하고, 두 개의 헤더를 노출
                    configuration.setExposedHeaders(Arrays.asList(&quot;Set-Cookie&quot;, &quot;access&quot;, &quot;Authorization&quot;));

                    return configuration;
                }
            }));</code></pre><p>AllowedOrigins에 도메인을 적용한 사항을 반영을 하지 않은 것이다</p>
<p>굉장히 사소했지만, 에러를 만드는 주범이었다. 정말 별것도 아닌걸로 오래 걸렸다. 전에 OAuth를 통한 소셜로그인을 구현할 때에도 ExposedHeaders 로 인해서 고생 했었는데,정말 CORS는 자잘한 에러를 잘 터뜨리는 것 같다</p>
<p>이번 방학에는 CORS를 잘 공부해서 이런 상황에 시간을 오래 쓰지 않도록 노력해야겠다.</p>