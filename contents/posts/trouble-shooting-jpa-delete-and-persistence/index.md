---
title: "JPA 엔티티를 삭제할 때 영속성과 연관 관계가 중요한 이유"
description: "JPA로 연관 관계를 가진 엔티티들을 삭제하는 과정에서 겪은 삽질과 교훈을 기록했습니다."
date: 2023-10-13
update: 2023-10-13
tags:
- Spring Data JPA
- Hibernate
- JPQL
- 트러블슈팅
---
> 
> 이 글은 우테코 괜찮을지도의 `도이`가 작성하였습니다.

삭제 기능에 대한 리팩터링 중, 회원 차단에 대한 기존 테스트가 실패해 이를 해결해야 했는데요.  
JPA에 대한 지식이 부족한 상태에서 삽질을 하며 알게 된 것들을 기록하고자 합니다.

## 문제 상황 1 : 쿼리의 발생 시점 찾기

### 도메인: 회원 차단 기능
도메인에 대해 먼저 설명드리겠습니다.  
관리자 API에서 회원을 차단하면, 차단한 회원의 지도 `Topic`, 핀 `Pin`, 핀 이미지 `PinImage`를 삭제 상태(soft delete)로 변경합니다.  
그리고 매핑 테이블 역할을 하는 엔티티인 `Bookmark 즐겨찾기`, `Atlas 모아보기`, `Permission 권한`은 실제로 삭제(hard delete)합니다.


```java
    public void blockMember(Long memberId) {
        Member member = findMemberById(memberId);
        member.updateStatus(Status.BLOCKED);

        deleteAllRelated(member);
        }

    private void deleteAllRelated(Member member) {
        List<Long> pinIds = extractPinIdsByMember(member);
        Long memberId = member.getId();

        permissionRepository.deleteAllByMemberId(memberId);
        atlasRepository.deleteAllByMemberId(memberId);
        bookmarkRepository.deleteAllByMemberId(memberId);
        pinImageRepository.deleteAllByPinIds(pinIds);
        pinRepository.deleteAllByMemberId(memberId);
        topicRepository.deleteAllByMemberId(memberId);
    }
```

아래와 같이 동작하기를 기대했습니다.

1. 차단할 회원에 대한 정보 조회
2. 회원의 상태를 차단으로 변경하는 update 쿼리 발생
3. 매핑 테이블 엔티티들을 먼저 삭제해야 함, 이 때 delete 쿼리 발생
4. 주요 도메인 엔티티들을 삭제 상태로 변경하는 update 쿼리 발생

### 예상과 다른 영속 동작
테스트 코드는 아래와 같습니다.

```java
    @DisplayName("Member를 차단(탈퇴시킬)할 경우, Member가 생성한 지도, 핀, 핀 이미지를 삭제 상태(soft delete)로 변경한다.")
    @Test
    void blockMember_Success() {
        //given
        // ...    
        // 객체 생성, 저장 및 기존 상태 검증 코드 생략
        // ...
            
        //when
        adminCommandService.blockMember(member.getId());

        //then
        Member blockedMember = memberRepository.findById(member.getId()).get();

        assertAll(
            () -> assertThat(blockedMember.getStatus()).isEqualTo(Status.BLOCKED), // 실패
            () -> assertThat(bookmarkRepository.existsByMemberIdAndTopicId(member.getId(), topic.getId()))
            .isFalse(), // 실패
            () -> assertThat(atlasRepository.existsByMemberIdAndTopicId(member.getId(), topic.getId())).isFalse(), // 실패
            () -> assertThat(permissionRepository.existsByTopicIdAndMemberId(topic.getId(), member.getId()))
            .isFalse() // 실패
            () -> assertThat(topicRepository.existsById(topic.getId())).isFalse(), 
            () -> assertThat(pinRepository.existsById(pin.getId())).isFalse(),
            () -> assertThat(pinImageRepository.existsById(pinImage.getId())).isFalse(),
        );
    }
```


하지만 테스트가 실패해 로그를 보니, 실제 동작은 아래와 같았습니다. 😧😧

1. 차단할 회원에 대한 정보 조회
2. ~~회원의 상태를 차단으로 변경하는 update 쿼리 발생~~
3. ~~매핑 테이블 엔티티들을 먼저 삭제해야 함, 이 때 delete 쿼리 발생~~
4. 주요 도메인 엔티티들을 삭제 상태로 변경하는 update 쿼리 발생

`// when` 절의 코드를 호출한 뒤 entityManager로 flush, clear를 해주어도 마찬가지였습니다.  

## 원인
이전에는 잘 통과하던 테스트인데, 왜 갑자기 예상과 다르게 작동할까요?  
JPA 영속성 컨텍스트의 쓰기 지연 때문이었습니다.  

1. hard delete 메서드를 통해 해당 id를 가진 엔티티를 영속성 컨텍스트에서 제거한다.
2. soft delete 메서드를 통해 update를 호출하면서, 연관된 엔티티들이 모두 영속화된다.  
   ➡️ (1)에서 delete 해도, 2에서 조회할 때 함께 불러와지는 `member`, `permission`, `atlas`, `bookmark`까지 다시 영속화된다.   
3. 영속성 컨텍스트에는 `(차단 상태가 아닌)member`, `permission`, `atlas`, `bookmark`가 존재한다.
4. `blockMember()` 호출 후 flush를 할 때, 영속성 컨텍스트의 상태를 기준으로 쿼리가 발생한다.  
   ➡️ member의 차단 상태에 대한 변경 감지도 되지 않고, delete 쿼리도 나가지 않는다.

그래서 `blockMember()` 메서드를 호출한 뒤 flush를 해줘도 소용이 없었던 것입니다.  


## 해결

단순히 해결부터 해보자면,  
soft delete 메서드로 인한 영속화가 되기 이전에 flush해서 member의 변경, hard delete에 대한 쿼리를 발생시키면 됩니다.  


### flush로 쿼리 발생 시점 조정하기

아래 처럼 서비스 단에서 해주거나, `@Modifying` 어노테이션을 사용할 수 있습니다.
```java
    private void deleteAllRelated(Member member) {
        permissionRepository.deleteAllByMemberId(memberId);
        atlasRepository.deleteAllByMemberId(memberId);
        bookmarkRepository.deleteAllByMemberId(memberId);
        // flush!
        bookmarkRepository.flush();
        pinImageRepository.deleteAllByPinIds(pinIds);
        pinRepository.deleteAllByMemberId(memberId);
        topicRepository.deleteAllByMemberId(memberId);
    }
```
## 문제 상황 2 : 테스트에서 발생하지 않는 일부 쿼리

그런데도 테스트는 성공하지 않았습니다. 🥲  
`delete bookmark~` 쿼리는 여전히 발생하지 않고 있었습니다.  

테스트 메서드와 `blockMember()`는 같은 트랜잭션으로 묶이기 때문에 `// when`절에서의 영속화 상태 때문일 것이라 짐작했습니다.  
`bookmark`를 참조하는 `topic`, 또는 `member` 객체가 영속화되어있기 때문에 delete 쿼리가 나가지 않는 것이라 생각했습니다.  

그래서 이에 대해서는 `blockMember()` 호출 전에 `testEntityManager.clear()`로 영속성 컨텍스트를 초기화해두는 것으로 해결했습니다.  


**하지만 정확한 원인이 궁금했습니다. 
왜 `atlas`, `permission`은 삭제되고 `bookmark`만 삭제되지 않았을까요?**  

## 원인
다른 매핑 테이블 엔티티들과 `bookmark`의 차이점을 살펴보았습니다.  
해당 엔티티만이 `topic`과의 연관 관계에서 `CasecadeType.PERSIST`가 걸려 있었습니다.

```java
    // Topic.class
    @OneToMany(mappedBy = "topic", cascade = CascadeType.PERSIST, orphanRemoval = true)
    private List<Bookmark> bookmarks = new ArrayList<>();
```

테스트 메서드의 트랜잭션 내에서, 영속성 컨텍스트에 존재하는 `topic`에 `bookmark`가 살아있기 때문에  
`PERSIST OPERATION`이 발생하고 `bookmark`에 대한 delete 쿼리는 무시됩니다.  

> [JPA 2.2 specification](https://download.oracle.com/otndocs/jcp/persistence-2_2-mrel-eval-spec/index.html) 문서 3.2 장 Entity Instance's Life Cycle에 따르면,  
> flush가 발생할 때 `CascadeType.PERSIST`나 `CascadeType.ALL`이 있을 경우 자식에 연쇄적으로 `PERSIST OPERATION`이 발생합니다. 
> `PERSIST OPERATION`은 연관 관계 매핑된 list의 엔티티에 대해 모두 이루어지며, 기존에 없던 값이면 새로 저장합니다.

## 해결
이에 대해서 엔티티에 걸어놓은 조건에 따라 정상적으로 동작하도록 하려면,   
`bookmarkRepository`를 통해 delete 쿼리를 호출하는 대신 연관 관계를 제거하는 방식으로 삭제해주었어야 했던 것입니다.

```java
    // Topic.class
    public void removeAllBookmarks() {
        bookmarks.clear();
        bookmarkCount = 0;
    }

    // AdminCommandService.class
    private void deleteAllRelatedMember(Member member) {
        List<Long> pinIds = extractPinIdsByMember(member);
        Long memberId = member.getId();

        permissionRepository.deleteAllByMemberId(memberId);
        atlasRepository.deleteAllByMemberId(memberId);
        atlasRepository.flush();
        topicRepository.findAllByCreatorId(memberId) // 변경된 부분
                .forEach(Topic::removeAllBookmarks);
        pinImageRepository.deleteAllByPinIds(pinIds);
        pinRepository.deleteAllByMemberId(memberId);
        topicRepository.deleteAllByMemberId(memberId);
    }
```

아래와 같이 `bookmark`에 대한 삭제 로직을 수정하니,  
테스트 코드에서 별도의 `clear`를 호출하지 않아도 잘 통과하는 것을 확인할 수 있었습니다.


## 결론
이번 삽질을 계기로 JPA를 잘 학습한 뒤 사용해야 한다는 교훈을 다시 한 번 몸소 느꼈습니다..  

엔티티의 생명 주기에 대해 잘 이해하고 객체를 생성 및 삭제해야한다는 것, 삭제할 때에도 연관 관계의 관리가 중요하다는 것을 알았습니다.
이처럼 예상하지 못한 동작을 피하기 위해 삭제 로직에서도 연관 관계 편의 메서드를 정의하는 방식으로 코드를 잘 작성할 필요가 있어 보입니다. 


## 참고 자료
- [[Spring boot] JPA Delete is not Working, 영속성와 연관 관계를 고려했는가.](https://velog.io/@jsb100800/spring-12)  
- [[jpa] CascadeType.PERSIST를 함부로 사용하면 안되는 이유](https://joont92.github.io/jpa/CascadeType-PERSIST%EB%A5%BC-%ED%95%A8%EB%B6%80%EB%A1%9C-%EC%82%AC%EC%9A%A9%ED%95%98%EB%A9%B4-%EC%95%88%EB%90%98%EB%8A%94-%EC%9D%B4%EC%9C%A0/)

