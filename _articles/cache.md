---
id: 0
title: "cache"
subtitle: "cache에 대해 알아보자""
date: "2020.03.18"
tags: "network","http 완벽가이드"
---

# CACHE



네트워크는 지구 반대편에 있는 리소스를 손쉽게 볼 수 있게 해주는 엄청난 존재다. 인터넷이라는 망을 통해 물리적으로 접근하기 힘든 정보를 간편하게 접할 수 있다. 그만큼 네트워크는 우리에게 많은 이익을 제공한다.

그러나 네트워크는 기본적으로 비용이 비싸다. 사실 한번의 네트워크 통신으로 많은 비용이 발생하지는 않는다. 하지만 이 트래픽이 많아지면 많아질수록 비용은 기하급수적으로 올라간다. 그래서 이 문제를 효율적으로 해결하기 위해서 나온 개념 중에 하나가 캐시다.

캐시는 네트워크 비용을 줄이고자 리소스의 사본을 캐시를 담당하는 프록시 서버나 웹 어플리케이션(클라이언트) 내장 메모리에 저장하여 그 다음 요청 시 적은 비용으로 리소스를 내려줄 수 있는 기능을 제공한다. 캐시를 효율적으로 사용하면 네트워크 비용을 줄일 수 있으며 인터넷 브라우저를 사용한다면 원하는 페이지를 빠르게 볼 수 있을 것이다.    

​    

## 캐시의 기본 흐름

* 클라이언트의 리소스 접근 요청을 받는다.
* 해당 리소스가 캐시 저장소에 존재하는지 확인한다.
* 있다면 기본적으로 캐시된 리소스르 내려주고 없다면 원 서버에 클라이언트의 요청 리소스 접근 요청을 보낸다.
  * 만약 캐시된 리소스가 있는 경우에 신선도 검사(재검사) 과정이 추가될 수 있다.
  * 캐시된 리소스를 클라이언트에게 내려줄 만큼 신선하다면 그대로 내려준다.
  * 그렇지 않다면 캐시된 리소스는 원 서버에게 신선도 검사를 받아야한다. 신선하지 않다는 것은 해당 리소스가 원서버에서 변경됐을 가능성이 있기 때문이다.
  * 신선도 검사 후 해당 리소스의 신선도가 여전히 유지되도 된다거나 그 사이에 변경이 일어났으니 원 서버로부터 새로운 리소스를 받고 저장한 다음에 캐시는 클라이언트에게 요청 리소스를 내려주게 된다.
* 캐시된 리소스에 한해서 이전에 원 서버로 부터 받았던 응답 헤더들과 새롭게 추가되는 헤더들로 응답 메세지를 만들어 클라이언트에게 내려준다.



## 신선도 - 유효기간과 나이

서버는 `Expires`나 `Cache-Control: max-age` 헤더를 통해 해당 리소스의 캐시 유효기간을 명시한다.

* 캐시 서버는 이런 헤더 정보를 바탕으로 리소스의 재검사를 해야하는지 여부를 확인한다. 캐시된 리소스를 접근하게 되는 시간이 `Expires` 값으로 명시된 날짜보다 지나있다면 이는 신선도가 떨어졌다고 확인된다.
* 특정 리소스를 접근하는 시점의 age 값이 `Cache-Control: max-age` 값보다 크다면 이 또한 신선도가 떨어졌다는 뜻이다.



## 신선도 검사 (재검사)

캐시 서버는 리소스의 신선도가 떨어졌다고 판단되는 순간 원 서버에 재검사를 요청한다. 이는 일반 HTTP 메세지를 보내는 행위지만 특정 헤더 값을 추가하여 신선도 검사를 위한 HTTP 통신임을 명시한다.

* `If-Modified-Since:` <date>
  * 캐시 서버는 리소스를 원 서버로 내려받았을 때 헤더 값으로 받았던 `Expires`나 `Cache-Control: max-age`값으로 이 헤더의 값을 명시한다. -> 추측
  * 원서버는 이 헤더값으로 명시된 날짜 이후에 리소스가 변경되었는지 확인한다.
  * 만약 변경이 이뤄지지 않았다면 body값 없이 `Expires`나 `Cache-Control: max-age`와 같이 필요한 헤더 값을 내려준다. (`304 Not Modified`)
    * 캐시 서버는 이렇게 받은 값을 가지고 다시 캐시된 리소스의 유효기간을 갱신시킨다.
  * 반대로 재검사 시 리소스의 변경이 이뤄진 경우라면 `200 OK` 응답으로 body 값을 포함하여 `Last-Modified` 헤더 값을 함께 내려준다.
    * 캐시 서버는 `Last-Modified` 헤더 값을 가지고 리소스의 정보를 갱신한다. -> 이후 원서버에 재검사 요청 시 `If-Modified-Since` 헤더의 값으로 `Last-Modified`로 받았던 값을 할당한다. (추측)
* `If-None-Match:` <tags>
  * 원 서버는 캐시 서버에게 리소스를 내려줄 때 다양한 헤더 값을 내려준다. 그 중 하나가 `Etag` 다.
  * `Etag`는 리소스의 변경을 감지할 수 있는 유효성 검사 토큰이다.
  * 캐시 서버는 특정 리소스에 대해 신선도가 만료됐는지 여부를 확인한 후 재검사를 필요로할 때 가지고 있는 `Etag`를 같이 조건부 요청에 `If-None-Match` 해더로 담아 보낸다.
    * 원 서버는 조건부 요청에 대상인 리소스의 `Etag` 값과 요청 헤더로 넘어온 `Etag` 값을 비교하여 같으면 변경이 안됐다는 뜻임으로 body가 없는 `304 Not Modified` 응답을 내려주고 그 값이 다른 경우 변경된 리소스의 전체 데이터를 담은 `200 OK` 응답을 내려준다.
  * 날짜로 리소스의 변경을 감지하는 것 보다 훨씬 효율적이다.
    * 주석 제거와 같은 중요하지 않은 데이터의 변경을 유도리 있게 감지 안할 수 있다.
      * 서버가 발행하는 Etag는 리소스의 중요 데이터를 기반으로 만들어내는 토큰이다. 즉, 무시할만한 데이터 변경에 대해서는 패스할 수 있다.
    * 서버마다 다를 수 있는 시간에 오차 문제로부터 벗어날 수 있다.



## Cache-Control



### no-cache vs no-store

* `no-store`
  * 캐시 서버에 파일을 저장하지 않겠다는 응답 헤더다.
  * 원 서버 이외에 서버가 알 필요가 없는 중요 데이터나 비밀 데이터에 대해 설정할 수 있는 캐시 제어 값이다.
  * 캐시 제어 값을 명시하지 않고 그냥 응답을 내려주면 이 응답을 받는 클라이언트는 자체적으로 설정해놓은 캐시 설정에 의해 리소스를 캐싱할지 말지 결정한다.
    * 브라우저는 자체적으로 대부분 리소스에 대해 캐싱을 한다.
    * 그렇기 때문에 중요한 정보가 담긴 리소스에 대해서는 왠만하면 `no-store` 헤더를 명시하고 응답을 내려줘야한다.
* `no-cache`
  * `max-age=0` 으로 설정해놓은 상태와 같은 캐시 제어 해더 값이다.
  * 즉, 일단 기본적으로 리소스에 대한 캐싱을 허락한다.
  * 하지만 캐싱된 리소스에 대한 접근이 필요할 때 캐시 서버 내부에서 신선한지를 확인하는 것이 아닌 바로 원 서버에 재검사 요청을 보낸다. (`max-age=0`)
  * 즉 느린 캐시 적중이 매번 일어난다. 캐싱된 리소스의 최상의 신선도를 보장받을 수 있다.
  * 캐시를 아예 안한 것과 다를게 뭐가 있냐라는 의문이 들 수 있다.
    * `no-cache` 제어는 일단 캐싱을 하기 때문에 원서버에 조건부 검사를 보낸다.
    * 다시 말해 신선도 검사 통과 시(true) 원 서버 응답에 리소스 데이터가 들어가 있지 않은 단순한 `304 Not Modified` 응답이 내려온다.
    * 캐싱된 리소스의 변경이 이뤄지지 않은 한에서 적은 네트워크 비용으로 리소스를 클라이언트에게 내려줄 수 있다.

![](https://user-images.githubusercontent.com/30451129/76437444-e8fca000-63fc-11ea-8be5-f05e0446179c.png)


​    

## 캐시 업데이트

정적 리소스는 1년의 `max-age`로 설정되어 캐시에 보관될 수 있다. HTML에서 css 같은 경우는 자주 바뀌지 않기 때문에 1년이라는 시간동안 캐싱을 해도 큰 문제가 되지 않기 때문이다.

하지만 1년이라는 캐싱 기간이 설정된 css 파일에 대해 기간 만료 전 변경이 일어났을 경우에는 어떻게 될까? 해당 HTML 파일을 브라우저로 보게 됐을 때 최근에 리소스를 받은 사용자는 변경된 최신 css 가 입혀진 페이지를 볼 것이고 약 6개월 전에 리소스를 받아서 사용하고 있는 사용자는 변경된 css의 모습을 구경하지 못할 것이다. (기본적으로 브라우저는 GET 요청에 대해 로컬 캐시를 먼저 확인한다.)

왜냐하면 6개월 전에 리소스를 받은 사용자의 브라우저는 아직 6개월 정도 age가 남은 css를 원 서버로부터 조건부 요청을 보내지 않을 것이기 때문이다. 사실 css와 같은 정적 파일은 변경에 둔감하다. 그래서 치명적인 문제나 오류를 일으킬 경우는 매우 드믈기 때문에 큰 문제가 일어나진  않을것이다. 하지만 캐시로 인해 사용자에게 올바른 리소스를 보여주지 못한다면 주객이 전도된 상황이 아닐까?

*캐시된 리소스의 최신 버전을 유지하기 위해서는 어떻게 해야할까?*

가장 많이 사용되는 방법은 캐싱하는 리소스 이름에 `Etag`나 날짜를 추가하는 것이다.

![](https://user-images.githubusercontent.com/30451129/76437444-e8fca000-63fc-11ea-8be5-f05e0446179c.png)

위 상황을 살펴보면 일단 HTML 내부에서 css, js, jpg 정적 파일을 불러오는 상황이다. 각 리소스에 대해서 캐시 전략이 다르다. 

* HTML의 경우 no-cache 제어로 매번 요청 시 원 서버에 재검사를 실시한다.
* style.3da37df.css 의 경우 1년이라는 시간의 캐시 기간이 설정되어있다.
* script.8sd34ff.js의 경우 CDN이 아닌 개인 로컬 캐시에만 저장할 수 있고 1년이라는 캐시 기간이 설정되어있다.
* photo.jpg의 경우 1시간의 캐시 기간이 설정되어있다.

위 파일들의 눈여겨봐야할 점은 css, js 파일 이름에 Etag 값이 추가되어있다는 점이다. 이는 HTML 파일이 `no-cache` 제어에 의해 매 요청이 원 서버에 검사를 받게되는데 css, js 파일에 변경이 일어났다면 이전 요청의 `Etag`값과 달라지기 때문에 브라우저의 로컬 캐시 메모리에서 해당 리소스를 찾지 못하고 원서버에서 다시 받아오게 될 것이다.

즉, 아무리 6개월전에 리소스를 받았던 브라우저라도 HTML을 매번 검사를 하기 때문에 정적 파일에 변경이 일어났다면 Etag 값이 이름에 붙는 css 나 js 파일 이름이 변경될 것이다. 즉 브라우저의 로컬 캐시 메모리에는 style.3da37df.css 파일이 있었지만 css의 업데이트가 일어나면서 새롭게 받은 HTML 문서 상에는 style.18yyaf77.css(예시) 파일을 불러오는 요청이 들어있을 것이다. 브라우저는 로컬 캐시 메모리에는 style.3da37df.css는 있지만 style.18yyaf77.css는 없으므로 캐시 적중이 안되어 원서버에 해당 리소스를 달라는 요청을 보내게 된다. 그래서 오래전에 리소스를 받았던 클라이언트라도 캐시 만료 전 업데이트된 리소스를 최신의 상태로 갱신할 수 있게 된다.



참고 자료

* https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching