---
title: Java 3퍼센트 문제(부동소수점 0으로 나누기)
tags: [Java]
excerpt_separator: <!--more-->
---

오늘 인터넷에서 본 9급 전산 객관식 시험에서 정답률 3%짜리의 문제를 보고 풀어보았습니다.

쉬워보여서 당연히 맞을 줄 알았는데... 

틀렸습니다.

쉽다고 생각한 것도 다시 풀어봅시다..

아래 코드를 보고 출력될 값들을 고르는 문제입니다.


 <!--more-->
 
## Java에서 0으로 나누면 왜 Infinity가 나올까?

```java
public class Main {

    public static void div(double a, double b) {
        try {
            System.out.println(a / b);
        } catch (ArithmeticException e) {
            System.out.println("divide by zero");
        }
    }

    public static void main(String[] args) {
        div(3, 2);
        div(2, 0);
    }
}
```

```
[1]
    1
    Infinity

[2]
    1.5
    Infinity

[3]
    1
    divide by zero

[4]
    1.5
    divide by zero
```

제가 처음 생각한 답은 4번이었는데...
```
[4]
    1.5
    divide by zero
```

double끼리 나누니까 첫번쨰는 double로 값이 리턴 될 것이고 1.5로 생각했습니다. 이거는 아마 모두 잘 풀었을 것 같은데...

`div(2,0)`이 실행되면 당연히 `ArithmeticException / by zero` 가 발생할 것이라고 생각했습니다.

그런데 정답은  2번이었습니다
```
[2]
    1.5
    Infinity
```

Infinity는 한 번도 출력으로 본 적이 없었던 터라.. 당황스러웠습니다.

## 왜 예외가 발생하지 않을까?

메서드의 파라미터 타입은 `double`입니다. (div에서 double / double를 하니까요)
```java
2.0 / 0.0
```

자바의 `double`과 `float`는 IEEE 754 부동소수점 표준을 따른다고 합니다. 

이 표준에서는 0으로 나누는 상황을 예외로 처리하지 않고 특별한 값을 리턴한다고 합니다.


```java
System.out.println(2.0 / 0.0);   // Infinity
System.out.println(-2.0 / 0.0);  // -Infinity
System.out.println(0.0 / 0.0);   // NaN
```

- 양수 / 0.0 → `Infinity`
- 음수 / 0.0 → `-Infinity`
- 0.0 / 0.0 → `NaN` (Not a Number)

## `ArithmeticException`은 언제 발생할까?

정수 연산에서만 발생합니다.

```java
System.out.println(2 / 0);
```

실행 결과:

```text
Exception in thread "main" java.lang.ArithmeticException: / by zero
```

- `int`, `long` → `/ by zero` 예외 발생
- `float`, `double` → `Infinity`, `-Infinity`, `NaN` 반환

## 필요한 경우 직접 예외를 던지기

```java
public static void div(double a, double b) {
    if (b == 0.0) {
        throw new ArithmeticException("divide by zero");
    }

    System.out.println(a / b);
}
```

## 정리

백엔드를 준비하면서 다른 걸 주로 알아보다보니 정작 언어의 가장 기본적인 동작을 당연하게 가정하고 넘어간 순간들이 있는 것 같습니다

솔직히 `Infinity`를 콘솔에서 본 건 자바를 공부하면서 처음인거 같습니다.

"0으로 나누면 예외가 발생한다"는 것은 맞지만, 그것이 모든 숫자 타입에 동일하게 적용된다고 생각하고 있었습니다.

기초는 이미 안다고 착각하기 쉽지만 이런 작은 점들이 오히려 언어의 기본기를 다시 점검하게 만드는 것 같습니다.

오늘도 하나 배웠습니다.