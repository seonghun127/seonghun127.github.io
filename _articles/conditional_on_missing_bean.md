---
id: 8
title: "@ConditionalOnMissingBean"
subtitle: "모르는 어노테이션 정리"
date: "2020.04.07"
tags: "spring","annoation"
---


## @ConditionalOnMissingBean 동작 과정



개발 중에 `@ConditionalOnMissingBean` 이라는 것을 알게 됐다.

위 어노테이션은 `@Bean` 으로 빈등록을 할 때 사용하는 어노테이션인데 ApplicationContext에 해당 타입의 빈이 없을 때만 빈으로 등록되는 조건을 주는 어노테이션이다. 만약 빈 등록을 할 때 같은 타입의 빈이 여럿 등록되고 우선순위에서 밀린다면 이 어노테이션을 추가하여 불필요하게 빈으로 등록되는 설정을 피할 수 있다.

`@ConditionalOnMissingBean` 어노테이션이 어떻게 작동되는지 간단하게 테스트를 해봤다.

> 테스트는 [이전 포스팅]()에서 확인했던 ObjectMapper를 기준으로 진행할 예정이다. autoConfiguration으로 등록되는 External Library의 jacksonObjecMapper() 메서드에 `@ConditionalOnMissingBean` 어노테이션이 붙어있는 것을 확인할 수 있었다. 

![](https://user-images.githubusercontent.com/30451129/78672467-a48cf300-791b-11ea-933b-05fcb139bf11.png)

일단 ApplicationContext에 등록되는 모든 빈의 이름을 출력하는 메서드를 만들어봤다. 위 코드는 [github](https://github.com/seonghun127/archive/tree/conditional_on_missing_bean)에서 확인할 수 있다. 

![](https://user-images.githubusercontent.com/30451129/78672476-a8207a00-791b-11ea-8d49-844452a34b19.png)

이 상태로 어플리케이션을 실행시키면 아래와 같이 콘솔창에 등록된 빈들의 이름을 확인할 수 있다

![](https://user-images.githubusercontent.com/30451129/78672486-ab1b6a80-791b-11ea-966d-e17978a0fc5a.png)



`JacksonAutoConfiguration` 클래스에서 등록하고 있는 `jacksonObjectMapper` 빈이 있는 것을 볼 수 있다. 그럼 여기서 다른 곳에서 `ObjectMapper` 타입의 빈을 등록하면 `@ConditionalOnMissingBean` 어노테이션에 의해 이 빈이 등록되지 않을 것이다.

프로젝트 내부에 `ObjectMapper` 를 직접 빈으로 등록해보자.

![](https://user-images.githubusercontent.com/30451129/78672493-ad7dc480-791b-11ea-9b52-04b9c95f6e49.png)

그리고 다시 프로젝트를 돌려보면

![](https://user-images.githubusercontent.com/30451129/78672502-b078b500-791b-11ea-954f-ac40dcadad20.png)

![](https://user-images.githubusercontent.com/30451129/78672510-b2db0f00-791b-11ea-91ad-4b06c326f379.png)

예상대로 `jacksonObjectMapper` 이름을 가진 빈은 존재하지 않다는 것을 확인할 수 있다.

그리고 콘솔창을 스크롤로 올리면 `SpringConfig` 에서 직접 등록한 `customObjectMapper` 빈이 있는 것을 확인할 수 있다.

![](https://user-images.githubusercontent.com/30451129/78672522-b53d6900-791b-11ea-8845-e0f2968a905a.png)

`customObjectMapper` 를 보면 빈 이름 출력 리스트의 상단에 위치하고 있다. 이는 ComponentScan에 의해 등록되는 빈이기에 External Library에서 등록될 빈들보다 먼저 등록된다. 그리고 난 후 autoConfiguration에 의해 Exteranl Library의  `jacksonObjectMapper` 와 같은 빈들이 등록된다. 그렇기 때문에 `@ConditionalOnMissingBean` 으로 보다 늦게 등록되는 빈은 빈으로 등록되지 않는다.

`JacksonAutoConfiguration` 의 `jacksonObjectMapper` 는 json 데이터와 java Object의 변환을 위해 등록되는 것인데 외부 라이브러리를 사용하는 사용자의 편의를 위해 `@ConditionalOnMissingBean` 으로 등록되는 것으로 보인다.

기본적으로 `ObjectMapper`를 빈으로 자동 등록해주면서 만약 해당 빈을 커스텀하게 사용하려는 사용자들에게는 빈 등록의 혼돈을 주지 않을 수 있는 이점을 가질 수 있다.
