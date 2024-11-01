<h2 id="cicd-구현">CI/CD 구현</h2>
<br />


<p>github actions를 이용하여 CI/CD를 실행 시킬 수 있는 스크립트를 작성해줍니다</p>
<pre><code>name: DOCKER_CD

on:
  push:
    branches: [ &quot;main&quot;, &quot;develop&quot; ]

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest # ubuntu 최신 버전에서 script를 실행
    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: &quot;adopt&quot;

      # application.yml 생성 (dev, release에 따라 다른 application 추가 해야함)
      # 환경변수 전부 따로 관리하고 쓰기 VS application.yml 직접 넣기 뭐가 좋을지 고민해보기
      - name: Make application.yml
        run: |
          mkdir -p src/main/resources
          echo &quot;${{ secrets.APPLICATION }}&quot; &gt; src/main/resources/application-dev.yml

      - name: Build with Gradle
        run: |
          chmod +x ./gradlew
          ./gradlew clean build -x test

      - name: Docker BUILD_PUSH
        run: |
          docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}
          docker build -f Dockerfile -t ${{ secrets.DOCKER_REPO }} .
          docker push ${{ secrets.DOCKER_REPO }}


      # 현재는 도커만 사용한 버전, 추후 도커 컴포즈를 사용하게 되면 많이 수정해야함
      # ELB와 함께 사용한다면 어떻게 수정해야할까
      - name: Deploy_EC2
        uses: appleboy/ssh-action@master
        id: deploy
        with:
          host: ${{ secrets.HOST }}
          username: ubuntu
          key: ${{ secrets.KEY }}
          script: |
            sudo docker rm -f $(sudo docker ps -qa) || true
            sudo docker pull ${{ secrets.DOCKER_REPO }}
            sudo docker run -d -p 8080:8080 --name myapp ${{ secrets.DOCKER_REPO }}
            sudo docker image prune -f
</code></pre><p>스크립트의 실행단계는 다음과 같습니다
<br /><br /></p>
<pre><code>on:
  push:
    branches: [ &quot;main&quot;, &quot;develop&quot; ]
</code></pre><ul>
<li>main과 develop브랜치에 푸쉬가 될 때 깃허브 액션이 실행되도록 합니다</li>
<li>해당 트리거는 실제 프로젝트 진행시 수정 될 가능성이 높습니다
<br /><br /></li>
</ul>
<pre><code>jobs:
  build:
    runs-on: ubuntu-latest # ubuntu 최신 버전에서 script를 실행
    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: &quot;adopt&quot;
</code></pre><ul>
<li>깃허브에서 필요한 파일을 체크아웃합니다</li>
<li>이로써 깃허브에 있는 코드와 파일을 깃허브 액션에서 사용 할 수 있게 됩니다</li>
<li>JDK 17을 설치해줍니다
<br /><br /></li>
</ul>
<pre><code># application.yml 생성 (dev, release에 따라 다른 application 추가 해야함)
      # 환경변수 전부 따로 관리하고 쓰기 VS application.yml 직접 넣기 뭐가 좋을지 고민해보기
      - name: Make application.yml
        run: |
          mkdir -p src/main/resources
          echo &quot;${{ secrets.APPLICATION }}&quot; &gt; src/main/resources/application-dev.yml</code></pre><ul>
<li>application.yml을 직접 주입해줍니다</li>
<li>해당 부분에서 환경변수를 따로 관리하여 넣어주는게 좋을지, 아니면 application.yml을 통째로 주입해주는게 좋을지에 대해 고민이 있습니다</li>
<li>평소에는 따로 관리하였지만 이번에는 통째로 주입하는 방식을 사용하였습니다
<br /><br />
```</li>
<li>name: Build with Gradle<pre><code>  run: |
    chmod +x ./gradlew
    ./gradlew clean build -x test</code></pre>```</li>
<li>코드를 빌드합니다
<br /><br />
```</li>
<li>name: Docker BUILD_PUSH<pre><code>  run: |
    docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}
    docker build -f Dockerfile -t ${{ secrets.DOCKER_REPO }} .
    docker push ${{ secrets.DOCKER_REPO }}</code></pre>```</li>
<li>빌드한 코드에 대한 도커 이미지를 레포에 푸시합니다</li>
<li>이때 사용되는 도커파일은 다음과 같습니다<br />

</li>
</ul>
<pre><code>FROM openjdk:17-jdk-slim

COPY build/libs/docker_cicd-0.0.1-SNAPSHOT.jar app.jar

CMD [&quot;java&quot;, &quot;-Dspring.profiles.active=dev&quot;, &quot;-jar&quot;, &quot;app.jar&quot;]</code></pre><ul>
<li>현재 application.yml의 profile을 dev라는 이름으로 관리하고 있습니다</li>
<li><strong>프로필이 변경 될 시, 도커파일을 알맞게 수정해줘야 합니다</strong>
<br /><br /><pre><code># 현재는 도커만 사용한 버전, 추후 도커 컴포즈를 사용하게 되면 많이 수정해야함
    # ELB와 함께 사용한다면 어떻게 수정해야할까
    - name: Deploy_EC2
      uses: appleboy/ssh-action@master
      id: deploy
      with:
        host: ${{ secrets.HOST }}
        username: ubuntu
        key: ${{ secrets.KEY }}
        script: |
          sudo docker rm -f $(sudo docker ps -qa) || true
          sudo docker pull ${{ secrets.DOCKER_REPO }}
          sudo docker run -d -p 8080:8080 --name myapp ${{ secrets.DOCKER_REPO }}
          sudo docker image prune -f</code></pre></li>
<li>만들어놓은 EC2에 원격접속하여 만들어놓은 도커 이미지를 pull하여 실행까지 시키고 스크립트를 마무리합니다</li>
</ul>
<p><br /><br /><br /><br /><br /></p>
<h2 id="trouble-shooting">TROUBLE SHOOTING</h2>
<br />

<h3 id="도커가-ec2에-설치가-안되어있어요">도커가 EC2에 설치가 안되어있어요</h3>
<ul>
<li>처음 EC2 서버를 개설 한 후 그냥 순수한 서버로는 바로 사용 할 수 없다</li>
<li>도커를 EC2 서버에 직접 접속하여 도커를 설치해주어야한다<br /></li>
<li>이러한 기능도 스크립트에 넣을 수 있지만 그러면 설치가 되어있는지 검증하고 설치를 하는 스크립트를 추가적으로 작성해야한다</li>
<li>이는 불필요한 스크립트의 증가 -&gt; 빌드 시간의 증가이므로 그냥 처음에 직접 설치하고 끝내도록 하자
<br /><br /><h3 id="도커에-접근-할-수-있는-권한이-없어요">도커에 접근 할 수 있는 권한이 없어요</h3>
</li>
<li>EC2가 도커에 접근 할 수 있는 권한이 없어서 생기는 오류<pre><code>sudo usermod -aG docker $USER</code></pre></li>
<li>코드를 추가해서 권한을 부여한다
<br /><br /><h3 id="pem키를-제대로-복사하는-방법">pem키를 제대로 복사하는 방법</h3>
</li>
<li>pem키는 보안처리 때문에 인터페이스에서 직접 접근이 불가능하다</li>
<li>터미널에서 직접 pem을 cat 파일관리자를 통해 열람하여 복사해서 사용하자
<br /><br /><h3 id="도커의-실행-로그를-잘-보아야해요">도커의 실행 로그를 잘 보아야해요</h3>
</li>
<li>docker logs 이미지명</li>
<li>해당 명령어로 실행되고 있는 이미지에 대한 로그를 확인 할 수 있다</li>
<li>RDS를 추가하든 로그를 확인해야하는 작업을 한다면 이렇게 확인하도록 하자
<br /><br /><br /><br /><br /><h2 id="cicd-develop-idea">CI/CD DEVELOP IDEA</h2>
<br />

</li>
</ul>
<h3 id="elastic-beanstalk를-이용한-cicd와-비교했을때-어떤-방법이-보다-효율적일까">Elastic Beanstalk를 이용한 CI/CD와 비교했을때 어떤 방법이 보다 효율적일까?</h3>
<ul>
<li><p>이전에 CI/CD를 Elastic Beanstalk를 이용하여 구현해본 경험이 있다</p>
</li>
<li><p>넣어야할 코드도 훨씬 많고 구현도 복잡하였다</p>
</li>
<li><p>하지만 로드밸런서와 롤링의 구현까지도 자동화 된다는 점에서 우수한 점이 존재한다</p>
</li>
<li><p>두 개를 한 번에 사용하는 방법도 존재하기 때문에 이러한 방법도 연구해봐야겠다
<br /><br /></p>
<h3 id="docker-compose">Docker Compose?</h3>
</li>
<li><p>내가 아는 도커 컴포즈는 여러개의 빌드를 묶어 한 번에 빌드를 할 수 있도록 도와주는 도커 툴이다</p>
</li>
<li><p>하지만 MSA로 다중 서버를 구축한 것이 아닌 시점에서 굳이 도커 컴포즈까지 필요할까 하는 생각이 든다</p>
</li>
<li><p>RDS도 application.yml에서 주입하기 때문에 따로 이미지 빌드도 필요없고...</p>
</li>
<li><p>이에 대해서는 나중에 공부해봐야겠다
<br /><br /></p>
<h3 id="왜-docker를-쓰라는거야">왜 Docker를 쓰라는거야?</h3>
</li>
<li><p>도커를 사용하면 내가 만든 이미지에 한해서, 기존에 구성된 환경과 별개로 빌드되기 때문에 환경에서 자유롭다</p>
</li>
<li><p>라고 추상적으로 이해하였지만, 와닿지 않는다</p>
</li>
<li><p>이것만이 장점이라면 다양한 장점이 존재하는 ELB와 같은 방법이 훨씬 효율적이고 좋아보인다 (물론 둘 다 쓰는게 Best이겠지만)</p>
</li>
<li><p>이에 대한 이해를 위해선 운영체제와 컴퓨터 구조에 대한 학습이 선행되어야 할 것 같다</p>
</li>
</ul>