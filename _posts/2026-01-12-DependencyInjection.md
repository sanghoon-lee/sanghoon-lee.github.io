---
layout: post
title: "[학습기록] 스프링에서 의존성 주입(Dependency Injection) 방식 비교: 생성자 주입 vs 필드 초기화"
date: 2026-01-12
categories: 학습기록
---

<style>
.main-image {
  width: 50%;
  max-width: 380px;
  height: auto;
  margin-bottom: 1.4rem;
  border-radius: 12px;
  display: block;
}
</style>

스프링에서는 의존성 주입(Dependency Injection, DI)을 통해 객체 간의 결합도를 낮추고, 코드의 재사용성을 높이며, 테스트 용이성을 증가시킬 수 있습니다. 이 글에서는 **생성자 주입(Constructor Injection)**과 **필드 초기화(Field Initialization)** 방식의 차이점과, 올바른 의존성 주입 방식을 사용하는 이유에 대해 살펴보겠습니다.

<img class="main-image" src="/assets/images/springboot.png" alt="스프링부트">

### 1. 필드 초기화 방식 (문제 코드)

먼저 필드 초기화 방식을 살펴보겠습니다.

```java
public class SomeService {
    @Autowired
    private SomeDependency someDependency;

    private String name = someDependency.getName();  // 의존성 주입 전에 사용됨
}
```

이 코드는 문제가 될 수 있는 코드입니다. 그 이유는 name 필드를 초기화할 때 someDependency가 아직 주입되지 않았을 수 있기 때문입니다. @Autowired 애너테이션은 Spring의 의존성 주입을 사용하여 객체가 생성된 후에 의존성을 주입합니다. 그러나 위 코드에서는 name을 필드 선언 시점에 초기화하려고 하므로, someDependency가 null일 가능성이 존재하며, 이로 인해 **NullPointerException**이 발생할 수 있습니다.

문제점:

필드 초기화 시점에 의존성이 주입되기 전에 getName()을 호출하려고 시도하므로, NullPointerException이 발생할 수 있습니다.

의존성 주입의 순서가 불확실하여 코드의 안정성에 문제가 생길 수 있습니다.

### 2. 생성자 주입 방식 (개선된 코드)

다음으로 생성자 주입 방식을 사용한 개선된 코드를 살펴보겠습니다.

```java
public class SomeService {
    private final SomeDependency someDependency;
    private final String name;

    @Autowired
    public SomeService(SomeDependency someDependency) {
        this.someDependency = someDependency;
        this.name = someDependency.getName();  // 생성자에서 의존성 주입 후 초기화
    }
}
```

위 코드는 **생성자 주입(Constructor Injection)**을 사용하여 의존성을 주입하는 방식입니다. 이 코드에서 someDependency는 생성자를 통해 주입되며, name 변수는 생성자 내부에서 의존성 주입이 완료된 후 초기화됩니다.

장점:

안전한 의존성 주입: 생성자에서 의존성 주입이 이루어지고, 그 후에 name 변수를 초기화하므로 NullPointerException을 방지할 수 있습니다.

불변성 보장: final 키워드를 사용하여 someDependency와 name을 초기화 후 변경할 수 없게 만듭니다. 이로 인해 객체가 **불변성(Immutable)**을 가지게 되어, 상태가 변경되지 않음을 보장할 수 있습니다.

테스트 용이성: 생성자 주입은 의존성 주입이 명확하게 드러나므로 테스트 코드에서 필요한 의존성을 모킹(Mocking) 하거나 주입하기 용이합니다.

컴파일 시점 의존성 검증: 생성자를 통한 의존성 주입은 컴파일 시점에 의존성이 누락되었는지 바로 확인할 수 있어 런타임 오류를 줄일 수 있습니다.

문제 해결:

의존성 주입 후 변수 초기화: 생성자에서 의존성 주입이 이루어진 후, 그 값을 이용하여 변수 name을 초기화합니다. 이 방식은 의존성이 안전하게 주입된 후에 값을 사용하므로 안정적입니다.

### 3. 생성자 주입 vs 필드 초기화 방식

| 특징 |	필드 초기화 방식 (@Autowired 필드 주입)	 | 생성자 주입 (Constructor Injection) |
| --- | --- | --- |
| 주입 시점	| 객체 생성 후, 필드 초기화 시점에 의존성 주입됨 |	객체 생성 시, 생성자에서 의존성 주입됨 |
| 의존성 관리 |	의존성 주입 전에 변수 사용 시 예외 발생 가능 |	의존성이 명확하게 주입되고, 생성자에서 초기화됨 |
| 불변성 |	불변성 보장되지 않음 |	final 키워드를 사용하여 불변성 보장 |
| 테스트 용이성	| 의존성 주입이 명확하지 않아서 테스트가 어려울 수 있음	 | 생성자 주입을 통해 의존성 주입이 명확하여 테스트가 용이 |
| 컴파일 타임 검증	| 의존성 주입 순서나 누락에 대한 검증이 어려움	 | 컴파일 시점에 의존성 누락 문제를 발견할 수 있음 |

### 4. 결론

생성자 주입 방식은 의존성 주입 후 변수 초기화를 보장하고, 불변성을 유지하며, 컴파일 타임에 의존성 문제를 해결할 수 있는 강력한 방식입니다. 반면, 필드 초기화 방식은 의존성이 주입되기 전에 필드를 사용할 가능성이 있어 예기치 않은 오류가 발생할 수 있습니다.

따라서 Spring에서 의존성 주입을 사용할 때는 생성자 주입 방식을 사용하는 것이 바람직합니다. 이 방식은 의존성 관리와 객체의 안정성, 테스트 용이성을 높이는 데 유리합니다. @Autowired를 생성자에 사용하여 의존성을 명확히 주입받고, 객체의 상태를 불변으로 유지하는 방식으로 개발하는 것을 권장합니다.

이 글이 Spring 의존성 주입 방식을 이해하는 데 도움이 되었기를 바랍니다! :)
