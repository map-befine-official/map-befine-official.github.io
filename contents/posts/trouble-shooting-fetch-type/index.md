---
title: "FetchType.EAGER, FetchType.LAZY 에 대해서 알아보자!"
description: "쿼리 개선 과정 중에 겪었던 문제에 대해서 정리하였습니다."
date: "2023-10-04"
update: "2023-10-04"
tags:
- Spring Data JPA
- JPQL
---

> 이 글은 우테코 괜찮을지도의 `매튜`가 작성하였습니다.

### 배경

#### 쿼리 개선... 필요할지도?

9월 말, 추석을 앞두고 사용자 유치 계획 실행을 앞두고 있었다.

우리는 행복회로를 돌리며, 많은 사용자들이 서비스에 들어올 것이라고 생각했고, 사용자들에게 좋은 경험을 선사하고 싶었다.

하지만, 문제가 있었다.

많은 사용자가 들어옴에 따라 평소보다 많은 트래픽이 발생할 것이고, 조금 더 좋은 경험을 제공하기 위해 추가한 지도와 핀으로 인해 데이터는 방대해졌다. 

자칫하면 사용자에게 좋지 않은 경험을 선사할 수 있다는 생각에 성능 개선을 목표로 삼았다.

때문에 우리는 현재 발생하고 있는 `N + 1` 문제를 해결하고, `인덱스`를 활용하여 빠른 성능 개선을 계획했다.

그 중에서 `N + 1` 문제를 해결하는 과정 중에 해당 글을 작성하게 된 문제가 발생하였다.

#### 문제 상황

유저가 `지도` 목록을 조회하려는 경우 비정상적으로 쿼리가 많이나갔다.

쉽게 예상할 수 있듯 당연히 `N + 1` 문제였다.

우리는 해당 `N + 1` 문제를 해결하기 위해서 `fetch join` 을 사용했다.

#### N + 1 을 해결해보자!

- Topic Entity 구조 (`지도 == Topic`)
```java
@Entity  
@NoArgsConstructor(access = PROTECTED)  
@Getter  
public class Topic extends BaseTimeEntity {  
  
    @Id  
    @GeneratedValue(strategy = GenerationType.IDENTITY)  
    private Long id;  

	... 생략
  
    @ManyToOne  
    @JoinColumn(name = "member_id")  
    private Member creator;  
  
    @OneToMany(mappedBy = "topic")  
    private List<Permission> permissions = new ArrayList<>();  
  
    @OneToMany(mappedBy = "topic", cascade = CascadeType.PERSIST)  
    private List<Pin> pins = new ArrayList<>();  
  
    @OneToMany(mappedBy = "topic", cascade = CascadeType.PERSIST, orphanRemoval = true)  
    private List<Bookmark> bookmarks = new ArrayList<>();  
  
    @Column(nullable = false)  
    @ColumnDefault(value = "0")  
    private int pinCount = 0;  
  
    @Column(nullable = false)  
    @ColumnDefault(value = "0")  
    private int bookmarkCount = 0;  

	... 생략

}
```

- Topic Repository 
```java
@Repository  
public interface TopicRepository extends JpaRepository<Topic, Long> {  
  
    @EntityGraph(attributePaths = {"permissions"})  
    List<Topic> findAll();

	... 생략
}
```

앞선 `Entity`, `Repository` 는 `지도` 목록을 조회할 때 사용된 `Entity` 및 `Repository` 이다.

마지막으로 `Topic` 에 필요한 정보는 아래와 같다.

- Topic Response
```json
{  
  "id" : 1,  
  "name" : "토픽 이름",  
  "image" : "토픽 이미지 링크",  
  "creator" : "토픽을 만든자",  
  "pinCount" : 3,  
  "bookmarkCount" : 5,  
  "updatedAt" : "2023-10-02T18:00:55.95188832"  
}
```

여기서 사용자에게 `Topic` 목록을 제공하기 위해 부수적으로 필요한 `Entity` 는 `Member` , `Permission` 이었다.

위에서 볼 수 있듯, `Member` 는 `@ManyToOne 기본 fetch 설정`으로 인해 `FetchType.EAGER` 로 설정되고, `Permission` 은 `@OneToMany 기본 fetch 설정` 인 `FetchType.LAZY` 이지만, `@EntityGraph` 를 통해서 `fetch join` 을 해주었기 때문에 당연히 쿼리는 한번만 나갈 줄 알았다.

하지만, 결과는....

```text
// 데이터 10만개를 기준으로 진행되었음
[http-nio-8080-exec-2] 16496 INFO  com.mapbefine.mapbefine.common.filter.LatencyLoggingFilter - Latency : 4.649s, Query count : 8, Request URI : /topics
```

위와 같았고, 우리는 멘붕이 올 수 밖에 없었다.

우리 예상대로라면 분명히 `Query Count` 가 `1`이어야 하는데??

도대체 왜 ?? ㅠㅠㅠ

#### 해결했는데 이유를 모르겠어

다른 문제들도 많은데, 해당 문제까지 발생하여 몇 시간 동안 골머리를 앓았다.

그러다 조금의 시간이 흘렀고, 마음을 가다듬고 천천히 쿼리를 뜯어보기 시작했고, 원인을 찾을 수 있었다.

```sql
Hibernate: 
    select
		... 수많은 컬럼들 ...
    from
        topic t1_0 
    left join
        permission p1_0 
            on t1_0.id=p1_0.topic_id
Hibernate: 
    select
		... 수많은 컬럼들 ...
    from
        member m1_0 
    where
        m1_0.id=?

	... 7 번을 member 를 찾아옴
```

우리는 `@ManyToOne` 의 `기본 fetch 설정`인 `FetchType.EAGER` 로 인해 `Member` 를 `join` 을 통해 가져올 것이라고 생각했지만, 실상은 `Topic` 을 가져올 때 `permission` 은 `join`을 잘해서 가져오지만, `member` 는 `select 쿼리`를 통해 따로 가져오고 있었다.

이로 인해서 `Query Count` 가 `1`이 아니었던 것이었다.

```java
@EntityGraph(attributePaths = {"creator", "permissions"})  
List<Topic> findAll();
```

해당 문제를 해결하기 위해, 위와 같이 `attributePaths` 에 `creator` 를 추가해주었더니

```text
[http-nio-8080-exec-5] 16826 INFO  com.mapbefine.mapbefine.common.filter.LatencyLoggingFilter - Latency : 5.941s, Query count : 1, Request URI : /topics
```

드디어 `Query Count` 를 `1`로 만들 수 있었다.

도대체 왜 그런걸까???

차근차근 알아가보자!

### ManyToOne 테스트

#### 테스트를 통해서 생각을 굳혀보자!

일단 이전 상황들로 미루어 보았을 때, `Eager` 와 `Lazy` 는 단순 `select 시기`를 `결정`하는 것 같다.

그렇기 때문에, `Eager` 로 설정하든 `Lazy` 로 설정하든, 따로 `select` 쿼리를 통해서 가져오는 것이다.

이를 증명하기 위해 몇 가지 테스트를 진행해보자.

`@ManyToOne` 을 진행하기 위해 `대상 Entity` 를 정하고 `Test` 하기 위한 `테스트 코드`를 짜보자!

- Permission (@ManyToOne 컬럼밖에 없어서 딱 테스트하기 좋은 Entity Class 라고 판단)

```java
@Entity  
@NoArgsConstructor(access = PROTECTED)  
@Getter  
public class Permission extends BaseTimeEntity {  
  
    @Id  
    @GeneratedValue(strategy = GenerationType.IDENTITY)  
    private Long id;  
  
    @ManyToOne // 여기서 Eager or Lazy 로 진행할 것 
    @JoinColumn(name = "topic_id", nullable = false)  
    private Topic topic;  
  
    @ManyToOne  
    @JoinColumn(name = "member_id", nullable = false)  
    private Member member;

}
```

- ManyToOne 을 테스트 하기 위한 테스트 코드 (네이밍은 봐주세영!)

```java
@ServiceTest  
class TestClass {
  
    @Autowired  
    private TopicRepository topicRepository;  
  
    @Autowired  
    private MemberRepository memberRepository;  
  
    @Autowired  
    private PermissionRepository permissionRepository;  
  
    @Autowired  
    private TestEntityManager testEntityManager;  
  
    private Member member;  
    private Topic topic;  
    private Permission permission;  
  
    @BeforeEach  
    void beforeEach() {  
        // 멤버를 저장한다.  
        member = memberRepository.save(MemberFixture.create("member", "member@naver.com", Role.USER));  
        // 토픽을 저장한다.  
        topic = topicRepository.save(TopicFixture.createPublicAndAllMembersTopic(member));  
        // 권한을 저장한다.  
        permission = permissionRepository.save(Permission.createPermissionAssociatedWithTopicAndMember(topic, member));  
  
        // 영속성 컨텍스트 초기화 (초기화 안하면 findById 때 쿼리가 안 날라감)
        testEntityManager.clear();  
    }  
  
    @Test  
    void permissionManyToOneFindById() {  
        Permission permissionByFindById = permissionRepository.findById(permission.getId()).get();  
          
        assertThat(permissionByFindById.getTopic()).isEqualTo(topic);  
        assertThat(permissionByFindById.getMember()).isEqualTo(member);  
    }
}
```

위와 같이 테스트 코드를 짜고, `@ManyToOne fetchType` 은 `Eager` (기본 설정) 로 설정하고 테스트를 진행했다.

당연히 추가적인 `select` 쿼리가 날아가겠지? ㅎㅎ

#### 예상과 다른 결과

```sql
Hibernate: 
    select
		... 수많은 컬럼들
    from
        permission p1_0 
    join
        member m1_0 
            on m1_0.id=p1_0.member_id 
    join
        topic t1_0 
            on t1_0.id=p1_0.topic_id 
    left join
        member c1_0 
            on c1_0.id=t1_0.member_id 
    where
        p1_0.id=?
```

아니 도대체 왜 ???

예상대로라면 `select` 쿼리가 나가야 하는데...

이해가 되지 않는다.

왜 `select` 했다가 지 마음대로 `join` 해서 오는지 갈대같은 `JPA` 의 마음을 알 수가 없다.

이 결과를 보고 이전과 다른게 뭐지... 하면서 곰곰히 생각해보았다.

이 때 내가 발견한 차이점은 단 하나였다.

# 이전에는 목록 조회(findAll()), 이번에는 단건 조회(findById())이다.

#### 호다닥 findAll 로 테스트

```java
@Test  
void permissionManyToOneFindAll() {  
    List<Permission> permission = permissionRepository.findAll();  
}
```

위와 같이 테스트를 실행해보았다.

결과는??

```sql
Hibernate: 
    insert 
    into
        permission
        (created_at,member_id,topic_id,updated_at,id) 
    values
        (?,?,?,?,default)
Hibernate: 
    select
	    ... 수많은 컬럼들 ...
    from
        permission p1_0
Hibernate: 
    select
	    ... 수많은 컬럼들 ...
    from
        member m1_0 
    where
        m1_0.id=?
Hibernate: 
    select
	    ... 수많은 컬럼들 ...
    from
        topic t1_0 
    left join
        member c1_0 
            on c1_0.id=t1_0.member_id 
    where
        t1_0.id=?
```

역시나 예상대로 `findAll` 인 경우는 똑같이 `Eager` 이더라도 `join` 을 해서 가져오지 않는다.

`Eager` 로 `findById`, `findAll` 을 테스트 해봤으니

`Lazy` 로 더 테스트를 진행해서 가설을 사실로 굳혀보자!

#### Lazy 테스트

먼저 `findById`로 테스트를 진행해보자.

```java
@Test  
void permissionManyToOneFindById() {  
    Permission permissionByFindById = permissionRepository.findById(permission.getId()).get();  
  
    assertThat(permissionByFindById.getTopic()).isEqualTo(topic);  
    assertThat(permissionByFindById.getMember()).isEqualTo(member);  
}
```

```sql
Hibernate: 
    select
		... 수많은 컬럼들 ...
    from
        permission p1_0 
    where
        p1_0.id=?
Hibernate: 
    select
		... 수많은 컬럼들 ...
    from
        topic t1_0 
    left join
        member c1_0 
            on c1_0.id=t1_0.member_id 
    where
        t1_0.id=?
```

결과는 당연하다.

`Lazy` 하게 가져오니 `findById` 로 가져오더라도 `join` 을 하지 않은 것이고, `getTopic()` 을 할 때 `Topic` 을 가져왔기 때문에 `select` 쿼리가 나갔다.

근데 `member` 를 조회하는 `select` 쿼리가 없다. 

왜 그럴까?

이유는, 이번에 처음 알았는데 `assertJ` 로 테스트를 진행할 때 단 하나라도 예상한 결과가 나오지 않아 테스트가 실패하게 되면 그 즉시 테스트가 종료되는 것 같다.

그래서 뒤에 있는 `getMember()` 구문은 실행되지 않은 것이다.

어쨌든, 본론으로 돌아와 `findAll` 도 테스트를 진행해보자.

```java
@Test  
void permissionManyToOneFindAll() {  
    List<Permission> permission = permissionRepository.findAll();  
  
    assertThat(permission.get(0).getTopic()).isEqualTo(topic);  
    assertThat(permission.get(0).getMember()).isEqualTo(member);  
}
```

위와 같이 테스트를 진행했고, 나간 쿼리는 당연히 위와 동일할 것이다.

```sql
Hibernate: 
    select
		... 수많은 컬럼들 ...
    from
        permission p1_0
Hibernate: 
    select
		... 수많은 컬럼들 ...
    from
        topic t1_0 
    left join
        member c1_0 
            on c1_0.id=t1_0.member_id 
    where
        t1_0.id=?
```

여윽시 동일하다.

#### ManyToOne 테스트를 통해 확실하게 유추할 수 있는 것

`findAll` 과 같은 목록 조회는 `Eager`, `Lazy` 가 정말 `select 시기`만을 결정하지만, `findById` 과 같은 단건 조회는 `Eager` 로 설정하게 되면, `fetchType` 이 `select 시기 결정`을 넘어 `join` 여부까지 결정할 수 있는 것이다. (최적화)

결과야 뻔하긴 하지만, 남은 `OneToMany` 테스트들도 진행해서 위 사실을 더욱 더 굳혀보자.

### OneToMany

#### 테스트 준비

`ManyToOne` 테스트와 마찬가지로 `OneToMany` 테스트를 진행할 `대상 Entity` 와 `Test 코드`를 짜보자!

- Member Entity (OneTOMany 컬럼만 있어서 딱 적합)
```java
@Entity  
@NoArgsConstructor(access = PROTECTED)  
@Getter  
public class Member extends BaseTimeEntity {  
  
    @Id  
    @GeneratedValue(strategy = GenerationType.IDENTITY)  
    private Long id;  

	... 생략 ...
  
    @OneToMany(mappedBy = "creator")  
    private List<Topic> createdTopics = new ArrayList<>();  

	... 생략 ...

}
```

- Test 코드
```java
@Test  
void memberOneToManyFindById() {  
    Member memberByFindById = memberRepository.findById(member.getId()).get();  
    assertThat(memberByFindById.getCreatedTopics().get(0)).isEqualTo(topic);  
}  
  
@Test  
void memberOneToManyFindAll() {  
    List<Member> members = memberRepository.findAll();  
    assertThat(members.get(0).getCreatedTopics().get(0)).isEqualTo(topic);  
}
```

#### Eager 에 대해서 먼저 테스트 해보자!

- findById 로 테스트
```sql
Hibernate: 
    select
		... 수많은 컬럼들 ...
    from
        member m1_0 
    left join
        topic c1_0 
            on m1_0.id=c1_0.member_id 
    where
        m1_0.id=?
```

이쯤되면 당연히 예상할 수 있듯이 `join` 을 해서 가져온다.

- findAll 로 테스트
```sql
Hibernate: 
    select
		... 수많은 컬럼들 ...
    from
        member m1_0
Hibernate: 
    select
		... 수많은 컬럼들 ...
    from
        topic c1_0 
    where
        c1_0.member_id=?
```

이것도 당연히 예상할 수 있듯, `join` 이 아닌 `select` 을 해서 가져오고, 실제로 `Topic` 에 `createdTopics` 에 접근하지 않더라도 `select` 구문이 발생하게 된다.

왜 ? 지금까지 계속보았듯 `findAll` 과 같은 `목록 조회`에서는 `Eager`, `Lazy` 가 `select 시기`만을 결정하니까

#### Lazy 도 테스트 해보자.

정말 뻔하니 그냥 별다른 코멘트 없이 테스트 결과만을 나열하겠다.

- findById
```sql
Hibernate: 
    select
		... 수많은 컬럼들 ...
    from
        member m1_0 
    where
        m1_0.id=?
Hibernate: 
    select
		... 수많은 컬럼들 ...
    from
        topic c1_0 
    where
        c1_0.member_id=?
```

당연히 `join` 하지 않고

- findAll
```sql
Hibernate: 
    select
		... 수많은 컬럼들 ...
    from
        member m1_0 
    where
        m1_0.id=?
```

당연히 조회하지 않는다면 더 이상 `select` 쿼리를 발생시키지 않는다.

### 최종적인 결론

- `findById` 는 `FetchType` 이 `Eager` 라면 `join` 을 해서 가져와준다 (최적화를 알아서 해주는 것)
- `findAll` 은 `findById` 와 다르게 `fetch type` 이 `Eager` 인지 `Lazy` 인지에 따라 해당 데이터를 가져오기 위한 `select 쿼리 발생 시기`만을 결정한다. (즉, `fetch type` 으로 인해 `join 여부`가 결정되지 않음)

