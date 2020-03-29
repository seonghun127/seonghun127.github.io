# Spring Redis 실습

redis에 대해 공부를 시작하면서 spring에 적용 시키기 위해 간단한 spring redis 실습을 진행해봤습니다.

모든 코드는 [github](https://github.com/seonghun127/archive/tree/redis)에 있습니다.

> spring은 spring-data-redis라는 모듈을 통해 손쉽게 spring 에서 redis를 사용할 수 있도록 제공하고 있습니다. 다시한번 spring의 서비스 추상화에 속으로 감탄해봅니다.

우선 필요한 의존성을 추가해줍니다.
```groovy
    dependencies {
    	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'   // 1
    	implementation "org.springframework.boot:spring-boot-starter-data-redis" // 2
    	implementation "redis.clients:jedis"                                     // 3
    	implementation 'org.springframework.boot:spring-boot-starter-web'
    	compileOnly 'org.projectlombok:lombok'
    	runtimeOnly 'com.h2database:h2'
    	annotationProcessor 'org.projectlombok:lombok'
    	testImplementation('org.springframework.boot:spring-boot-starter-test') {
    		exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    	}
    }
```
1. jpa 의존성 추가입니다. redis 사용에 반드시 필요한 것은 아니지만 추후 다른 실습을 할 때 필요할 수 있을거 같아서 추가해줬습니다.
2. 위에서 언급한 spring-data-redis입니다. spring에서 redis를 처음 사용하는 사람이라도 코드 몇줄로 간단하게 사용할 수 있도록 제공해주는 모듈입니다.
3. jedis라는 모듈은 redis 메모리에 접근하기 위해 사용하는 모듈입니다. jedis에서 제공하는 Connection 으로 redis 메모리와 통신을 주고받을 수 있습니다.

*본격적으로 시작하기에 앞서 redis를 설치해줘야합니다. 본인 로컬 환경에서 진행하게 된다면 로컬 환경에 redis를 다운받아야 하며 aws 를 사용하신다면 aws의 redis 리소스를 먼저 구축하셔야합니다.*

> redis 설치는 구글에 검색하시면 쉽게 방법을 알 수 있기 때문에 여기서는 설명하지 않고 넘어가겠습니다. 🙏

이제 redis 설정을 추가해줍니다.

```java
    @Configuration
    public class RedisConfig {
    
    		// 1
        @Bean
        public JedisConnectionFactory jedisConnectionFactory() {
            return new JedisConnectionFactory();
        }
    
    		// 2
        @Bean
        public RedisTemplate<String, Object> redisTemplate(JedisConnectionFactory jedisConnectionFactory) {
            RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
            redisTemplate.setConnectionFactory(jedisConnectionFactory);
            return redisTemplate;
        }
    }
```
1. redis 메모리와 커넥션을 맺기 위한 설정입니다. jedis라는 모듈이 손쉽게 커넥션을 맺을 수 있는 기능을 제공해주기 때문에 우리는 `new JedisConnectionFactory()` 로 객체를 생성하여 빈으로 등록만 해주면 됩니다.
    - 만약 로컬에서 redis를 설치하시고 실습하신다면 호스트 주소나 포트번호는 따로 설정하지 않으셔도 됩니다. `JedisConnectionFactory()`에 기본 호스트 주소와 포트번호가 설정되어 있기 때문입니다. *(물론 redis를 설치하실 때 따로 설정 값들을 주셨다면 그에 맞게 코드에서도 설정을 해주셔야합니다.)*

        ![](https://user-images.githubusercontent.com/30451129/77851626-22167c00-7215-11ea-8fee-c641b0db9c8b.png)

2. 1번에서 빈으로 등록한 `JedisConnectionFactory`를 주입받아 `RedisTemplate`을 빈으로 등록해줍니다. `RedisTemplate`은 spring-data-redis 모듈에서 제공하는 헬퍼 클래스입니다. 우리는 이 클래스를 통해 손쉽게 redis에 데이터를 탐색, 추가, 삭제, 수정을 할 수 있습니다.

간단하게 엔티티와 레퍼지토리를 만들어봅시다.
```java
    @Getter
    @NoArgsConstructor
    @EqualsAndHashCode
    @ToString
    public class Member implements Serializable {
    
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;
    
        private String memberNo;
    
        private Integer count;
    
        @Builder
        public Member(String memberNo, Integer count) {
            this.memberNo = memberNo;
            this.count = count;
        }
    }
```
```java
    public interface MemberRepository extends JpaRepository<Member, Long> {
    }
```
그리고 `RedisTemplate`으로 redis를 사용할 수 있는 간편 클래스를 하나 만들어봅시다.
```java
    @Component
    public class MemberCacheStore {
    
        private RedisTemplate<String, Object> redisTemplate;
        private ValueOperations<String, Object> valueOperations;    // 1
    
        public MemberCacheStore(RedisTemplate<String, Object> redisTemplate) {
            this.redisTemplate = redisTemplate;
            this.valueOperations = redisTemplate.opsForValue();     // 2
        }
    
        public Member get(String key) {
            return (Member) valueOperations.get(key);               // 3
        }
    
        public void set(String key, Object value) {
            valueOperations.set(key, value);                        // 4
        }
    }
```
1. 이 실습에서는 redis의 제일 기본적인 자료구조인 String 자료구조를 사용해볼 것입니다. `ValueOperation`클래스가 String 자료구조로 redis에 데이터를 적재시켜줄 명령어 클래스입니다. 다양한 자료구조에 맞춰 사용할 수 있는 Operation 클래스도 다양합니다.
    - redis에는 다양한 자료구조가 있습니다. 자료구조에 대한 공부가 필요하신 분은 [여기](https://seonghun127.github.io/article/3.html)를 보시면 됩니다.
2. 생성자를 통해 `RedisTemplate`을 주입 받아서 `RedisTemplate`이 가지고 있는 `ValueOperation` 객체를 `opsForValue()`라는 메서드로 주입시켜줍니다.
3. String 자료구조에서 key로 데이터를 조회해오는 메서드입니다.
4. String 자료구조에서 key와 value를 저장하는 메서드입니다.

이제부터 우리는 `MemberCacheStore` 클래스로 실습을 진행해볼 것입니다.

간단한 테스트를 만들어봅시다.
```java
    @SpringBootTest
    class SpringPracticeApplicationTests {
    
    	@Autowired
    	private MemberRepository memberRepository;
    
    	@Autowired
    	private MemberCacheStore memberCacheStore;
    
    	@Test
    	void redis_저장_조회_테스트() {
    		// 1. Given
    		Member member = Member.builder()
    				.memberNo("123")
    				.count(100)
    				.build();
    		Member newMember = memberRepository.save(member);                // 1
    
    		// 2. When
    		memberCacheStore.set(newMember.getMemberNo(), newMember);        // 2
    
    		// 3. Then
    		assertThat(memberCacheStore.get(newMember.getMemberNo())).isEqualTo(newMember);    // 3
    	}
    }
```
1. 사실 없어도 되는 로직입니다. 원치않으시면 삭제하셔도됩니다.
2. 주입받은 `MemberCacheStore` 객체의 `set()` 메서드로 `Member` 클래스가 가지고 있는 memberNo 값으로 key로,  새롭게 생성한 `newMembe` 객체를 value로 저장해줍니다.
3. 그리고 `newMember` 객체의 memberNo 값으로 redis 메모리에서 값을 조회한 뒤 `newMember`와 같은지 테스트를 돌려봅니다. 테스트는 정상적으로 성공합니다.

    ![](https://user-images.githubusercontent.com/30451129/77851631-25aa0300-7215-11ea-935a-70fe1cfe7392.png)

해당 테스트는 직접 spring boot를 띄워 실제 로직을 수행시킨 테스트이기 때문에 redis 메모리 실제 데이터가 적재됐을 것입니다. 직접 redis 메모리에 접속해서 데이터를 확인해봅시다. (redis-cli로 터미널에서 확인해보려합니다.)

![](https://user-images.githubusercontent.com/30451129/77851632-26429980-7215-11ea-9055-a554adfd1f66.png)

터미널에서 확인해보니 데이터는 분명 들어가 있는데 위와같은 상황이 발생했습니다. 사람이 알 수 없는 문자열 형태로 데이터가 저장되어 직접 메모리 조회 시에는 뭐가 뭔지 알 수 없는 상황이었습니다.

해당 이슈를 확인해보니 `RedisTemplate`을 빈으로 등록할 때 특별히 객체의 serializer 를 설정해주지 않았기 때문입니다. `RedisTemplate`의 디폴트 serializer는 `JdkSerializationRedisSerializer`입니다.

![](https://user-images.githubusercontent.com/30451129/77851635-26db3000-7215-11ea-92f5-66ccd91e7996.png)

위 `RedisTemplat`클래스의 `afterProPertiesSet()`에서 디폴트 serializer를 등록해주고 있습니다. 해당 메서드는 빈으로 등록될 때 실행되는 메서드입니다. 만약 커스텀하게 serializer를 `RedisTemplate`에 설정해주지 않았다면 빨간 네모칸에 로직이 수행될 것입니다.

그래서 새롭게 serializer 설정을 추가해줍시다.
```java
    @Configuration
    public class RedisConfig {
    
        @Bean
        public JedisConnectionFactory jedisConnectionFactory() {
            return new JedisConnectionFactory();
        }
    
        @Bean
        public RedisTemplate<String, Object> redisTemplate(JedisConnectionFactory jedisConnectionFactory) {
            RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
            redisTemplate.setConnectionFactory(jedisConnectionFactory);
            // 1
            redisTemplate.setKeySerializer(redisTemplate.getStringSerializer());
            // 2
            redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());
            return redisTemplate;
        }
    }
```
1. 우선 redis key의 serializer를 등록해줍니다. key는 String 형태를 유지해주는 serializer로 등록해주면 됩니다. 보통 key는 문자로 저장되는 것이 사람이 보는데 제일 이해하기 쉽기 때문입니다. 사실 `RedisTemplate`을 보면  해당 serializer를 가지고 있습니다.

    ![](https://user-images.githubusercontent.com/30451129/77851636-2773c680-7215-11ea-87fd-0a85a8edd325.png)

    그래서 이 값을 그대로 사용해주면 됩니다. 사실 `StringRedisSerializer` 객체를 새롭게 만들어 `setKeySerializer()`의 인자로 넣어줘도 되지만 굳이 새롭게 만들지말고 이미 `RedisTemplate`에 존재하는 객체를 사용해줍시다.

2. key의 경우 문자 형태로 입력되고 저장도 문자 형태로 되기 때문에 상관 없었지만 여기서 value는 객체가 저장됩니다. 현재 실습에선 `Member`라는 객체를 저장하려하기 때문에 객체를 json 형태의 문자열 형태로 바꿔주는 `GenericJackson2JsonRedisSerializer`로 설정해줍니다. (json 형태가 사람이 보기에도 제일 읽기 쉬운 형태의 구조이기 때문에 이걸로 설정해줬습니다.)

이제 다시 테스트를 돌려줍시다. (스프링에 정상적으로 뜬다면 테스트 로직을 바꾼 것이 없기 때문에 테스트는 정상적으로 성공할 것입니다.)

그리고 터미널로 redis 메모리의 데이터를 확인해봅시다.

![](https://user-images.githubusercontent.com/30451129/77851637-280c5d00-7215-11ea-87f6-cc5ba67486e2.png)

데이터가 설정대로 잘 저장된 것을 확인할 수 있습니다.

지금까지 간단하게 spring에서 redis를 사용해보는 실습이었습니다.
