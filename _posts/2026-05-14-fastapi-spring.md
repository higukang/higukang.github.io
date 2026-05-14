---
title: FastAPI와 Spring Boot 비교하기
tags: [framework]
excerpt_separator: <!--more-->
---

Spring Boot와 Python의 FastAPI를 비교해봅시다..
두 프레임워크는 사용하는 언어는 다르지만, 웹 애플리케이션을 구성하는 핵심 개념들은 비슷한 점이 있습니다.

<!--more-->

## WAS: Apache Tomcat ↔ Uvicorn

웹 애플리케이션은 클라이언트의 HTTP 요청을 받아 처리하고 그 결과를 다시 HTTP 응답으로 반환해야 합니다. 이 역할을 담당하는 소프트웨어를 WAS(Web Application Server)라고 부릅니다.

Spring Boot에서는 주로 Apache Tomcat이 이 역할을 하고, FastAPI에서는 Uvicorn이 같은 역할을 맡습니다.

| 역할 | Spring Boot | FastAPI |
|------|------------|---------|
| WAS | Apache Tomcat | Uvicorn |

Tomcat과 Uvicorn은 모두 다음과 같은 공통된 역할을 수행합니다.

1. TCP 포트를 열어 HTTP 요청을 기다린다.
2. 요청을 웹 애플리케이션에 전달한다.
3. 애플리케이션이 생성한 응답을 클라이언트에 반환한다.

두 소프트웨어 모두 애플리케이션 앞단에서 네트워크 요청을 받아 프레임워크에 전달하는 서버 역할을 담당합니다.

### Spring Boot

```bash
java -jar app.jar
```

Spring Boot 애플리케이션을 실행하면 내부적으로 내장된 Tomcat이 함께 실행됩니다. Tomcat은 요청을 받아 DispatcherServlet에 전달하고, DispatcherServlet이 적절한 컨트롤러를 호출합니다.

```text
Client → Tomcat → DispatcherServlet → Controller
```

### FastAPI

```bash
uvicorn main:app --reload
```

이 명령을 실행하면 Uvicorn이 시작됩니다. Uvicorn은 HTTP 요청을 받아 FastAPI 애플리케이션 객체(`app = FastAPI()`)에 전달합니다.

```text
Client → Uvicorn → FastAPI → Path Operation Function
```

### 정확히 말하면

> Tomcat은 Java 진영의 Servlet Container이자 WAS이고, Uvicorn은 Python 진영의 ASGI Server입니다.

용어는 다르지만 실질적인 역할은 매우 유사합니다.

- Tomcat: Servlet API 기반 애플리케이션 실행
- Uvicorn: ASGI 기반 애플리케이션 실행

따라서 Spring 개발자의 관점에서는 다음과 같이 이해하면 된다.

> Uvicorn은 Python 생태계에서 Tomcat과 비슷한 역할을 수행하는 서버이다.

## 웹 애플리케이션 표준 인터페이스: Servlet API ↔ ASGI

Tomcat과 Uvicorn이 모두 HTTP 요청을 처리한다고 해서, 애플리케이션과 임의의 방식으로 통신하는 것은 아닙니다.

각 생태계에는 웹 서버와 애플리케이션이 서로 통신하기 위한 표준 인터페이스가 존재합니다.

| 비교 기준 | Spring Boot | FastAPI |
|---------|------------|---------|
| 웹 애플리케이션 표준 인터페이스 | Servlet API | ASGI |

즉,

- Tomcat은 Servlet API 규격을 구현한 서버
- Uvicorn은 ASGI 규격을 구현한 서버

라고 볼 수 있습니다.

### Servlet API

Java 진영에서는 Servlet API가 웹 애플리케이션의 표준 인터페이스 역할을 합니다.

Tomcat은 HTTP 요청을 받으면 이를 `HttpServletRequest`와 `HttpServletResponse` 객체로 변환하여 애플리케이션에 전달합니다.

```java
@GetMapping("/hello")
public String hello(HttpServletRequest request,
                    HttpServletResponse response) {
    return "hello";
}
```

Spring MVC는 이 Servlet API 위에서 동작합니다.

```text
Client → Tomcat → Servlet API → Spring MVC → Controller
```

### ASGI

Python 비동기 웹 생태계에서는 ASGI(Asynchronous Server Gateway Interface)가 표준 인터페이스 역할을 합니다.

Uvicorn은 HTTP 요청을 ASGI 규격에 맞는 형태로 변환하여 FastAPI 애플리케이션에 전달합니다.

```text
Client → Uvicorn → ASGI → FastAPI → Path Operation Function
```

FastAPI는 이 ASGI 인터페이스를 기반으로 동작합니다.

### 왜 이런 표준이 필요한가?

웹 서버와 프레임워크가 공통 규약을 따르면 서로 자유롭게 조합할 수 있습니다.

예를 들어:

- Spring Boot 애플리케이션은 Tomcat, Jetty, Undertow 위에서 실행할 수 있습니다.
- FastAPI 애플리케이션은 Uvicorn, Hypercorn, Daphne 위에서 실행할 수 있습니다.

즉, 서버와 프레임워크가 느슨하게 결합됩니다.

### 한 줄로 정리

Servlet API와 ASGI는 모두 웹 서버와 애플리케이션 사이의 표준 인터페이스입니다.

- Servlet API: Java 웹 표준
- ASGI: Python 비동기 웹 표준

따라서 Spring 개발자의 관점에서는 다음과 같이 이해할 수 있습니다.

> ASGI는 Python 생태계의 Servlet API라고 볼 수 있습니다.



## 웹 프레임워크: Spring Boot ↔ FastAPI
---
title: FastAPI와 Spring Boot 비교하기
tags: [framework]
excerpt_separator: <!--more-->
---

Spring Boot와 Python의 FastAPI를 비교해봅시다..
두 프레임워크는 사용하는 언어는 다르지만, 웹 애플리케이션을 구성하는 핵심 개념들은 비슷한 점이 있습니다.

<!--more-->

## WAS: Apache Tomcat ↔ Uvicorn

웹 애플리케이션은 클라이언트의 HTTP 요청을 받아 처리하고 그 결과를 다시 HTTP 응답으로 반환해야 합니다. 이 역할을 담당하는 소프트웨어를 WAS(Web Application Server)라고 부릅니다.

Spring Boot에서는 주로 Apache Tomcat이 이 역할을 하고, FastAPI에서는 Uvicorn이 같은 역할을 맡습니다.

| 역할 | Spring Boot | FastAPI |
|------|------------|---------|
| WAS | Apache Tomcat | Uvicorn |

Tomcat과 Uvicorn은 모두 다음과 같은 공통된 역할을 수행합니다.

1. TCP 포트를 열어 HTTP 요청을 기다린다.
2. 요청을 웹 애플리케이션에 전달한다.
3. 애플리케이션이 생성한 응답을 클라이언트에 반환한다.

두 소프트웨어 모두 애플리케이션 앞단에서 네트워크 요청을 받아 프레임워크에 전달하는 서버 역할을 담당합니다.

### Spring Boot

```bash
java -jar app.jar
```

Spring Boot 애플리케이션을 실행하면 내부적으로 내장된 Tomcat이 함께 실행됩니다. Tomcat은 요청을 받아 DispatcherServlet에 전달하고, DispatcherServlet이 적절한 컨트롤러를 호출합니다.

```text
Client → Tomcat → DispatcherServlet → Controller
```

### FastAPI

```bash
uvicorn main:app --reload
```

이 명령을 실행하면 Uvicorn이 시작됩니다. Uvicorn은 HTTP 요청을 받아 FastAPI 애플리케이션 객체(`app = FastAPI()`)에 전달합니다.

```text
Client → Uvicorn → FastAPI → Path Operation Function
```

### 정확히 말하면

> Tomcat은 Java 진영의 Servlet Container이자 WAS이고, Uvicorn은 Python 진영의 ASGI Server입니다.

용어는 다르지만 실질적인 역할은 매우 유사합니다.

- Tomcat: Servlet API 기반 애플리케이션 실행
- Uvicorn: ASGI 기반 애플리케이션 실행

따라서 Spring 개발자의 관점에서는 다음과 같이 이해하면 된다.

> Uvicorn은 Python 생태계에서 Tomcat과 비슷한 역할을 수행하는 서버이다.

## 웹 애플리케이션 표준 인터페이스: Servlet API ↔ ASGI

Tomcat과 Uvicorn이 모두 HTTP 요청을 처리한다고 해서, 애플리케이션과 임의의 방식으로 통신하는 것은 아닙니다.

각 생태계에는 웹 서버와 애플리케이션이 서로 통신하기 위한 표준 인터페이스가 존재합니다.

| 비교 기준 | Spring Boot | FastAPI |
|---------|------------|---------|
| 웹 애플리케이션 표준 인터페이스 | Servlet API | ASGI |

즉,

- Tomcat은 Servlet API 규격을 구현한 서버
- Uvicorn은 ASGI 규격을 구현한 서버

라고 볼 수 있습니다.

### Servlet API

Java 진영에서는 Servlet API가 웹 애플리케이션의 표준 인터페이스 역할을 합니다.

Tomcat은 HTTP 요청을 받으면 이를 `HttpServletRequest`와 `HttpServletResponse` 객체로 변환하여 애플리케이션에 전달합니다.

```java
@GetMapping("/hello")
public String hello(HttpServletRequest request,
                    HttpServletResponse response) {
    return "hello";
}
```

Spring MVC는 이 Servlet API 위에서 동작합니다.

```text
Client → Tomcat → Servlet API → Spring MVC → Controller
```

### ASGI

Python 비동기 웹 생태계에서는 ASGI(Asynchronous Server Gateway Interface)가 표준 인터페이스 역할을 합니다.

Uvicorn은 HTTP 요청을 ASGI 규격에 맞는 형태로 변환하여 FastAPI 애플리케이션에 전달합니다.

```text
Client → Uvicorn → ASGI → FastAPI → Path Operation Function
```

FastAPI는 이 ASGI 인터페이스를 기반으로 동작합니다.

### 왜 이런 표준이 필요한가?

웹 서버와 프레임워크가 공통 규약을 따르면 서로 자유롭게 조합할 수 있습니다.

예를 들어:

- Spring Boot 애플리케이션은 Tomcat, Jetty, Undertow 위에서 실행할 수 있습니다.
- FastAPI 애플리케이션은 Uvicorn, Hypercorn, Daphne 위에서 실행할 수 있습니다.

즉, 서버와 프레임워크가 느슨하게 결합됩니다.

### 한 줄로 정리

Servlet API와 ASGI는 모두 웹 서버와 애플리케이션 사이의 표준 인터페이스입니다.

- Servlet API: Java 웹 표준
- ASGI: Python 비동기 웹 표준

따라서 Spring 개발자의 관점에서는 다음과 같이 이해할 수 있습니다.

> ASGI는 Python 생태계의 Servlet API라고 볼 수 있습니다.


### 내부 동작에서 가장 중요한 차이

Servlet API와 ASGI의 차이를 이해할 때 가장 중요한 포인트는 동시성을 처리하는 방식입니다.

쉽게 말하면 Tomcat은 기본적으로 스레드 기반으로 요청을 처리하고, Uvicorn은 이벤트 루프 기반으로 요청을 처리합니다.

| 항목 | Tomcat | Uvicorn |
|------|--------|---------|
| 기본 실행 모델 | 요청마다 스레드 할당 | 이벤트 루프 기반 비동기 처리 |
| 동시성 단위 | 스레드 | 코루틴 / Task |
| I/O 처리 방식 | 블로킹 I/O 중심 | 논블로킹 I/O 중심 |
| 대표 프로그래밍 모델 | Spring MVC | FastAPI + `async/await` |

Tomcat 기반의 Spring MVC에서는 요청이 들어오면 보통 하나의 요청을 하나의 스레드가 처리합니다.

```text
Request 1 → Thread 1
Request 2 → Thread 2
Request 3 → Thread 3
```

이 방식은 이해하기 쉽고 코드 작성도 단순합니다. 하지만 DB 조회나 외부 API 호출처럼 I/O 대기가 발생하면, 해당 스레드는 응답이 올 때까지 대기 상태로 점유됩니다.

반면 Uvicorn은 이벤트 루프 위에서 여러 코루틴을 실행합니다.

```text
Event Loop
 ├─ Task 1
 ├─ Task 2
 ├─ Task 3
 └─ Task N
```

하나의 작업이 I/O 응답을 기다리는 동안 이벤트 루프는 다른 작업을 실행할 수 있습니다. 그래서 Uvicorn은 많은 연결을 적은 수의 스레드로 처리하는 데 유리합니다.

정리하면 다음과 같습니다.

- Tomcat: 요청마다 스레드를 배정해서 처리한다.
- Uvicorn: 이벤트 루프가 여러 비동기 작업을 번갈아 처리한다.

따라서 두 서버의 가장 큰 내부 동작 차이는 단순히 사용하는 표준이 Servlet API냐 ASGI냐가 아니라, 동시성을 처리하는 실행 모델에 있습니다.

> Tomcat은 스레드 중심이고, Uvicorn은 이벤트 루프와 코루틴 중심입니다.


## 웹 프레임워크: Spring Boot ↔ FastAPI

웹 프레임워크는 라우팅, 요청 처리, 데이터 검증, 의존성 관리 같은 기능을 제공하여 웹 애플리케이션 개발을 쉽게 만들어 줍니다.

| 비교 기준 | Spring Boot | FastAPI |
|---------|------------|---------|
| 웹 프레임워크 | Spring Boot | FastAPI |

Spring Boot가 Java 기반 웹 애플리케이션 개발을 지원하듯, FastAPI는 Python 기반 웹 애플리케이션 개발을 지원합니다.

## 프론트 컨트롤러: DispatcherServlet ↔ FastAPI Application/Router

모든 HTTP 요청을 먼저 받아 적절한 처리 함수로 전달하는 중앙 진입점입니다.

| 비교 기준 | Spring Boot | FastAPI |
|---------|------------|---------|
| 프론트 컨트롤러 | DispatcherServlet | FastAPI Application / Router |

Spring MVC에서는 DispatcherServlet이 이 역할을 수행하고, FastAPI에서는 `app = FastAPI()` 객체와 Router가 같은 역할을 담당합니다.

## 라우팅 애너테이션: @GetMapping ↔ @app.get()

URL과 HTTP 메서드를 특정 함수에 연결하는 기능입니다.

| 비교 기준 | Spring Boot | FastAPI |
|---------|------------|---------|
| 라우팅 | `@GetMapping` | `@app.get()` |

```java
@GetMapping("/hello")
public String hello() {
    return "hello";
}
```

```python
@app.get("/hello")
def hello():
    return "hello"
```

## 컨트롤러: @RestController ↔ Path Operation Function

실제로 HTTP 요청을 처리하고 응답을 반환하는 코드입니다.

| 비교 기준 | Spring Boot | FastAPI |
|---------|------------|---------|
| 컨트롤러 | `@RestController` | Path Operation Function |

FastAPI에서는 데코레이터가 붙은 함수 자체가 컨트롤러 역할을 합니다.

## DTO: Java Class/Record ↔ Pydantic Model

요청과 응답 데이터를 검증하고 구조화하는 객체입니다.

| 비교 기준 | Spring Boot | FastAPI |
|---------|------------|---------|
| DTO | Java Class / Record | Pydantic Model |

Pydantic은 타입 검증과 직렬화를 자동으로 수행합니다.

## 의존성 주입(DI): 생성자 주입 ↔ Depends()

필요한 객체를 직접 생성하지 않고 외부에서 주입받는 방식입니다.

| 비교 기준 | Spring Boot | FastAPI |
|---------|------------|---------|
| 의존성 주입 | 생성자 주입 | `Depends()` |

Spring의 생성자 주입과 FastAPI의 `Depends()`는 같은 목적을 가집니다.

## 비동기 처리: @Async ↔ async def

오래 걸리는 작업이나 I/O 작업을 효율적으로 처리하기 위한 기능입니다.

| 비교 기준 | Spring Boot | FastAPI |
|---------|------------|---------|
| 비동기 처리 | `@Async` | `async def` |

FastAPI는 Python의 `async`와 `await`를 활용합니다.

## ORM: Spring Data JPA ↔ SQLAlchemy

객체와 데이터베이스 테이블을 매핑하는 기술입니다.

| 비교 기준 | Spring Boot | FastAPI |
|---------|------------|---------|
| ORM | Spring Data JPA | SQLAlchemy |

두 기술 모두 객체 중심으로 데이터베이스를 다룰 수 있게 해줍니다.

## 트랜잭션 처리: @Transactional ↔ Session 기반 트랜잭션 관리

여러 데이터베이스 작업을 하나의 논리적 단위로 묶어 처리합니다.

| 비교 기준 | Spring Boot | FastAPI |
|---------|------------|---------|
| 트랜잭션 처리 | `@Transactional` | Session 기반 트랜잭션 관리 |

Spring은 선언적으로 트랜잭션을 관리하고, FastAPI에서는 SQLAlchemy Session을 통해 트랜잭션을 관리합니다.


## 정리

| 비교 기준 | Spring Boot | FastAPI |
|---------|------------|---------|
| WAS | Apache Tomcat | Uvicorn |
| 웹 애플리케이션 표준 인터페이스 | Servlet API | ASGI |
| 웹 프레임워크 | Spring Boot | FastAPI |
| 프론트 컨트롤러 | DispatcherServlet | FastAPI Application / Router |
| 라우팅 애너테이션 | `@GetMapping` | `@app.get()` |
| 컨트롤러 | `@RestController` | Path Operation Function |
| DTO | Java Class / Record | Pydantic Model |
| 의존성 주입(DI) | 생성자 주입 | `Depends()` |
| 비동기 처리 | `@Async` | `async def` |
| ORM | Spring Data JPA | SQLAlchemy |
| 트랜잭션 처리 | `@Transactional` | Session 기반 트랜잭션 관리 |