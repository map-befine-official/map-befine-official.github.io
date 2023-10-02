---
title: "@Modifying 어노테이션의 옵션이 정상 동작하지 않는 문제"
description: "삭제 메서드를 호출하여도 영속성 컨텍스트의 데이터가 유지되어 발생했던 문제 해결 과정을 정리해보았습니다."
date: "2023-10-02"
update: "2023-10-02"
tags:
- Spring Data JPA
- Test
- @Modifying
---

> 이 글은 우테코 괜찮을지도의 `쥬니`가 작성하였습니다.

### 배경

괜찮을지도 서비스에서는 테스트를 `Layer To End` 방식으로 수행하고 있습니다.<br>
`Layer To End`란 우리가 알고 있는 `E2E` 테스트에서 따온 네이밍입니다. (서비스 -> 레포지토리까지의 테스트만으로도 E2E라고 불리는지는 잘 모르겠습니다)
<br> 이때, Layer는 각 계층(Service, Repo 등)을 말합니다.

서비스 계층에서 데이터를 삭제하는 테스트를 수행하는 도중, 데이터가 삭제되지 않고 조회되는 문제가 발생하였습니다.
<br> 지금부터, 그 이야기를 시작해 보려 합니다.

### 문제 상황

문제를 직면한 상황은 `사용자의 즐겨찾기 목록을 모두 삭제하는 상황`이었습니다.
<br> 테스트를 위해, 각 Repositry를 사용하여 아래와 같이 데이터를 넣어주는 작업을 수행했습니다.
```java
    @Test
    @DisplayName("즐겨찾기 목록에 있는 모든 토픽을 삭제할 수 있다")
    void deleteAllBookmarks_Success(){
            // 회원 저장
            Member member = memberRepository.save(MemberFixture.create(
            "member",
            "member@naver.com",
            Role.USER
            ));
            
            Topic topic1 = TopicFixture.createPrivateAndGroupOnlyTopic(member);
            Topic topic2 = TopicFixture.createPrivateAndGroupOnlyTopic(member);

            // 지도 저장
            topicRepository.save(topic1);
            topicRepository.save(topic2);
            
            Bookmark bookmark1 = Bookmark.createWithAssociatedTopicAndMember(topic1, member);
            Bookmark bookmark2 = Bookmark.createWithAssociatedTopicAndMember(topic2, member);

            // 즐겨찾기 등록
            bookmarkRepository.save(bookmark1);
            bookmarkRepository.save(bookmark2);
}
```
위 코드의 결과로, 사용자(Member)와 지도(Topic)가 생성되어 있을 것이고, 사용자는 자신이 만든 지도를 즐겨찾기(Bookmark)로 등록해 놓은 상황일 것입니다.
<br> 이후, 아래와 같은 코드를 수행하면 해당 사용자의 즐겨찾기 목록이 전부 삭제되어, 테스트를 통과할 것이라 예상하였습니다.
```java
        // 생략 ...
        
        // 통과 !
        assertThat(creatorBefore.getBookmarks()).hasSize(2);

        // 해당 회원의 즐겨찾기 목록 전체 삭제
        AuthMember user = MemberFixture.createUser(creatorBefore);
        bookmarkCommandService.deleteAllBookmarks(user);
        
        // 실패 !
        assertThat(bookmarkRepository.findById(creatorBefore.getId())).isEmpty();
```
테스트 실패 메시지는 아래와 같았습니다.
```java
java.lang.AssertionError: 
Expecting empty but was: [com.mapbefine.mapbefine.bookmark.domain.Bookmark@23504729,
    com.mapbefine.mapbefine.bookmark.domain.Bookmark@2447e2e]
```
즉, 즐겨찾기가 존재하지 않을 것이라고 예상하였지만, 데이터가 존재한다는 의미였습니다.
<br> 테스트에 사용된 메서드들의 로직적인 오류를 재차 확인하였지만, 발견할 수 없었습니다.
<br> 그렇다면, 도대체 왜 데이터가 삭제되지 않고 조회되는 것일까요 ?


### 원인 파악

위 문제의 원인을 찾기 위해 여러 방법을 시도하던 중, `TestEntityManager`를 통해 즐겨찾기 삭제 메서드 호출 전, 후에 다음과 같은 로직을 추가하였습니다.
```java
    @Test
    @DisplayName("즐겨찾기 목록에 있는 모든 토픽을 삭제할 수 있다")
    void deleteAllBookmarks_Success() throws InterruptedException {
        // 생략
        testEntityManager.flush();
        testEntityManager.clear();

        bookmarkCommandService.deleteAllBookmarks(user);

        testEntityManager.flush();
        testEntityManager.clear();
        // 생략
    }
```
즉, 삭제 메서드를 호출한 뒤, 명시적으로 영속성 컨텍스트를 flush & clear 해준 것입니다.
<br> 위와 같은 로직을 추가하자, 테스트가 성공적으로 통과했습니다.
<br> 이로써, 문제 원인은 영속성 컨텍스트와 관련이 있음을 알게 되었습니다.

하지만, 한 가지 의문점이 생겼습니다.
<br> 분명, 우리는 데이터의 변경이 일어나는 `Repository`의 메서드에는 `@Modifying` 어노테이션과 함께, `clearAutomatically = true`로 설정해 둔 상태였습니다.
<br> 테스트에서 직면한 문제처럼, 데이터를 수정(삭제)하였더라도 1차 캐시 내부에서는 수정 전 데이터가 존재할 수 있음을 인지하고 있었습니다.
<br> 이로인해, 수정 관련 쿼리가 실행된 후, 명시적으로 영속성 컨텍스트를 비워주었습니다.
```java
public interface BookmarkRepository extends JpaRepository<Bookmark, Long> {

    @Modifying(clearAutomatically = true)
    void deleteAllByMemberId(Long memberId);

}
```

위와 같은 설정을 해두었음에도 불구하고, 영속성 컨텍스트가 비워지지 않았다는 사실을 납득하기 어려웠습니다.
하지만, 그 원인은 생각보다 쉽게 찾을 수 있었습니다.
우리가 사용한 `@Modifying` 어노테이션을 확인해 보니, 다음과같이 쓰여있었습니다.
> Indicates a query method should be considered as modifying query as that changes the way it needs to be executed. <br>
> **This annotation is only considered if used on query methods defined through a {@link Query} annotation.**<br> It's not 
> applied on custom implementation methods or queries derived from the method name as they already have control over 
> the underlying data access APIs or specify if they are modifying by their name.

우리가 주목할 점은, 위 설명의 두 번째 줄이었습니다.
<br> 간단하게 설명하자면, `@Modifying` 어노테이션은 `@Query` 어노테이션과 함께 사용될 때만 효력이 있다는 것입니다.
<br> 즉, 아래와 같은 `NamedQeury`를 사용할 때는 옵션값을 넣어주더라도 동작하지 않았던 것이죠.
```java
public interface BookmarkRepository extends JpaRepository<Bookmark, Long> {
    @Modifying(clearAutomatically = true)
    void deleteAllByMemberId(Long memberId);
}
```

### 문제 해결

(1) `@Query` 어노테이션을 사용하여 직접 쿼리를 작성하는 방법과, (2) 테스트 코드에서 flush & clear를 명시적으로 수행하는 방법이 있었습니다.
<br> 단순히 테스트 통과에 목적을 둔 것이 아니기 때문에, 실제 프로덕트 코드에서도 예상치 못한 동작을 방지하기 위해 (1)번 방법을 선택하였습니다.
<br> 이에 따라, `Named Query -> JPQL`로 변경함으로써 문제를 해결할 수 있었습니다.
```java
public interface BookmarkRepository extends JpaRepository<Bookmark, Long> {
    @Modifying(clearAutomatically = true)
    @Query("delete from Bookmark b where b.member.id = :memberId")
    void deleteAllByMemberId(@Param(value = "memberId") Long memberId);
}

```
