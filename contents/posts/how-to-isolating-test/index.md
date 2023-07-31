---
title: "인수테스트에서 테스트 격리하기!"
description: "인수테스트 진행시 어떻게 하면 효율적으로 각 테스트들을 독립적으로 다룰 수 있을까에 대해 다룬 글입니다."
date: 2023-07-31
update: 2023-07-31
tags:
  - 테스트
  - 데이터베이스
---

> 이 글은 우아한테크코스 괜찮을지도팀의 `매튜`가 작성했습니다I

## 테스트 간 격리.. 왜 필요할까?

우리는 프로덕션 코드의 신뢰성을 보장하기 위해서 테스트 코드를 작성한다.

그렇기 때문에, 테스트 코드는 100번을 실행시키던 100만번을 실행시키던, 동일한 결과를 내뱉어야한다.

테스트 코드를 아무리 잘 작성하더라도, 매번 테스트의 결과가 다르다면 의미가 없다.

예를 들어 아래와 같은 테스트가 있다고 해보자.

```java
@DataJpaTest
public class ExampleTest {  

    @Autowired  
    private PinRepository pinRepository;  
      
    @Test  
    void 모든_핀을_조회한다() {  
        // given  
        Pin pin = new Pin(...);  
        pinRepository.save(pin);  
          
        // when  
        List<Pin> pins = pinRepository.findAll();  
  
        // then  
        assertAll(  
                () -> assertThat(pins).hasSize(1),  
                () -> assertThat(pins.get(0)).isEqualTo(pin)  
        );  
    }  

}
```

정상적인 경우라면, 위 테스트는 통과해야한다.

하지만, 테스트 간 격리를 진행하지 않은 상태에서 이 테스트 이전에 다른 테스트에서 pin 을 저장하는 동작을 수행했고, 데이터를 지워주지 않았다면?

해당 테스트는 실패하게 될 것이다.

위 테스트는 이전에 수행된 테스트들의 동작에도 영향을 받는, 독립적이지 못한 테스트가 된 것이다.

이것이 바로 테스트 간 격리가 필요한 이유이다.

그렇다면 위와 같은 상황을 @Transactional 어노테이션만으로 완벽하게 예방할 수 있을까?

결론부터 말하자면 그럴 수 없다.

@SpringBootTest 를 사용하는 인수테스트 같은 경우는 Port 를 지정하여 서버를 띄우게 되는데, 이 때 HTTP 클라이언트와 서버는 각기 다른 스레드에서 실행되게 된다.

그렇기 때문에 테스트 코드에 @Transactional 있더라도 호출되는 쪽은 다른 스레드에서 새로운 트랜잭션을 커밋하기 때문에, 롤백 전략은 무색해지게 되고, 테스트 간 격리도 제대로 이행될 수가 없는 것이다.

## 그렇다면 격리를 위해 사용할 수 있는 방법들은 무엇이 있을까?

이번 포스트를 통해서 다뤄볼 방법은 3가지이다.

바로 `Dirtiest Context`,  `@Sql 어노테이션`, `Entity Manager` 이다.

하나씩 짚어보면서 넘어가보자.

### DirtiesContext

```java
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_EACH_TEST_METHOD)  
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class ExampleTest {
	...
}
```

`DiriesContext` 는 현재 테스트를 실행하고자 하는 컨텍스트에 빈이 로드되어 있으면 Dirties 를 확인하고 Bean 들을 Reload 하게 된다.

즉, 테이블도 다시 만들기 때문에 테스트 간의 격리가 가능하다.

하지만, 매번 테스트하기 이전에 컨텍스트를 Reload 하게 된다면, 테스트 시간은 한없이 길어지게 될 것이기 때문이다.

테스트의 장점은 프로덕션 코드의 신뢰성을 보장함에도 존재하지만, 개발자가 개발을 진행중에 코드를 올바르게 작성중인지 바로 바로 응답받기 위한 수단이기도 하다.

따라서, 응답 속도는 개발 진행 속도와 크게 연관되어 있는 것이다.

하지만, 인수테스트에 DirtiesContext 를 난사하게 되면, 매 테스트 실행마다 속이 터지는 경험을 하게 될 것이다.

### @Sql 어노테이션

해당 방법은 꽤나 획기적인 방법이다.

간단한 sql 구문만으로 테스트 간의 격리를 이뤄낼 수 있다. 

어떤 이는 외래키 제약 조건으로 인해 한번에 데이터를 삭제하는 것이 불가능하다고 생각할 수도 있지만, 아래 코드와 같이 외래키 제약 조건을 해제해주고 데이터를 삭제하는 것이 가능하다. 

```sql
SET FOREIGN_KEY_CHECKS = 0;  
TRUNCATE TABLE 테이블이름;  
...
SET FOREIGN_KEY_CHECKS = 1;
```

```java
@DataJpaTest
@Sql("/truncate.sql")
public class ExampleTest {  
	...
}
```

하지만, 해당 방법에도 단점은 존재한다.

바로, 테이블이 추가될 때마다, 해당 sql 구문을 수정해주어야 한다는 것이다.

큰 단점은 아니지만, 항상 신경써주어야 한다는 점에서 조금은 아쉽다는 생각이 든다. 

### Entity Manager

위에서도 언급하였듯 `@Sql` 어노테이션은 정말 강력하지만, 테이블이 추가될 때마다 sql 구문을 다시 수정해주어야 한다는 단점이 존재했다.

이 때, `@Sql` 어노테이션의 장점을 모두 가져가면서, 위에서 언급한 단점도 보완할 수 있는 방법이 있다.

바로 `Entity Manager` 를 활용하는 방법이다.

도대체 어떻게? `Entity Manager` 를 통해서 Data 를 지운다는 것일까?

우리는 개발자이니 코드를 통해 살펴보자. 

```java
@Componenet  
public class DatabaseCleanup implements InitializingBean {  
  
    @PersistenceContext  
    private EntityManager entityManager;  
  
    private List<String> tableNames;  
  
    @Override  
    public void afterPropertiesSet() {  
        tableNames = entityManager.getMetamodel().getEntities().stream()  
                .filter(entityType -> entityType.getJavaType().getAnnotation(Entity.class) != null)  
                .map(entityType -> convertTableNameFromCamelCaseToSnakeCase(entityType.getName()))  
                .toList();  
    }  

    private String convertTableNameFromCamelCaseToSnakeCase(String tableName) {  
        StringBuilder tableNameSnake = new StringBuilder();  
  
        for (char letter : tableName.toCharArray()) {  
            addUnderScoreForCapitalLetter(tableNameSnake, letter);  
            tableNameSnake.append(letter);  
        }  
  
        return tableNameSnake.substring(1).toLowerCase();  
    }  
  
    private void addUnderScoreForCapitalLetter(StringBuilder tableNameSnake, char letter) {  
        if (Character.isUpperCase(letter)) {  
            tableNameSnake.append(UNDERSCORE);  
        }  
    }  
  
}
```

Entity Manager 를 통해 데이터를 관리하기 위해 DatabaseCleanup 이라는 객체를 생성해줬다.

`InitializingBean` 을 implements 해 `afterPropertiesSet` 메서드를 구현하게되면, 프로퍼티가 모두 초기화되었을 때, BeanFactory에 의해 자동으로 해당 메서드가 호출되게 된다.

그러니, 해당 메서드의 내부 구현으로 Entity 들의 ClassName 을 이용하여 모든 테이블명을 생성해내어 저장하면 된다. (Entity Class 명을 Camel Case -> Snake Case 로 변환해준다. `convertTableNameFromCamelCaseToSnakeCase` 가 해당 동작을 수행해주고 있다.)

위와 같이 모든 테이블명을 생성해내어 저장했다면, 해당 테이블 명들을 이용하여 Data 를 모두 지워주는 execute 를 구현해주자.

```java
@Service  
public class DatabaseCleanup implements InitializingBean {  

	private static final String SET_REFERENTIAL_INTEGRITY_SQL_MESSAGE = "SET REFERENTIAL_INTEGRITY %s";  
	private static final String TRUNCATE_SQL_MESSAGE = "TRUNCATE TABLE %s";  
	private static final String ID_RESET_SQL_MESSAGE = "ALTER TABLE %s ALTER COLUMN ID RESTART WITH 1";  
	private static final String UNDERSCORE = "_";
  
    @PersistenceContext  
    private EntityManager entityManager;  
  
    private List<String> tableNames;  

	...

	@Transactional  
	public void execute() {  
	    entityManager.flush();  
	    entityManager.createNativeQuery(String.format(SET_REFERENTIAL_INTEGRITY_SQL_MESSAGE, false)).executeUpdate();  
	  
	    for (String tableName : tableNames) {  
	        entityManager.createNativeQuery(String.format(TRUNCATE_SQL_MESSAGE, tableName)).executeUpdate();  
	        entityManager.createNativeQuery(String.format(ID_RESET_SQL_MESSAGE, tableName)).executeUpdate() ;  
	    }  
	  
	    entityManager.createNativeQuery(String.format(SET_REFERENTIAL_INTEGRITY_SQL_MESSAGE, true)).executeUpdate();  
	}
	  
}
```

위의 코드를 순서대로 간단하게 설명해보자면

1. 외래키 제약조건을 비활성화 해준다. 
2. `afterPropertiesSet` 을 통해 생성해놓은 모든 테이블 명들을 이용하여 Data 들을 모두 Truncate 해주고, ID 값을 다시 세팅해준다.
3. 외래키 제약조건을 활성화해준다. 

이런 Flow 로 흘러가는 execute 메서드를 구현해주었다면

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)  
public class ExampleTest {  
  
    @LocalServerPort  
    private int port;  
  
    @Autowired  
    private DatabaseCleanup databaseCleanup;  
  
    @BeforeEach  
    public void setUp() {  
        RestAssured.port = port;  
    }  
  
    @AfterEach  
    public void tearDown() {  
        databaseCleanup.execute();  
    }  
  
}
```

위 코드와 같이 @AfterEach 를 통해 매 인수테스트 동작 이후 실행시켜주면, 모든 테스트들을 격리할 수 있게 되는 것이다.


## 결론

지금으로서는 `Entity Manager` 를 통해 테스트를 격리하는 것이 최선의 방법으로 보인다.

하지만, 추후에 이보다 더 좋은 방법을 발견하면, 면밀히 검토해보고 바꿀 의사가 충분하다고 생각한다.
