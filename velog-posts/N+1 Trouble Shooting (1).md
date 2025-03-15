<h1 id="차례">차례</h1>
<p>-<a href="https://api.velog.io/rss/gdbs1107#%EB%B0%B0%EA%B2%BD">배경</a>
-<a href="https://api.velog.io/rss/gdbs1107#N+1-%EC%9D%B4%EB%9E%80">N+1 이란</a>
-<a href="https://api.velog.io/rss/gdbs1107#N+1%EC%9D%80-%EC%99%9C-%EB%B0%9C%EC%83%9D%ED%95%A0%EA%B9%8C">N+1은 왜 발생할까</a>
-<a href="https://api.velog.io/rss/gdbs1107#N+1-%ED%95%B4%EA%B2%B0">N+1 해결</a></p>
<h1 id="배경">배경</h1>
<p>이번 방학 프로젝트를 진행하며 SQL 최적화에 대해 다시 한 번 생각해보게 되었다.</p>
<p>이전엔 무관심하게 생각했던 쿼리의 발생량으로 인해서 메인페이지의 조회가 생각보다도 너무 오래걸린다는 것을 알게되었고,
이를 해결하기 위해서 QueryDSL 도입을 결정하였지만 QueryDSL은 근본적인 해결책이 아니라는 사실이 기억이 났다.</p>
<br />

<p>QueryDSL은 JPQL의 빌더 역할을 해줄 뿐이다. (물론 이것만으로도 매우 유용하지만) 그러면 JPQL은 왜 사용할까?
JPA의 기능적 맹점으로 인해 개발자들이 쿼리를 의도대로 커스텀 해야하는 필요성이 대두되었고 그에따라 개발된 기능이 JPQL이다.</p>
<p>JPQL의 강력한 기능인 fetch join으로 성능 병목현상의 90프로를 차지하는 N+1을 크게 개선 할 수 있는데,</p>
<blockquote>
<p>난 N+1 문제에 대해서 제대로 대답 할 수 있는가?</p>
</blockquote>
<p>로 부터 시작된 이번 시리즈</p>
<br />

<ol>
<li>fetch join을 이용한 N+1 문제 해결 </li>
<li>fetch join의 한계와 돌파 </li>
<li>QueryDSL로의 리팩터링</li>
</ol>
<br />
로 성능을 최적화해보자

<p><br /><br /></p>
<h1 id="n1-이란">N+1 이란</h1>
<p>N+1에 대해 먼저 정의해보자면</p>
<p>하나의 엔티티, 그리고 그와 매핑된 엔티티의 정보를 조회해야하는 경우에 그 엔티티의 데이터 수 만큼의 쿼리가 추가적으로 발생하는 것을 의미한다.</p>
<p>말로는 어려우니 쿼리로 직접 보자</p>
<pre><code>@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Setter
@ToString(of = {&quot;id&quot;,&quot;username&quot;,&quot;age&quot;})
public class Member {

    @Id
    @GeneratedValue
    private Long id;

    private String username;

    private int age;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = &quot;team_id&quot;)
    private Team team;</code></pre><br />

<pre><code>@Entity
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@ToString(of = {&quot;id&quot;,&quot;name&quot;})
public class Team {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @OneToMany(mappedBy = &quot;team&quot;)
    List&lt;Member&gt; members = new ArrayList&lt;&gt;();</code></pre><br />

<p>일대다로 매핑된 회원과 팀 엔티티가 있다.
여기서 회원의 username과 소속 팀의 name을 조회하고 싶다면</p>
<br />



<pre><code>    @GetMapping(&quot;/all-members&quot;)
    public List&lt;TestDTO&gt; getAllMembers() {
        List&lt;Member&gt; members = memberRepository.findAll();

        return members.stream()
                .map(member -&gt; TestDTO.builder()
                        .memberName(member.getUsername())
                        .teamName(member.getTeam().getName())
                        .build())
                .collect(Collectors.toList());
    }</code></pre><p>이런 식으로 조회 할 수 있을 것이다.</p>
<p>이렇게 해서 회원이 두 명일 경우의 조회를 하면 쿼리가</p>
<br />

<pre><code>2025-03-14T09:07:41.445+09:00 DEBUG 23630 --- [nio-8080-exec-9] org.hibernate.SQL                        : 
    select
        m1_0.id,
        m1_0.age,
        m1_0.team_id,
        m1_0.username 
    from
        member m1_0
2025-03-14T09:07:41.454+09:00 DEBUG 23630 --- [nio-8080-exec-9] org.hibernate.SQL                        : 
    select
        t1_0.id,
        t1_0.name 
    from
        team t1_0 
    where
        t1_0.id=?
2025-03-14T09:07:41.456+09:00 DEBUG 23630 --- [nio-8080-exec-9] org.hibernate.SQL                        : 
    select
        t1_0.id,
        t1_0.name 
    from
        team t1_0 
    where
        t1_0.id=?</code></pre><br />

<p>이렇게 발생된다</p>
<br />

<p><img alt="" src="https://velog.velcdn.com/images/gdbs1107/post/a002e477-9dbe-4404-b32d-a63322622c76/image.png" /></p>
<p>이상하다 쿼리는 회원 조회 (1번) + 팀 조회 (1번) = 2번 발생해야 하는데</p>
<p>실제 쿼리는 세 번이 발생하였다.
만약 회원이 1000000명이었다면 1000001번 발생했을 것이다.</p>
<p>이것이 N+1 트러블이다.</p>
<p><br /><br /></p>
<h1 id="n1은-왜-발생할까">N+1은 왜 발생할까</h1>
<p>이런 현상은 왜 발생하는 걸까?</p>
<p>주요 요인으로는 조회해야하는 엔티티와 매핑된 데이터를 조회 할 때 발생하는 것인데, 다시 엔티티를 보면</p>
<pre><code>    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = &quot;team_id&quot;)
    private Team team;</code></pre><p>이렇게 LAZY (지연로딩)으로 매핑이 되어 있는 것을 볼 수 있다.</p>
<br />

<p>지연로딩이란, 데이터를 조회 할 때 그와 연결된 데이터를 한 번에 조회 하는 것이 아닌 필요할 때만 실제로 조회하여 영속성 객체에 저장하고 매핑데이터는 프록시 객체로서 조회하는 것을 의미한다.</p>
<p>이렇게 함으로써 일반적으로는 쿼리를 크게 줄일 수 있어 유용하지만 Join 쿼리의 경우는 다르다.
2개의 데이터가 존재하는 조회 쿼리가 발생하면 2개의 데이터 조회 시, 매번 매핑된 데이터를 조회하는 문제가 발생하여 조회되는 데이터(N개) 만큼의 쿼리가 추가로 터지는 것이다.</p>
<p>만약 조회해야하는 매핑 엔티티가 세 개라면 1+N+N+N......</p>
<p><br /><br /></p>
<h1 id="n1-해결">N+1 해결</h1>
<p>N+1 문제를 해결하는 방법은 간단하다.
매핑된 데이터를 조회 시, 한 번에 가져오면 된다. 이를 위해서 직관적인 방법은 매핑을 EAGER를 통해서 하면 된다 싶지만 문제가 있다.</p>
<br />

<ol>
<li><p>예측치 못한 쿼리 발생
즉시 로딩으로 데이터를 조회하면 쿼리가 마구잡이로 나가게 된다. 얘 조회했다가 얘 조회했다가...
이는 동적인 상황에 문제를 발생 시킬 수 있고 쿼리를 추적하는 것도 매우 어려워진다는 단점이 존재한다</p>
</li>
<li><p>지연로딩의 장점을 포기
지연로딩으로서 얻을 수 있는 많은 장점을 이로 인해서 포기하게 된다. 이는 득보다 실이 많은 선택이라고 할 수 있다</p>
</li>
</ol>
<br />

<p>그래서 우리는 JPQL의 강력한 무기인 fetch join을 이용한다. 이는 쿼리 속에 있는 데이터만 즉시 조회하여 한 방에 쿼리를 날릴 수 있는 방법이다.</p>
<p>코드로 예시를 들어보면</p>
<pre><code>    @Query(&quot;select m from Member m join fetch m.team&quot;)
    List&lt;Member&gt; findAllWithTeam();</code></pre><br />

<pre><code>    @GetMapping(&quot;/all-members2&quot;)
    public List&lt;TestDTO&gt; getAllMembers2() {
        List&lt;Member&gt; members = memberRepository.findAllWithTeam();

        return members.stream()
                .map(member -&gt; TestDTO.builder()
                        .memberName(member.getUsername())
                        .teamName(member.getTeam().getName())
                        .build())
                .collect(Collectors.toList());
    }</code></pre><p>이렇게 team 데이터를 fetch join으로 조회하게 된다. 그러면 쿼리는</p>
<br />


<pre><code>2025-03-14T09:07:59.805+09:00 DEBUG 23630 --- [nio-8080-exec-1] org.hibernate.SQL                        : 
    select
        m1_0.id,
        m1_0.age,
        t1_0.id,
        t1_0.name,
        m1_0.username 
    from
        member m1_0 
    join
        team t1_0 
            on t1_0.id=m1_0.team_id</code></pre><p>이렇게 단 하나만 발생한다.</p>
<p>회원 데이터 두 개에 매핑데이터 하나였기 때문에 극적인 변화가 아니어보일 수 있지만 실제 서비스였다면 정말 큰 성능 최적화가 됐을 것이다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/gdbs1107/post/f887ffb3-ce64-44b1-b965-e6c1d96e88a0/image.png" /></p>
<p>자 이제 N+1이 모두 해결 된 것 같지만...
N+1도 무적은 아니라는 점..</p>
<ol>
<li>fetch join이 된 메서드를 이용하면 페이징이 불가능하고</li>
<li>양방향 매핑관계에 있어서는 fetch join을 자유롭게 쓸 수 없다</li>
</ol>
<p><br /><br /></p>
<p>이러한 문제점은 다음 포스팅에서 자세히 서술해보도록 하겠다</p>
<p><br /><br /></p>