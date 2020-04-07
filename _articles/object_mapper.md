---
id: 7
title: "Object Mapper"
subtitle: "ObjectMapper로 삽질하기"
date: "2020.04.06"
tags: "spring","object mapper"
---


최근에 `ObjectMapper`를 가지고 삽질한 적이 있다. 편리하게 사용하고만 있던 이 녀석에 대해 간단하게나마 공부를 한 내용을 공유하고자 한다.

사건의 발단은 이랬다. 멀티모듈 환경에서 다른 시스템과 통신을 주고 받는 로직을 third party 모듈로 옮기는 작업을 진행하고 있었는데 그곳에서 빈으로 등록하는 `ObjectMapper`가 말썽이었다. 자꾸 로그에 `com.fasterxml.jackson.databind.exc.UnrecognizedPropertyException`가 뜨는 것이었다. 무슨 일이고 보니 데이터에서 전혀 모르는 필드가 인식되어 파싱할 수 없다는 이야기였다.

사실 이 부분에 대해 깊게 생각해본 적이 없다. 나는 ObjectMapper를 사용하는 `@RequestBody`에 대한 객체 변환 시에도 모르는 필드에 대해서는 가볍게 무시되고 존재하는 필드에 대해서만 값들이 잘 assign된다는 걸 생각해서 당연히 `ObjectMapper`에 별다른 추가 설정없이 빈으로 등록했다.

오류를 해결하기 위해 구글링을 했다. 답은 간단했다. `ObjectMapper`를 빈으로 등록할 때 unknown properties를 무시하라는 설정을 추가해주면 됐다.

```java
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
```

위 설정만 추가해주면 필드에 없는 데이터가 들어와도 무시가 된다.

문제는 간단히 해결되는 듯 했지만 다른 문제가 발생했다. third party 모듈에서 등록하는 `ObjectMapper`빈을 상위 모듈에서 전부 사용하게 된다는 것. 당장의 무슨 문제를 일으키는 것은 아니지만 다음과 같은 상황에 문제가 될 수 있다.

- 누군가가 상위 모듈에 같은 빈을 등록하려고 할 경우 같은 빈을 여러개 등록하는 상황 발생
- 그렇다면 위 상황을 구분지어야할텐데 서로 다른 모듈에서 등록되기 때문에 관리 포인트가 늘어남
- 그전까지 third party의 상위 모듈은 전부 third party 에서 등록하는 빈을 사용하고 있었는데 새롭게 빈을 등록하게 되는 순간 빈의 설정 상태에 따라 다양한 오류 발생 가능

위 문제는 등록하는 빈과 주입되는 빈의 이름을 명시하는 방법으로 해결할 수 있겠지만 만약 이를 명시하지 않은 코드가 다양하게 퍼져있을 땐 이를 감당하는 것은 매우 힘든 일이 될 것이다.

이 부분에 대해 third party에 꼭 `ObjectMapper`를 빈으로 등록해야되냐는 동료 개발자 분의 피드백에 흠칫 놀랐다. 당연지사라고 생각했던 부분이 당연하지 않아도 될거 같단 기분이 들었기 때문이다.

그런데 여기서 든 의문은 그럼 Controller에서 `@RequestBody`가 붙은 객체를 파싱할 때는 어떤 `ObjectMapper`를 사용할까? 라는 생각이 들었다. 그래서 일단 그 `ObjectMapper`부터 찾아나섰다. 누가 어디서 등록해줄까?

그의 답은 External Library에 있다.  `org.springframework.boot.autoconfigure.jackson` 경로에 있는 `JacksonAutoConfiguration` 클래스에서  `jacksonObjectMapper`로 등록하고 있었다. (이는 autoConfiguration에 의해 등록되는 빈이다.)
![](https://user-images.githubusercontent.com/30451129/78565354-c07b9080-7858-11ea-839b-d5bc6652a8c5.png)

위 코드를 보면 `ObjectMapper` 등록 시 인자로 `Jackson2ObjectMapperBuilder`를 받는데 이 클래스의 spring docs를 보면 아래와 같이 나와있다.

![](https://user-images.githubusercontent.com/30451129/78565361-c40f1780-7858-11ea-8a97-6ada570375af.png)

기본적으로 `ObjectMapper`를 생성하는데 사용되는 이 클래스는 `DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES` 가 비활성화되어있다. default 값은 true다. 만약 true인 상태에서 unknown properties 값을 역직렬화 시 예외를 던진다.

[레퍼런스](http://tutorials.jenkov.com/java-json/jackson-objectmapper.html#ignore-unknown-json-fields)에 아래와 같이 나와있다.

![](https://user-images.githubusercontent.com/30451129/78565362-c4a7ae00-7858-11ea-8457-ea488ff6ea84.png)

그렇다면 기본적으로 등록되는 `ObjectMapper`빈이 있으니 추가로 빈 등록을 해줄 필요가 있을까? 본론으로 돌아와 아까 피드백을 주신 동료 개발자분의 이야기 답을 하자면...

***추가로 빈을 등록해줄 필요는 없다!***

다른 추가 설정이 필요한 경우에는 등록하는 가치가 있겠지만 사실 현재 그런 상황도 아닌지라 기존에 등록되는 빈 그대로 사용해도 무방하다.

그렇게 문제는 `ObjectMapper` 빈 등록 설정을 지우면서 해결했다. 허무맹랑하지만 이렇게 문제는 잘 마무리 되었...😂
    

지만 몇가지가 궁금했던 터라 간단한 테스트를 진행해봤다.
아래 실습 코드는 모두 [github](https://github.com/seonghun127/archive/tree/object_mapper)에 있다.

우선 테스트 코드로 unknown properties를 줘봤다.

일단 테스트 할 클래스 하나를 만들었다.

```java
@Getter
@NoArgsConstructor                  // ObjectMapper는 기본생성자를 이용하여 역직렬화를 하기 때문에 기본 생성자를 추가해줬다.
@EqualsAndHashCode
public class TestEntity {

    private String value;

    public TestEntity(String value) {
        this.value = value;
    }
}
```

테스트 코드 하나를 작성해보자.

```java
@Test
void unknown_properties_test() throws JsonProcessingException {
    
    ObjectMapper objectMapper = new ObjectMapper();

    String json = "{\\"value\\":\\"test\\"}";

    TestEntity object = objectMapper.readValue(json, TestEntity.class);

    assertThat(object).isEqualTo(new TestEntity("test"));
}
```

위 테스트는 초록불이 들어온다. 기본 `ObjectMapper`를 사용했지만 `TestEntity`필드의 이름도 정확히 맞춰줬기 때문에 별 문제 없는 코드다.

여기서 String json에 다른 claim을 추가해보자.

```java
@Test
void unknown_properties_test() throws JsonProcessingException {

    ObjectMapper objectMapper = new ObjectMapper();    
    
    String json = "{\\"value\\":\\"test\\", \\"value2\\":\\"test2\\"}";

    TestEntity object = objectMapper.readValue(json, TestEntity.class);

    assertThat(object).isEqualTo(new TestEntity("test"));
}
```

![](https://user-images.githubusercontent.com/30451129/78565363-c5404480-7858-11ea-88e1-3d3e28bb12e8.png)

그러면 바로 위와 같은 오류가 난다. `Unrecognized field "value2" (class com.example.springpractice.entity.TestEntity), not marked as ignorable (one known property: "value"])` 오류를 보면 "value2"라는 필드를 알 수 없다고 한다.

위 테스트를 성공시키기 위해 unknown properties를 무시하라는 설정을 추가해보자.

```java
@Test
void unknown_properties_test() throws JsonProcessingException {
	
    ObjectMapper objectMapper = new ObjectMapper();
	
    objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);   // 설정 추가

    String json = "{\\"value\\":\\"test\\", \\"value3\\":\\"test2\\"}";

    TestEntity object = objectMapper.readValue(json, TestEntity.class);

    assertThat(object).isEqualTo(new TestEntity("test"));
}
```

`objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);`이렇게 간단한 설정으로 위 테스트는 잘 돌아가게 된다.

추가로 위에서 이야기 했던 `@RequestBody`는 어떻게 될지도 테스트 해보자.

기본 `ObjectMapper`를 빈으로 등록해주자.

```java
@SpringBootApplication
public class SpringPracticeApplication {

    public static void main(String[] args) {
	SpringApplication.run(SpringPracticeApplication.class, args);
    }

    @Bean
    public ObjectMapper objectMapper() {
	return new ObjectMapper();               // 아무 설정 없는 ObjectMapper 등록
    }
}
```

간단하게 하기 위해 따로 @Configuration 클래스를 만들지 않고 @SpringBootApplication 클래스에 추가해줬다.

그리고 간단하게 Controller도 만들어주자.

```java
@RestController
public class TestController {

    @GetMapping("/")
    public String index(@RequestBody TestEntity testEntity) {
        return testEntity.getValue();
    }
}
```

그리고 http 파일을 만들어 테스트를 진행해보자.

```http
### test
GET localhost:8080/
Content-Type: application/json

{
  "value": "test"
}

###
```

그리고 위 파일을 실행시켜보면 잘 돌아가는 것을 확인할 수 있다.

![](https://user-images.githubusercontent.com/30451129/78565365-c5404480-7858-11ea-8c1b-cc9d3b801834.png)

200 응답이 내려오고 응답 바디에 test가 담겨져 있는 것을 확인할 수 있다. test 값은 컨트롤러에서 내려주는 값이다.

그럼 이번에 json payload에 unknown claim을 추가하고 돌려보자.

그 결과는 아래와 같다.

![](https://user-images.githubusercontent.com/30451129/78565366-c5d8db00-7858-11ea-96c2-bb98dc6b543e.png)

400 응답과 함께 value2라는 필드를 읽을 수 없다는 에러 메세지가 반환됐다.

이로써 `ObjectMapper`는 기본적으로 존재하지 않는 필드에 대한 데이터 파싱을 할 수 없다는 것을 눈으로 확인할 수 있었다. 그리고 추가로 `org.springframework.boot.autoconfigure.jackson` 경로에 있는 `JacksonAutoConfiguration` 클래스에서 등록하는  `jacksonObjectMapper`빈으로 spring은 json 데이터와 Object를 변환시키고 있었다는 것을 알 수 있었다. (그리고 이 빈은 unknown properties를 무시하는 설정이 되어 있다는 점도 알 수 있었다 👏)

정말 간단 내용이지만 나같이 신경쓰지 않으면 여러 방면으로 삽질하게 될 수 있을거라 생각한다.

**이래서 기본이 정말 중요하구나라고 다시 느끼게 된다. 그리고 공부해야할 부분이 많다고 느낀다. 🤣**
