---
title: "HikariCP 적용기"
description: "괜찮을지도 서비스의 HikariCP Size를 적용 과정을 정리한 글입니다."
date: 2023-10-10
update: 2023-10-10
tags:
  - 데이터베이스
  - HikariCP
  - 커넥션 풀
---

> 이 글은 우아한테크코스 괜찮을지도팀의 `쥬니`가 작성했습니다.
### 배경

CPU 코어가 하나인 컴퓨터라도 수십 또는 수백 개의 쓰레드를 `동시에` 지원할 수 있습니다.
<br> 하지만, 실제로 하나의 코어는 하나의 쓰레드만 실행할 수 있습니다.
<br> `쓰레드 스케줄링`을 통해, 하나의 코어에서도 여러 개의 쓰레드를 동시에 지원할 수 있는 것이죠.

물론, 두 개의 쓰레드 A, B가 존재할 때 이를 순차적으로 실행하는 것이 성능적으로는 더 빠릅니다.
<br> `Context Switching Overhead`가 없기 때문이죠 !

그렇다면, 어플리케이션과 관련된 쓰레드의 개수를 설정할 때, 코어의 개수와 동일하게 가져가면 될까요 ?
<br> **결론부터 이야기하면, 그렇지 않습니다 !**
<br> 실제 어플리케이션에서는 데이터를 저장하고 있는 `디스크`를 고려해야 하기 때문입니다.

어플리케이션에서 사용하는 데이터베이스에는 디스크가 존재합니다.
<br> 데이터를 읽기 위해서는, 물리적인 `Disk Arm과 Spindle`이 동작합니다.
<br> 원하는 데이터를 찾기 위해 위와 같은 동작을 수행하며, 이때 드는 비용은 상당히 큽니다.
<br> 이 시간을, `I/O 대기 시간`이라고 부르며, 해당 시간 동안 쓰레드들은 `waiting` 상태로 대기하게 됩니다.

그렇기 때문에, I/O 대기 시간으로 인해 waiting 상태인 쓰레드가 생기게 됩니다.
<br> 즉, 일하지 않고 놀게 되는 코어가 생기게 됩니다.

그렇다면, 데이터베이스 Connection과 관련된 `Thread Pool`의 크기는 어느 정도로 관리하는 게 좋을까요 ?

### Hikari Connection Pool

`HikariCP` 공식 문서에서는 아래와 같은 추천 공식을 제공하고 있습니다.

`Connection Pool Size = (Core count * 2) + Effective spindle count`

위 공식에서, `Effective spindle count`의 값은 사실상 하드디스크의 개수와 같다고 보면 됩니다.
<br> 디스크 1개당, 1개의 물리적 spindle을 가지고 있기 때문이죠.

`괜찮을지도` 서비스의 운영 서버 EC2 환경은 2개의 코어와 1개의 디스크를 사용하고 있습니다.
<br> 이를 위 공식에 대입해 보면, `Connection Pool Size = 2 * 2 + 1 = 5`

**즉, `HikariCP`에서 추천하는 커넥션 풀 사이즈는 5가 나오게 됩니다.**

### 성능 테스트
`HikariCP`에서 추천하는 `괜찮을지도` 서비스의 커넥션 풀 사이즈는 `5`임을 알 수 있었습니다.
<br> 하지만, 이는 단순히 이론적인 내용일 뿐 맹신할 수는 없습니다.
<br> 그래서, `JMETER`를 이용한 성능 테스트를 수행하였습니다.
<br> 테스트에서는 다른 설정을 동일하게 두고, 커넥션 풀 사이즈만 변경하며 진행했습니다.

Connection Pool Size는 아래와 같이 yml 파일을 작성하여 수정할 수 있습니다.
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 5
```

**- Connection Pool Size 10**
<br> 기본 설정값인 10으로 설정한 뒤, 테스트를 수행하였습니다.
![connection_pool_size_10.png](.index_image%2Fconnection_pool_size_10.png)

**- Connection Pool Size 5**
<br> HikariCP에서 제공하는 공식을 통해 도출된 값으로 설정한 뒤, 테스트를 수행하였습니다.
![connection_pool_size_5.png](.index_image%2Fconnection_pool_size_5.png)

유의미한 TPS 차이를 통해, 커넥션 풀 사이즈가 5일 때의 성능이 좋다는 것을 확인할 수 있었습니다.
<br> 물론, 트래픽 양에 따라, 다른 값을 적용하였을 때가 더 최적의 성능을 나타낼 수 있습니다.

`괜찮을지도` 서비스에서는 성능 테스트 목표치량을 기준으로 테스트를 수행하였기 때문에, 위와 같은 결과가 도출되었습니다.

### 마치며
**위에서도 이야기했지만, 공식을 통해 도출된 값이 항상 최적인 것은 아닙니다.**
<br>**각 서비스의 특성에 맞게, 커넥션 풀 사이즈를 설정해 보시면서 테스트를 수행하여 최적의 값을 도출해 내시길 바랍니다.**

또한, 커넥션 풀 외에도 HikariCP에서 튜닝해야할 설정을 추천하고 있습니다.
<br> 이 부분도 [참고](https://github.com/brettwooldridge/HikariCP/wiki/MySQL-Configuration)하면 좋을 것 같습니다.

### 참고
[HikariCP docs](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
<br>[HikariCP MySQL Configuration](https://github.com/brettwooldridge/HikariCP/wiki/MySQL-Configuration)
