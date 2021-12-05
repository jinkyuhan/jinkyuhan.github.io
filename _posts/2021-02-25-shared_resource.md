---
layout: post
title: Shared resource를 동시에 업데이트하기
date: 2020-09-01
summary: 두 요청이 동시에 한 리소스를 업데이트 할 때
categories: study java
---

# 문제가 무엇이었나요?

---

&nbsp;고객의 주문 요청이 완료될 때, 상점의 포인트를 증가 시킨 뒤 증가한 포인트가 일정 기준을 넘으면 상점의 등급을 높여주는 레벨 시스템을 구현할 때 였습니다.<br>
&nbsp;해당 API가 E2E테스트 진행 시에는 정상적으로 작동하다가, 스트레스 테스트를 하는 과정에서 등급이 계속적으로 증가하는 버그가 발생하였습니다.

<br>

# 왜 그런 문제가 생긴 거죠?

---

&nbsp;E2E 테스트는 단일 건에 대한 요청의 정상작동여부만을 테스트하였고, 스트레스 테스트는 여러개의 고객 계정이 하나의 상점 계정에 동시에 많은 주문을 하는 시나리오였습니다.<br>
&nbsp;문제는 각 요청들이 포인트를 증가시키기 위해 리소스를 읽어갈때 앞선 트랜잭션이 커밋하지 않아서 **다음 트랜잭션이 변경 이전의 값을 읽어가는 문제**가 발생했습니다. 요청들이 순서대로 포인트를 누산 했어야
하나, 각 요청이 비동기적으로 작동하여 동시에 읽어간 값을 기준으로 DB를 업데이트 시키는 것이었습니다.

```
// 문제 시나리오
1. 요청 A가 현재 포인트 n 읽어감.
2. 요청 A가 끝나기전에 요청 B가 들어옴, 현재 포인트 n 읽어감.
3. 요청 A가 DB를 n + a 로 업데이트, 등급 업 조건 충족하여 등급 업.
4. 요청 B가 DB를 n + a 로 업데이트, 등급 업 조건 충족하여 등급 업.

- 기대결과 : 포인트 n + 2a 달성, 등급 1 up
- 실제    : n + a, 등급 2 up,
```

&nbsp;문제가 매 케이스 마다 다른 형태로 나타나는 것으로 보아 동시성과 관련된 문제임을 파악할 수 있었습니다. 이 문제는 이러한 "한 리소스에 동시에 접근하는 요청들은 어떻게 처리 되어야 하는 가"라는 고민으로
이어졌습니다.

<br>

# 어떻게 해결하였나요?

---

&nbsp;고민과 공부 끝에 두가지의 고려할만한 해결방안이 생겼습니다. 하나는 Database의 Transaction Isolation level을 통한 **Locking 조절을 통한 동시성 및 실행 순서 확보**
이고, 또 하나는 현재 Cache로 사용중인 Redis를 이용하여 **MessagingQueue**를 만드는 방법이었습니다.

## 1. Transaction Isolation Level

&nbsp;DB에 트랜잭션을 통한 쿼리를 날릴 때, 트랜잭션의 격리수준(Transaction Isolation Level)을 설정 할 수 있습니다. 이는 각 Database에서 SQL 인터페이스로도 존재합니다.
&nbsp;사용하는 라이브러리에 따라 격리수준을 설정하는 방법은 다를 수 있지만, 보통 제공되는 격리수준은 다음과 같습니다.

- READ UNCOMMITTED
  - 실행되고 있는 다른 트랜잭션이 리소스를 변경시키고 커밋하기 이전이라도 값을 읽어갈 수 있습니다.
  - Resource에 대해 shared lock이 발생하지 않아서, Rollback 될 데이터도 읽어 올 수 있으므로 데이터의 일관성이 어긋날 수 있습니다.
  - 어떠한 lock도 걸지 않습니다.
- READ COMMITTED
  - 이때까지 커밋된 데이터만을 읽는 방식입니다.
  - 동일 트랜잭션 내에서 일관성을 보장하지 않습니다. 한 트랜잭션에 두번의 읽기를 할경우 값이 다르게 나올 수 있습니다.
  - 트랜잭션동안 접근하는 행에 대해 shared lock이 걸립니다.
- REPEATABLE READ
  - 동일 트랜잭션 내에서 일관성을 보장합니다
  - 읽기 시 snapshot을 만들어 놓고 한 트랜잭션 내에서는 커밋하기전까지 그 snapshot 만을 읽는다.
  - 트랜잭션 동안 접근하는 행에 대해 shared lock 이 발생합니다
- SERIALIZABLE
  - 트랜잭션 동안 다른 트랜잭션은 해당 테이블에 insert or update 가 불가능 합니다.
  - 리소스가 속한 테이블 전체에 shared lock을 겁니다.

&nbsp;제가 부딪힌 문제를 해결하기 위해서는 기본값으로 설정된 READ UNCOMMITED로 **완전히 lock을 걸지 않거나**, **SELECT FOR UPDATE 를 통해 명시적인 exclusive
lock을 걸어서** 두번째 트랜잭션이 아예 값을 읽어가지 못하고 기다리도록 하는 방법을 고려할 수 있습니다.<br>
&nbsp;하지만 READ UNCOMMITTED를 사용하는 것은 포인트라는 중요한 데이터의 정합성을 깨뜨릴 수 잇어서 절대 사용할 수 없었고, SELECT FOR UPDATE 트랜잭션 중 block이 걸리기 때문에
동시 요청이 많아 질 경우 Long Transaction으로 응답이 느려질 가능성이 있어 꺼려졌습니다.

## 2. Queueing

&nbsp;그 다음으로 고려된 것은 요청의 앞에 Queue를 두는 것입니다. 요청이 들어오면 '주문완료' 메시지를 하나씩 Queue에 집어넣고 순서대로 하나씩 꺼내서 처리를 하는 것입니다.<br>
&nbsp;요청이 들어올 때는 Queue에 넣기만 하고 응답을 주기 때문에 응답이 지연되지 않습니다. 또한 Queue의 특성상 하나씩 선입선출을 보장하기 때문에 한개의 메시지 큐를 두면 동시에 들어오는 많은 요청을 순서를 보장하며 처리할 수 있습니다.<br>

&nbsp;**Redis의 List 자료구조를 사용하여 이벤트 메시지 큐를 구현**하고, blocking pop 인터페이스를 이용한 무한 루프로 consumer를 구현하여 직렬로 처리하니 들어온 **많은 요청에 대한 순서를 보장**할 수 있었습니다.

```javascript
// 당시 작성했던 NodeJS 코드.

function getNewProducer(queueName) {
  // 새로운 이벤트 메시지를 큐에 넣는 함수를 생성
  const redisClient = require('./redis-wrapper')(`PUB_${queueName}`);
  return async (eventMessage) =>
    await redisClient.lpushAsync(queueName, JSON.stringify(eventMessage));
}

async function startEventListening(client, queueName, cb) {
  // blocking pop 으로 새로운 이벤트가 있을 때까지 block 상태로 기다림
  client.brpop(queueName, 10000, async (err, poppedMessage) => {
    if (err) {
      console.error(err);
    }
    if (poppedMessage != null) {
      const [_, eventMessageInString] = poppedMessage;
      const eventMessage = JSON.parse(eventMessageInString);
      if (eventMessage != null) {
        if (cb.constructor.name === 'AsyncFunction') {
          await cb(eventMessage);
        } else {
          cb(eventMessage);
        }
      }
    }
    // 재귀함수로 무한 루프
    startEventListening(client, queueName, cb);
  });
}

function addNewConsumer(queueName, cb) {
  // 해당 이벤트가 있을 때마다 콜백 함수 실행하도록 consumer 등록
  const redisClient = require('./redis-wrapper')(`SUB_${queueName}`, () => {
    startEventListening(redisClient, queueName, cb);
  });
}
```

<br>

# 이런 것을 느꼈어요

---

&nbsp;서버는 늘 N개의 요청을 상대해야 합니다. 그리고 각 요청이 비동기적으로 돈다는 것은 굉장히 중요하며, 이러한 사실을 완벽이 이해하고 염두에 두어야만 최적화된 프로그래밍을 할 수 있습니다. <br>
&nbsp;서버를 구성하는 환경마다 이 동시성을 구현한 방법은 많이 다르겠지만, 이를 통해 이후에 같은 실수를 하지 않을 수 있게 되었습니다.<br>
&nbsp;또한 개념적으로만 알고있었던, Java는 1요청당 1스레드를 만든다는 말을 비로소 체감 할 수 있었고, Node의 싱글스레드가 어떻게 여러 요청의 동시성을 지원하는지를 공부하는 계기가 되었습니다.
