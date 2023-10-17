---
title: "N + 1 문제 해결 도중 맞닥뜨린 Set 과 List 의 차이"
description: "성능 개선을 위해 N + 1 문제를 해결하던 와중 Set, List 으로 인해 겪었던 문제를 탐구해본 글입니다!"
date: "2023-10-15"
update: "2023-10-15"
tags:
- Spring Data JPA
- Hibernate
---

> 이 글은 우테코 괜찮을지도의 `매튜`가 작성하였습니다.

### 서론

## 간단한 도메인 설명

저희 서비스에는 `Topic` 이라는 도메인이 존재하고, 이는 `지도`를 의미합니다.

그리고 해당 `지도`는 `Permission(권한)`, `Pin(핀)`, `Bookmark(즐겨찾기)` 들과 `1:N` `연관관계`를 이루고 있습니다.


## 문제 발생 상황 

### 문제 상황

이 글에서 주로 탐구하고 있는 문제는 서비스 홍보를 앞두고 성능 개선을 하기 위해 `N + 1` 문제를 해결하고 있던 와중 발생하였습니다.

일단 유의미한 성능 차이를 보기 위해, 우선적으로 `Topic` 과 `Pin` 데이터를 각각 `10만개`씩 넣고 진행하였습니다.

또한 요청은 `PostMan` 을 이용하여 테스트 하였습니다.

아래는 그 때의 `코드`입니다.

- Topic
```java
@Entity  
@NoArgsConstructor(access = PROTECTED)  
@Getter  
public class Topic extends BaseTimeEntity {  

	... 생략
  
    @ManyToOne(fetch = FetchType.LAZY)  
    @JoinColumn(name = "member_id")  
    private Member creator;  
  
    @OneToMany(mappedBy = "topic")  
    private List<Permission> permissions = new ArrayList<>();  
  
    @OneToMany(mappedBy = "topic", cascade = CascadeType.PERSIST)  
    private List<Pin> pins = new ArrayList<>();  
  
    @OneToMany(mappedBy = "topic", cascade = CascadeType.PERSIST, orphanRemoval = true)  
    private List<Bookmark> bookmarks = new ArrayList<>();  

	... 생략

}
```

- TopicRepository
```java
@Repository  
public interface TopicRepository extends JpaRepository<Topic, Long> {  

    @EntityGraph(attributePaths = {"creator", "permissions", "bookmarks", "pins"})  
    List<Topic> findAll();  

}
```

당연히 위 코드는 `MultipleBagFetchExcepion` 예외가 발생하였습니다. (`MultipleBagFetchExcepion` 자세한 설명은 https://map-befine-official.github.io/jpa-multibag-fetch-exception/ 해당 글을 확인해주세요!)

### MultipleBagFetchExcepion 해결

우리는 해당 예외를 해결하기 위해 `Topic` 의 `Collection` 들의 `자료구조`를 `Set` 으로 바꿔주었습니다.

```java
@Entity  
@NoArgsConstructor(access = PROTECTED)  
@Getter  
public class Topic extends BaseTimeEntity {  

	... 생략
  
    @ManyToOne(fetch = FetchType.LAZY)  
    @JoinColumn(name = "member_id")  
    private Member creator;  
  
    @OneToMany(mappedBy = "topic")  
    private Set<Permission> permissions = new HasSet<>();  
  
    @OneToMany(mappedBy = "topic", cascade = CascadeType.PERSIST)  
    private Set<Pin> pins = new HashSet<>();  
  
    @OneToMany(mappedBy = "topic", cascade = CascadeType.PERSIST, orphanRemoval = true)  
    private Set<Bookmark> bookmarks = new HashSet<>();  

	... 생략

}
```

이렇게 해서 단 한번의 `Query` 로 존재하는 모든 `Topic` 을 불러 올 수 있었습니다.

하지만, `카테시안 곱` 으로 인해 요청에 대한 응답시간이 어마어마 했습니다. (대략 `20초` 정도?)

### Topic findAll성능 개선 완료

`Topic`을 전체 조회할 때 사실 `Bookmark(즐겨찾기)`, `Pin(핀)` 의 세부 정보가 아닌, 이들의 개수만이 필요하기 때문에, 반정규화를 통해 이 문제를 해결하였습니다.

결론적으로 아래와 같이, `Collection` 중에는 `Permission` 만을 `join` 해오면 되는 거죠!

```java
@Repository  
public interface TopicRepository extends JpaRepository<Topic, Long> {  

    @EntityGraph(attributePaths = {"creator", "permissions"})  
    List<Topic> findAll();  

}
```

근데 이렇게 반정규화를 진행하던 도중, 어쩌다가 `Topic` 을 아래와 같이 바꾸는 일이 있었습니다.

```java
@Entity  
@NoArgsConstructor(access = PROTECTED)  
@Getter  
public class Topic extends BaseTimeEntity {  

	... 생략
  
    @ManyToOne(fetch = FetchType.LAZY)  
    @JoinColumn(name = "member_id")  
    private Member creator;  
  
    @OneToMany(mappedBy = "topic")  
    private Set<Permission> permissions = new HasSet<>();  
  
    @OneToMany(mappedBy = "topic", cascade = CascadeType.PERSIST)  
    private List<Pin> pins = new ArrayList<>(); // Set --> List 로 바꿈
  
    @OneToMany(mappedBy = "topic", cascade = CascadeType.PERSIST, orphanRemoval = true)  
    private Set<Bookmark> bookmarks = new HashSet<>();  

	... 생략

}
```

이 때 `Collection` 의 자료구조를 `Set` 만을 썼을때보다 속도가 굉장히 빨라졌었습니다.

이 때 당시에 모두 이에 대해 왜 이런 것이지? 하는 의문을 가졌었지만, 시간이 없어 어쩔 수 없이 넘어갔었습니다.

그리고 시간이 지나, 어느정도 여유가 생긴 지금, 해당 문제에 대해 탐구해보고자 글을 작성하게 된 것입니다.

## 재연

### 상황을 그때와 동일하게 구성해보자.

`프로시저를` 통해 `Local DB` 에다가 `Topic`, `Bookmark` 데이터를 `10만개` 가량을 넣어주고 테스트를 진행했습니다. (내 컴퓨터 살려..)

`Pin` 데이터를 넣지 않은 이유는, 현재 `Pin` 은 `반정규화`가 진행되어 있어 `Topic` 전체 목록을 조회할 때, 성능에 전혀 영향을 끼치지 않기 때문입니다.

그렇다고 `Pin` `반정규화`를 풀자니, 요청과 응답 시간이 비 정상적으로 너무 길어졌습니다. (대략 1분 30초 정도)

그렇기 때문에 일단 `Pin` 은 일단 `반정규화`를 유지하였고, `Bookmark` 만 `반정규화`를 해제하고 진행하였습니다.

그렇기 때문에 테스트를 위해서 `자료구조`를 변경하게 될 `Collection` 은 사실상 `Permission` 과 `Bookmark` 뿐인 것 입니다.

### 테스트 진행

말씀드린 두 `Collection`의  `자료구조`를 바꿔가며 테스트 해본 결과는 아래와 같습니다. (`Permission`, `Bookmark` 모두 `Set` 이 아닌 경우는 `MultipleBagFetchException`  이 발생하기 때문에 테스트하지 않았습니다.)

- Permission, Bookmark 모두 Set
```txt
[http-nio-8080-exec-2] 2042 INFO  com.mapbefine.mapbefine.common.filter.LatencyLoggingFilter - Latency : 5.442s, Query count : 1, Request URI : /topics
```

- Permission  만 Set 
```txt
[http-nio-8080-exec-3] 2181 INFO  com.mapbefine.mapbefine.common.filter.LatencyLoggingFilter - Latency : 7.074s, Query count : 1, Request URI : /topics
```

- Bookmark 만 Set
```txt
[http-nio-8080-exec-1] 2072 INFO  com.mapbefine.mapbefine.common.filter.LatencyLoggingFilter - Latency : 5.348s, Query count : 1, Request URI : /topics
```

위 테스트 결과들만으로는 `유의미한 차이`가 보이지 않아 `원인`을 추론해보기 어려웠습니다.

### 지금까지 무의미한 데이터로 테스트 해본 것은 아닐까?

위와 같이 테스트하다가, 문득 `Permission` 에 `데이터`도 넣어봐야 유의미하지 않을까? 란 생각이 머리를 스쳐 지나갔고, `Permission` 데이터도 추가해주었습니다.

하지만, `Permission` 을 추가해주니, `카제인 곱`이 엄청나게 발생되어, `Permission`, `Bookmark` 각각 데이터 개수가 `3000개`만 넘어가도 `Java Heap` 이 터지는 예외가 발생하게 되어, 적당히 `2000` 개 가량의 데이터를 각각 넣어주고, 이어서 테스트를 진행해본 결과 아래와 같은 결과가 나오게 되었습니다.

- Permission 과 Bookmark 모두  Set
```txt
[http-nio-8080-exec-3] 1863 INFO  com.mapbefine.mapbefine.common.filter.LatencyLoggingFilter - Latency : 4.924s, Query count : 1, Request URI : /topics
```

- Permission 만 Set 일 때
```txt
[http-nio-8080-exec-2] 1952 INFO  com.mapbefine.mapbefine.common.filter.LatencyLoggingFilter - Latency : 5.159s, Query count : 1, Request URI : /topics
```

- Bookmark 만 Set 일 때
```txt
[http-nio-8080-exec-3] 2000 INFO  com.mapbefine.mapbefine.common.filter.LatencyLoggingFilter - Latency : 7.253s, Query count : 1, Request URI : /topics
```

테스트 데이터를 변경하더라도, 역시나 `유의미한 차이`를 볼 수 없었습니다.

### 내린 결론 (가설)

`Set` 자료구조를 사용함에 따라, `List` 보다는 부하가 더 발생할 수 있다고 생각합니다.

자료구조 특성상 `Set` 은 중복을 제거해주는 연산을 실행해주어야 하니까요.

하지만, 어짜피 `List` 를 사용하더라도 `hibernate` 에서 `distinct` 를 통해 `중복`을 제거해주기 때문에 더더욱이 유의미한 성능상의 차이를 가져오지 못하는 것 같습니다.

정말 많이 테스트해보면서, 가끔 컴퓨터의 상태에 따라 응답시간이 비정상적으로 길어지는 경우가 있었습니다.

저희는 그것을 보았던 것 아닐까요??

## 이대로 끝내기는 아쉬우니까!

### JPA 에서 Set 을 사용할 때 주의할 점

`문제`를 탐구하다가 `재미있는 글`을 발견했습니다.

[재미있는 글](https://www.inflearn.com/questions/321256/collection-type%EC%9C%BC%EB%A1%9C-set-%EB%8C%80%EC%8B%A0-list%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%98%EB%8A%94-%EC%9D%B4%EC%9C%A0%EA%B0%80-%EC%9E%88%EB%8A%94%EC%A7%80%EC%9A%94)

질문은 아래와 같았습니다.

```txt
Collection type으로 Set 대신 List를 사용하시는 이유가 궁금합니다.

지금까지 나온 Collection들이 모두 unique한 Entity(또는 값 타입)들의 collection이기 때문에, Set을 활용할 경우 중복 확인 관련 부분이 깔끔해지고, 다른 질문의 답변에서 답해주신대로 값 타입 컬렉션에도 row를 모두 날리고 다시 넣는 문제를 막을 수 있어 Set에 대해 좋은 인상을 가지게 되었습니다.

그런데 기본적으로 예제가 List를 사용하여, Set을 사용하였을 때 제가 놓친 문제가 있는지 의문이 들었습니다.
```

그에 대한 영한님의 답변은 이랬습니다.

```txt
안녕하세요. Catnip님

좋은 질문입니다. Set이 개념적으로 좋지만 실무에서는 성능 이슈가 있습니다.

Set은 중복을 제거해야 하는데, 그렇다는 것은 기존 데이터 중에 중복이 있는지 비교를 해야 합니다. 이게 일반적으로는 크게 문제가 없는데, 지연 로딩으로 컬렉션을 조회했을 때 문제가 됩니다.

컬력션이 아직 초기화 되지 않은 상태에서 컬렉션에 값을 넣게 되면 프록시가 강제로 초기화 되는 문제가 발생합니다. 왜냐하면 중복 데이터가 있는지 비교해야 하는데, 그럴러면 컬렉션에 모든 데이터를 로딩해야 하기 때문입니다.

반면에 List는 이런 중복 체크가 필요없이 때문에 데이터를 추가할 때 초기화가 발생하지 않습니다.

감사합니다.
```

아주 흥미로웠습니다.

이를 제대로 확인해보기 위해서 테스트를 진행하였습니다.

### 테스트

```java
@Test  
@Transactional  
void Topic의_Collection_의_자료구조에_따른_초기화를_확인해보자() {  
    //given  
    Member savedMember = memberRepository.save(MemberFixture.create("member", "member@naver.com", Role.USER));  
    Topic savedTopic = topicRepository.save(TopicFixture.createPublicAndAllMembersTopic(savedMember));  
    Location savedLocation = locationRepository.save(LocationFixture.create());  
  
    // when  
    entityManager.clear();  
    Topic findTopic = topicRepository.findById(savedTopic.getId()).get();  
    Pin savedPin = pinRepository.save(PinFixture.create(savedLocation, savedTopic, savedMember));  
  
    // then  
    System.out.println("=============Add Pin 이전");  
    findTopic.addPin(savedPin);  
    System.out.println("=============Add Pin 이후");  
    entityManager.flush();  
}
```

이와 같이 테스트 코드를 짜고 `Pins` 를 `List` 혹은 `Set` 으로 진행해보았다.

- Pins 가 List 일 때 쿼리
```txt
... 이전 쿼리들
=============Add Pin 이전
=============Add Pin 이후
Hibernate: 
    update
        topic 
    set
	    ... 수 많은 컬럼들
    where
        id=?
```

- Pins 가 Set 일 때 쿼리
```txt
... 이전 쿼리들
=============Add Pin 이전
Hibernate: 
    select
		... 수 많은 컬럼들
    from
        pin p1_0 
    left join
        member c1_0 
            on c1_0.id=p1_0.member_id 
    left join
        location l1_0 
            on l1_0.id=p1_0.location_id 
    where
        p1_0.topic_id=? 
        and (
            p1_0.is_deleted = false
        ) 
=============Add Pin 이후
Hibernate: 
    update
        topic 
    set
		... 수 많은 컬럼들
    where
        id=?
```

영한님의 말씀대로 `List` 는 `Collection` 에 값을 `추가` 를 진행할 때, 기존의 데이터가 필요 없으니, 초기화를 진행하지 않지만, `Set` 을 쓰는 경우 중복을 방지하기 위해 기존의 데이터가 필요하기 때문에 `select` 를 통해 값을 가져와 초기화를 진행해주는 것을 볼 수 있었습니다.

즉, 이렇게 `fetch` 전략으로 `Lazy Loading` 을 사용하는 경우 `자료구조`로 `Set` 을 사용하는 경우, `연관관계 매핑`을 하게 되었을 때, 해당 `부작용`이 발생할 수 있는 것입니다.

조심해서 써야겠습니다.

## 최종적인 결론

이 글의 최종적인 결론은 아래와 같습니다.

- 사실 `Set` 과 `List` 로 인한 `성능 차이`는 `유의미하지 않은` 것 같다. 우리가 착각했던 것일지도..?
- 하지만, `Set` 을 무지성으로 써도 되는 것은 아니다, 이로부터 얻는 부작용이 상당히 많으니, 고심해서 사용하자 (순서 보장 x, 위에서 설명한 초기화 문제)

긴 글 봐주셔서 정말 감사합니다~!
