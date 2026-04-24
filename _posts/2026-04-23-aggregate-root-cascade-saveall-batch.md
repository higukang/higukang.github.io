---
title: Aggregate Root가 있는데 자식 엔티티를 별도 JPARepository로 저장해야 할까
tags: [ddd, jpa, database]
excerpt_separator: <!--more-->
---

> **DDD에서 `Aggregate Root`를 두고 있는데, 자식 엔티티를 저장할 때도 별도 `JpaRepository`를 직접 호출해야 할까?**

프로젝트를 리팩토링하다가 궁금한 점이 생겼습니다.

단순히 `cascade`, `saveAll`, `batch insert`, `IDENTITY` 같은 JPA 키워드의 역할을 정리하기 위해 작성했다기 보다는

실제 코드에서

- `Question`을 `Aggregate Root`로 두고 있었고
- 자식인 `AnswerOption`도 함께 저장해야 했으며
- 성능 최적화까지 조금 신경 쓰고 싶었지만
- 동시에 도메인 설계도 망가뜨리고 싶지 않았던

그 상황에서 어떤 기준으로 판단했는지를 정리한 글입니다.

특히 이번 고민에서 중요했던 건 아래 두 가지에 대해 고민해보았습니다.

1. **설계 관점**  
   `Aggregate Root`가 있는데 자식 엔티티를 별도 저장소로 직접 저장하는 것이 맞는가
2. **성능 관점**  
   `saveAll()`을 쓴다고 정말 배치 저장이 되는가  
   그리고 `GenerationType.IDENTITY`에서는 왜 기대와 다르게 동작하는가

<!--more-->

---

## 궁금해진 상황

코드를 리팩토링하면서 사용자의 설문을 받는 온보딩 질문을 관리하는 도메인이 있었습니다.

- `Question`: 온보딩 질문
- `AnswerOption`: 객관식 질문의 선택지

도메인 모델상으로는 `Question`이 `Aggregate Root`입니다.  
선택지(`AnswerOption`)는 독립적으로 존재하는 것이 아니라, 질문(`Question`)에 종속된 하위 구성요소로 보고 있었습니다.

그런데 구현을 보니 저장 방식이 조금 애매했습니다.

질문을 먼저 저장하고,
그 다음 선택지들을 따로 `AnswerOptionRepository`로 저장하고 있었기 때문입니다.

대략 이런 흐름이었습니다.

```java
@Transactional
public Question registerQuestion(RegisterQuestionRequest command) {
    Question question = questionStore.store(command.toEntity());

    if (question.getQuestionType() == QuestionType.SUBJECTIVE) {
        return question;
    }

    List<AnswerOption> answerOptions = command.getAnswerOptionsRequestList().stream()
            .map(req -> req.toEntity(question))
            .toList();

    answerOptionStore.storeAll(answerOptions);

    return question;
}
```

이 코드를 보다가 이런 생각이 들었습니다.

> **`Question`이 aggregate root인데, 왜 자식인 `AnswerOption`을 별도 repository로 저장하고 있지?**  
> **이미 `cascade` 옵션도 걸려 있는데 그냥 루트에서 관리하면 되는 것 아닌가?**

이런 궁금증에서 시작해서 자연스럽게 아래 주제들까지 같이 고민해보게되었습니다.

- `cascade`와 영속성 컨텍스트 변경 감지
- `saveAll()`의 실제 의미
- batch insert가 정확히 무엇인지
- `saveAll`과 batch insert는 왜 다른지
- `GenerationType.IDENTITY`에서는 왜 기대한 최적화가 잘 안 되는지

---

## 엔티티 매핑

질문과 선택지의 연관관계는 다음과 같았습니다.

```java
@Entity
public class Question {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(
            fetch = FetchType.EAGER,
            mappedBy = "question",
            cascade = CascadeType.ALL,
            orphanRemoval = true
    )
    private List<AnswerOption> answerOptions = new ArrayList<>();
}
```

```java
@Entity
public class AnswerOption {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "question_id")
    private Question question;
}
```

`Question.answerOptions`에는

- `cascade = CascadeType.ALL`
- `orphanRemoval = true`

가 설정되어 있었습니다.

이 매핑만 보면 `Question`을 저장하거나 수정할 때 자식 `AnswerOption`도 같이 관리하겠다는 의도는 이미 있었습니다.

그래서 더 헷갈렸습니다.

> **그렇다면 왜 굳이 `AnswerOptionRepository`를 별도로 두고 직접 `saveAll()`을 호출하고 있었을까?**

---

## 먼저 정리해야 했던 것: 비교 대상이 사실 다 다르다

처음에는 저도 머릿속에서 다음 네 가지를 비슷한 계층의 선택지처럼 놓고 생각했습니다.

1. `cascade`로 저장
2. `saveAll()`로 저장
3. batch insert 사용
4. 둘을 성능 비교

그런데 생각해보니 이 네 가지는 사실 같은 레벨의 개념이 아니었던 것 같습니다.

### 1. `cascade`(영속성 전이)

`cascade`는 **연관된 엔티티를 함께 영속화할 것인지**에 대한 JPA 매핑 옵션입니다.


- 부모를 저장할 때 자식도 함께 저장할지
- 부모를 삭제할 때 자식도 함께 삭제할지

같은 **엔티티 생명주기 전파 규칙**에 가깝습니다.

### 2. `saveAll()`

`saveAll()`은 Spring Data JPA의 **편의 API**입니다.  
엔티티를 여러 개 받아 저장해주는 메서드일 뿐입니다.

중요한 점은:

> **`saveAll()`은 “여러 엔티티를 저장해달라”는 API일 뿐, SQL을 하나로 묶어 보낸다는 뜻이 아닙니다.**

`saveAll()`을 쓴다고 자동으로 batch insert가 되는 것은 아니라는 것이었습니다.

### 3. batch insert

batch insert는 JDBC/Hibernate 레벨의 **SQL 전송 최적화**입니다.

여러 insert를

- 애플리케이션 코드에서는 여러 번 수행하더라도
- DB에는 가능한 한 묶어서 보내서

네트워크 왕복과 statement 실행 비용(DB가 SQL 한 번 실행할 때 드는 오버헤드 전체)을 줄이는 방식입니다.

batch insert는 저장 API 이름이 아니라 **실행 전략**입니다.

### 4. `GenerationType.IDENTITY`

이건 PK 생성 전략입니다.  
그런데 놀랍게도 이 전략 하나가 batch insert 가능성에 꽤 큰 영향을 줍니다.

`saveAll()`과 `batch`를 구분해서 생각해야 하고,
거기에 PK 생성 전략도 같이 봐야 합니다.

---

## 설계 관점에서 다시 보면 왜 어색했나

DDD에서 `Aggregate Root`의 핵심 역할 중 하나는 **하위 엔티티의 생명주기를 통제하는 것**입니다.

이번 예시라면:

- 외부에서는 `Question`을 통해서만 `AnswerOption`에 접근하고
- `AnswerOption`은 독립적으로 저장/삭제되기보다
- `Question`의 규칙 아래에서 생성되고 제거되는 편이 더 자연스럽습니다.

예를 들어 이런 규칙이 있었습니다.

- `SUBJECTIVE` 질문에는 선택지가 있으면 안 된다
- `CHOICE -> SUBJECTIVE`로 바뀌면 기존 선택지는 삭제되어야 한다

이 규칙은 선택지 저장소가 아니라 `Question`이 가장 잘 알고 있는 규칙입니다.

그렇다면 저장도 가능하면 이런 의도가 드러나는 편이 좋았습니다.

```java
Question question = command.toEntity();
question.addAnswerOption(...);
question.addAnswerOption(...);
questionStore.store(question);
```

이 흐름을 보면 의미가 분명합니다.

- 질문이 선택지를 소유하고 있고
- 질문이 선택지를 추가하고
- 질문이 한 번 저장되면서
- 자식도 같이 영속화된다

반면 아래 흐름은 기술적으로는 동작하더라도 aggregate 경계가 조금 흐려집니다.

```java
Question question = questionStore.store(...);
answerOptionStore.storeAll(...);
```

이 경우에는 `Question`에 종속된 자식 엔티티인데 `AnswerOption`이 별도의 저장 단위를 가진 것처럼 읽힙니다.

그래서 리팩토링을 하며 **설계 관점에서는 aggregate root를 통해 관리하는 쪽이 더 자연스럽다**고 판단했습니다.

---

## 그렇다면 `Question.answerOptions.add(...)`만 하면 되는가

여기서도 한 번 더 짚고 넘어가야 할 점이 있었습니다.

JPA 양방향 연관관계에서는 단순히 컬렉션에만 추가한다고 끝나지 않습니다.

```java
question.getAnswerOptions().add(answerOption);
```

이 코드만으로는 부족할 수 있습니다.

왜냐하면 **연관관계의 주인**은 `@ManyToOne`(일대 다의 '다') 쪽인 `AnswerOption.question`이기 때문입니다.

실제 외래키(`question_id`)를 쓰는 쪽은 `AnswerOption`입니다.

그래서 메모리상 컬렉션과 DB 반영 기준 필드를 모두 맞추려면 보통 두 작업이 같이 필요합니다.

```java
answerOption.setQuestion(question);
question.getAnswerOptions().add(answerOption);
```

그래서 저는 aggregate root 쪽에 편의 메서드를 두는 방식으로 정리했습니다.

```java
public void addAnswerOption(String content, Integer displayOrder, String optionTag) {
    validateChoiceType();

    AnswerOption answerOption = new AnswerOption(content, displayOrder, optionTag);
    answerOption.setQuestion(this);
    answerOptions.add(answerOption);
}
```

이렇게 하면 외부에서는 무조건 `Question`을 통해서만 자식을 추가하게 됩니다.

이 방식의 장점은 두 가지였습니다.

1. **JPA 연관관계 일관성 유지**
2. **도메인 규칙을 aggregate root가 통제**

연관관계의 주인은 여전히 `AnswerOption`이지만,
연관관계 변경의 **도메인 진입점**은 `Question`이 되는 구조입니다.

이 부분이 이번 리팩토링에서 개인적으로 가장 중요하다고 느낀 지점이었습니다.

---

## `assignQuestion()`을 왜 public으로 두지 않았나

이 질문도 자연스럽게 따라왔습니다.

> 연관관계 주인이 `AnswerOption`이면, 그냥 외부에서 `answerOption.setQuestion(question)` 호출하면 되는 것 아닌가?

기술적으로는 그럴 수 있습니다.  
하지만 그렇게 열어두면 aggregate root를 우회할 수 있습니다.

예를 들어 외부 코드가 이렇게 해버릴 수 있습니다.

```java
answerOption.setQuestion(question);
```

이러면 다음과 같은 문제가 생길 수 있습니다.

- `Question.answerOptions` 컬렉션에는 안 들어가 있을 수 있다
- `SUBJECTIVE` 질문에도 연결해버릴 수 있다
- `Question`의 도메인 규칙을 거치지 않고 자식이 붙을 수 있다

JPA 관계는 맞췄을지 몰라도 도메인 규칙은 깨질 수 있습니다.

그래서 이번 구조에서는:

- 외부 공개 메서드: `Question.addAnswerOption(...)`
- 내부 보조 메서드: `AnswerOption.setQuestion(...)`

이렇게 역할을 나눴습니다.

연관관계의 주인은 `AnswerOption`이지만,
생명주기와 규칙 통제는 `Question`이 하는 구조입니다.

---

## 그럼 `saveAll()`은 왜 아쉬웠나

여기서 오해하기 쉬운 부분이 하나 있습니다.

`saveAll()` 자체가 나쁜 것은 아닙니다.

오히려 아래처럼 “여러 자식 엔티티를 한 번에 저장한다”는 의도는 꽤 직관적입니다.

```java
answerOptionRepository.saveAll(answerOptions);
```

다만 이번 상황에서는 두 가지가 걸렸습니다.

### 1. aggregate root 관점이 흐려진다

앞서 말한 것처럼 `Question`이 root인데
`AnswerOptionRepository`를 별도로 호출하면
읽는 입장에서 저장 단위가 둘처럼 느껴집니다.

### 2. `saveAll()`이 곧 batch insert는 아니다

이 부분은 저도 다시 정확히 정리할 필요가 있었습니다.

Spring Data JPA의 `saveAll()`은 결국 내부적으로 각 요소를 순회하며 저장하는 개념에 가깝습니다.

- API는 한 번 호출했지만
- 내부적으로는 여러 엔티티를 각각 영속화 대상으로 등록하고
- flush 시점에 각 엔티티에 대한 insert SQL이 생성될 수 있습니다

여기서 진짜 batch insert가 되려면 별도의 조건이 필요합니다.

`saveAll()`과 batch insert는 같은 말이 아닙니다.

---

## batch insert는 무엇이고, 어떻게 해야 하나

batch insert는 **여러 insert SQL을 JDBC batch로 묶어 보내는 최적화**입니다.

Spring Boot + JPA 환경에서 보통은 Hibernate 설정으로 활성화합니다.

예를 들면 대략 이런 설정을 합니다.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
```

이런 설정이 있으면 Hibernate가 가능한 경우 insert들을 묶어서 보낼 수 있습니다.

하지만 여기서 중요한 건 **가능한 경우**라는 점입니다.

다음 조건들이 같이 맞아야 기대한 효과가 잘 나옵니다.

- Hibernate batch 설정이 켜져 있을 것
- insert 순서를 Hibernate가 정리할 수 있을 것
- PK 생성 전략이 batch에 불리하지 않을 것
- flush 타이밍이 너무 자주 발생하지 않을 것

`saveAll()`만으로는 부족하고,
실제로는 JPA 설정과 PK 전략까지 같이 봐야 합니다.

---

## `saveAll()`과 batch insert는 어떻게 다른가

이건 이번에 꼭 구분해서 정리하고 싶었던 부분입니다.

| 항목 | 의미 |
| --- | --- |
| `saveAll()` | Spring Data JPA API |
| batch insert | JDBC/Hibernate SQL 전송 최적화 |

조금 더 풀어 쓰면 이렇습니다.

### `saveAll()`

- 자바 코드에서 여러 엔티티를 한 번에 넘긴다
- 저장 책임을 편의상 하나의 API로 감싼다
- 하지만 SQL이 하나로 나간다는 뜻은 아니다

### batch insert

- 실제 SQL 전송을 묶는다
- 네트워크 왕복 비용과 statement 실행 비용을 줄인다
- 설정과 PK 전략에 따라 가능 여부가 달라진다


> **`saveAll()`은 “API 수준의 묶음”이고, batch insert는 “SQL 실행 수준의 묶음”이다.**

이 둘을 같은 것으로 생각하면 의사결정을 잘못하기 쉽습니다.

---

## 테스트로 직접 확인

여기까지는 개념 설명이었고, 저도 실제로는 “그래도 로그를 보면 혹시 다르게 보이지 않을까?”가 궁금했습니다.

그래서 `@DataJpaTest`를 하나 추가해서 두 가지 경우를 직접 확인했습니다.

1. `Question`에 `AnswerOption`을 붙이고 cascade로 저장하는 경우
2. `Question`을 먼저 저장하고 `AnswerOption`을 `saveAll()`로 저장하는 경우

테스트 코드는 대략 이런 식이었습니다.

```java
@DataJpaTest(properties = {
        "spring.jpa.show-sql=true",
        "spring.jpa.properties.hibernate.format_sql=true",
        "logging.level.org.hibernate.SQL=DEBUG",
        "logging.level.org.hibernate.orm.jdbc.bind=TRACE"
})
@Import(JpaAuditingConfig.class)
class OnboardingPersistenceSqlLogTest {

    @Test
    void cascadePersistLogsOneInsertPerAnswerOption(CapturedOutput output) {
        Question question = Question.builder()
                .content("질문")
                .purpose("목적")
                .displayOrder(1)
                .questionType(Question.QuestionType.CHOICE)
                .build();

        question.addAnswerOption("선택지 1", 1, "tag1");
        question.addAnswerOption("선택지 2", 2, "tag2");

        questionRepository.saveAndFlush(question);
    }

    @Test
    void saveAllStillLogsSeparateInsertStatements(CapturedOutput output) {
        Question question = Question.builder()
                .content("질문")
                .purpose("목적")
                .displayOrder(1)
                .questionType(Question.QuestionType.CHOICE)
                .build();

        questionRepository.saveAndFlush(question);

        AnswerOption option1 = new AnswerOption("선택지 1", 1, "tag1");
        option1.setQuestion(question);

        AnswerOption option2 = new AnswerOption("선택지 2", 2, "tag2");
        option2.setQuestion(question);

        answerOptionRepository.saveAll(List.of(option1, option2));
        answerOptionRepository.flush();
    }
}
```

이 테스트를 돌려보고 로그를 보면, cascade 저장에서도 insert가 각각 따로 나가는 것을 볼 수 있었습니다.

```sql
insert
into
    onboarding_questions
    (content, created_at, display_order, purpose, question_type, updated_at, id)
values
    (?, ?, ?, ?, ?, ?, default)

insert
into
    answer_options
    (content, display_order, option_tag, question_id, id)
values
    (?, ?, ?, ?, default)

insert
into
    answer_options
    (content, display_order, option_tag, question_id, id)
values
    (?, ?, ?, ?, default)
```

그리고 `saveAll()`로 저장한 경우에도 마찬가지로 `answer_options` insert가 요소별로 각각 나갔습니다.

```sql
insert
into
    answer_options
    (content, display_order, option_tag, question_id, id)
values
    (?, ?, ?, ?, default)

insert
into
    answer_options
    (content, display_order, option_tag, question_id, id)
values
    (?, ?, ?, ?, default)
```

현재 설정에서는

- cascade로 저장한다고 해서 선택지 insert가 자동으로 batch로 묶이는 것은 아니다
- `saveAll()`을 쓴다고 해서 선택지 insert가 한 번의 SQL로 처리되는 것도 아니다
- 결국 둘의 차이는 “설계 의도” 쪽이 더 크고, “SQL 개수 자체”는 지금 설정에선 큰 차이가 없었다

이 테스트는 블로그 글을 쓰면서 개인적으로 꽤 도움이 됐습니다.  
머릿속 추론만으로 결론을 내리는 것보다 짧은 테스트로 실제 로그를 확인하고 나니 판단 기준이 훨씬 분명해졌기 때문입니다.

---

## `GenerationType.IDENTITY`이면 왜 기대만큼 묶이지 않을까

프로젝트에서는 `Question`과 `AnswerOption` 모두 아래처럼 `IDENTITY` 전략을 쓰고 있었습니다.

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

`IDENTITY` 전략은 DB가 insert 시점에 PK를 생성합니다.

애플리케이션은 insert를 날린 뒤에

- 이 엔티티의 id가 몇인지
- 다음 연산에 어떤 FK를 넣어야 하는지

를 알 수 있습니다.

이 특성 때문에 Hibernate는 insert를 batch로 공격적으로 묶기 어렵습니다.

특히 부모와 자식이 같이 있는 경우에는:

1. 먼저 `Question` insert
2. DB가 생성한 `question.id` 확보
3. 그 값을 `AnswerOption.question_id`에 넣어야 함

이 흐름이 필요합니다.

PK를 미리 아는 전략(`SEQUENCE` 등)보다 batch 처리에 불리할 수밖에 없습니다.

그래서 `saveAll()`을 쓰더라도 실제로는 insert SQL이 개별로 나갈 가능성이 큽니다.

---

## MySQL, PostgreSQL에서는 어떻게 봐야 하나

`IDENTITY` 전략의 문제를 이해할 때 MySQL과 PostgreSQL을 같이 떠올리게 됩니다.

둘 다 디테일은 다르지만, 핵심은 비슷합니다.

- insert 시점에 PK가 확정된다
- JPA/Hibernate는 그 PK 값을 알아야 다음 관계를 처리할 수 있다
- 그래서 insert를 자유롭게 묶기 어렵다

“DB가 키를 만들어주는 전략”은 보통 JPA batch insert에 불리합니다.

물론 실제 내부 구현이나 드라이버 수준 동작은 DB마다 다르지만,
애플리케이션 레벨에서 판단할 때는 아래 정도로 정리해두는 것이 실용적이었습니다.

> **`IDENTITY` 전략에서는 `saveAll()`을 쓴다고 해서 기대한 batch insert가 쉽게 보장되지 않는다.**

---

## 성능 관점 비교

이제 처음 고민했던 선택지를 성능 관점에서 다시 정리해보면 이렇습니다.

### 1. `cascade + aggregate root`

장점:

- 설계가 자연스럽다
- aggregate 경계가 분명하다
- 자식 생명주기를 root가 통제한다

단점:

- batch insert 자체를 보장하지는 않는다
- 결국 PK 전략과 Hibernate 설정의 영향을 받는다

### 2. `saveAll()`

장점:

- 코드만 보면 “여러 개를 한 번에 저장한다”는 의도가 명확하다
- 단순 CRUD에서는 실용적이다

단점:

- aggregate root 관점에서는 자식이 별도 저장 단위처럼 보일 수 있다
- batch insert와 혼동하기 쉽다

### 3. batch insert

장점:

- 대량 insert에서는 성능상 이점이 있을 수 있다
- 네트워크 왕복과 statement 실행 비용을 줄일 수 있다

단점:

- 설정, flush 전략, PK 생성 전략의 영향을 크게 받는다
- 소량 데이터에서는 체감 이점이 거의 없을 수 있다

“무엇이 더 빠를까”보다 **“지금 문제에서 정말 그 정도 최적화가 필요한가”** 에 대해 더 생각해보았습니다

---

## 선택한 방식

이번 온보딩 도메인에서는 `AnswerOption`이 많아야 4개 정도였습니다.

이 숫자를 보고 나서 생각이 조금 더 분명해졌습니다.

> **이 정도 개수라면 성능 미세 최적화보다 aggregate 일관성을 우선하는 편이 맞다.**

이번 상황에서는:

- `saveAll()`을 쓰는 쪽이 특별히 더 빠를 가능성도 크지 않았고
- `IDENTITY` 전략 때문에 batch insert도 크게 기대하기 어려웠고
- 자식 개수 자체도 매우 적었기 때문에

도메인 모델을 더 깔끔하게 만드는 쪽이 낫다고 판단했습니다.

그래서 최종적으로는 아래 구조를 선택했습니다.

```java
@Transactional
public Question registerQuestion(RegisterQuestionRequest command) {
    Question question = command.toEntity();

    if (question.getQuestionType() == Question.QuestionType.CHOICE) {
        command.getAnswerOptionsRequestList().forEach(request ->
                question.addAnswerOption(
                        request.getContent(),
                        request.getDisplayOrder(),
                        request.getOptionTag()
                )
        );
    }

    return questionStore.store(question);
}
```

이 방식으로 바꾸면서

- `AnswerOptionRepository`
- `AnswerOptionStore`
- `AnswerOptionStoreImpl`

도 온보딩 도메인에서는 제거할 수 있었습니다.

저장 전략이 더 단순해졌을 뿐 아니라,
도메인 모델의 의도도 조금 더 잘 드러나게 되었습니다.

---

## 어떤 상황에서 무엇을 고르면 좋을까

이번 경험을 기준으로 정리해보면 저는 대략 이렇게 판단할 것 같습니다.

### 1. 자식 엔티티 수가 적고 aggregate 경계가 중요하다

이 경우에는 `cascade + aggregate root` 방식이 더 좋다고 생각합니다.

- root가 생명주기를 통제
- 도메인 규칙이 한 곳에 모임
- 저장 흐름이 더 자연스럽게 읽힘

### 2. 자식 엔티티가 독립적이고 대량 저장이 중요하다

이 경우에는 batch insert를 진지하게 검토할 수 있습니다.

다만 이때는 꼭 같이 봐야 합니다.

- `hibernate.jdbc.batch_size`
- `order_inserts`
- flush 전략
- PK 생성 전략 (`IDENTITY`인지 아닌지)

### 3. 단순한 CRUD 편의성이 더 중요하다

이 경우에는 `saveAll()`도 충분히 좋은 선택입니다.

다만 `saveAll() = batch insert`라고 생각하면 안 됩니다.

### 4. `IDENTITY` 전략을 쓰고 있다

이 경우에는 대량 insert 성능 최적화 기대치를 조금 낮추는 것이 좋다고 생각합니다.  
batch insert를 정말 활용하고 싶다면 PK 전략 자체를 다시 볼 필요가 있습니다.

---

## 마무리

이번 고민을 하면서 느낀 건
JPA에서는 기능 하나하나를 따로 이해하는 것도 중요하지만,
실제로는 이들이 전부 연결되어 있다는 점이었습니다.

- `cascade`는 연관 엔티티 생명주기 전파 규칙이고
- `saveAll()`은 저장 API이며
- batch insert는 SQL 실행 최적화이고
- `IDENTITY`는 그 최적화 가능성에 영향을 주는 PK 전략입니다

그리고 여기에 DDD의 `Aggregate Root` 관점까지 들어오면
이건 단순 성능 문제가 아니라 **설계와 구현의 일관성 문제**가 됩니다.

이번 경우에는

- 자식 개수가 매우 적고
- root가 자식 생명주기를 강하게 통제해야 했고
- `IDENTITY` 전략 때문에 batch 이점도 크지 않았기 때문에

최종적으로는 **aggregate root를 통해 자식을 저장하는 방식**을 선택했습니다.

이번 경험을 통해 다시 정리하게 된 건 아래 한 줄입니다.

> **JPA 저장 전략은 성능만 보고 정할 문제가 아니라, 도메인 경계와 PK 전략까지 같이 보고 정해야 한다.**

---

## 참고

- Spring Data JPA 공식 문서
- Hibernate User Guide - batching
- JPA 연관관계 주인과 양방향 연관관계 관련 자료
