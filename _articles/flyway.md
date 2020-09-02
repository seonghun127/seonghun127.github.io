---
id: 11
title: "Flyway 적용해보기"
subtitle: "flyway 적용기"
date: "2020.09.03"
tags: "spring","flyway"
---

# Flyway란?

flyway란 홈페이지에 나온 설명에 의하면 DB의 Version Control를 가능케하여 Migration을 편리하게 할 수 있는 도구라고 한다.

## 왜 Flyway를 사용할까?

여러명이서 어플리케이션을 개발할 때 필수적으로 요구되는 것이 코드에 대한 Version Control이다. 그래서 더욱이 CI (Continuous Integration)가 필요하며 원활히 개발하기 위해 서로 다른 코드의 Version(or Revision)을 맞춰나가야한다.

이를 위해 흔히 git이라는 기술로 형상관리를 많이 한다. github, gitlab 등이 유명한 이유가 이것이다. 그러나 데이터베이스에 대한 Version Control은 항상 논외였다. 운영 환경에 배포가 되기전까지 많은 stage(or phase)가 존재하는게 일반적인데, 각각의 환경에서 데이터베이스 Version Control은 수동으로 처리해야만 했다.

> 사실 수동으로 하는게 더 나을 수 있는 것이, 새로운 시스템 개발이 아니라면 DB 스키마 변경이 자주 일어나지 않기도 하며, DBA가 (또는 같이 상의하며) 튜닝을 하기 때문에 데이터베이스 스키마 변경은 오히려 자동화를 하는 것이 어설픈 데이터베이스가 될 수 있는 위험이 있기도 하다.

하지만 여러 환경 중 로컬, 테스트 환경 등은 스키마 변경 시 DB 성능보다는 제대로 실행되는지에 대한 빠른 피드백을 원하는 경우가 많기 때문에 **손쉬운** Migration이 절실히 필요하다. 이를 위해 flyway는 좋은 제안이 될 수 있다고 생각한다.

## Flyway는 어떻게 작동할까?

flyway은 Version Control을 하기 위해서 별도의 테이블을 사용합니다. Migration하는 database 내에 `flyway_schema_history` 이라는 테이블을 두어 Migration 히스토리를 저장합니다.

데이터베이스 Migration을 하기위해 DDL 또는 DML의 sql 스크립트를 짜두면 flyway가 이 스크립트를 실행하는 방식으로 진행되는데, sql 스크립트 파일 명을 바탕으로 Version, description 등을 저장한다.

![](https://user-images.githubusercontent.com/30451129/92002043-61c03180-ed7a-11ea-96bc-8044a35b62b4.png)

![](https://user-images.githubusercontent.com/30451129/92002082-697fd600-ed7a-11ea-8b43-7dd88656627e.png)


- *이를 위해 스크립트 파일 명의 컨벤션이 존재한다. (R에 대한 내용 추가 필요)*

    `V<version>__<description>.sql`

    - 맨 앞에는 V가 와야한다. 소문자 v여도 상관없다.
    - <version> 자리에는 숫자가 오는데, 보통 version 관리는 소수점을 이용하기에 소수점을 `_` 하나로 구분한다. 즉, 3.1 버전이라면 3_1로 <version> 자리를 채워야한다. ( `V3_1_<description>.sql` )
    - <description> 자리에는 해당 sql 파일에 들어있는 스크립트 내용을 명시해준다. git으로 따지면 commit message와 비슷한 역할이다. <description>에는 별도의 컨벤션이 있지 않다. 다만, <version> 명시 시 소수점을 `_` 로 구분하기 때문에 <version>과 <description>은 `_` 2개로 구분한다. <decription> 내부에 `_` 는 띄어쓰기 역할을 한다. 즉, 새로운 테이블을 추가하는 sql이라면 mesage로 'add table'이라고 줄 수 있는데, 이럴 경우 `V3_1__add_table.sql` 와 같은 형식으로 이름을 지어주면 된다.

    **위 컨벤션을 바탕으로 `flyway_schema_history` 에는 아래와 같이 데이터가 저장된다.**

    ![](https://user-images.githubusercontent.com/30451129/92002032-5ec54100-ed7a-11ea-8929-a58437c7c1f5.png)

flyway는 `flyway_schema_history` 테이블의 내용으로 현재 데이터베이스의 상태를 트랙킹한다. Migration 실행 시 filesystem이나 application classapath 를 스캔하여 Migration 시킬 스크립트를 찾고 해당 스크립트 파일 명을 바탕으로 `flyway_schema_history` 에 각각의 정보를 저장한 후 version 기준으로 정렬한다. 그런 다음에 파일을 적용하여 Migration을 실시한다.

만약 새로운 version의 Migration을 실시하기 위해 여러 스크립트 파일을 작성 후 Migration을 수행시킨다면 flyway는 `flyway_schema_history` 테이블을 보며 실제 수행되어야하는 스크립트를 version 기준으로 분류하여 실행시킨다. 현재 `flyway_schema_history` 테이블 내에 위 사진과 같이 데이터가 저장되있다면, 가장 최신 version인 3 이하의 version 스크립트 파일은 무시한다.

새로 추가된 스크립트 파일(version이 3보다 높은)들은 `flyway_schema_history` 테이블에 저장되어 정렬되고 순서대로 Migartion이 적용된다. 

*(version 3 이하인 파일들이 무시되는데, 나머지 version 이 높은 파일들은 pending 된다고 docs에 설명되고 있다. → 정확하게 이해하진 못했지만 version 이 낮은 파일들이 무시되는 동안 해당 파일들은 pending 상태가 된다고 이해했다. 물론, 추후에 다시 success 상태가 되면서 스크립트가 정상 실행된다.)*

## 어플리케이션에 flyway 적용해보기

spring framework를 사용해서 간단한 web application을 만들고 flyway를 적용해서 쉽게 필요한 데이터베이스 환경을 구축해보자.

**실습에 사용한 기술 스택**

- flyway plugins
- kotlin
- spring boot (gradle)
- mysql
- jpa (혹시 몰라 추가)

*실습 코드는 모두 [여기](https://github.com/seonghun127/archive-kotlin/tree/master/flywayanddbunit) 있다.*

실습에서 사용할 엔티티부터 만들어보자.

*Member.kt*

```kotlin
@Entity
class Member(

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long,

    @Column
    var name: String,

    @Column
    var age: Long,

    @ManyToOne
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

엔티티를 다 만들었으면 이제 flyway를 적용해보자.

flyway를 적용할 수 있는 방법은 여러가지 있다.

- Command-Line
- API (Java / Android)
- Maven
- Gradle
- Community Plugins

각각의 사용 방법은 [홈페이지](https://flywaydb.org/documentation/)*(USAGE 카테고리)* 에 자세히 나와있다.

이번 실습에는 gradle 빌드 도구를 사용하기에 Gradle을 방법을 사용하겠다.

flyway는 community edition, pro edition, enterprise edition 그래들 플러그인이 있다. 각각의 edition이 지원하는 Java 버전 및 gradle 버전은 홈페이지를 참조하자.

필자는 comunity edition 을 사용하겠다. 이를 위해 `build.gradle.kts`에 아래와 같이 설치 스크립트를 추가해준다.

*build.gradle.kts*

```groovy
plugins {
		...

    id("org.flywaydb.flyway") version "6.5.5"

		...
}
```

이 flyway 플러그인을 설치하면 해당 플러그인이 제공하는 Task를 사용할 수 있다. 이 Task들은 `./gradlew` 로 실행시킬 수 있다. Task 종류는 아래와 같다.

![](https://user-images.githubusercontent.com/30451129/92002047-6258c800-ed7a-11ea-84f8-a026446e9f68.png)

플러그인을 설치했으니 flyway가 해당 데이터베이스로 접근할 수 있도록 설정 스크립트를 추가해주자. (이번 실습은 single database 로 이뤄지고 있다. multiple database 도 가능하니 이 부분은 [도큐먼트](https://flywaydb.org/documentation/gradle/)를 참조하자.)

*build.gradle.kts*

```groovy
flyway {
    url = "jdbc:mysql://localhost:13306/testdb"
    user = "root"
    password = "1234"
    locations = arrayOf("filesystem:${file("src/migration").absolutePath}")
}
```

*참고로 필자는 docker로 local에 mysql을 띄운 상태이다. (포트에 오타난 것이 아니다.)*

위 설정 필드를 보면 다른건 다 알거라 생각하고 locations 부분만 설명하겠다.

위에서 말했듯이 flyway는 Migration할 sql 스크립트 파일을 filesystem이나 application classpath에서 찾는다고 했다. 해당 설정은 filesystem 방식으로 위치를 찾는 것이며 절대 경로 설정이다.

이 경로에 맞춰 폴더와 sql 파일을 생성해주자.

![](https://user-images.githubusercontent.com/30451129/92002049-62f15e80-ed7a-11ea-81f1-a04dc2a2295a.png)

*V1__create_table.sql*

```sql
CREATE TABLE member (
    id          BIGINT          NOT NULL PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(15)     COMMENT '이름',
    age         BIGINT          COMMENT '나이'
);

CREATE TABLE team (
    id          BIGINT          NOT NULL PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(15)     COMMENT '이름'
);
```

위에서 생성한 엔티티에 맞게 테이블을 생성하는 Mysql DDL 문이다. flyway로 해당 스크립트를 기준으로 Migration한다면 데이터베이스에 member, team 테이블이 생길 것이다.

여기까지 했다면 flyway 설정이 다 된 것이다. (매우 간단하다.)

이제 Task를 돌려 데이터베이스에 Migration해보자.

Migration Task는 `flywayMigration`이다.

터미널에서 프로젝트의 루트 경로로 이동하여 `./gradlew flywayMigration` 명령어를 실행시켜보자.

![](https://user-images.githubusercontent.com/30451129/92002052-6389f500-ed7a-11ea-82c5-da1ae3115216.png)

BUILD SUCCESSFUL이 잘 떴다면 데이터베이스로 들어가서 확인해보자.

![](https://user-images.githubusercontent.com/30451129/92002053-64228b80-ed7a-11ea-81e6-86210816961d.png)

데이터베이스에 새로 생성된 테이블을 보면 sql 스크립트에서 생성하려했던 member, team 테이블과 더불어 Version Control을 위해 여러 정보를 저장하는 flyway_schema_history 테이블도 생성된 것을 볼 수 있다.

flyway_schema_history 테이블에는 아래와 같이 정보가 저장돼있을 것이다.

![](https://user-images.githubusercontent.com/30451129/92002054-64228b80-ed7a-11ea-978f-d87256ae1351.png)

이로써 flyway가 어플리케이션에 잘 적용됐다. 추가로 flyway 플러그인의 다른 Task도 봐보자.

- **flywayClean**
    - 이름에서 유추해볼 수 있듯이 데이터베이스에 있는 모든 스키마를 삭제(Clean)시키는 Task다.
    - `./gradlew flywayClean` 명령어를 실행시켜보면

        ![](https://user-images.githubusercontent.com/30451129/92002056-64bb2200-ed7a-11ea-9d8b-49a92ebc5778.png)

        ![](https://user-images.githubusercontent.com/30451129/92002059-6553b880-ed7a-11ea-9f1f-10343176e259.png)

        이와같이 해당 데이터베이스에는 모든 스키마가 삭제된다.

    - 이는 더이상 Version Control 보다는 데이터베이스를 아예 새롭게 구축하는 것이 더 효율적이다고 판단될 때 사용할 수 있는 Task 라고 생각한다.
- **flywayInfo**
    - 이 Task는 Version Control 정보를 확인할 수 있는 명령어다. `./gradlew flywayInfo`

        ![](https://user-images.githubusercontent.com/30451129/92002061-6553b880-ed7a-11ea-8b4c-7e8e3ebc4e45.png)

    - 다시 아까 스크립트를 Migration 시키고 정보를 조회해보면 아래와 같이 Success 상태가 된 것을 볼 수 있다.

        ![](https://user-images.githubusercontent.com/30451129/92002062-65ec4f00-ed7a-11ea-8842-1454f1d0ff7e.png)

    - 추가 정보를 조회해보기 위해 새로운 파일을 만들어 Migration을 수행해보자.

        *V2__add_data.sql*

        ```sql
        INSERT INTO member (name, age) values ('name', 28);
        ```

        ![](https://user-images.githubusercontent.com/30451129/92002064-6684e580-ed7a-11ea-8ea8-cb78f96d7fc8.png)

        ![](https://user-images.githubusercontent.com/30451129/92002067-6684e580-ed7a-11ea-9ac5-c50a99d36204.png)

        데이터 추가 쿼리는 잘 수행돼 member 테이블에 데이터가 추가된 것을 볼 수 있고 flywayInfo에서도 Version이 2인 정보가 Success 

    - 현재 Schema version 보다 낮은 version의 sql 스크립트를 Migration 하면 어떻게 될까?

        *V1_1__lower_version_test.sql*

        ```sql
        INSERT INTO team (name) values ('team_name_test');
        ```

        ![](https://user-images.githubusercontent.com/30451129/92002068-671d7c00-ed7a-11ea-8869-ddc8dbd977be.png)

        Task는 Build에 실패하고 현재 Schema version 보다 낮은 1.1 version은 데이터베이스에 적용되지 않았다는 에러 메세지가 떴다.

        해당 Revision은 Ignored 상태가 됐다. 그렇기 때문에 실제 데이터베이스 team 테이블에는 아무 데이터도 추가되지 않았다.

        ![](https://user-images.githubusercontent.com/30451129/92002071-67b61280-ed7a-11ea-8021-e7c7ec8606c7.png)

        ![](https://user-images.githubusercontent.com/30451129/92002073-67b61280-ed7a-11ea-8df8-cde7cdccc1cc.png)

    - 이번엔 현재 version에 해당하는 Revision 파일인 V2__add_data.sql 파일을 **삭제**하고 그보다 높은 version 의 V2_1__add_team_data.sql 파일을 만들어 Migration 해보자.

        *V2_add_data.sql 삭제*

        ...

        *V2_1__add_team_data.sql*

        ```sql
        INSERT INTO team (name) values ('team_name_test');
        ```

        ![](https://user-images.githubusercontent.com/30451129/92002074-684ea900-ed7a-11ea-816d-c4205df7cae1.png)

        현재 version에 해당하는 2에 해당하는 Revision 파일을 찾지 못해 에러가 발생했다고 메세지가 출력된다. 2.1 버전의 Revision은 pending 되어 데이터베이스에 적용되지 않았다. *(그대로 Schema version은 2다.)*

        ![](https://user-images.githubusercontent.com/30451129/92002075-684ea900-ed7a-11ea-9bc6-b55d8bec4446.png)

        ![](https://user-images.githubusercontent.com/30451129/92002077-68e73f80-ed7a-11ea-8899-d6a151366990.png)

        이 상황은 git으로 따지면 commit을 새롭게 하려는데, 부모 커밋을 찾지 못하는 상황과 비슷하다. 그래서 새로운 commit이 실패했다고 볼 수 있다. 이런 상황은 다시 현재 Schema version 에 해당하는 Revision 파일을 추가해서 다시 Migration Task 돌리면 해결된다. (→ 2.1 version으로 Migration에 성공한다.)

- **flywayBaseline**

    기존에 존재하는 데이터베이스에 flyway를 적용할 때 사용하는 명령어다. 아무것도 없던 데이터베이스에 처음부터 flyway를 적용한다면 이 명령어가 필요 없겠지만 이미 존재하던 데이터베이스에 flyway로 마이그레이션을 시도하면 오류가 난다. 왜냐하면 flyway를 적용하기 위해선 `flyway_schema_history` 테이블이 필요하기 때문이다.

    flyway를 처음 적용하는 데이터베이스라면 `flyway_schema_history` 테이블이 없기 때문에 이 테이블부터 만들어줘야한다. 현재 적용하려는 데이터베이스의 상태를 기준 version 으로 잡고 `flyway_schema_history` 생성하여 Version Control를 하겠다는 명령어가 flywayBaseline인 것이다.

    ![](https://user-images.githubusercontent.com/30451129/92002084-6a186c80-ed7a-11ea-84e6-cc52de2f761b.png)

    만약 `flyway_schema_history` 테이블이 없는 상황에서 Migration을 시도한다면 위와같이 오류가 난다.

    ![](https://user-images.githubusercontent.com/30451129/92002086-6ab10300-ed7a-11ea-818c-7041f291e2ce.png)

    당연히 프로젝트에 있는 sql 파일들에 대한 상태 또한 pending 상태가 되어 데이터베이스에 반영이 되지 않는다. 이를 위해 `./gradle flywayBaseLine` 명령어를 실행해줘야한다.

    ![](https://user-images.githubusercontent.com/30451129/92002087-6ab10300-ed7a-11ea-8dc8-590d8801adec.png)



### Reference

* https://flywaydb.org/
