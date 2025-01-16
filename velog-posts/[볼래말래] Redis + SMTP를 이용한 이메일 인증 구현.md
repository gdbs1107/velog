<p><img alt="" src="https://velog.velcdn.com/images/gdbs1107/post/9e1d95d3-2192-47ad-a68e-fca2a0d4b753/image.png" /></p>
<ul>
<li>배경</li>
<li>redis 구현</li>
<li>Mail API 구현</li>
<li>디벨롭 포인트</li>
</ul>
<p><br /><br /><br /></p>
<h2 id="배경">배경</h2>
<hr />
<p>기존 볼래말래의 기획안에서 “카카오 알림톡” 기능을 이용하여 사용자에게 대부분의 알림을 전달하려 했지만…</p>
<p>알림톡을 도와주는 중간 벤더들은 대부분 사업자등록증이 필요한 카카오 비즈니스 채널을 요구하였다. ( 당연하다 )</p>
<br />
<br />

<p>사업자등록증 발급이 어렵진 않지만, 많은 혜택을 포기해야하는 선택이기 때문에 일단 이메일을 통해서 1차적으로 구현을 하고 추후 카카오 알림톡으로 디벨롭하자는 방향으로 기획이 수정되었다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/gdbs1107/post/8a4c7d4e-5974-4963-b9cf-601fbf286755/image.png" /></p>
<p>나 구현 해봤는데 이메일.</p>
<br />

<p>이메일 인증을 위해서 구현을 해보았지만, 인증번호를 인증하는 절차를 그냥 List에 키밸류로 담아서 저장하고 인증을 진행했었다.</p>
<p>이메일 인증은 한사람이 몇번이고 하는건데 당연히 List로 다 담는건 현실적으로 불가능하고 중복데이터 검증도 너무 불안정한 구현형태였다.</p>
<p>그렇다고 RDB에 일일히 다 저장하자니 이것도 뭐 List랑 크게 다르지 않다고 생각하고,</p>
<p>데이터를 일정시간만 유지하고 삭제하는 로직을 추가해야하는데 그러면 mysql과 spring이 서로를 의존하게되는 기이한 형태를 지니게 된다.</p>
<br />

<p>결론적으로 인메모리 형태로 데이터를 read/write하여 빠르고 가벼우며, 자동타이머 메서드를 내장하고 있고 아직은 실력이 닿지 않지만 캐싱작업을 구현 할 수 있는 가능성이 열리는</p>
<p>또한 refreshToken도 정확히 동일한, 아니 오히려 더하는 정도로 같은 문제점을 지니지만 mysql로 관리하고 있었기 때문에 겸사겸사 Redis를 도입하기로 결정을 하였다.</p>
<h2 id="redis-연동-및-구현">Redis 연동 및 구현</h2>
<hr />
<p>우선 redis를 주입받자</p>
<pre><code class="language-java">    // redis
    implementation 'org.springframework.data:spring-data-redis:2.7.5'
    implementation 'io.lettuce:lettuce-core:6.2.1.RELEASE'</code></pre>
<br />

<p>그리고 이를 이용 할 수 있도록 yml도 작성해준다</p>
<pre><code class="language-java">  redis:
    host: localhost
    port: 6379</code></pre>
<br />

<p>그리고 config 를 구현해주자</p>
<pre><code class="language-java">@Configuration
@EnableRedisRepositories
@Profile(&quot;local&quot;)
public class RedisConfig {

    @Value(&quot;${management.redis.host}&quot;)
    private String redisHost;

    @Value(&quot;${management.redis.port}&quot;)
    private int redisPort;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(redisHost, redisPort);
    }

    @Bean
    public RedisTemplate&lt;?, ?&gt; redisTemplate() {
        RedisTemplate&lt;byte[], byte[]&gt; redisTemplate = new RedisTemplate&lt;&gt;();
        redisTemplate.setConnectionFactory((redisConnectionFactory()));
        return redisTemplate;
    }

    @Bean
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory) {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
}</code></pre>
<p>레디스를 이용 할 수 있는 내장 메서드를 구현해주는데</p>
<p>이게 좀 신기했다. JPA를 사용하여 쓰던 mysql과는 되게 달랐다</p>
<br />
<br />

<pre><code class="language-java">@RequiredArgsConstructor
@Service
public class RedisUtil {
    private final StringRedisTemplate template;

    /**

     key 값을 이용하여 value를 가져오는 메서드

     * */
    public String getData(String key) {
        ValueOperations&lt;String, String&gt; valueOperations = template.opsForValue();
        return valueOperations.get(key);
    }

    /**

     key에 해당하는 데이터가 있는지 조회하는 메서드

     */
    public boolean existData(String key) {
        return Boolean.TRUE.equals(template.hasKey(key));
    }

    /**

     key/value로 데이터를 저장하는 메서드
     + 만료시간을 설정 할 수 있다

     */
    public void setDataExpire(String key, String value, long duration) {
        ValueOperations&lt;String, String&gt; valueOperations = template.opsForValue();
        Duration expireDuration = Duration.ofSeconds(duration);
        valueOperations.set(key, value, expireDuration);
    }

    public void deleteData(String key) {
        template.delete(key);
    }
}</code></pre>
<br />

<p>메서드를 설명해보자면</p>
<ul>
<li>getData()<ul>
<li>key/value 타입으로 데이터를 저장한다</li>
<li>key를 반환한다</li>
</ul>
</li>
<li>existData()<ul>
<li>key를 이용해서 해당하는 데이터가 존재하는지 조회한다</li>
<li>존재하면 Boolean.True를 반환한다</li>
</ul>
</li>
<li>setDataExpire<ul>
<li>데이터의 만료시간을 설정한다</li>
<li>아주 유용한 메서드</li>
<li>이를 이용해서 refreshToken이나 인증번호를 별도의 Batch 작업 필요없이 구현이 가능해진다<ul>
<li>물론 정말 대용량의 데이터를 일시에 처리할때는 Spring Batch를 이용하는게 좋다고 한다</li>
</ul>
</li>
</ul>
</li>
<li>deleteData<ul>
<li>데이터를 삭제한다</li>
</ul>
</li>
</ul>
<p>이렇게 메서드를 이용하여 데이터를 다룰 수 있게된다.</p>
<p>막상 구현을 해보니 데이터 다루는 기본원리가 참 비슷하다는 생각이 든다</p>
<p><del>(추상화가 또 생각이 나네)</del>
<br />
<br /></p>
<h2 id="smtp-구현">SMTP 구현</h2>
<hr />
<p>Google SMTP 비밀번호를 발급받는건 어렵지 않으니 구글링을 통해 찾아보자</p>
<p>절대 스크린샷이 귀찮은게 아니야</p>
<br />
<br />

<pre><code class="language-java">implementation 'org.springframework.boot:spring-boot-starter-mail'

implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
implementation 'ognl:ognl:3.3.1'</code></pre>
<p>mail-starter와 mail form을 위한 thymeleaf를 주입받는다</p>
<p>인증메일 폼을 제작해주고</p>
<pre><code class="language-java">&lt;!DOCTYPE html&gt;
&lt;html xmlns:th=&quot;http://www.thymeleaf.org&quot;&gt;
&lt;body&gt;
&lt;div style=&quot;margin:120px&quot;&gt;
    &lt;div style=&quot;margin-bottom: 10px&quot;&gt;
        &lt;h1&gt;인증 코드 메일입니다.&lt;/h1&gt;
        &lt;br/&gt;
        &lt;h3 style=&quot;text-align: center;&quot;&gt; 아래 코드를 사이트에 입력해주십시오&lt;/h3&gt;
    &lt;/div&gt;
    &lt;div style=&quot;text-align: center;&quot;&gt;
        &lt;h2 style=&quot;color: crimson;&quot; th:text=&quot;${code}&quot;&gt;&lt;/h2&gt;
    &lt;/div&gt;
    &lt;br/&gt;
&lt;/div&gt;
&lt;/body&gt;
&lt;/html&gt;</code></pre>
<p>서비스 코드를 작성해준다
<br /></p>
<pre><code class="language-java">@Service
@RequiredArgsConstructor
@Profile(&quot;local&quot;)
public class MailServiceImpl implements MailService {

    private final JavaMailSender mailSender;
    private final RedisUtil redisUtil;

    @Value(&quot;${management.mail.username}&quot;)
    private String configEmail;

    @Override
    public void sendEmail(String toEmail) throws MessagingException {
        if (redisUtil.existData(toEmail)) {
            redisUtil.deleteData(toEmail);
        }

        MimeMessage emailForm = createEmailForm(toEmail);

        mailSender.send(emailForm);
    }

    @Override
    public Boolean verifyEmailCode(String email, String code) {

        String codeFoundByEmail = redisUtil.getData(email);

        if (codeFoundByEmail == null) {
            throw new MailHandler(ErrorStatus.MAIL_NOT_SEND);
        }

        if (!code.equals(codeFoundByEmail)) {
            throw new MailHandler(ErrorStatus.MAIL_NOT_VALID);
        }

        return true;
    }

    private String setContext(String code) {
        Context context = new Context();
        TemplateEngine templateEngine = new TemplateEngine();
        ClassLoaderTemplateResolver templateResolver = new ClassLoaderTemplateResolver();

        context.setVariable(&quot;code&quot;, code);

        templateResolver.setPrefix(&quot;templates/&quot;);
        templateResolver.setSuffix(&quot;.html&quot;);
        templateResolver.setTemplateMode(TemplateMode.HTML);
        templateResolver.setCacheable(false);

        templateEngine.setTemplateResolver(templateResolver);

        return templateEngine.process(&quot;mail&quot;, context);
    }

    private MimeMessage createEmailForm(String email) throws MessagingException {

        String authCode = createdCode();

        MimeMessage message = mailSender.createMimeMessage();
        message.addRecipients(MimeMessage.RecipientType.TO, email);
        message.setSubject(&quot;[볼래말래] 회원가입 이메일 2차확인 인증번호&quot;);
        message.setFrom(configEmail);
        message.setText(setContext(authCode), &quot;utf-8&quot;, &quot;html&quot;);

        redisUtil.setDataExpire(email, authCode, 60 * 5L);

        return message;
    }

    @Override
    public String makeMemberId(String email) throws NoSuchAlgorithmException {
        MessageDigest md = MessageDigest.getInstance(&quot;SHA-256&quot;);
        md.update(email.getBytes());
        md.update(LocalDateTime.now().toString().getBytes());
        return Arrays.toString(md.digest());
    }

    private String createdCode() {
        int leftLimit = 48; // number '0'
        int rightLimit = 122; // alphabet 'z'
        int targetStringLength = 6;
        Random random = new Random();

        return random.ints(leftLimit, rightLimit + 1)
                .filter(i -&gt; (i &lt;=57 || i &gt;=65) &amp;&amp; (i &lt;= 90 || i&gt;= 97))
                .limit(targetStringLength)
                .collect(StringBuilder::new, StringBuilder::appendCodePoint, StringBuilder::append)
                .toString();
    }
}</code></pre>
<br />
<br />

<ul>
<li><p>sendEmail</p>
<ul>
<li>제작한 이메일을 전송한다</li>
<li>이때 전송이력이 존재한다면, 이전에 있던 데이터를 삭제한다</li>
</ul>
</li>
<li><p>verifyEmailCode</p>
<ul>
<li>데이터를 검증한다</li>
<li>데이터가 일치하지 않으면 정해놓은 에러를 반환한다</li>
</ul>
</li>
<li><p>setContext</p>
<ul>
<li>정적인 이메일 폼을 작성한다</li>
<li>타임리프와 연동하여 폼을 제작&amp;전송한다</li>
</ul>
</li>
<li><p>createEmailForm</p>
<ul>
<li>동적인 데이터를 다수 포함한다</li>
<li><strong>여기서 인증번호의 유효시간을 설정한다</strong></li>
<li>시간이 만료되면 value는 null이 된다 → 인증이 실패</li>
</ul>
</li>
<li><p>createdCode</p>
<ul>
<li><p>인증 코드를 생성한다</p>
<br />
<br />

</li>
</ul>
</li>
</ul>
<p>그냥 서비스를 호출하는 컨트롤러 코드</p>
<pre><code class="language-java">@RestController
@RequiredArgsConstructor
@RequestMapping(&quot;/emails&quot;)
@Tag(name = &quot;이메일 인증 API&quot;)
public class MailController {

    private final MailService emailService;

    @GetMapping(&quot;/{email_addr}&quot;)
    @Operation(summary = &quot;인증 이메일 전송 API&quot;)
    public ApiResponse&lt;String&gt; sendEmailPath(@PathVariable String email_addr) throws MessagingException {

        emailService.sendEmail(email_addr);

        return ApiResponse.onSuccess(&quot;이메일이 전송되었습니다&quot;);
    }

    @PostMapping(&quot;/{email_addr}&quot;)
    @Operation(summary = &quot;인증 번호 검증 API&quot;)
    public ApiResponse&lt;String&gt; sendEmailAndCode(@PathVariable String email_addr,
                                                   @RequestBody EmailRequestDto dto) throws NoSuchAlgorithmException {

        emailService.verifyEmailCode(email_addr, dto.getCode());

        return ApiResponse.onSuccess(&quot;이메일 인증에 성공하였습니다&quot;);
    }
}
</code></pre>
<p>(우리 볼래말래 이메일 주소도 있다구요~)</p>
<p><img alt="" src="https://velog.velcdn.com/images/gdbs1107/post/8247eb68-f6af-4d9a-a781-358e26d283d7/image.png" />
<br /></p>
<p>이렇게 구현이 마무리 된다</p>
<p>폼 개구려 윽</p>
<h2 id="디벨롭">디벨롭</h2>
<hr />
<h3 id="배포된-서버에-레디스를-어떻게-얹어야할지-모르겠다">배포된 서버에 레디스를 어떻게 얹어야할지 모르겠다</h3>
<p>서버를 열기에는 관리하는게 실질적으로 리프레시 토큰이 전부라서 자원이 낭비되는 것 같다고 생각한다. </p>
<p>그래서 그냥 배포된 EC2에 레디스를 설치하고 사용하는게 좋을 것 같다는 막연한 생각 뿐</p>
<br />

<h3 id="리프레시토큰-과-인증번호를-구별하는-법">리프레시토큰 과 인증번호를 구별하는 법</h3>
<p>내가 진짜 레디스를 몰라서 테이블의 개념이 있는지도 모르겠다</p>
<p>그냥 DB에 데이터가 들어가는건 눈으로 보긴 했는데…테이블이 안보여…</p>
<p>물론 키값이 username과 email이기에 아예 다르긴 하지만…어떻게 해야할지 고민이다</p>
<br />

<h3 id="테스트-코드-작성">테스트 코드 작성</h3>
<p>mysql은 정말 학교수업에서도 하고 다양한 프로젝트에서 계속 쓴 제품이라 너무 익숙한데, 레디스는 아직 너무 낯설다.</p>
<p>모킹을 하지 않는 테스트 코드는 당장은 포기해야하지 않을까라는 생각까지 든다</p>
<br />


<p><img alt="" src="https://velog.velcdn.com/images/gdbs1107/post/27a241f7-900e-422e-a8f0-7101cae387d4/image.png" /></p>
<p>이제 시작이라니</p>