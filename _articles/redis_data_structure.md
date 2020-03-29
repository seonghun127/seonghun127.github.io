---
id: 3
title: "redis data structure"
subtitle: "레디스 자료구조에 대해 간단히에 알아보자""
date: "2020.03.29"
tags: "redis"
---


# Redis 자료구조와 시간복잡도

> redis를 처음 첩해보면서 기본적인 자료구조에 대해 정리해봤다.
> redis 레퍼런스를 번역한 내용이 거의 대부분이다.
> 정리한 자료구조는 String, List, Hash, Set, Sorted Set이다.

### String

- 제일 간단한 자료구조
    - redis는 String을 기본 자료형으로 사용하며 이는 memcached와 같은 구조를 가지는 자료구조다.
    - 많은 사용 케이스가 있다.
        - HTML 프래그먼트, 페이지를 캐싱할 수 있다.
    - binary data도 저장이 가능하다.
        - jpeg 이미지도 value로 저장될 수 있다. 용량은 512MB가 최대다.
- 명령어
    - set, get, incr(by), decr(by), mset, mget(returns an array of values)
    - incr, decr에 대한 동시성 이슈를 보장한다. (싱글 스레드)
        - 기본적으로 여러명의 클라이언트가 하나의 데이터를 읽는 경우는 일어날 수 없다.
        - Redis는 싱글 스레드이기 때문에 위와상황에서 값을 변경하는 incr(decr) 명령어에 대해 동시성을 보장받을 수 있다.
        - read-increment-set 명령어는 다른 모든 클라이언트가 동시간에 명령어를 실행시키지 않을 때 수행된다.
    - incr, decr 명령어는 숫자 형태의 값이 아닌 경우에는 에러가 난다.(물론 저장은 String 형태로 저장됨)
        - key에 저장된 String 값은 base-10 64bit의 부호를 갖는 정수로 해석되어 명령어가 실행된다.
- 시간 복잡도
    - mget, mset 등 1개 이상의 값을 설정하거나 불러오는 명령어를 제외하면 단일 값에 대한 명령어의 시간 복잡도는 대개 O(1)를 갖는다.
        - multi value를 다루는 명령어는 O(n)의 시간 복잡도를 갖는다.

### List

- Redis는 Linked List 로 구현됐다.
- index로 사용 시 매우 빠른 시간내로 데이터에 접근할 수 있는 Array와 달리 그만큼의 스피드를 링크드 리스트는 내질 못한다. (linked list 자료구조의 특징)
- 데이터베이스 특성 상 빠른 시간내로 새로운 데이터를 저장하는 것이 중요하기 때문에 위 단점에도 불구하고 링크드 리스트로 리스트 자료구조를 구현했다.
- 만약 대규모 데이터 구조에서 중간에 빠르게 접근해야하는 경우라면 리스트 보다는 sorted set 자료구조를 사용하는 것이 훨씬 효율적이다.
- 명령어
    - lpush, rpush, lrange(extracts ranges of elements from lists), rpop(retrieving and eliminating), ltrim(similar to lrange, but it sets specified range as the new list value. all the elements outside the given range are removed)
- 리스트 사용 케이스
    - 소셜 네트워크에 가장 최신에 업데이트된 게시글 저장
    - producer가 아이템을 리스트에 담고 consumer(worker)가 그 아이템들을 컨슘하는 구조의 consumer-pattern 패턴
- key 생성, 제거에 대한 자동화
    - 사용자가 직접 데이터를 적재 시킬 때 특정 자료구조를 새롭게 생성하거나 삭제할 필요가 없다.
        - 다시말해 자료구조의 생성과 삭제는 오로지 redis의 책임으로 전가되어 있다.
        - 사용자가 새로운 데이터를 추가하려할 때 해당 key가 존재하지 않는다면 redis는 데이터를 추가하기 전에 새로운 자료구조를 생성한다.
        - 또한 사용자가 데이터를 삭제할 경우 해당 자료구조에 데이터가 더이상 존재하지 않는다면 redis 해당 자료구조를 삭제한다.
    - 물론 이는 특정 자료구조에 국한된 것이 아닌 list, set, hash, stream 등에 모두 해당되는 특성이다.
- 시간 복잡도
    - redis list는 linked list로 구현됐기 때문에 백만개의 데이터가 리스트에 있다하더라도 첫부분이나 마지막부분에 데이터를 삽입하는 것은 상수 시간밖에 걸리지 않는다. → **insert data in the head or tail : O(1)**
    - 상수 길이의 데이터를 상수시간내로 가져올 수 있다는 장점을 가지고 있다.
        - lrange 명령어는 O(n)의 시간복잡도를 가지지만 작은 범위이면서 리스트의 head나 tail에 접근하는 것은 상수 시간의 시간복잡도를 가진다.
        - String의 경우 단일 값(key와 value가 1:1관계)이라도 range(getrange)를 명령어를 실행시킬 경우 O(n)의 시간 복잡도를 갖는다.

### Hash

- redis key에 field-value 가 짝을 이루어 redis value에 저장되는 구조다.
- hash는 객체를 표현하는데 매우 편리하지만 동시에 hash에 저장할 수 있는 데이터 수는 사용가능한 메모리 내에서 제한이 없기 때문에 매우 다양하게 활용될 수 있다.
- 명령어
    - hmset, hmget, hset, hget
- 시간복잡도
    - 단일 값에 대한 명령어는 O(1)의 시간복잡도를 갖는다.
        - multi value에 대한 명렁어는 데이터의 갯수 만큼(n)의 시간복잡도 (O(n))를 갖는다.

### Set

- 순서가 보장되지 않는 String 집합이다.
- 데이터의 unique를 보장한다. (중복 허용 x)
- 명령어
    - sadd, smember, sismember
- set은 객체 사이의 관계를 나타내는데 좋은 자료구조다.
    - 예를 들어 tags를 구현하는데 많이 사용된다.
```java
sadd news:1000:tags 1 2 5 77
(integer) 4
```
- 시간복잡도
    - 단일 데이터에 대한 명령어는 O(1)의 시간복잡도를 잗는다.
        - sadd, srandmember, spop, sismember

### Sorted Set

- sorted set은 set, hash를 섞어 놓은 자료구조다.
- 데이터에 대한 unique, 중복 허용x 특징을 가지는 set의 성질을 그대로 가지며 동시에 순서를 보장받을 수 있는 자료구조다. (Set은 순서를 보장받을 수 없다.)
    - sorted set의 데이터들은 모두 score라는 점수를 가지고 있다.
    - 이러한 점이 hash와 비슷하다고 볼 수 있다.
- 명령어
    - zadd(sadd와 비슷하지만 추가로 인자 하나를 더 받는다 - score), zrange,  zrevrange, zremrangebyscore
- 시간 복잡도
    - sorted set은 스킵 리스트, 해시 테이블를 포함하는 듀얼 포트 자료구조로 구현됐기 때문에 데이터 추가는 O(log(n))의 시간복잡도를 가진다.
        - skip list의 탐색, 추가, 삭제 명령은 모두 O(log(n))의 시간복잡도를 갖는다. [참고](https://brilliant.org/wiki/skip-lists/)
        - hash table의 경우 탐색, 추가, 삭제 명령 모두 상수 시간의 시간복잡도를 갖는다.

            (물론 O(n)의 시간복잡도를 갖는 예외 케이스가 있긴하다. [참고](https://stackoverflow.com/questions/9214353/hash-table-runtime-complexity-insert-search-and-delete))

            → skip list 시간복잡도에 맞춰 sorted set 자료구조의 시간복잡도가 O(log(n))으로 맞춰진 것으로 보인다. 

        - List에서 이야기 했던 것처럼 HEAD, TAIL에 추가하는 명령어는 빠른 시간을 갖는 장점의 List지만 데이터를 리스트 중간에 추가할 경우 추가해야하는 자리를 탐색하는 시간(O(n))이 걸리기 때문에 시간 복잡도가 커지기에 이와 같은 경우 O(log(n))의 시간 복잡도를 갖는 sorted set을 사용하는 것이 훨씬 효율적이다.


### reference
- https://redis.io/topics/data-types-intro#strings
