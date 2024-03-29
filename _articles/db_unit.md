---
id: 12
title: "DB Unit 적용해보기"
subtitle: "DB Unit 적용기"
date: "2020.09.03"
tags: "spring","dbunit"
---



## DB Unit 이란?

DB Unit는 JUnit 확장 라이브러리다. DB Unit을 사용하면 테스트 수행 시 발생할 수 있는 DB의 데이터 오염을 방지할 수 있다. 테스트 수행 전과 후 테스트하려는 데이터 처리를 대신해주어 실제 테스트하려는 로직에 더 집중할 수 있게 해준다.



## DB Unit을 사용하면 뭐가 좋을까?

흔히 로컬 환경에서 (통합) 테스트를 수행하기 위해 로컬에 설치한 데이터베이스에 데이터를 미리 넣어두고 원하는 결과가 나오는지에 테스트를 한다. 이런 경우 테스트의 갯수가 많지 않다면 관리하기 용이하겠지만 테스트가 점점 많아질수록 데이터 관리하는 것이 까다로워지고 단일로는 성공적으로 돌아가던 테스트가 전체 테스트 수행 시에는 실패하는 경우도 발생할 수 있다.

DB Unit은 이런 경우를 예방하고 테스트를 위한 데이터를 관리(오염 방지)하기 편리한 라이브러리라고 생각하면 된다. 단위 테스트에서는 실제 데이터베이스에 데이터를 저장하는 경우는 적을 수 있지만 실제 조회 쿼리가 잘 수행되는지 확인하기 위해 수행하는 통합 테스트에서는 상당히 효과적이라고 볼 수 있다.



## 어플리케이션에 DB Unit을 적용해보자.

**실습 환경**

- DB Unit
- spring boot 2.3.3
- kotlin
- JPA
- Mysql
- JUnit 5

*실습 코드는 모두 [여기](https://github.com/seonghun127/archive-kotlin/tree/master/flywayanddbunit)에 있다.*

실습에서 사용할 엔티티부터 만들어보자.

*Member.kt*

```kotlin
@Table(name = "members")    // table 이름을 members라고 한 이유는 로컬 DB에
@Entity                     // member 테이블이 이미 존재해서 구분짓기 위함
class Member(

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long,

    @Column
    var name: String,

    @Column
    var age: Long,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id", nullable = false, updatable = false)
    var team: Team
)
```

*Team.kt*

```kotlin
@Entity
class Team(

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long,

    @Column
    var name: String,

    @OneToMany(mappedBy = "team")
    val members: MutableList<Member> = mutableListOf()
)
```

*엔티티 클래스와 더불어 Repository로 같이 만들어주자. (코드는 생략햐겠다.)*

spring에서 사용할 수 있는 DB Unit 의존성 추가를 해주자.

*build.gradle.kts*

```groovy
dependencies {
		...

    testImplementation("com.github.springtestdbunit:spring-test-dbunit:1.3.0")
    testImplementation("org.dbunit:dbunit:2.6.0")

		...
}
```

로컬에 Mysql를 설치하고 connection을 맺기 위한 설정과 JPA 설정을 yml에 추가해준다.

*/main/resources/application.yml*

```yaml
spring:
  datasource:
    driverClassName: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:13306/testdb
    username: root
    password: 1234
	jpa:
    hibernate:
      ddl-auto: none
    show-sql: true              // 테스트를 진행하면서 날라가는 쿼리를 보기 위함
    properties:
      hibernate:
        format_sql: true        // 포맷팅된 쿼리를 보기 위함
```

이제 로컬 데이터베이스에 테이블을 생성해주자. (flyway 사용했음)

```sql
CREATE TABLE members (    // member 엔티티의 매핑 테이블이름을 members라고 설정해놓은 상황
    id          BIGINT          NOT NULL PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(15)     COMMENT '이름',
    age         BIGINT          COMMENT '나이',
    team_id     BIGINT          COMMENT '팀 ID'    // 실제 테이블간 reference는 주지 않음
);                                                // flyway에서 외래키 설정 시 migration이 되지 않는 상황이 발생
                                                  // 실제 외래키를 설정해줘도 상관 x
CREATE TABLE team (
    id          BIGINT          NOT NULL PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(15)     COMMENT '이름'
);
```

이제 테스트 코드를 짜보자.

*MemberRepositoryTest.kt*

```kotlin
@SpringBootTest                                       // 1
@TestExecutionListeners(value = [                     // 2
    TransactionDbUnitTestExecutionListener::class])   // 3
@Transactional
@DatabaseSetup(                                       // 4
		value = ["member.xml"], type = DatabaseOperation.CLEAN_INSERT  // 5
)
class MemberRepositoryTest @Autowired constructor(
    private val memberRepository: MemberRepository
) {

    @Test
    fun findById() {
        // given
        val id = 2L

        // when
        val member = memberRepository.findByIdOrNull(id)

        // then
        assertThat(member!!.id).isEqualTo(id)
        assertThat(member.name).isEqualTo("test1")
        assertThat(member.age).isEqualTo(28)
    }

}
```

1. 프로덕션 코드에서 설정해준 dataSource 를 가지고 로컬 데이터베이스를 접근할 것이기 때문에 테스트 컨텍스트의 스프링 부트를 띄워준다.
2. 스프링에서 DB Unit을 사용하기 위해서는 `TestExecutionListener` 어노테이션을 이용해야한다. 

    `@TestExecutionListener`는 TestContextManager에 등록되어야하는 TestExecutionListener들을 설정하는 클래스 레벨의 메타 데이터를 정의하는 어노테이션이다.

    `TestExecutionListener`는 TestContextManager에 의해 발행되는 테스트 실행 이벤트들에 반응하는 리스너 API를 정의하는 인터페이스다.

3. `TransactionDbUnitTestExecutionListener` 는 `TestExecutionListener` 를 구현하고 있는 `TransactionalTestExecutionListener` 과 `DbUnitTestExecutionListener` 를 체이닝하고 있는 클래스다.

    ![](https://user-images.githubusercontent.com/30451129/92079254-8fe55600-edfa-11ea-8e08-761ed099427d.png)

    - `TransactionalTestExecutionListener`는 스프링에서 가지고 있는 `TestExecutionListener`의 구체 클래스다. 이 클래스는 `@Transactional` 어노테이션이 붙어있는 테스트 메서드가 하나의 트랜잭션에서 수행될 때 자동으로 테스트 수행 후 트랜잭션을 롤백시키는 리스너 클래스다. 이 리스너를 사용해서 실제 데이터베이스를 사용하는 테스트를 수행할 때 데이터가 오염되는 것을 방지할 수 있다.
    - `DbUnitTestExecutionListener`는 springtestdbunit 라이브러리에 구현된 `TestExecutionListener`의 구체 클래스다. 이 클래스는 `@DatabaseSetup`, `@DatabaseTearDown`, `@ExpectedDatabase` 를 지원해준다. 이 어노테이션을 이용해서 테스트 수행 전, 후에 데이터를 쉽게 넣고 비교할 수 있다. DB Unit을 적용한 테스트를 작성한다면 반드시 필요한 리스너다.

    `@TestExecutionListener`에 `TransactionalTestExecutionListener` 와 `DbUnitTestExecutionListener` 를 각각 선언해줘도 상관없다. 어차피 똑같이 동작하기 때문이다. 두개의 클래스를 적어주는 것이 번거롭기 때문에 springtestdbunit 라이브러리에 이 두 클래스를 묶은 `TransactionDbUnitTestExecutionListener` 를 추가해준 것 같다.

4. `DbUnitTestExecutionListener`를 사용함으로써 지원되는 어노테이션이다. `@DatabaseSetup` 어노테이션은 이름에서 알 수 있듯이 테스트가 수행전에 데이터베이스가 Setup 되어야하는 상태를 관리하는 어노테이션이다.
5. `@DatabaseSetup` 어노테이션의 속성 값으로 value에 xml 파일 이름을 넣어주면 xml 파일에 적혀있는 내용에 맞춰 데이터베이스를 Setup 해준다. 두번째 속성 값으로 type이 있는데 이는 데이터베이스에 Setup할 때 어떤 operation을 수행할 것인 정의한다. 여기서는 `DatabaseOperation.CLEAN_INSERT` 로 설정해줬는데, 이는 데이터베이스의 테이블 로우를 전부 삭제하고 순차적으로 xml 파일의 컨텐츠를 삽입하는 operation이다.

이제 위 테스트가 제대로 돌아가게 하기 위해 xml 파일을 추가해주자.

*member.xml*

```xml
<dataset>
    <members id="2" name="test1" age="28" team_id="1" />
</dataset>
```

- `team_id="1"` 부분은 이미 필자는 team 테이블에 id 가 1인 데이터를 넣어둔 상태이기 때문에 가능한 것이기에 해당 xml을 짜기 전에 team 데이터를 하나 넣어두자.
- 위 파일은 `resources` 디렉토리 하위에 넣어줘야하는데 테스트 코드에서 경로가 아닌 파일명만 명시해줬기 때문에 테스트 코드 파일과 동일한 경로에 위치해야한다. (물론 절대경로로 호출한다면 테스트 코드 파일과 경로가 달라도 된다.)

    ```kotlin
    @DatabaseSetup(
    		value = ["member.xml"], type = DatabaseOperation.CLEAN_INSERT
    )
    ```

![](https://user-images.githubusercontent.com/30451129/92079268-9247b000-edfa-11ea-8115-8c910ec3a4ec.png)

이제 테스트를 실행시키기 위한 준비가 끝났다. 테스트를 돌려보면 정상적으로 동작하고 있다는 것을 알 수 있다.

아래 사진을 보면 테스트 성공과 함께 마지막에 롤백이 됐다는 거까지 확인할 수 있다.

![](https://user-images.githubusercontent.com/30451129/92079270-92e04680-edfa-11ea-9039-c60228627f23.png)

이런 테스트 환경을 구성하면서 **주의할 점** 몇가지만 정리해보자면,

- `TransactionalTestExecutionListener`는 반드시 `@Transactional` 이 붙은 테스트 메서드의 경우에만 자동 롤백시킨다는 점이다. 즉, `@Transactional` 어노테이션이 붙어있지 않으면 롤백되지 않고 커밋된다. 그래서 실제 데이터베이스에 데이터가 변경될 수 있다.
    
    - 기본적으로 `TransactionDbUnitTestExecutionListener` 는 트랜잭션 범위를`@DatabaseSetup` 이전에 시작해서 `@DatabaseTearDown` 이나 `@ExpectedDatebase` 이후에 끝나도록 설정하는데, 위 테스트 코드를 보면 `@DatabaseSetup` 를 사용하고 있어 `findById` 테스트 함수가 실행되면서 같은 트랜잭션으로 묶이게 된다. 여기서 `@Transactional` 을 붙이지 않고 돌리게 되면 테스트 코드의 트랜잭션이 커밋되어 `@DatabaseSetup` 에서 insert한 데이터들이 실제 데이터베이스에 추가된다. → 이는 이후 다른 테스트 코드에도 영향을 끼칠 수 있기 때문에 조심해야한다.
- `@DatabaseTearDown` 는 `TransactionDbUnitTestExecutionListener`를 사용한다면 무의미하다. `@DatabaseTearDown`는 이름에서 알 수 있듯이 테스트 수행 후 데이터베이스를 허무는 설정이다. 만약

    ```kotlin
    @DatabaseTearDown(value = ["member.xml"], type = DatabaseOperation.DELETE_ALL)
    ```

    이와 같이 설정해줬다면 테스트 수행 후 테이블의 있는 모든 로우를 삭제한다. 이는 `@DatabaseSetup` 이나 테스트 수행 과정에서 추가되는 데이터를 삭제하는데 용이한데 `TransactionDbUnitTestExecutionListener` 로 테스트 트랜잭션에서 일어나는 모든 행위를 롤백되기 때문에 해당 설정은 무의미할 수 있다.

    실제 `@DatabaseTearDown` 설정을 주고 테스트를 돌렸을 때 로컬 데이터베이스가 찍히는 로그는 아래와 같다.

    ![](https://user-images.githubusercontent.com/30451129/92079274-94117380-edfa-11ea-9362-e05594489bd8.png)

    롤백될 트랜잭션 내부 쿼리들이지만 데이터를 삭제하는 `delete from members` 쿼리가 추가된다. (`@DatabaseTearDown` 설정이 없다면 해당 쿼리는 찍히지 않는다.)  이 쿼리 하나 추가된다고 테스트 성능이 달라지는 것은 아니지만 만약 수행될 필요가 없는 상황이라면 해당 설정을 추가해주는 것이 무의미할 수 있다는 이야기다.
