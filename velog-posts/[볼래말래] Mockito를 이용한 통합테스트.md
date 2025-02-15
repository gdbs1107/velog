<h1 id="1-들어가며">1. 들어가며</h1>
<p><a href="https://api.velog.io/rss/%5Bhttps:/velog.io/@gdbs1107/%EB%B3%BC%EB%9E%98%EB%A7%90%EB%9E%98-Mockito-H2%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-%EB%8B%A8%EC%9C%84%ED%85%8C%EC%8A%A4%ED%8A%B8%5D(https:/velog.io/@gdbs1107/%EB%B3%BC%EB%9E%98%EB%A7%90%EB%9E%98-Mockito-H2%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-%EB%8B%A8%EC%9C%84%ED%85%8C%EC%8A%A4%ED%8A%B8)">이전 글을 참고해주세요</a></p>
<p>우리는 이전에 Mockito와 H2를 활용하여 각 레이어의 기능만을 독립적으로 테스트하는 단위테스트를 구현해보았다.
이를 하고 나니 각 계층에서의 에러를 디버깅하고, 숨어있는 의존성을 파악하는 등 실제 기능과 객체지향을 점검하는데에 큰 역할을 할 수 있었다.</p>
<p>하지만 이게 끝일까?</p>
<p>각 계층이 상호작용하며 예상치 못한 문제가 발생하진 않을까?
Controller 계층은 제대로 작동하는걸까?
다양한 엣지 포인트를 코드를 리팩터링하며 단위테스트만으로 정말 전부 커버할 수 있을까?
…</p>
<p>이를 해결하기 위해서는 다양한 엣지 케이스를 구상하여 swagger나 postman을 이용하여 요청을 직접 보내보며 테스트 해볼 수 있지만</p>
<p><img alt="" src="https://velog.velcdn.com/images/gdbs1107/post/030fd7e5-a2a9-4741-b638-6d7e56689228/image.png" /></p>
<p><del>아 그걸 어떻게 해 에바잖아</del></p>
<p>말이 안된다. 그냥 테스트 코드안에 그걸 포함시켜놓고 test-run해서 모든걸 한 번에 테스트 해볼 순 없을까?</p>
<hr />
<p><br /><br /></p>
<h1 id="2-통합-테스트-코드-구현">2. 통합 테스트 코드 구현</h1>
<p>이는 통합 테스트로서 해결이 가능하다</p>
<pre><code class="language-java"> @Override
    public MemberJoinDTO.MemberJoinResponseDTO joinMember(@Valid MemberJoinDTO.MemberJoinRequestDTO request){

        authenticateAgreement(request);
        authenticateUsernameValid(request.getUsername());

        String encodedPassword = bCryptHolder.encode(request.getPassword());
        Member newMember= MemberConverter.toMember(request,encodedPassword);
        Member savedMember = memberRepository.save(newMember);

        Agreement newAgreement = Agreement.JoinDTOto(request,savedMember);
        agreementRepository.save(newAgreement);

        return MemberJoinDTO.MemberJoinResponseDTO.builder()
                .memberId(savedMember.getId())
                .build();
    }</code></pre>
<p>이전 포스팅에서 이용했던 회원가입을 예시로 들어보자</p>
<p>회원가입의 예외사항은 정말 많다. 비밀번호 형식이 틀릴수도, 아이디가 중복일 수도, 이미 등록한 회원일지도 모른다.</p>
<p>이렇게 다양한 예외사항을 모두 점검하기 위해서 통합테스트를 구현해보면,</p>
<br />

<pre><code class="language-java">@SpringBootTest
@AutoConfigureMockMvc(addFilters = false)
@ActiveProfiles(&quot;test&quot;)
@Transactional
class MemberControllerTest {

    @Autowired
    protected MockMvc mockMvc;</code></pre>
<p>MockMVC를 사용한다.</p>
<p>Mock요청을 구현하도록 지원해주는 의존성이다. 이를 이용해서</p>
<br />

<pre><code class="language-java">    @Test
    @DisplayName(&quot;join()을 통해 회원가입을 진행 할 수 있다&quot;)
    void join_success() throws Exception {
        // given
        MemberJoinDTO.MemberJoinRequestDTO request = MemberJoinDTO.MemberJoinRequestDTO.builder()
                .username(&quot;test123&quot;)
                .password(&quot;Test123!&quot;)
                .name(&quot;test&quot;)
                .gender(Gender.FEMALE)
                .birthDate(LocalDate.of(1, 1, 1))
                .email(&quot;test@naver.com&quot;)
                .phoneNumber(&quot;010-1111-1111&quot;)
                .serviceAgreement(true)
                .privacyAgreement(true)
                .financialAgreement(true)
                .advAgreement(true)
                .build();

        // when &amp; then
        mockMvc.perform(post(&quot;/members/join&quot;)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(jsonPath(&quot;$.isSuccess&quot;).value(true))
                .andExpect(jsonPath(&quot;$.code&quot;).value(&quot;COMMON200&quot;))
                .andExpect(jsonPath(&quot;$.message&quot;).value(&quot;성공입니다.&quot;))
                .andExpect(jsonPath(&quot;$.result.memberId&quot;).exists());
    }</code></pre>
<p>이렇게 가짜 요청을 보낼 수 있다.
<br /></p>
<p>그러면 서버는 해당 API를 실제로 동작시켜 다양한 에러를 검증 할 수 있도록하고 응답도 반환시킨다.
물론 이를 효과적으로 활용하기 위해서는 H2와 같은 테스트 DB도 적극적으로 할용해야하기 때문에</p>
<pre><code class="language-java">@Autowired
    private MemberRepository memberRepository;</code></pre>
<br />
memberRepository를 실제로 주입받아서 동작 할 수 있도록 한다.

<p>이때 @DataJpaTest와 동시에 사용이 불가능하니 주의하도록 하자 (@SpringBootTest와 기능이 충돌한다)</p>
<hr />
<p><br /><br /></p>
<h1 id="3-정리">3. 정리</h1>
<p>이런식으로 통합테스트를 구현하면</p>
<pre><code class="language-java"> @Test
    @DisplayName(&quot;username이 중복되면 정해진 예외를 반환한다&quot;)
    public void join_fail_username_duplicate() throws Exception {
        MemberJoinDTO.MemberJoinRequestDTO request = MemberJoinDTO.MemberJoinRequestDTO.builder()
                .username(&quot;test12&quot;)
                .password(&quot;Test123!&quot;)
                .name(&quot;test&quot;)
                .gender(Gender.FEMALE)
                .birthDate(LocalDate.of(1, 1, 1))
                .email(&quot;test@naver.com&quot;)
                .phoneNumber(&quot;010-1111-1111&quot;)
                .serviceAgreement(true)
                .privacyAgreement(true)
                .financialAgreement(true)
                .advAgreement(true)
                .build();

        // when &amp; then
        mockMvc.perform(post(&quot;/members/join&quot;)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andDo(print())
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath(&quot;$.isSuccess&quot;).value(false))
                .andExpect(jsonPath(&quot;$.code&quot;).value(&quot;COMMON400&quot;))
                .andExpect(jsonPath(&quot;$.message&quot;).value(&quot;잘못된 요청입니다.&quot;))
                .andExpect(jsonPath(&quot;$.result.username&quot;).value(&quot;중복된 username입니다&quot;));
    }

    @Test
    @DisplayName(&quot;비밀번호 패턴에 불일치 할 시, 정해진 예외를 반환한다&quot;)
    void join_fail_password() throws Exception {
        // given
        String ERROR_PASSWORD = &quot;error_password&quot;;

        MemberJoinDTO.MemberJoinRequestDTO request = MemberJoinDTO.MemberJoinRequestDTO.builder()
                .username(&quot;test123&quot;)
                .password(ERROR_PASSWORD)
                .name(&quot;test&quot;)
                .gender(Gender.FEMALE)
                .birthDate(LocalDate.of(1, 1, 1))
                .email(&quot;test@naver.com&quot;)
                .phoneNumber(&quot;010-1111-1111&quot;)
                .serviceAgreement(true)
                .privacyAgreement(true)
                .financialAgreement(true)
                .advAgreement(true)
                .build();

        // when &amp; then
        mockMvc.perform(post(&quot;/members/join&quot;)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andDo(print())
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath(&quot;$.isSuccess&quot;).value(false))
                .andExpect(jsonPath(&quot;$.code&quot;).value(&quot;COMMON400&quot;))
                .andExpect(jsonPath(&quot;$.message&quot;).value(&quot;잘못된 요청입니다.&quot;))
                .andExpect(jsonPath(&quot;$.result.password&quot;).value(&quot;비밀번호는 8~12자의 영문, 숫자, 특수문자를 포함해야 합니다&quot;));
    }

    ...(더 많은 엣지케이스)</code></pre>
<p>이렇게 다양한 엣지케이스를 한 번에 검증 할 수 있게 된다.</p>
<p>이를 통해 느끼는 점은 두가지가 있다.</p>
<br />

<h3 id="테스트-케이스의-중복">테스트 케이스의 중복</h3>
<p>아무래도 통합테스트에서의 테스트 케이스는 단위테스트에서의 테스트 케이스와 중복 될 수 밖에 없다. 통합테스트에서의 에러를 단위 기능이 검증하기 때문이다.</p>
<p>앞서 말했듯이 레이어간의 상호작용이나 커버되지 못한 케이스에서의 에러가 발생 할 수도 있기 때문에 통합테스트가 필요하기도 하지만…그래도 너무 반복적인 코드 작성에 피로함을 느끼곤 하였다.</p>
<p>하지만 그럼에도 테스트가 반드시 필요한 이유는…..</p>
<br />

<h3 id="정책">정책</h3>
<p>이 서비스에 대한 개발은 과연 우리만 할까?</p>
<p>아니다 다른 개발자와 협업을 할 수도 있고, 내 자리를 다른 사람이 대신하게 될 지도 모른다. 그럼 그때 개발자가 내 코드를 보고 이 기능을 다 인지 할 수 있을까?
<br /></p>
<p><strong>단언컨데, 절대 못한다</strong>
<br /></p>
<p>특히나 자바와 같은 객체지향의 속성이 강한 언어와 프레임워크의 경우는 더 그렇다고 생각한다. 하나의 기능을 위한 요소가 곳곳에 흩어져 있기 때문에 정확히 참고하는데에 어려움이 있다.</p>
<p>이런 상황에 신규 개발자가 기능을 개발하고 이를 단순히 혼자서 스웨거를 통해 테스트하고 메인에 머지를 한다?</p>
<p><img alt="" src="https://velog.velcdn.com/images/gdbs1107/post/e894b54b-8660-435b-98c1-9964a9c39667/image.png" /></p>
<p>재앙이라고 생각한다. 그 기능이 정말 기존 기능과 완전히 분리되어 있다면 상관없겠지만, 그럴리 없을 뿐더러 그걸 누가 장담 할 수 있을까. 우리는 그 신규 개발자가 우리 서비스, 서버 코드에 대해서 완전히 이해하고 개발하기를 바란다. 그리고 그 지점을 테스트가 해결 할 수 있다.</p>
<br />

<hr />
<p>우리가 작성한 다양한 테스트 케이스는 하나의 <strong>정책</strong>이 된다. 회원의 username 패턴이나 중복검증 여부, 비밀번호 패턴과 필요한 다양한 요소들을 검증하는 그 테스트 케이스가 그 서버만의 정책이 되는 것이다.</p>
<p>그렇게 테스트를 잘 작성해 놓은 서버라면 신규 개발자가 오더라도 두려움이 덜 할 수 있다. 테스트 케이스를 쭉 점검하고 정책을 확인한 뒤에 자신이 개발할 부분을 개발하고 테스트를 돌려 자신의 기능이 무결함을 검증 받을 수 있기 때문이다.</p>
<p>그래서 테스트 코드는 중요하다고 생각하고 반드시 필요하다고 생각한다.</p>
<p><br /><br /></p>
<p>그리고 이러한 마음가짐은 나를 단순히 CRUD를 개발하는 개발자에서 유지보수를 생각하는 개발자로 성장할 수 있도록 도와주었고, 이는 곧 객체지향이 닿고자하는 목표와 같다는 것을 이해했다.</p>
<p>테스트를 공부하기 전까지는 객체지향이 무엇인지 정확히 알 지 못했다. 아니 모른다기 보단 이걸 왜 해야하는지 이해를 못했다. 하지만 우리의 서비스를 구성하는 다양한 요소가 언제든지 다른 것으로 대체 될 수 있다는 것을 알고 나서는 객체지향적이지 않도록 개발한 서버는 수명이 길 수 없다 라는 나만의 결론에 도달하였다.</p>
<p>물론 모든걸 객체지향 하는 것 보다는 서비스의 특성에 맞게 정도를 조절하는 것이 당연히 좋겠지만… 아무튼아무튼 자코코를 이용하든 TDD를 적용하든 BDD를 적용하든 테스트는 꼭 적용하는 우리가 되어보자.</p>
<p><br /><br /><br /></p>
<p><img alt="" src="https://velog.velcdn.com/images/gdbs1107/post/c09e9300-bdd0-46f5-bfe9-0f68a4377944/image.png" />
다시 개발하러갈게요</p>