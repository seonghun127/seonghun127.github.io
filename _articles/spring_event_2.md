---
id: 10
title: "Spring Event 2"
subtitle: "spring event를 적용해보면서 했던 삽질 2"
date: "2020.05.03"
tags: "spring","event","mockito"
---


# Spring Event-2



*이 글은 [Spring Event 1](https://seonghun127.github.io/article/9.html) 글과 이어지는 내용입니다. 이전 글을 읽지 않으신 분은 앞선 글을 읽으셔야 아래 내용이 이해되실겁니다.*

이제 새로운 조건이 생겼다.

`count`가 100 이상인 `Member`에 경우에만 이벤트가 이를 구독하여 `count`를 1증가시켜야한다.

사실 이벤트 구독 시 조건을 추가하는 것은 매우 쉬운 일이다.

조건을 추가해보자.

아래와 같이  `@TransactionalEventListener`에 condition을 추가해주자.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class MemberEventSubscriber {

    private final MemberIncreaseService memberIncreaseService;

    @TransactionalEventListener(condition = "#memberEvent.member.count >= 100")
    public void processEvent(MemberEvent memberEvent) {
        log.info("Event received! {}", memberEvent.toString());
        memberIncreaseService.increase(memberEvent.getMember());
    }
}
```

condition 값으로 들어간 `"#memberEvent.member.count >= 100"` 는 SqEL 이다.

SpEL에 대해 간단하게 정리하자면 런타임 시에 객체에 접근할 수 있는 유용한 언어다. 여기서 사용한 `"#memberEvent.member.count >= 100"` 을 하나씩 해석해보면 

1. 이벤트 구독 메서드인 `processEvent()`의 파라미터로 선언된 `memberEvent`에 `#memberEvent`로 접근한다.
2. 해당 인스턴스가 가지고 있는 `getMember()`를 `.member`로 호출해서 실제 `memberEvent` 이벤트 객체 안에 있는 회원 `member` 객체를 꺼낸다. 
3. 그런 뒤 `member` 객체가 가지고 있는 `getCount()` 메서드를 `.count`로 호출하여 그 안에 값을 주어진 100 숫자와 크기 비교를 한다.

그래서 위 조건이 true 인 경우에만 이벤트를 구독하는 `processEvent()` 메서드가 실행된다.

SpEL 문법을 잘 몰라도 사실 어느정도 추론이 가능하다고 생각한다. SpEL에 대해서는 [레퍼런스](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/expressions.html)를 참고하길 바란다.

이제 간단하게 테스트 해볼 수 있게 새로운 API를 하나 만들어보자.

`PathVariable`로 전달하는 숫자만큼의 `count` 를 가지는 `Member` 생성 API이다.

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class MemberEventService {

    private final MemberRepository memberRepository;
    private final ApplicationEventPublisher applicationEventPublisher;

    @Transactional
    public void save() {
        Member member = memberRepository.save(Member.builder()
                .count(100)
                .build());

        publishEvent(member);
    }

	// 새로 추가한 메서드
    @Transactional
    public void saveBy(int count) {
        Member member = memberRepository.save(Member.builder()
                .count(count)
                .build());

        publishEvent(member);
    }

		...
	// publishEvent() 생략
}
```

```java
@RestController
@RequiredArgsConstructor
public class MemberController {

    private final MemberEventService memberEventService;

    @GetMapping
    public void publishEvent() {
        memberEventService.save();
    }

	// 회원의 count를 인자로 받아서 새롭게 생성하는 API다.
    @GetMapping("/members/{count}")
    public void publishMemberEvent(@PathVariable int count) {
        memberEventService.saveBy(count);
    }
}
```

해당 API를 100이상의 인자로 호출 시에는 새롭게 생성된 회원의 count 값이 1 증가하는 것을 확인할 수 있고 99 이하의 숫자를 인자로 전달한 경우에는 회원은 새롭게 생성되지만 count 값이 1 증가하지 않는 것을 확인할 수 있다.

*PathVariable로 100을 전달한 경우*

![](https://user-images.githubusercontent.com/59458270/80915448-673a4a80-8d8d-11ea-9320-ce1208036982.png)

![](https://user-images.githubusercontent.com/59458270/80915450-6b666800-8d8d-11ea-82db-bce5f3f51b17.png)

*PathVariable로 99를 전달한 경우*

![](https://user-images.githubusercontent.com/59458270/80915451-6dc8c200-8d8d-11ea-86e7-fdbf58241cbb.png)

![](https://user-images.githubusercontent.com/59458270/80915453-6f928580-8d8d-11ea-8885-9b71137fd87a.png)

이렇게 우리는 SpEL 로 이벤트 구독의 새로운 조건을 쉽게 추가해줬다.

그러면 이제 이벤트를 구독하는 `MemberEventSubscriber` 의 테스트 코드를 작성해보자.

이벤트 구독자를 어떻게 테스트를 할 수 있을까 고민해보다가 생각한 방법은 mockito를 사용하여 이벤트가 발행한 시점에 `MemberEventSubscriber`의 `processEvent()` 메서드의 인자로 전달된 이벤트 객체가 실제 발행된 이벤트와 같은 지를 비교하는 방식으로 구현하기로 했다.

```java
@SpringBootTest(classes = MemberEventSubscriber.class)
public class MemberEventSubscriberTest {

    @MockBean
    private MemberEventSubscriber memberEventSubscriber;              // 1

    @Autowired
    private ApplicationEventPublisher applicationEventPublisher;      // 2

    @Captor
    private ArgumentCaptor<MemberEvent> argumentCaptor;               // 3

    @Test
    public void 이벤트_발행_시_제대로_구독하는지_테스트() {
        // 1. Given
        Member member = Member.builder()
                .count(100)
                .build();
        MemberEvent event = MemberEvent.builder()
                .member(member)
                .build();

        // 2. When
        applicationEventPublisher.publishEvent(event);

        // 3. Then
        verify(memberEventSubscriber, times(1)).processEvent(argumentCaptor.capture());            // 4
        MemberEvent publishedEvent = argumentCaptor.getValue();
        assertThat(publishedEvent).isEqualTo(event);
    }
}
```

1. 조금은 어색할 수 도 있지만 실제 테스트할 `MemberEventSubscriber` 를 목킹했다. 이 이유는 나중에 `verify()`로 해당 객체를 확인하기 위함이다. 목킹했더라도 해당 클래스의 인스턴스의 메서드로 들어온 인자를 확인하는데는 전혀 지장이 없기 때문이다.
2. 실제 이벤트를 발행하는 객체다. `MemberEventService`에서도 이 `ApplicationEventPublisher` 빈을 주입받아서 이벤트를 발행했기 때문에 테스트에서도 똑같이 이벤트를 발행시키위해 빈으로 주입받는다.
3. `mockito` 라이브러리에 있는 `Captor`다. 이 객체를 통해 `verify()`에서 메서드 인자로 들어온 값을 캡쳐할 수 있다.
4. 본 테스트의 핵심 부분이다. `verify()`로 테스트하려는 `memberEventSubscriber` 객체의 실제 실행 횟수와 더불어 `processEvent()` 메서드의 인자로 들어온 객체를 `arguemntCapter.capture()`로 캡쳐하고 있다.

테스트는 성공적으로 돌아간다.

위 테스트 방식은 복잡해보이지만 사실 매우 간단하다. 실제 테스트할 객체를 목킹한다는게 조금 어색할 수 있지만 이벤트 구독 메서드가 잘 동작하고 있다는 것을 확인할 수 있다.

그럼 이제 새롭게 추가해줬던 조건에 맞게 100 미만의 `count` 값을 가지는 회원 객체의 경우 이벤트 구독자가 이벤트를 처리하지 않는지 테스트를 해보자.

```java
	@Test
    public void 이벤트_조건이_부합하지않은경우_이벤트_처리하지_않는지_테스트() {
        // 1. Given
        int notConditionCount = 99;
        Member member = Member.builder()
                .count(notConditionCount)
                .build();
        MemberEvent event = MemberEvent.builder()
                .member(member)
                .build();

        // 2. When
        applicationEventPublisher.publishEvent(event);

        // 3. Then
        verify(memberEventSubscriber, never()).processEvent(argumentCaptor.capture());
    }
```

이 테스트 역시 잘 돌아간다. 99 라는 count 값으로 조건에 부합하지 않는 회원에 대한 이벤트가 발생했을 때 `memberEventSubscriber.processEvent()` 메서드는 실행되지 않는다는 것을 잘 확인할 수 있다.

테스트까지 순탄하게 완성했지만 문제가 하나 있다.

위 실습 환경은 스프링 부트 2.2.6 인데 실제 문제를 맞닥뜨린 환경은 스프링 부트 1.5.7이었다. 또한 멀티모듈 환경이었다.

> 원래는 테스트가 깨질 줄 알고 짠 코드였지만 너무나 잘 돌아가서 당황했다. 분명 마주했던 문제와 코드는 크게 다르지 않았기 때문이다. 그래서 그 당시 환경과 동일한 구조로 바꿔봤다.

변경한 환경은 이렇다.

- 스프링 부트 버전 변경 : 2.2.6 → 1.5.7
- 멀티 모듈 환경
    - mutli-module-api 모듈과 multi-module-core 모듈 두개를 만들었다.
    - multi-module-api
        - `MemberController`
        - `MemberEventService`
        - `MemberIncreaseService`
        - `MemberEventSubscriber`
        - `MemberEventSubscriberTest`
    - multi-module-core
        - `Member`
        - `MemberRepository`
        - `MemberEvent`

이렇게 실습 환경을 수정하고 다시 테스트를 돌려봤다.

테스트는 보란듯이 실패했다.

![](https://user-images.githubusercontent.com/59458270/80915454-70c3b280-8d8d-11ea-9c43-93c1b71d3bc4.png)

SpEL 에서 `member` 객체로 접근했던 부분이 null 상태라는 오류메세지다.

**이 오류 메세지의 원인을 파악하고 디버깅해보면서 어느 지점에서 발생했는지 파악하기까지 정말 오랜 시간이 걸렸다.**

일단 위 문제가 발생한 원인은 클래스가 mocking 될 때 CGLIB, ByteBuddy(Mockito 버전에 따라 다르다.) 라이브러리에 의해 mock 클래스가 해당 클래스를 상속하여 만들어진다.

즉, 위 예시에서 `@MockBean`어노테이션이 붙은 `MemberEventSubscriber`클래스를 상속하는 Mocking `MemberEventSubscriber`가 생성되는 것이다.

그리고 SpEL은 새롭게 생성된 Mocking `MemberEventSubscriber` 기준으로 파라미터를 확인하며 파싱된다. 

`"#memberEvent.member.count >= 100"` 를 보면 런타임 시 `MemberEventSubscriber`클래스가 가지고 있는 `processEvent()` 메서드의 파리미터인 `memberEvent`에 접근하게 된다.

여기서 접근하는 `processEvent()`메서드는 아까 Mockito에 의해 새롭게 상속하면서 만들어진 Mocking `MemberEventSubscriber` 클래스의 메서드가 된다.

> 여기서 스프링은 해당 메서드의 파리미터 이름을 확인하기 위해 Java의 Reflection을 사용하는 `ParameterNameDiscoverer` 인터페이스를 사용한다.

위 오류의 원인을 살펴보자. 해당 테스트를 디버깅하면서 하나씩 확인해본 결과 아래 스크린샷에 나온 지점에 도달할 수 있었다.

![](https://user-images.githubusercontent.com/59458270/80915456-728d7600-8d8d-11ea-81b3-66f08b945476.png)

런타임 시에 접근하는 SpEL 특성 상 `processEvent(MemberEvent memberEvent)` 메서드의 파라미터로 선언된 `memberEvent` 로 접근하게되는데 막상 접근해보니 해당 메서드의 파리미터 이름은 `arg0`으로 나와있어 SpEL로 선언된 `memberEvent`는 알 수 없는 존재가 돼버린 것이다.

다시 스프링 버전을 이전으로 높이고 테스트 디버깅을 해보았다.

![](https://user-images.githubusercontent.com/59458270/80915457-74573980-8d8d-11ea-8c4b-8e0879719f25.png)

버전이 올라가면서 `StandardReflectionParameterNameDiscoverer`내부 코드의 구조는 변경됐지만 로직이 변한 부분은 없다.

디버깅 값을 보니 파리미터의 이름으로 `memberEvent`라는 것이 확인됐다.

해당 이슈로 버전이 낮은 상태에서 오류가 나고 있음을 확인할 수 있었다.

그러나 해당 이슈에 대해 더 정확한 이유를 파악하려했지만 속시원할만한 이유를 알아내지 못한채 다른 해결 방법을 찾았다.

> 속으로 너무 아쉬워 이후에도 계속 찾아보고 mockito 그룹스 메일로도 문의를 해본 상황이다. 😭

그럼 위 버전을 사용하는 사람들이 나와 같은 문제에 시간을 허비하는 일이 없도록 해결방안에 대해 설명해보겠다.

사실 매우 간단하다.

위 문제는 런타임 시 Mocking된 메서드의 파라미터 이름으로 SpEL을 파싱할 수 없으니 발생한 것이다. 꼭 해당 메서드의 파라미터 이름으로 접근하는 방법이 아닌 `root`라는 것을 이용해 현재 발행된 이벤트에 대해 접근하는 방식으로 해결해보겠다.

`@TransactionalEventListener(condition = "#root.args[0].member.count >= 100")`

condition에 SpEL을 위와같이 작성해주면 된다. 테스트는 정상적으로 동작할 것이다.

하나씩 살펴보자.

- `root`라는 것은 `EventExpressionRootObject` 를 가리킨다. 이 클래스는 이벤트가 발행되는 시점에 타겟이 되는 메서드의 정보를 바탕으로 만들어지는 이벤트 루트 객체다. 여기서 타겟이 되는 메서드는 `MemberEventSubscriber` 클래스의 `processEvent()` 메서드를 말한다.

    이 객체는 `EventExpressionEvaluator` 클래스가 만들어주는데 생성자 파라미터 인자로 넘겨주는 것은 두 가지다.

    1. `ApplicationEvent` : 발행된 이벤트의 상위 인터페이스다. 실제 발행된 `MemberEvent`는  `PayLoadApplicationEvent` 라는 구현체에 담겨 전달된다.
    2. `args` : 타겟 메서드의 파리미터로 선언된 순서대로 발행된 이벤트가 배열에 담겨있다. 이벤트 타겟 메서드의 파라미터는 해당 메서드가 구독할 이벤트의 종류를 나타내기 때문이다. 위 실습에서는 타겟 메서드의 파리미터는 하나이기 때문에 `args` 배열의 길이는 1이며 `MemberEvent` 하나가 담기게 된다.

    ![](https://user-images.githubusercontent.com/59458270/80915459-76b99380-8d8d-11ea-8067-b2bf93a1bcf9.png)

- `root`가 의미하는 `EventExpressionRootObject`에는 `getArgs()` 메서드가 있다. 해당 객체가 가지고 있는 이벤트 종류를 인덱스로 가져올 수 있다. 그래서 `.args[0]`은 현재 타겟 메서드의 파리미터로 하나 선언된 `MemberEvent` 를 가져오겠다는 뜻이다. 만약 타겟 메서드의 파리미터로 여러개가 선언되어 있다면 선언된 순서에 맞게 인덱스로 접근할 수 있다.
- 그렇게 꺼낸 `MemberEvent`의 `.member` 를 꺼내고 `.count` 꺼내는 방식으로 SpEL은 작동할 것이다.

`@TransactionalEventListener(condition = "#root.event.payload.member.count >= 100")`

위와 같이 선언해도 문제없이 돌아간다.

`root.event.payloay`는 `EventExpressionRootObject`에 생성자 인자로 같이 전달됐던 `PayLoadApplicationEvent`으로 접근하게 되는 것이다. 그래서 `PayLoadApplicationEvent` 객체가 가지고 있는 `getPayLoad()`로 `MemberEvent`를 가져올 수 있다.

```java
	@TransactionalEventListener(condition = "#root.args[0].member.count >= 100")
    public void processEvent(MemberEvent memberEvent) {
        log.info("Event received! {}", memberEvent.toString());
        memberIncreaseService.increase(memberEvent.getMember());
    }
```

하지만 위와 같은 방식이 더 좋다고 생각한다. 이벤트 구독 메서드의 파리미터가 여러개라면 선언된 순서로 인덱스 접근하는 방식이 더 명확하게 느껴지기 때문이다.

이렇게 스프링의 이벤트 발행 / 구독 방식을 예제를 통해 구현해보았고 이를 테스트하는 방법도 같이 살펴봤다.

이 모든 과정에 best practice가 아닐 수 있지만 처음 이 내용을 접해보는 사람들에게 좀 더 쉽게 다가갈 수 있었으면 좋겠다.

### reference

- [https://spring.io/blog/2015/02/11/better-application-events-in-spring-framework-4-2](https://spring.io/blog/2015/02/11/better-application-events-in-spring-framework-4-2)
- [https://www.baeldung.com/spring-events](https://www.baeldung.com/spring-events)
- [https://github.com/mockito/mockito/wiki/What's-new-in-Mockito-2](https://github.com/mockito/mockito/wiki/What%27s-new-in-Mockito-2)
- [https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-testing](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-testing)
