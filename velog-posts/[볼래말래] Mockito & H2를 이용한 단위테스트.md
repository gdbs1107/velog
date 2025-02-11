<br />

<p><img alt="" src="https://velog.velcdn.com/images/gdbs1107/post/860059c8-c0c6-4bcb-aac4-f5cf289bd6b5/image.png" /></p>
<p><br /><br /><br /></p>
<h2 id="1-헥사고날-못하겠어요ㅜㅜ">1. 헥사고날 못하겠어요ㅜㅜ</h2>
<p><img alt="" src="https://velog.velcdn.com/images/gdbs1107/post/006ff402-a71d-4553-84ec-7fc62e8a35e1/image.png" /></p>
<p>어딜가나 헥사고날을 떠들던 내가 아키텍쳐를 버리게 될 줄이야</p>
<p>연유는 다음과 같았다</p>
<h3 id="jpa와의-이별">JPA와의 이별</h3>
<p>아키텍쳐에 너무 신경쓰는 나머지, 개발 속도를 잡을 수 없었다. 그 중 날 가장 힘들게 만든 것은 JPA의 많은 부가기능을 직접적으로 사용 할 수 없게 된다는 것이었다.</p>
<p>이는 장기적으로는 외부의 의존성을 끊어낼 수 있다는 큰 장점이었지만, 난 아직도 JPA에 대해 제대로 이해하지 못했구나 하는 포인트가 너무나도 많았다.</p>
<p>특히나 변경감지의 경우, 습관적으로 이용하던 것을 직접 로직에 넣어야하니 수시로 예상치 못한 오류가 발생하였다. 여러 도메인이 얽혀있는 로직의 경우는 고작(하지만 대단한) 변경감지를 디버깅하느라 긴 시간을 보내기도 하였다.
<br /></p>
<h3 id="매핑-레이어의-등장">매핑 레이어의 등장</h3>
<pre><code class="language-java">// movie 엔티티
@OneToMany
List&lt;MoviePosterEntity&gt; moviePosters = new ArrayList&lt;&gt;();</code></pre>
<pre><code class="language-java">
// movie 도메인
List&lt;MoviePoster&gt; moviePosters = new ArrayList&lt;&gt;();</code></pre>
<p>과연 둘 중 어디서 배열의 초기화가 되어야 할까?
배열의 초기화 후에는 이를 매핑 할 수 있는 메서드가 추가도 해야하지 않을까?</p>
<p>그런 관계가 하나면 다행이지만 보통의 서비스는 일대다,다대일 등 다양한 매핑관계로 엮여있다. 그 중 순환참조라도 발생한다면?
<br />
제대로 분리해놓지 않았다면 에러를 잡아내는데에 상당한 비용이 소모된다.</p>
<p>그래서 도메인과 영속성 객체를 매핑레이어를 통해 분리하여 이를 구현하고자 했었다. 
<br />
그치만 이렇게 구현하고 나니 매핑레이어가 영속성 객체에 의존하게 되고 결국 다른 벤더의 제품을 사용하게 되었을 때 수정사항이 더욱 늘어나게 됐다.</p>
<p>구현속도가 몇배 느려진 것은 덤이다.
<br /><br /></p>
<h3 id="테스트의-불변함">테스트의 불변함?</h3>
<p>헥사고날 아키텍쳐를 통해 서비스를 구현했을 때의 큰 장점 중 하나는 Mock 객체의 실행 로직을 직접 구현하여 테스트 구현 시 Mockito &amp; H2와의 강결합을 피할 수 있게 된다는 점이었다.</p>
<p>이는 결과적으로 외부의 요인으로 인해서 발생하는 테스트의 실패나 실행 시간을 줄여준다는 장점에 도달 할 수 있었지만, 우선 난 <strong>Mock의 정확한 실행 원리도 모를 뿐더라 Mock을 사용해본 적도 없었다.</strong>
<br />
Mockito는 커녕 정확한 단위테스트의 기준을 알지도 못하니 당연히 Mock객체를 직접 만들던 어쩌던 테스트는 엉망일 수밖에 없었다.</p>
<p>테스트는 로직이 추가될 때마다 터지기 일수였고, 로직 구현보다 아득히 많은 시간을 테스트를 보수하는데에 사용해야 했다.
<br />
이러한 이유로 나와 동료 개발자간의 협의 끝에 다시 레이어 아키텍쳐로 돌아가고, Mockito와 H2를 적극 활용한 테스트를 구현하기로 했다.</p>
<p>그리고 왜 이러한 구조가 문제인지 직접 느끼고 파악해보는 것으로 결정하였다.
<br /><br /></p>
<hr />
<h2 id="2-서비스-레이어-테스트">2. 서비스 레이어 테스트</h2>
<pre><code class="language-java">    @Override
    public MemberJoinDTO.MemberJoinResponseDTO joinMember(@Valid MemberJoinDTO.MemberJoinRequestDTO request){

                // 필수 약관동의 검증
        authenticateAgreement(request);
        // username 중복 검증
        authenticateUsernameValid(request.getUsername());

        String encodedPassword = bCryptPasswordEncoder.encode(request.getPassword());
        Member newMember= MemberConverter.toMember(request,encodedPassword);
        Member savedMember = memberRepository.save(newMember);

        Agreement newAgreement = Agreement.JoinDTOto(request,savedMember);
        agreementRepository.save(newAgreement);

        return MemberJoinDTO.MemberJoinResponseDTO.builder()
                .memberId(savedMember.getId())
                .build();
    }</code></pre>
<p>회원가입을 위한 메서드이다. 이를 어떻게 테스트 코드로 구현 할 수 있을까?</p>
<pre><code class="language-java">@ActiveProfiles(&quot;test&quot;)
@ExtendWith(MockitoExtension.class)
class MemberServiceImplTest {

    @Mock
    MemberRepository memberRepository;

    @Mock
    AgreementRepository agreementRepository;

    @Mock
    BCryptPasswordEncoder bCryptPasswordEncoder;

    @Mock
    LocalDateHolder localDateHolder;

    @InjectMocks
    MemberServiceImpl memberService;

    MemberProfileImage memberProfileImage;

    Member testMember;

    }</code></pre>
<p>우선 필요한 의존성을 한 번에 주입받자</p>
<pre><code class="language-java">@ActiveProfiles(&quot;test&quot;)
@ExtendWith(MockitoExtension.class)</code></pre>
<p>해당 클래스가 실행될때 사용할 프로필을 test로 설정하고</p>
<p>Mockito를 사용 할 수 있도록 어노테이션을 추가해준다
<br /></p>
<pre><code class="language-java">    @Mock
    MemberRepository memberRepository;

    @Mock
    AgreementRepository agreementRepository;

    @Mock
    BCryptPasswordEncoder bCryptPasswordEncoder;

    @Mock
    LocalDateHolder localDateHolder;

    @InjectMocks
    MemberServiceImpl memberService;</code></pre>
<p>의존성 주입이 필요한 대상을 @InjectMock을 이용하여 선언하고</p>
<p>그에 필요한 의존성을 @Mock을 통해 자동 주입한다</p>
<p>이렇게 주입된 여러 객체들은 실제 service가 아니라 프록시 객체로서 작동한다. 즉 반환되는 로직을 Mockito 메서드를 이용하여 직접 구현해줘야한다.</p>
<br />

<pre><code class="language-java"> @Test
    @DisplayName(&quot;joinMember() 메서드를 이용하여 회원가입을 할 수 있다&quot;)
    public void joinMember_test(){
        // given
        MemberJoinDTO.MemberJoinRequestDTO requestDTO = MemberJoinDTO.MemberJoinRequestDTO.builder()
                .username(&quot;test123&quot;)
                .password(&quot;Test123!&quot;)
                .name(&quot;test&quot;)
                .gender(Gender.MALE)
                .birthDate(LocalDate.of(1995, 5, 20))
                .email(&quot;test@example.com&quot;)
                .phoneNumber(&quot;010-1234-5678&quot;)
                .serviceAgreement(true)
                .privacyAgreement(true)
                .financialAgreement(true)
                .advAgreement(false)
                .build();

        Member mockMember = Member.builder()
                .id(1L)
                .username(&quot;testUser&quot;)
                .password(&quot;encodedPassword&quot;)
                .build();

        Agreement mockAgreement = Agreement.builder()
                .id(1L)
                .member(mockMember)
                .serviceAgreement(true)
                .build();

        ㅊ
        when(memberRepository.save(Mockito.any(Member.class))).thenReturn(mockMember);
        when(agreementRepository.save(Mockito.any(Agreement.class))).thenReturn(mockAgreement);

        // when
        MemberJoinDTO.MemberJoinResponseDTO responseDTO = memberService.joinMember(requestDTO);

        // then
        assertThat(responseDTO).isNotNull();
        assertThat(responseDTO.getMemberId()).isEqualTo(mockMember.getId());

        // Verify
        verify(memberRepository, Mockito.times(1)).save(Mockito.any(Member.class));
    }</code></pre>
<p>보면 실제 로직에서 사용되는 다양한 메서드에 대한 로직이
<br /></p>
<pre><code class="language-java">when(bCryptPasswordEncoder.encode(anyString())).thenReturn(&quot;encodedPassword&quot;);
when(memberRepository.save(Mockito.any(Member.class))).thenReturn(mockMember);
when(agreementRepository.save(Mockito.any(Agreement.class))).thenReturn(mockAgreement);</code></pre>
<p>이런식으로 구현되어 있는 것을 알 수 있다.</p>
<p>모킹된 객체는 메서드를 호출할 뿐, 실제 동작까지는 할 수 없기 때문에 null을 포함해도 문제가 생기지 않는 로직이라면 이렇게 실제 로직을 구현해줘야한다.
<br /></p>
<p>바로 여기서 내가 가졌던 의문이 발생한다.</p>
<p><strong>그러면 이 테스트는 내가 실제 서비스 로직에 무슨 일을 해도 깨지지 않는 코드가 되는거 아닌가?</strong></p>
<p>맞다. 아마 내가 when으로 구현한 메서드들의 동작방식이 달라져도 테스트는 깨지지 않을 것이다. 테스트에서는 그 메서드들의 동작을 모두 모킹하고 있기 때문이다.</p>
<p><strong>그리고 바로 그게 단위테스트라는 것을 깨달았다.</strong></p>
<p><img alt="" src="https://velog.velcdn.com/images/gdbs1107/post/a02369cb-c7c8-4037-99f6-34c5d0fabc6e/image.png" /></p>
<p>단위 테스트에서 리파지토리 메서드가 제대로 동작하는지 알아야 할까? BCrypt 메서드가 제대로 실행되는지 알아야할까?</p>
<p>아니다. 내가 구현한 서비스 로직 안에서 필요한 로직만 제대로 실행되고 정해진 예외를 반환 할 수 있는지만 검증하면 된다. </p>
<p>그 외의 메서드들의 테스트는 그 메서드의 단위테스트에서 검증하면 되는 것이다.</p>
<p>그리고 Mockito는 얘가 없었으면 전부 내가 구현하여야 했을 리파지토리나 주입된 클래스의 메서드를 간편하게 동작하게 할 수 있도록 도와주는 라이브러리인 것이다.</p>
<hr />
<p><br /><br /></p>
<h2 id="3-리파지토리-레이어-테스트">3. 리파지토리 레이어 테스트</h2>
<p>그러면 이를 이용하여 간단한 리파지토리 레이어도 테스트해보자</p>
<pre><code class="language-java">@Query(&quot;SELECT m FROM Member m WHERE m.status = :status AND m.inactiveDate &lt;= :cutoffDate&quot;)
    List&lt;Member&gt; findInactiveMembersForDeletion(Status status, LocalDateTime cutoffDate);</code></pre>
<p>이 메서드의 경우 비활성화 상태가 지정일 만큼 지난 회원을 삭제하는 메서드이다
<br />
이를 통합테스트, 혹은 mock 객체를 spy객체로 변환하여 실제 메서드를 호출하는 등의 테스트를 하면 너무 많은 메서드가 지나서 해당 메서드에 도달하기 때문에 해당 메서드가 제대로 동작한 것인지 파악 할 수 없다.</p>
<p>또한 실제 로직 상에서 @Schedule을 이용해서 메서드가 실행되기 때문에 더더욱 통합테스트가 힘든 상황이다. </p>
<p>그렇기 때문에 정확한 메서드의 기능을 확인하기 위해서 <strong>단위테스트</strong>의 도입이 필요하다.
<br /></p>
<p><img alt="" src="https://velog.velcdn.com/images/gdbs1107/post/6ac80dc8-e15b-4bc5-ba77-a64e81e7609a/image.png" /></p>
<p>우선 작성 전에 H2를 연동해주도록 하자. 굳이 H2는 아니어도 좋지만 인메모리에서 작동하도록 설정 할 수 있어 속도가 빠르기 때문에 테스트용 DB로 선호된다고 한다.</p>
<br />


<pre><code class="language-java">@DataJpaTest
class MemberRepositoryTest {

    @Autowired
    MemberRepository memberRepository;
    }</code></pre>
<pre><code class="language-java">@DataJpaTest</code></pre>
<p>해당 어노테이션은 테스트 환경에서 Spring 설정을 제외한 데이터베이스 연동에 필요한 다양한 설정값을 연동해준다. DB를 테스트 할때 주로 사용되는 어노테이션이다.</p>
<p>여기에 실제 MemberRepository를 @AutoWired를 통해 주입하여 메서드에 사용한다.</p>
<pre><code class="language-java">    @Test
    @DisplayName(&quot;findInactiveMembersForDeletion() 를 이용하여 비활성화 상태가 30일이 넘은 회원을 조회할 수 있다&quot;)
    public void findInactiveMembersForDeletion_success() {
        // given
        LocalDateTime now = LocalDateTime.of(2024, 2, 1, 0, 0);
        LocalDateTime cutoffDate = now.minusDays(30);

        Member member1 = Member.builder()
                .username(&quot;test1&quot;)
                .password(&quot;Test123!&quot;)
                .name(&quot;test&quot;)
                .role(Role.ROLE_USER)
                .phoneNumber(&quot;010-1234-5678&quot;)
                .birthday(LocalDate.of(1995, 5, 20))
                .email(&quot;test1@example.com&quot;)
                .status(Status.INACTIVE)
                .gender(Gender.MALE)
                .alarmAccount(0)
                .bookmarkAccount(0)
                .subStatus(SubStatus.UNSUBSCRIBE)
                .inactiveDate(now.minusDays(31))
                .build();

        Member member2 = Member.builder()
                .username(&quot;test2&quot;)
                .password(&quot;Test123!&quot;)
                .name(&quot;test&quot;)
                .role(Role.ROLE_USER)
                .phoneNumber(&quot;010-1234-5678&quot;)
                .birthday(LocalDate.of(1995, 5, 20))
                .email(&quot;test2@example.com&quot;)
                .status(Status.INACTIVE)
                .gender(Gender.MALE)
                .alarmAccount(0)
                .bookmarkAccount(0)
                .subStatus(SubStatus.UNSUBSCRIBE)
                .inactiveDate(now.minusDays(40))
                .build();

        Member member3 = Member.builder()
                .username(&quot;test3&quot;)
                .password(&quot;Test123!&quot;)
                .name(&quot;test&quot;)
                .role(Role.ROLE_USER)
                .phoneNumber(&quot;010-1234-5678&quot;)
                .birthday(LocalDate.of(1995, 5, 20))
                .email(&quot;test3@example.com&quot;)
                .status(Status.INACTIVE)
                .gender(Gender.MALE)
                .alarmAccount(0)
                .bookmarkAccount(0)
                .subStatus(SubStatus.UNSUBSCRIBE)
                .inactiveDate(now.minusDays(25))
                .build();

        memberRepository.save(member1);
        memberRepository.save(member2);
        memberRepository.save(member3);

        // when
        List&lt;Member&gt; result = memberRepository.findInactiveMembersForDeletion(Status.INACTIVE, cutoffDate);

        // then
        assertThat(result).hasSize(2);
        assertThat(result).extracting(Member::getUsername)
                .containsExactlyInAnyOrder(&quot;test1&quot;, &quot;test2&quot;);
    }</code></pre>
<p>그 후 Junit5를 적극 사용하여 테스트 로직을 작성해준다.</p>
<p>실제 Repository를 주입받아 사용하므로 모킹의 여지가 없다는 점이 특징이다.</p>
<p>이를 통해 해당 메서드의 독립적인 안정성이 보장된다.</p>
<p><br /><br /></p>
<hr />
<h2 id="4-후기">4. 후기</h2>
<p>우리는 이를 통해 각각의 계층에서의 안정성을 보장하고 추후 코드의 변동이 생길때 모든 테스트를 돌려보며 정확히 어느 포인트에서 문제가 발생하고 있는지 파악할 수 있어졌다.</p>
<p>하지만 여기서 끝이 아닌, 우리는 테스트 코드를 작성하며 무엇이 객체지향적인 코드인지도 배울 수 있게되었다.</p>
<h3 id="숨어있는-의존성-발견">숨어있는 의존성 발견</h3>
<p>위의 회원가입 서비스 테스트 코드 중에서</p>
<pre><code class="language-java">when(bCryptPasswordEncoder.encode(anyString())).thenReturn(&quot;encodedPassword&quot;);</code></pre>
<p>해당 메서드를 단독으로 테스트 할 수 있을까?</p>
<p>LocalDate.now(), UUID 등 호출마다 동적으로 달라지는 데이터를 검증 할 수 있을까?</p>
<p>물론 다양한 Mock 메서드들을 떡칠하면 불가능하지 않지만, 이렇게 Mock에 강하게 의존하면 추후 리팩터링이나 예측치 못한 변수에 취약해 질 수 있다.</p>
<p>이를 해결하기 위해서 우리는 무엇을 할 수 있을까?</p>
<br />


<p><strong>의존성 역전을 통해 BCryptPasswordEncoder를 추상화하고</strong>
<strong>의존성 주입을 통해 해당 회원가입 메서드에 주입하고</strong>
<strong>그리고 테스트 상에서는 추상화된 BCrypt를 직접 구현하여 해싱할때 실제 BCrypt가 아닌 임의의 값을 반환하도록 모킹한다.</strong></p>
<p>우리는 이를 통해 하나의 포인트를 더 잡을 수 있게된다.
<br /></p>
<h3 id="테스트와-객체지향">테스트와 객체지향</h3>
<p>위의 예시에서 보았듯이, 테스트가 불가능한 코드가 존재하고 이는 저 의존성은 대체가 불가능한 메서드임을 의미한다. 변동과 확장에 매우 제한적인 의존성인 것이다.</p>
<p>테스트에서는 이를 반영한다. 무언가 테스트에서 구현할때 너무 지엽적이게 된다면 의심 할 수 있다.</p>
<br />


<p>repository의 경우도 그렇다. 현재 리파지토리 테스트에서 @DataJPA를 제외하고 테스트가 가능한가?</p>
<p>불가능하다. 있어도 너무 지엽적인 방법일 것이다.</p>
<p>이를 이전 헥사고날 포스팅과 같이 repository 클래스를 한 번 더 추상화 할 수 있다면, Mockito, H2와 분리된 테스트가 가능해진다.</p>
<br />


<p>그리고 이는 곧 해당 메서드는 JPA에 의존적이지 않고 거기에 정상적으로 작동 할 수 있는 다른 외부 의존성이 붙어도 구현이 된다는 것을 의미하고</p>
<p>이는 곧 변경과 확장에 용이한 유지보수에 특화된 객체지향적 서비스 임을 의미하게 된다.</p>
<p><br /><br /><br /><br /><br /><br /><br /></p>
<p><img alt="" src="https://velog.velcdn.com/images/gdbs1107/post/beba088f-a524-400f-a392-9a29a2d770b0/image.png" />
다음엔 통합테스트와 자코코에 대한 글을 써보도록 하겠습니다.</p>