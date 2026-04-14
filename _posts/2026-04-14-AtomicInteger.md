---
title: 동시성3. AtomicInteger
tags: [concurrency]
excerpt_separator: <!--more-->
---

> synchronized의 **스레드 대기로 인한 성능 저하**라는 치명적인 단점이 있다. 
이 문제를 해결하기 위해 `java.util.concurrent.atomic` 패키지를 활용하고, 그중 정수형(`int`)을 다룰 때  **AtomicInteger**을 사용할 수 있다.
<!--more-->

## 락(Lock) 없는 동기화, AtomicInteger
`AtomicInteger`는 이름 그대로 '원자적인 정수'이다. 
<br>이 클래스를 사용하면 멀티스레드 환경에서도 synchronized 키워드 없이 안전하게 값을 증가시키거나 변경할 수 있다.

## 예시 코드
### 1. `synchronized` 방식
```java
public class SyncCounter {
    private int count = 0;

    // 스레드들이 이 메서드 앞에서 기다려야 함 (병목 발생)
    public synchronized void increment() {
        count++; 
    }

    public synchronized int getCount() {
        return count;
    }
}
```
### 2. `AtomicInteger` 방식
```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounter {
    // 1. 일반 int 대신 AtomicInteger 객체 생성
    private AtomicInteger count = new AtomicInteger(0); 

    public void increment() {
        // 2. synchronized 없이 메서드 호출
        count.incrementAndGet(); 
    }

    public int getCount() {
        return count.get();
    }
}
```

`incrementAndGet()` 메서드를 호출하면 내부적으로 메모리 가시성 문제와 원자성 문제를 한 번에, 락을 걸지 않고 해결한다.

## `AtomicInteger`의 핵심 메서드
- `incrementAndGet()`: 값을 1 증가시키고, 증가된 후의 값을 리턴. (++i 와 동일)

- `getAndIncrement()`: 값을 1 증가시키지만, 증가되기 전의 값을 리턴. (i++ 와 동일)

- `addAndGet(int delta)`: 원하는 값(delta)만큼 더하고, 그 결과를 리턴.

- `get()`: 현재 값을 리턴.

## 락(Lock) 없이 가능한 이유
`AtomicInteger` 클래스는 내부적으로 `CAS (Compare-And-Swap)` 연산을 활용한다.  
또한 `volatile` 기반으로 메모리 가시성을 보장하면서, CAS를 통해 락 없이 원자성을 확보한다.

하지만 CAS는 충돌 시 재시도(spin)가 발생하므로 경쟁이 심한 환경에서는 성능 비용이 증가할 수 있다.

이러한 방식은 낙관적 락과 유사한 개념으로 Redis나 데이터베이스 기반 동시성 제어를 이해하는 데 도움이 된다...