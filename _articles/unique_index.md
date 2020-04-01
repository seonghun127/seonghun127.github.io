---
id: 6
title: "Unique Index"
subtitle: "Index를 몰라 더 삽질했던 Unique Index"
date: "2020.04.01"
tags: "database","index","spring"
---



*이 글은 부족한 지식에서 출발한 삽질의 연속을 담은 글이다. (데이터베이스에 대한 지식이 많이 부족하다는 것을 처참히 느꼈다...)*

나는 인덱스에 대해 막연하게만 알고 있다. 속으로 공부해야지, 해야지를 수십번 다짐했지만 결국 나중에 라는 결론을 내리면서 지금 이런 상황이다.😅

그런 와중에 Unique Index라는 것을 접하게 됐다.

그래서 일단 Unique Index에 찾아보았다. 개념은 인덱스 값의 중복을 허용하지 않는 인덱스를 말한다. 정의는 되게 간단하다. 하지만 궁금한 점이 생겼다. Unique Index를 걸면 유니크 제약조건이 자동으로 먹히나...?

현재 JPA를 사용하고 있어 @Index 어노테이션을 사용하여 인덱스를 걸어주고 있었는데 이 어노테이션의 속성을 까보다가 boolean unique 속성이 있다는 것을 알게 됐다.

![](https://user-images.githubusercontent.com/30451129/78143502-571b0c80-7469-11ea-81a7-46e6098a2a81.png)

음...인덱스에 unique 속성이라... 문뜩 든 생각은 unique 속성을 가지는 인덱스라면 해당 칼럼에 유니크함을 보장하게 되지 않을까? 라는 것이다.

그렇다면 굳이 따로 유니크 제약조건을 명시하지 않아도 자동으로 유니크 제약 조건이 생길테니까!

> 사실 유니크 제약 조건을 명시하는 것이 어려운 것은 아니기에 인덱스도 걸어주고 유니크 제약조건도 걸어주면 되지만 그냥 궁금해서 이것저것 해보고 있다... 😅

그래서 간단하게 테스트를 만들어보고 확인해봤다. 이 코드는 모두 [github](https://github.com/seonghun127/archive/tree/unique_index)에 있다.

> 테스트는 h2 DB를 사용하고 있으며 jpa의 auto-ddl : create-drop로 테이블을 새롭게 생성하고 있다.

데이터 베이스 설정을 먼저 해주자.

```groovy
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driverClassName: org.h2.Driver
    username: sa
    password: 1234
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
      open-in-view: false
    hibernate:
      ddl-auto: create-drop
  h2:
    console:
      enabled: true
      path: /h2-console
```

테스트할 도메인 Member이다. (예제에선 EcoMember인데 이는 테스트 DB에 Member 테이블이 이미 있어 Eco라는 이름을 그냥 붙혀줬다... 사실 아무 의미 없다 😂)

우선 Unique Index를 적용하지 않고 memberNo 필드에 인덱스와 유니크 제약조건을 각각 걸어줬다. (pk에 인덱스를 명시해준 이유는 사실 없다. 테스트해보면서 pk에 명시하게 될 경우 어떻게 될까 궁금했을 뿐이다. 여기서는 memberNo 필드에 집중하면 된다.)

```java
@Entity
@Table(
        name = "eco_member",
        indexes = {
                @Index(name = "ik_eco_member_id", columnList = "id"),
                @Index(name = "ik_eco_member_member_no", columnList = "memberNo")
        }
        ,
        uniqueConstraints = {
                @UniqueConstraint(name = "uk_eco_member_member_no", columnNames = {"memberNo"})
        }
)
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class EcoMember {

    @Id
    @GeneratedValue
    private Long id;

    @Column(nullable = false, length = 12)
    private String memberNo;

    @Column(nullable = false)
    private Integer count;

    @Builder
    public EcoMember(String memberNo, Integer count) {
        this.memberNo = memberNo;
        this.count = count;
    }
}

public interface EcoMemberRepository extends JpaRepository<EcoMember, Long> {
}
```

테스트 코드를 작성해보자.

```java
@SpringBootTest
class SpringPracticeApplicationTests {

	@Autowired
	private EcoMemberRepository ecoMemberRepository;

	@Test
	void 유니크_제약조건_테스트() {
		// 1. Given
		String memberNo = "123";
		EcoMember ecoMember1 = EcoMember.builder()
				.memberNo(memberNo)
				.count(123)
				.build();
		EcoMember ecoMember2 = EcoMember.builder()
				.memberNo(memberNo)
				.count(123)
				.build();

		// 2. When
		ecoMemberRepository.save(ecoMember1);
		ecoMemberRepository.save(ecoMember2);         // 유니크 제약 조건 위반을 발생시키는 지점이다.

		// 3. Then
	}
}
```

그리고 테스트를 돌려보면 당연히 실패한다. 에러 메세지를 확인해보자.

![](https://user-images.githubusercontent.com/30451129/78143512-5a15fd00-7469-11ea-8431-b4340ed68b20.png)

보면 `Unique index or primary key violation: "PUBLIC.UK_ECO_MEMBER_MEMBER_NO_INDEX_D ON PUBLIC.ECO_MEMBER(MEMBER_NO)`라는 에러 메세지를 볼 수 있다. 유니크 제약 조건에 위배됐다는 말이다. 유니크 제약 조건 이름(UK_ECO_MEMBER_MEMBER_NO)을 보니 엔티티에서 명시한 제약 조건이 잘 적용된 것이 확인된다.

![](https://user-images.githubusercontent.com/30451129/78143518-5bdfc080-7469-11ea-9f32-52a22fe9e6b8.png)

그리고 하이버네이트에 의해 테이블이 생성될 때 명시한 인덱스와 유니크 제약조건이 잘 설정되고 있다는 것을 확인할 수 있다.

이제는 한번 엔티티에 유니크 제약조건을 제거한 뒤 @Index의 unique 속성을 true 바꿔주고 다시 테스트를 돌려보자.

```java
@Index(name = "ik_eco_member_member_no", columnList = "memberNo", unique = true)
Entity
@Entity
@Table(
        name = "eco_member",
        indexes = {
                @Index(name = "ik_eco_member_id", columnList = "id"),
                @Index(name = "ik_eco_member_member_no", columnList = "memberNo", unique = true)
        }
//        ,
//        uniqueConstraints = {
//                @UniqueConstraint(name = "uk_eco_member_member_no", columnNames = {"memberNo"})
//        }
)
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class EcoMember {

    @Id
    @GeneratedValue
    private Long id;

    @Column(nullable = false, length = 12)
    private String memberNo;

    @Column(nullable = false)
    private Integer count;

    @Builder
    public EcoMember(String memberNo, Integer count) {
        this.memberNo = memberNo;
        this.count = count;
    }
}
```

![](https://user-images.githubusercontent.com/30451129/78143522-5bdfc080-7469-11ea-8d43-1bce54621a95.png)

테스트를 결과를 보니 같은 에러가 발생했다. 에러 메세지는 `Unique index or primary key violation: "PUBLIC.IK_ECO_MEMBER_MEMBER_NO_INDEX_D ON PUBLIC.ECO_MEMBER(MEMBER_NO)`이다.

이전 상황과 같은 유니크 제약조건 위배의 에러메세지인 것을 확인할 수 있다. 여기서 다른 점은 제약 조건의 이름이 IK_ECO_MEMBER_MEMBER_NO라는 것이다. 이는 엔티티에서 인덱스를 걸어줄 때 설정한 인덱싱 이름이다.

이를 통해 확인할 수 있는 것은 Unique Index의 경우 해당 칼럼에 대해 유니크 설정을 같이 적용해준다는 것을 유추할 수 있다.

![](https://user-images.githubusercontent.com/30451129/78143526-5c785700-7469-11ea-9aaf-79228d7c58d4.png)

하이버네이트가 설정해주는 제약조건을 보면 이전과 다름을 알 수 있다. (음... 뭔가 이상한 느낌이 들긴하지만 일단 넘어가보자...) 보다 명확히 확인하기 위해 DB를 MySql로 바꾸어 설정 정보를 Workbench 에서 확인해보았다.

> MySql로 바꾸는 설정은 생략하겠다. 🙏

MySql 로 바꾸고 테스트를 돌려 생성된 테이블의 인덱스 정보를 조회해봤다.

![](https://user-images.githubusercontent.com/30451129/78143528-5d10ed80-7469-11ea-9124-c9ee39f4dc72.png)

확인해보니 엔티티에서 설정한 id (pk), memberNo 필드에 대해 인덱스가 걸려있는 것을 확인할 수 있다. 인덱스 정보를 보면 Non_unique 칼럼이 있는데 이 칼럼이 Unique Index인지에 대한 정보를 나타낸다. 1 : true, 0 : false 이므로 key_name이 PRIMARY와 ik_eco_member_member_no 인 인덱스 키가 유니크로 설정되어 있다.

> 참고로 PRIMARY는 기본키에 대한 인덱스 이름인데 DB는 테이블을 생성할 때 자동으로 기본키에 대해 인덱스를 걸어놓는다. 그래서 엔티티에서 명시했던 id(pk)에 대한 인덱스(ik_eco_member_id)와 DB가 걸어놓은 pk에 대한 인덱스(PRIMARY)도 조회되고 있다.

*여기까지 직접 실습하면서 이해가 잘 됐다. 크게 이상한 점도 없었기에 어느정도 마무리하면 되겠구나 라고 생각하면서 밑에 추가로 간단한 실습으로 마무리 지으려고 했다. 그런데...*

추가로 유니크 제약조건과 인덱스를 같이 명시한 상황에서의 인덱스 정보는 아래와 같다. (유니크 제약조건에 대한 정보와 인덱스 정보가 따로 조회되고 있다.)

![](https://user-images.githubusercontent.com/30451129/78143531-5da98400-7469-11ea-80b2-22829422268b.png)

이상한 점을 발견했다. 왜 유니크 제약조건을 걸어준 정보(uk_eco_member_member_no)가 인덱스 정보 조회에서 나오는지 이해가 가질 않았다.

```sql
show index from table 명;
```

나는 분명 위 쿼리로 인덱스 정보를 조회한 것인데 제약조건에 대한 정보도 나오냔 말이다?! 그래서 다시 구글링을 했다...

그렇게 구글링을 통해 새롭게 알게된 사실은 인덱스 자동 생성에 대한 새로운 정보다. 위에서 언급했듯이 인덱스 자동생성은 기본키에 대해 적용되는데, 추가로 제약조건이 걸린 칼럼에 대해서도 자동생성된다고 한다! (전혀 모르고 있었던 내용이다...😢)

그래서 유니크 제약 조건을 걸어줬던 member_no 칼럼에도 Unique Index가 자동 생성된다.

![](https://user-images.githubusercontent.com/30451129/78143526-5c785700-7469-11ea-9aaf-79228d7c58d4.png)

그래서 아까 이상한 조짐이 보였던 부분을 다시 보면, 분명 `@Index(name = "ik_eco_member_member_no", columnList = "memberNo", unique = true)`설정으로 memberNo 필드에 인덱스를 걸어줬는데 DB 쿼리로는 유니크 제약조건 설정 쿼리가 날라갔던 것이다. 그러면 인덱스 자동 생성에 의해 해당 칼럼에 Unique Index가 걸리게 된다.

정리해보자면 하나의 칼럼에 대해 유니크 제약조건을 설정해줘야하고 동시에 인덱스도 걸어줘야하는 상황이라면 직접 두가지를 모두 명시해줘도 되지만 Unique Index로 한번에 명시할 수 있다. 사실 이 이야기를 하기 위해 돌고 돌아왔다. 😅 (반대로 유니크 제약 조건만 명시해줘도 된다!)

확실히 인덱스와 제약조건을 같이 명시하는 것은 프로덕션 코드가 쉽게 길어진다. 상황에 맞게 하나만 명시해주는 것이 훨씬 간결한 코드를 유지할 수 있을 것이다. 어차피 동작은 같으니까!

만약 여러 칼럼에 각각의 인덱스를 걸어주고 있는 상황이라면 유니크 제약 조건을 줘야하는 칼럼에도 @Index의 unique 속성을 true 주는 것도 하나의 방법이겠다. (통일성을 위해! 🤛)

*추가로...*

사실 이 글을 읽으면서 의아할 수 있는 점이 있는데 memberNo 필드 하나에 대해 유니크 제약조건을 걸어줄 거면 @Column 어노테이션의 unique 속성을 true로 바꿔주면 보다 간결하게 설정을 줄 수 있다는 점이다.🤣

```java
@Column(unique=true)
```

나는 @UniqueConstraint 어노테이션로 유니크 제약조건을 줬는데 이는 제약 조건 이름을 같이 설정해줄 수 있기 때문이다. (동작에 차이는 없다.)

만약 이름을 따로 설정해주지 않는다면 `UK_ioorg3rqhkfhgwoha1uwabppg`와 같은 형태의 이름이 나온다. 이는 무슨 말을 하고 있는지 전혀 알 수 없기 때문에 직접 이름을 주는 것이 명확하다는 점과 @Column 어노테이션에서 줄 수 있는 유니크 설정은 단일 칼럼에 대해서만 적용된다는 점을 놓고 봤을 때 @UniqueConstraint 어노테이션이 더 명확하다고 생각한다.
