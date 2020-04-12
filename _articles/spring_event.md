---
id: 9
title: "Spring Event 1""
subtitle: "spring event를 적용해보면서 했던 삽질 1"
date: "2020.04.12"
tags: "spring","event"
---


# Spring Event


얼마전 개발을 하면서 처음으로 Spring Event라는 기술을 접하게 됐다. 말로만 들었던 이벤트 발행 / 구독을 직접 구현해본 경험을 공유하고자 한다.

> 사실 이벤트가 발행되고 이를 구독하는 쪽에서 어떻게 원하는 이벤트를 받아서 알맞게 처리하는지 그 개념이나 원리는 정확히 모른다. 개념을 먼저 익히고 기술 구현에 들어가고 싶었지만 그럴 시간은 없었고 구현부터 진행했기에 그 과정에서 수많은 시간을 삽질했던 내용을 중심으로 글을 쓰려고 한다. (개념, 원리에 대해서는 나중에 공부해보겠다.)



당시 구현하면서 삽질했던 시스템 구조와 비슷하면서 간소화한 상황은 다음과 같다.

* 회원(Member)이 api 요청에 의해 저장된다.
* 회원이 저장되면 무조건 회원 도메인의 count 필드 값이 1 증가된다.
  * 회원을 저장하는 서비스, 회원의 정보를 수정하는 서비스는 별도로 존재해야한다. (아래 예제에서 설명하겠다.)



예시가 뭔가(?) 이상하면서 어색하지만 실제 구현했던 상황은 조금 복잡했기 때문에 최대한 간소화 시켜서 그런거일 뿐이다.

코드는 전부 [github](https://github.com/seonghun127/archive/tree/master/spring-event)에 있다.



Spring Event를 좀 더 이해하기 위해 우선 예제를 Event 발행을 사용하지 않은 방식부터 시작해보겠다.

먼저 도메인인 `Member` 를 만들어주자.

```java
@Entity
@Getter
@NoArgsConstructor
@ToString
@EqualsAndHashCode
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    @Column(nullable = false)
    public int count;

    @Builder
    public Member(int count) {
        this.count = count;
    }

    public int increase() {
        return ++count;
    }
}
```



```java
public interface MemberRepository extends JpaRepository<Member, Long> {
}
```



이제는 회원을 저장하는 서비스를 만들어주자.

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class MemberSaveService {

    private final MemberRepository memberRepository;
    private final MemeberIncreaseService memeberIncreaseService;

    @Transactional
    public void save() {
        Member member = memberRepository.save(Member.builder()
                .count(100)
                .build());
      
		// 회원 count 1 증가
        memeberIncreaseService.increase(member);
    }
}
```



회원의 정보를 수정하는 서비스를 만들어주자.

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class MemberIncreaseService {

    private final MemberRepository memberRepository;

    @Transactional
    public Member increase(Member member) {
        member.increase();
        return memberRepository.save(member);
    }
}
```

위 예제를 보면 회원 객체를 인자로 받고 있다. `increase(Member member)` 그리고 난 후 회원 정보 수정 `member.increase() `이후 다시 레퍼지토리로 객체를 저장해주고 있다.

사실 `MembmerIncreaseService` 의  `increase()` 메서드를 호출하는 부분인  `MemberSaveService` 의 `save()` 메서드 자체가 회원 객체를 저장한 뒤 영속화된 객체를 인자로 전달해주기 때문에 `MemberInreaseService`의 `increase()` 메서드에서는 따로 저장을 하지 않아도 된다. *(또한 트랜잭션 전파 속성으로 두 메서드가 하나의 트랜잭션으로 묶여 있기도 하다.)*

하지만 이부분은 이따가 Event 발행 방식으로 바꾸면서 의존성을 제거 해줄 예정이기 때문에 일단 두자.

이제 컨트롤러 하나를 만들어주자.

```java
@RestController
@RequiredArgsConstructor
public class MemberController {

    private final MemberSaveService memberSaveService;

    @GetMapping
    public void publishEvent() {
        memberSaveService.save();
    }
}

```



이렇게 위의 요구사항을 모두 구현했다. 직접 실행시켜보고 데이터베이스를 확인해보면 count가 1인 회원 정보가 저장되어 있는 것을 확인할 수 있다. (날라간 DB 쿼리를 보면 저장 / 수정 쿼리가 날라간 것을 확인할 수 있다.)



자 그러면 이제 Spring Event 방식을 적용해보자.

Spring에서 이벤트를 발행하고 이를 구독하는 로직은 매우 간단하게 구현할 수 있다.

이벤트를 발행하는 로직은 `ApplicationEventPublisher` 를 이용하면 된다.

우리의 요구사항에서 이벤트를 발행해야하는 시점은 회원이 저장됐을 시점이다. 이 시점에 이벤트를 발행하여 이를 구독하는 쪽에서 회원의 count 정보를 1증가하게 만들면 된다.

그렇기에 회원을 저장하는 서비스의 로직을 조금 수정해보자.

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class MemberSaveService {

    private final MemberRepository memberRepository;
    private final ApplicationEventPublisher applicationEventPublisher;			// 1

    @Transactional
    public void save() {
        Member member = memberRepository.save(Member.builder()
                .count(100)
                .build());

        publishEvent(member);
    }

    private void publishEvent(Member member) {
        applicationEventPublisher.publishEvent(MemberEvent.builder()				// 2
                .id(member.getId())
                .member(member)
                .build());
        log.info("Event is published!");
    }
}
```

1. `AppilcationEventPublisher` 빈은 기본적으로 스프링이 컨텍스트에 등록하는 빈이다. Spring은 우리가 알게 모르게 자체적으로 이벤트를 발행하고 있기 때문에 그 때 사용되는 빈이 이 빈이다. 그래서 우리는 새로운 이벤트를 발행하려면 `ApplicationEventPublisher` 를 주입받아서 사용하면 된다.
2. `ApplicationEventPublisher` 의 `publish()` 메서드를 이용해 이벤트를 발행한다. 이 메서드의 인자로는 이벤트 객체를 넣어주면 된다. 이벤트 객체는 아래에서 새롭게 만들어 줄 예정이다. 그래서 이벤트를 구독하는 쪽에서는 이 인자 타입에 해당하는 이벤트가 발행되는 시점에 이벤트를 받아서 지지고 볶고를 할 수 있다.

발행시킬 이벤트 클래스를 추가하자.

```java
@Getter
@ToString
public class MemberEvent {

    private Long id;
    private Member member;

    @Builder
    public MemberEvent(Long id, Member member) {
        this.id = id;
        this.member = member;
    }
}
```



그러면 이제 이벤트를 구독하고 있을 클래스를 만들어보자.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class MemberEventSubscriber {

    private final MemberIncreaseService memberIncreaseService;

    @EventListener																												// 1
    public void processEvent(MemberEvent memberEvent) {										// 2
        log.info("Event received! {}", memberEvent.toString());
        memberIncreaseService.increase(memberEvent.getMember());
    }
}
```

1. 이벤트를 구독하는 로직 또한 매우매우 간단하다. 보시다시피 `@EventListener` 어노테이션 하나만 메서드 위에 붙여주면 끝이다. 물론 이 클래스 역시 빈으로 등록해줘야하기 때문에 `@Component` 어노테이션을 클래스 위헤 붙여줬다.
2. 이제 이벤트 구독자가 된 이 클래스는 `processEvent()` 메서드에서 파리미터로 선언된 `MemberEvent` 타입의 이벤트가 발행되면 이 이벤트 객체를 받아서 메서드를 실행시킬 수 있다. 메서드 로직은 `MemberIncreaseService` 의 `increase()`  로직이 끝이다.



끝이다. 정말 간단하다. 위 로직 또한 직접 실행시켜보고 직접 api 호출해보면 잘 동작하고 있다는 것을 확인할 수 있다.

여기서 주목할 점은 Spring Event를 통해 `MemberSaveService` 와 `MemberIncreaseService` 간의 의존성을 제거했다는 점이다. **이벤트 발행  / 구독 방식을 통해 복잡하게 얽혀있는 객체간의 의존성을 제거할 수 있다.**



분명 여기까지 읽다보면 이렇게 간단(?)한데 어떤 삽질을 했냐는 의문을 가질 수 있다. 이제 내가 경험한 삽질에 대해 공유해보겠다.

위 상황은 매우 간단하다. 그리고 직접 돌려보고 잘 동작하고 있다는 것을 확인할 수 있다.

하지만 문제가 있다.

일단 기본적으로 Spring Event는 동기 방식으로 처리된다. 이 말은 이벤트를 발행한 시점에 트랜잭션이 진행중이었다면 이벤트를 받아서 처리하던 구독쪽에 로직도 그 트랜잭션에 속하여 처리된다는 것이다.

물론 이 자체가 문제가 되지 않는다. 그리고 위 요구사항 또한 이벤트를 발행하는 쪽과 구독하는 쪽 모두 `Member`라는 도메인을 같이 사용하고 있기 때문에 하나의 트랜잭션으로 묶인다고 해서 안좋을 건 없다.

그런데 만약 회원 정보가 DB에 저장이 된 후에 정보가 업데이트가 되어야한다고 요구사항이 변경되면 어떻게 될까?

위 로직에 의해 수행되는 쿼리가 날라가는 상황을 보면 아래 스크린샷과 같다.

![](https://user-images.githubusercontent.com/30451129/79062386-f4c9d380-7cd4-11ea-9104-088887f76dc0.png) 

회원 정보를 저장하고 수정하는 로직이 하나의 트랜잭션으로 묶였기 때문에 JPA 특성상 트랜잭션이 커밋되는 시점에 한번에 쿼리가 날라가고 있다. 이 상황이라면 변경된 요구사항을 만족할 수 없다. `Member` 도메인을 같이 사용하고 있기 때문에 큰 문제가 될 건 없다고 생각할 수 있겠지만 만약 다른 도메인을 처리하는 로직이라면 문제가 발생할 것이다.

만약 회원 정보를 저장하고 수정까지 한 다음에 추가로 새로운 도메인에 대한 비지니스 로직이 처리되어야한다면 하나의 트랜잭션으로 묶이는게 문제가 될 것이다. 새로운 도메인 `NewDomain` 은 반드시 `Member` 가 저장된 후에 저장되어야 하는 상황이면 말이다.

일단 `Member` 정보는 저장되어야하는데 같이 하나의 트랜잭션으로 묶여버린 `NewDomain` 도메인에 대한 처리 과정에 문제가 발생 시 롤백되어 `Member` 도메인에 대한 처리까지 모두 원복되어 저장되지 않을 수 있기 때문이다. (수행되어야할 로직이 수행되지 않아버린다.)

이와 같은 문제 때문에 트랜잭션을 분리해야한다. 그래서 변경된 요구 사항을 반영해보자.

사실 매우 간단하다. 이벤트를 구독하던 곳에서 기존에 트랜잭션이 존재한다면 해당 트랜잭션이 모두 끝난 다음에 이벤트를 받아서 처리하게 끔하면 된다. 이는 기존 `@EventListener` 를 `@TransactionalEventListener` 로 바꾸면 된다.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class MemberEventSubscriber {

    private final MemberIncreaseService memberIncreaseService;

    @TransactionalEventListener
    public void processEvent(MemberEvent memberEvent) {
        log.info("Event received! {}", memberEvent.toString());
        memberIncreaseService.increase(memberEvent.getMember());
    }
}
```

`@TranactionEventListener` 는 트랜잭션 전파를 컨트롤할 수 있다. 이 어노테이션에는 `TransactionPhase` 속성 값이 있는데 이 값에는 아래와 같이 4자기 종류가 있다.

* AFTER_COMMIT : default 값이다. 트랜잭션이 성공적으로 commit 됐을 때 사용된다.
* AFTER_ROLLBACK : 트랜잭션이 롤백됐을 때 사용된다.
* AFTER_COMPLETION : 트랜잭션이 완료됐을 때 사용된다.
* BEFORE_COMMIT : 트랜잭션이 커밋되기 전에 사용된다.

위 4가지 속성으로 이벤트 구독 지점에서 이전에 진행됐던 트랜잭션에 전파되어 하나로 묶이는 부분을 해결할 수 있다. default 값인 AFTER_COMMIT 속성을 이용하여 앞선 트랜잭션이 끝난 다음에 `processEvent()` 메서드가 수행되게 하면 된다.

그렇게 문제는 쉽게 해결되는 줄 알았는데 제대로 작동하지 않고 있다는 것을 발견했다.

![](https://user-images.githubusercontent.com/30451129/79062388-f6939700-7cd4-11ea-8647-3039a60ac378.png)

쿼리와 로그를 확인해보니 이벤트 발행 시점의 트랜잭션이 끝났고 이벤트 구독하는 곳의 메서드가 수행은 됐는데 회원 정보 수정은 되지 않았다.

원래대로라면 회원 정보를 수정하는 update 쿼리가 날라가야하는데 그게 날라가지 않았다. update 로직을 수행하는 `MemberIncreaseService` 코드를 다시 보자.

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class MemberIncreaseService {

    private final MemberRepository memberRepository;

    @Transactional
    public Member increase(Member member) {
        log.info("MemberIncreaseService.increase is called!");
        member.increase();
        return memberRepository.save(member);
    }
}
```

일단 `@Transactional` 어노테이션으로 회원정보 수정 로직을 하나의 트랜잭션으로 묶었고 `memberRepository.save(member); ` 로 회원정보를 직접 저장까지 하기 때문에 도메인의 영속성 문제도 전혀 없다.

그렇다면 도대체 왜 회원 정보가 수정되지 않는 것일까? 구글링을 통해 다음과 같은 사실을 알게됐다.

![](https://user-images.githubusercontent.com/30451129/79062389-f7c4c400-7cd4-11ea-8c22-bb11c2056d8c.png)

위 내용을 대충 해석해보자면,

> 트랜잭션이 커밋됐다하더라도 트랜잭션 리소스는 아직 활성화 되어있고 접근가능하다. 그 결과, 그 시점에 발생된 데이터에 접근하는 코드는 명시적으로 새로운 트랜잭션에서 수행되어야한다는 선언이 없다면 기존 트랜잭션에 참여하게 된다. 결국 이 코드는 더이상 커밋 되지 않은채 수행된다.

결국 이벤트를 구독하는 지점에서는 이미 커밋된 트랜잭션에 참여하게 되어 회원정보를 수정하는 로직은 DB에 전혀 반영되지 않았던 것이다. 이 문제를 해결하기 위해 트랜잭션의 전파 속성을 바꿔줘야한다.

`@Transactional` 어노테이션의 전파 속성의 기본 값은 `Propagation.REQUIRED`이다. 

![](https://user-images.githubusercontent.com/30451129/79062390-f8f5f100-7cd4-11ea-82e9-eb010c97a15c.png)

이 속성은 만약 트랜잭션이 이미 존재하고 있다면 그 트랜잭션으로 묶이게 되고 현재 트랜잭션이 없는 경우에만 새로운 트랜잭션을 만든다는 값이다. 이 속성 상태 때문에 위와 같은 문제가 발생했다.

그래서 우리는 위에서 말했듯이 `Propagation.REQUIRES_NEW` 으로 설정해줘야한다.

![](https://user-images.githubusercontent.com/30451129/79062391-f98e8780-7cd4-11ea-93d9-5a91819f7f8d.png)

이 속성은 일단 새로은 트랜잭션을 생성한다. 만약 트랜잭션이 기존에 존재한다면 이를 미뤄버린다.

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class MemberIncreaseService {

    private final MemberRepository memberRepository;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public Member increase(Member member) {
        log.info("MemberIncreaseService.increase is called!");
        member.increase();
        return memberRepository.save(member);
    }
}
```

그리고 다시 스프링을 띄우고 확인해보면 잘되는 것을 확인할 수 있다.

![](https://user-images.githubusercontent.com/30451129/79062393-fabfb480-7cd4-11ea-8939-ee40e468a5c7.png)

`@Transactional(propagation = Propagation.REQUIRES_NEW)` 설정을 통해 문제를 해결했다.

정리하자면 이벤트 발행 시점의 트랜잭션과 이벤트 구독 시점의 트랜잭션이 맞물리게 되면 기본적으로 전파 속성으로 인해 하나의 트랜잭션으로 묶인다. Spring Event는 기본적으로 동기적으로 일어나기 때문이다. 그래서 `@EventListener` 으로 쉽게 이벤를 구독할 수 있지만 요구사항에 따라 트랜잭션을 분리해야한다면 `@TransactionalEventListener` 어노테이션을 사용해야한다.

`@TransactionalEventListener` 어노테이션은 트랜잭션 전파를 컨트롤할 수 있는데 이벤트 발행 시점의 트랜잭션의 커밋 이전, 이후, 롤백 이후 등의 옵션을 선택하여 상황에 맞게 동작할 수 있다. 하지만 Javadoc에서 설명하듯이 앞선 트랜잭션이 커밋되었다하더라도 트랜잭션의 리소스는 여전히 활성화 되어있고 접근 가능하기 때문에 언제든지 해당 트랜잭션으로 묶일 수 있다. 그렇기 때문에 만약 분리된 새로운 트랜잭션을 만들어줘야한다면 `@Transactional(propagation = Propagation.REQUIRES_NEW)` 명시를 해줘야한다.

이렇게 문제를 완전히 해결한다고 생각했지만 추가 요구사항이 들어왔다.

회원이 저장되는 시점에 회원 count가 100 이상인 경우에만 1 증가되어야한다. 즉 이벤트를 발행하지만 특정 조건에 경우에만 이벤트를 구독하는 방식으로 수정해야한다. 이 과정에서 겪었던 삽질은 다음 포스팅에서 다루겠다.



### reference

* https://www.baeldung.com/spring-events
* https://dzone.com/articles/transaction-synchronization-and-spring-application
