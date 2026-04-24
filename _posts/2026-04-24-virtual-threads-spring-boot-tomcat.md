---
title: Spring Boot에서 가상 스레드를 켜면 톰캣 스레드풀은 어떻게 달라질까
tags: [virtual-thread]
excerpt_separator: <!--more-->
---

> **`spring.threads.virtual.enabled=true`를 켜면 정말 스레드풀이 사라지는 걸까?**  
> **그리고 `@Async`, `Executor`, 톰캣 요청 처리 스레드는 서로 어떻게 다른 걸까?**


현재 설정은 이렇게 되어 있습니다.

```yaml
spring:
  threads:
    virtual:
      enabled: true
  main:
    keep-alive: true
```

그리고 코드에는 `@EnableAsync`와 `@Async`도 들어가 있습니다.

```java
@EnableAsync
@Configuration
public class AsyncConfig {
}
```

```java
@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handle(OnboardingCompleteEvent event) {
    ...
}
```

여기서 궁금해졌습니다.

- 가상 스레드를 끄면 기본 톰캣 스레드풀은 몇 개인가
- 가상 스레드를 켜면 그 스레드풀은 어떻게 바뀌는가
- `spring.main.keep-alive=true`는 왜 필요한가
- 가상 스레드를 쓰면 `@Async`나 `Executor`는 필요 없어진다고 봐도 되는가
- `saveAll`과 batch insert를 구분해야 했던 것처럼, 여기서도 “스레드”라는 단어 하나로 여러 개념이 섞여 있는 것 아닌가

이번 글은 이 질문을 정리한 글입니다.

<!--more-->

---

## 먼저 구분해야 할 것: 지금 이야기하는 스레드는 하나가 아니다

가상 스레드 얘기를 할 때 자꾸 헷갈리는 이유는 보통 아래 세 가지가 섞여서 말해지기 때문이라고 생각합니다.

1. **톰캣이 HTTP 요청을 처리하는 스레드**
2. **Spring이 `@Async`에 쓰는 `Executor` 스레드**
3. **가상 스레드를 실제로 실행시키는 JVM 내부 carrier thread**

이 셋은 같은 것이 아닙니다.

예를 들어 `spring.threads.virtual.enabled=true`를 켰다고 해서

- 톰캣의 worker thread 개념
- Spring `@Async`용 executor
- JVM 내부 가상 스레드 scheduler

가 전부 완전히 같은 방식으로 동작하는 것은 아닙니다.

그래서 이번 글은 일부러 이 셋을 나눠서 설명하려고 합니다.

---

## 가상 스레드를 끄면 기본 톰캣 스레드풀은?

Spring Boot에서 서블릿 애플리케이션을 띄우면 기본적으로 embedded Tomcat이 요청을 받습니다.

이때 많이 보는 설정이 이것입니다.

- `server.tomcat.threads.max`
- `server.tomcat.threads.min-spare`

Spring Boot의 공통 속성 문서를 보면 기본값은 다음과 같습니다.

- `server.tomcat.threads.max = 200`
- `server.tomcat.threads.min-spare = 10`

가상 스레드를 켜지 않은 일반적인 경우,
톰캣은 기본적으로 **최대 200개의 worker thread**를 사용하고
**최소 10개 정도는 spare thread**로 유지하는 구조로 볼 수 있습니다.

보면 보통 이런 그림입니다.

- 요청이 들어오면 톰캣 worker thread 하나가 붙는다
- 그 스레드는 요청 처리 동안 점유된다
- blocking I/O가 길어지면 그동안 OS thread도 같이 묶인다
- 동시 요청이 많아지면 결국 worker thread 수가 병목이 된다

전통적인 톰캣 모델은 사실상 **platform thread 기반 thread-per-request**에 가깝습니다.

---

## 그럼 가상 스레드를 켜면 톰캣 스레드풀은 몇 개가 되나

여기서 제일 많이 헷갈렸던 부분이 이거였습니다.

> **가상 스레드를 켜면 톰캣 스레드풀 크기는 몇 개가 되는가?**

결론부터 말하면:

> **고정된 “톰캣 worker thread pool 크기”라는 관점이 거의 의미가 없어집니다.**

Spring Boot 문서에서도 `server.tomcat.threads.max`, `server.tomcat.threads.min-spare`는
**virtual threads가 enabled일 때는 효과가 없다**고 되어 있습니다.

이 시점부터는

- `max=200`
- `min-spare=10`

같은 전통적인 톰캣 worker pool 튜닝이 핵심이 아니게 됩니다.

가상 스레드 모드에서는 “요청당 가상 스레드 하나”에 더 가까운 모델로 바뀝니다.

다만 여기서 조심해야 할 게 있습니다.

> **“스레드풀이 아예 없다”는 말과 “고정된 worker pool 개념이 약해진다”는 말은 다릅니다.**

왜냐하면 가상 스레드는 결국 JVM 내부 scheduler 위에서 돌아가고,
그 scheduler는 소수의 platform thread를 carrier로 사용하기 때문입니다.

JDK 공식 문서의 `VirtualThreadSchedulerMXBean` 설명을 보면,
JDK의 virtual thread scheduler는 `ForkJoinPool`이고
**target parallelism 기본값은 `available processors` 수**라고 되어 있습니다.

정리하면:

- **톰캣 요청 처리 관점**: 고정 worker pool 개념이 약해짐
- **JVM 내부 실행 관점**: carrier 역할을 하는 platform thread 집합은 존재
- **기본 target parallelism**: 사용 가능한 CPU 코어 수 기준

그래서 “가상 스레드를 켰을 때 스레드풀 개수는 몇 개냐”는 질문에는 이렇게 답하는 게 제일 정확하다고 생각합니다.

1. 톰캣 worker thread 풀 관점에서는 기존 `200/10` 같은 설정이 의미를 잃는다
2. 대신 요청당 가상 스레드가 만들어진다
3. 그 가상 스레드들은 JVM-wide scheduler 위에서 실행되고
4. scheduler의 기본 parallelism은 CPU 코어 개수 기준이다

**“톰캣 worker pool 몇 개”라고 딱 떨어지게 말하기보다, “요청당 virtual thread + JVM scheduler” 모델로 바뀐다**고 설명하는 게 맞습니다.

---

## `@Async` Executor는 별개

위까지는 **HTTP 요청을 처리하는 톰캣 쪽 이야기**였습니다.

그런데 제 코드에는 `@EnableAsync`와 `@Async`도 있습니다.

이건 또 다른 내용의 이야기입니다.

Spring Boot 문서를 보면, 별도의 `Executor` bean이 없을 때 Spring Boot는 `AsyncTaskExecutor`를 자동 설정합니다.

- virtual threads **비활성화**: `ThreadPoolTaskExecutor`
- virtual threads **활성화**: `SimpleAsyncTaskExecutor` that uses virtual threads

그리고 이 자동 구성된 executor는 다음에 사용됩니다.

- `@EnableAsync`
- Spring MVC async request processing
- Spring WebFlux blocking execution support
- 일부 GraphQL 비동기 처리

지금 제 `@Async` 이벤트 핸들러는
가상 스레드를 켜기 전후로 다른 executor를 타게 됩니다.

### 가상 스레드를 끄면

Spring Boot 기본 executor는 `ThreadPoolTaskExecutor`입니다.

공식 문서 기준으로 이 풀은 **core 8 threads**를 사용합니다.  
그리고 부하에 따라 grow/shrink 된다고 설명합니다.

`@Async` 쪽은 톰캣 기본값 200개와는 또 별개로,
기본적으로 **8개 core thread 기반 executor**를 쓰는 셈입니다.

### 가상 스레드를 켜면

Spring Boot 기본 executor는 `SimpleAsyncTaskExecutor`로 바뀌고,
이 executor는 virtual thread를 사용합니다.

중요한 점은:

> **이 시점에는 `spring.task.execution.pool.*` 같은 thread pool 관련 속성이 더 이상 의미가 없습니다.**

공식 문서에도 virtual threads enabled 시에는
thread pool 관련 속성들이 효과가 없다고 명시되어 있습니다.

즉 `@Async` 관점에서 보면:

- 예전: thread pool 기반
- 지금: task마다 virtual thread

로 바뀐 것입니다.

---

## 비동기(`@Async`)와 가상 스레드는 같은 개념이 아니다

이 부분도 꽤 헷갈리기 쉬웠습니다.

가상 스레드를 켰다고 해서 “이제 다 비동기야”라고 생각하면 안 됩니다.

둘은 목적이 다릅니다.

### `@Async` / 비동기

비동기는 보통 **호출한 스레드와 실행할 스레드를 분리**하는 개념입니다.

즉:

- 지금 요청 스레드에서 바로 하지 말고
- 다른 executor에 작업을 넘겨서
- caller는 빨리 돌아오게 하자

가 핵심입니다.

예를 들면:

- 이벤트 후처리
- 메일 발송
- 로그 적재
- 외부 시스템 호출

같이 “굳이 지금 요청 응답을 붙잡고 할 필요는 없는 일”을 분리하는 데 의미가 있습니다.

### 가상 스레드

가상 스레드는 **동기/블로킹 스타일 코드를 더 싸게 실행하는 방법**에 가깝습니다.

Oracle 문서에서도 virtual thread는

- high-throughput
- blocking I/O
- thread-per-request style

애플리케이션에 적합하다고 설명합니다.

가상 스레드는
“비동기 프로그래밍을 강제하는 기술”이 아니라
오히려 **동기식 코드를 유지한 채 더 많은 동시성을 감당하게 해주는 기술**에 가깝습니다.

그래서 이렇게 정리하는 게 좋았습니다.

- `@Async`는 **작업을 다른 실행 흐름으로 분리**하는 것
- virtual thread는 **스레드당 비용을 낮춰서 blocking 코드를 더 많이 동시에 처리**하는 것

둘은 겹칠 수는 있지만 같은 개념은 아닙니다.

---

## 왜 예전에는 스레드풀이 중요했고, 가상 스레드에서는 덜 중요해지는가

이 부분은 CS 관점에서 보면

전통적인 platform thread는 결국 OS thread와 연결됩니다.

즉 platform thread를 많이 만들수록 비용이 있습니다.

- 커널이 관리해야 할 스레드가 늘어남
- 스택 메모리 비용이 커짐
- context switching 비용이 늘어남
- blocking 중인 스레드도 OS 자원 관점에서는 비싼 자원임

그래서 기존에는 스레드를 함부로 많이 만들지 않고,
미리 정해진 개수의 thread pool을 재사용하는 방식이 중요했습니다.

스레드풀은 단순히 편의 기능이 아니라,
**비싼 platform thread를 재사용하기 위한 전략**이기도 했습니다.

반면 virtual thread는 비용 모델이 다릅니다.

Oracle 문서에서도 virtual thread는 lightweight 하며,
JVM 하나가 **millions of virtual threads**를 지원할 수 있다고 설명합니다.

그리고 virtual thread는 blocking I/O 시 carrier에서 unmount될 수 있기 때문에,
platform thread처럼 오래 붙잡고 있지 않습니다.

그래서 virtual thread 환경에서는
예전 platform thread 시대의 습관,
“무조건 thread pool로 개수를 제한해야 한다”는 사고를 그대로 가져오면 오히려 맞지 않을 수 있습니다.

가상 스레드는 보통 **task당 하나씩 만드는 방식**이 더 자연스럽고,
리소스 제한이 필요하면 thread pool보다 semaphore 같은 방식이 더 적절할 수 있습니다.

즉 정리하면:

### platform thread 시대

- 스레드 생성 비용 큼
- 블로킹 비용 큼
- context switch 비용 큼
- 그래서 thread pool 재사용이 중요

### virtual thread 시대

- 스레드 생성 비용 훨씬 낮음
- blocking I/O 친화적
- task당 thread 모델이 다시 현실적
- thread pool의 이유 중 “생성 비용 절감” 비중이 많이 낮아짐

---

## `spring.main.keep-alive=true`는 왜 필요한가

이건 이번에 공식 문서를 다시 보면서 알게된 내용입니다.

가상 스레드의 중요한 특성 중 하나는:

> **virtual thread는 daemon thread다**

라는 점입니다.

Java에서 daemon thread만 남으면 JVM은 종료될 수 있습니다.

Spring Boot 문서도 이 점을 강조합니다.

- 가상 스레드를 켜면
- 스케줄러 스레드도 virtual thread가 될 수 있고
- 이 경우 JVM을 붙잡아두지 못할 수 있다

그래서 공식 문서에서는

```yaml
spring:
  main:
    keep-alive: true
```

를 **권장**합니다.

문서 표현 그대로,
이 설정은 **all threads are virtual threads인 경우에도 JVM이 살아 있도록 보장**하기 위한 설정입니다.

제 설정에도 이걸 넣은 이유가 바로 이 때문입니다.

```yaml
spring:
  threads:
    virtual:
      enabled: true
  main:
    keep-alive: true
```

개인적으로는 이 설정을 보면 알 수 있는 점이 하나 더 있었습니다.

> 가상 스레드는 “켜기만 하면 끝”인 옵션이 아니라,  
> 런타임의 생명주기와 스레드 특성까지 같이 이해하고 써야 하는 옵션이다.

---

제 프로젝트 기준으로 정리하면 현재 구조는 대략 이렇습니다.

### 1. HTTP 요청 처리

- Spring Boot + embedded Tomcat
- `spring.threads.virtual.enabled=true`
- 따라서 전통적인 Tomcat worker pool `200/10` 튜닝보다
  virtual-thread-per-request 관점으로 보는 쪽이 더 맞음

### 2. `@Async` 후처리

- `@EnableAsync`가 켜져 있음
- 별도 custom `Executor` bean은 없음
- 따라서 Spring Boot auto-configured executor 사용
- virtual thread enabled 상태이므로 `SimpleAsyncTaskExecutor` 기반 virtual thread 사용

### 3. JVM 생명주기

- virtual thread는 daemon
- 그래서 `spring.main.keep-alive=true`를 같이 둠

지금 제 앱은 단순히 “톰캣에 가상 스레드 켰다”가 아니라,

- 요청 처리
- `@Async` 처리
- JVM keep-alive

까지 한 번에 바뀐 상태라고 보는 게 맞습니다.

---

## 그렇다면 언제 무엇을 봐야 하나

이번에 정리하면서 이렇게 정리해보았습니다.

### 1. HTTP 요청 처리 병목이 궁금하다

이 경우에는

- 톰캣 worker thread 개수
- blocking request 처리 방식
- virtual thread enabled 여부

를 봐야 합니다.

즉 이건 **웹 서버 요청 처리 모델** 문제입니다.

### 2. `@Async` 동작이 궁금하다

이 경우에는

- `Executor` bean이 있는지
- 없으면 Spring Boot가 무엇을 auto-configure 하는지
- virtual thread 켰을 때 `ThreadPoolTaskExecutor -> SimpleAsyncTaskExecutor`로 바뀌는지

를 봐야 합니다.

이건 **Spring task execution 모델** 문제입니다.

### 3. 앱이 왜 바로 종료되는지 궁금하다

이 경우에는

- virtual thread가 daemon인지
- `spring.main.keep-alive=true`를 넣었는지

를 봐야 합니다.

이건 **JVM 프로세스 생명주기** 문제입니다.

이 셋을 섞어 생각하면 헷갈리고,
나눠서 보면 오히려 꽤 정리가 됩니다.

---

## 정리

1. **가상 스레드를 켜기 전**
   - 톰캣은 기본적으로 worker thread max `200`, min spare `10`
   - Spring `@Async` 기본 executor는 core `8`의 `ThreadPoolTaskExecutor`

2. **가상 스레드를 켠 후**
   - 톰캣의 기존 worker pool 크기 개념은 핵심이 아니게 됨
   - 요청당 virtual thread 모델로 보는 편이 맞음
   - Spring `@Async` 기본 executor는 `SimpleAsyncTaskExecutor` 기반 virtual thread 사용
   - pool 관련 설정은 효과가 없어짐

3. **`spring.main.keep-alive=true`는 사실상 같이 봐야 한다**
   - virtual thread는 daemon이므로
   - 공식 문서도 keep-alive를 권장함

4. **비동기와 가상 스레드는 같은 개념이 아니다**
   - `@Async`는 작업 분리
   - virtual thread는 blocking 코드를 더 싸게 많이 돌리는 방식

5. **스레드풀의 필요성도 달라진다**
   - platform thread 시대에는 생성 비용과 context switching 때문에 pool 재사용이 중요
   - virtual thread 시대에는 “생성 비용 절감용 pool”의 의미가 많이 줄어든다


> **가상 스레드를 켰다고 해서 모든 스레드 관련 고민이 사라지는 것은 아니지만,  
> 어디서 어떤 스레드가 쓰이고 있는지 층위를 분리해서 보면 훨씬 명확해진다**

---

## 참고

- [Spring Boot Task Execution and Scheduling](https://docs.spring.io/spring-boot/reference/features/task-execution-and-scheduling.html)
- [Spring Boot Common Application Properties](https://docs.spring.io/spring-boot/appendix/application-properties/)
- [Apache Tomcat HTTP Connector](https://tomcat.apache.org/tomcat-10.1-doc/config/http.html)
- [Apache Tomcat Executor Configuration](https://tomcat.apache.org/tomcat-10.1-doc/config/executor.html)
- [Oracle Java Virtual Threads Guide](https://docs.oracle.com/en/java/javase/24/core/virtual-threads.html)
- [VirtualThreadSchedulerMXBean](https://docs.oracle.com/en/java/javase/24/docs/api/jdk.management/jdk/management/VirtualThreadSchedulerMXBean.html)
