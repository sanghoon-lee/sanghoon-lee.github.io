---
layout: post
title: "[학습기록] Spring 생성자 주입을 사용하는 이유: @Autowired 필드 주입과 차이점"
date: 2026-01-12
description: Spring 의존성 주입(DI) 방식인 생성자 주입과 @Autowired 기반 필드 주입의 차이점을 비교하고, 생성자 주입을 권장하는 이유와 장점, 필드 주입 방식의 한계를 정리한 글입니다.
image: /assets/images/springboot.png
categories: 학습기록
tags:
  - 자바
  - 스프링
---

Spring에서는 객체 간 의존 관계를 관리하기 위해 **의존성 주입(Dependency Injection, DI)**을 사용합니다. 이때 생성자 주입과 `@Autowired`를 이용한 필드 주입 중 어떤 방식을 선택해야 하는지 고민하는 경우가 많습니다.

<img class="main-image" src="/assets/images/springboot.png" alt="스프링부트">

기존 코드를 유지보수하거나 새로운 기능을 개발하는 과정에서 의존성 주입 방식에 대한 충분한 이해 없이 간편하다는 이유로 `@Autowired` 기반 필드 주입 방식을 사용하는 경우도 있습니다.

최근에는 필드 주입보다는 생성자 주입 방식이 권장되고 있습니다. 하지만 단순히 "권장된다"는 이유만으로 사용하기보다 그 이유와 차이점을 이해하는 것이 중요합니다.

저 역시 의존성 주입 방식의 차이를 명확히 이해하지 못했던 부분이 있었고, 이번 기회에 생성자 주입이 권장되는 이유와 필드 주입 방식의 한계를 정리해보았습니다.

---

## 1. Spring에서 의존성 주입 방식이 필요한 이유

Java와 같은 객체 지향 프로그래밍에서는 하나의 객체가 다른 객체를 필요로 하는 경우가 많습니다. 

```java
public class SomeService {
    private SomeDependency someDependency = new SomeDependency();
}
```

`SomeService`는 `SomeDependency` 없이 정상적으로 동작할 수 없습니다. 이처럼 한 객체가 또 다른 객체를 필요로 하는 경우를 **의존성**이 있는 관계라고 설명할 수 있습니다. 

전통적인 방식에서는 객체가 필요한 의존 객체를 직접 생성했습니다. 하지만 구현체의 변경이 어렵고, 테스트의 제약이 있으며 객체 간 결합도가 높아질 수 있습니다. 만약 `SomeDependency` 구현체가 변경되거나 테스트를 위해 다른 객체로 교체해야 한다면 기존 코드를 수정해야 합니다.

결국, 의존 관계를 어떻게 관리하느냐에 따라 코드의 품질이 크게 달라질 수 밖에 없었습니다. 

하지만, Spring과 같은 프레임워크가 등장하면서 이러한 흐름은 크게 뒤집혔습니다. Spring에서는 객체 생성과 의존 관계 관리를 컨테이너가 담당합니다. 개발자는 필요한 객체를 직접 생성하기보다, 컨테이너가 관리하는 객체를 주입받아 사용하는 방식으로 의존 관계를 구성합니다. 이를 **의존성 주입**이라고 합니다.  

<mark>“개발자가 객체를 관리한다” → “프레임워크가 객체를 관리한다”</mark>

Spring에서 제공하는 대표적인 의존성 주입 방식은 다음과 같습니다.

* 생성자 주입(Constructor Injection)
* 필드 주입(Field Injection)
* 수정자 주입(Setter Injection)
* 일반 메소드 주입(Method Injection)

가장 많이 비교되는 방식이 생성자 주입과 `@Autowired`를 이용한 필드 주입 방식입니다. 두 방식은 의존성이 주입되는 시점과 객체 설계 방식에 큰 차이가 있습니다.

---

## 2. 생성자 주입 vs 필드 주입 비교

생성자 주입과 필드 주입 방식의 차이점은 다음과 같습니다.

| 특징    | 생성자 주입             | 필드 주입                                   |
| ----- | ------------------ | --------------------------------------- |
| 주입 시점 | 객체 생성 시 생성자를 통해 주입 | 객체 생성 이후 Spring 컨테이너가 Reflection을 통해 주입 |
| 의존성 관리 | 생성자를 통해 의존성이 명확하게 전달되고 초기화됨 | 객체 생성과 의존성 주입 시점이 분리됨 |
| 의존 관계 변경 |	`final` 키워드 사용으로 외부 객체 참조 변경 방지 | 의존 객체 변경 가능 |	
| 테스트 용이성	| 생성자 주입을 통해 의존성 주입이 명확하여 테스트가 용이 | 의존성 주입이 명확하지 않아서 테스트가 어려울 수 있음	 | 
| 의존성 누락 방지	| 필요한 의존성이 생성자에 명확히 표현되어 누락 여부 확인이 용이 | 코드 구조상 의존 관계 확인이 어려움	 | 

최근 팀에서 담당하고 있는 서비스의 코드를 정적분석 도구인 SonarQube로 분석하면서, 의존성 주입 방식에 대한 개선 필요성을 확인했습니다. 특히 생성자 주입과 필드 주입의 차이를 명확히 이해하는 것이 필요하다고 느꼈습니다.

---

## 3. 생성자 주입을 권장하는 이유

Spring에서는 생성자 주입 방식을 사용하는 것이 가장 권장됩니다. 그 이유는 객체 설계 관점에서 더 안전하고 명확한 구조를 만들 수 있기 때문입니다.

생성자 주입 방식의 주요 장점은 다음과 같습니다.

* 객체 생성 시점에 필수 의존성 보장
* 의존 관계의 불변성 유지 가능
* 객체 간 의존 관계를 명확하게 표현
* 테스트 코드 작성 용이

---

### 3.1. 객체 생성 시점에 필수 의존성 보장

객체가 정상적으로 동작하기 위해 반드시 필요한 객체를 **필수 의존성**이라고 표현합니다. 

```java
@Service
public class SomeService {
    private final SomeDependency someDependency;
    private final SomeRepository someRepository;

    public SomeService(
            SomeDependency someDependency, 
            SomeRepository someRepository) {

        this.someDependency = someDependency;
        this.someRepository = someRepository;
    }

    public void run(){
        someDependency.run();
        someRepository.save();
    }
}
```

위 코드에서 `SomeDependency`와 `SomeRepository`는 `SomeService`가 정상적으로 동작하기 위해 반드시 필요한 **필수 의존성**입니다.

생성자 주입 방식에서는 객체 생성 시점에 필요한 의존성을 반드시 전달받아야 합니다. 따라서 `SomeDependency`와 `SomeRepository`가 전달되지 않는다면 `SomeService` 객체 자체를 생성할 수 없습니다. 

즉, 생성자가 객체 생성 조건을 강제하기 때문에 의존성이 누락된 불완전한 객체가 만들어지는 것을 방지할 수 있습니다.

반면 필드 주입 방식에서는 객체 생성 이후 Spring 컨테이너가 Reflection을 이용해 의존성을 주입합니다. 따라서 객체 생성 시점과 의존성 주입 시점이 분리되고, 객체가 완전히 초기화되기 전까지는 필요한 의존성이 존재하는지 명확하게 알기 어렵습니다.

참고로 필드 주입 방식에서 사용하는 Reflection은 Java에서 실행 중에 클래스 정보를 조회하고 객체의 필드에 접근할 수 있도록 제공하는 기능입니다. Spring은 Reflection을 활용해 런타임에 Bean의 메타데이터를 확인하고, 개발자가 직접 접근하기 어려운 private 필드에도 의존 객체를 주입할 수 있습니다.

---

### 3.2. 의존 관계의 불변성 유지 가능

생성자 주입 방식의 또 다른 장점은 의존성을 `final` 키워드로 선언하여 의존 관계를 안정적으로 관리할 수 있다는 점입니다. 여기서 말하는 불변성은 객체 내부의 모든 상태가 변경되지 않는다는 의미가 아니라, 객체가 참조하는 의존 관계가 생성 이후 변경되지 않는다는 의미입니다.

```java
@Service
public class SomeService {
    private final SomeDependency someDependency;
    
    public SomeService(SomeDependency someDependency) {
        this.someDependency = someDependency;
    }
}
```

`someDependency`는 `final`로 선언되어 있기 때문에 객체 생성 이후 다른 객체를 참조하도록 변경할 수 없습니다. 어떤 의존 객체를 사용하는지가 객체 생성 이후 변경되지 않기 때문에, 객체의 동작 흐름을 예측하기 쉬워집니다.

따라서 생성자 주입 방식은 객체가 필요로 하는 의존성을 생성 시점에 고정하고, 의존 관계가 변경되는 것을 방지하여 안정적인 객체 설계를 가능하게 합니다.

반면 일반적인 필드 주입 방식에서는 생성자에서 값을 초기화해야 하는 `final` 필드와 함께 사용할 수 없습니다.

```java
@Service
public class SomeService {

    @Autowired
    private SomeDependency someDependency;

}
```

`final` 필드는 객체 생성 과정에서 반드시 초기화되어야 하는데, 필드 주입은 객체 생성 이후 Spring 컨테이너가 값을 주입하는 방식이기 때문입니다.

---

### 3.3. 객체 간 의존 관계를 명확하게 표현

생성자 주입 방식의 또 다른 장점은 객체가 필요로 하는 의존 관계를 코드 수준에서 명확하게 표현할 수 있다는 점입니다.

생성자는 객체를 생성하기 위해 필요한 조건을 정의하는 역할을 합니다. 따라서 생성자에 선언된 의존성은 해당 객체가 정상적으로 동작하기 위해 반드시 필요한 요구사항이라고 볼 수 있습니다.

```java
@Service
public class SomeService {
    private final SomeDependency someDependency;
    private final SomeRepository someRepository;

    public SomeService(
            SomeDependency someDependency, 
            SomeRepository someRepository) {
        
        this.someDependency = someDependency;
        this.someRepository = someRepository;
    }
}
```

위 코드에서 `SomeService` 생성자를 통해 `SomeDependency`와 `SomeRepository`를 전달받고 있습니다. 즉, 코드를 확인하는 개발자는 `SomeService`가 두 객체를 필요로 한다는 사실을 바로 알 수 있습니다. 결과적으로 생성자 주입 방식은 단순히 의존성을 전달하는 방법을 넘어, 객체가 어떤 협력 객체를 필요로 하는지 명확하게 표현하는 설계 방식입니다.

반면 필드 주입 방식에서는 의존성이 클래스 내부 필드에 선언되어 있기 때문에, 객체 생성에 필요한 조건이 생성자에 명확하게 드러나지 않습니다.

```java
@Service
public class SomeService {
    
    @Autowired
    private SomeDependency someDependency;
    
    @Autowired
    private SomeRepository someRepository;

}
```

필드 주입 방식도 클래스 내부를 확인하면 의존 관계 자체는 파악할 수 있습니다. 하지만 생성자를 통해 객체 생성 조건을 명확하게 표현하지 않기 때문에, 해당 객체가 반드시 필요로 하는 의존성이 무엇인지 코드 구조만으로 드러내기 어렵습니다.

---

### 3.4. 테스트 코드 작성 용이

생성자 주입 방식은 테스트 코드 작성 측면에서도 장점을 제공합니다.

객체가 필요로 하는 의존성을 생성자를 통해 전달받기 때문에, 테스트 환경에서는 Mockito와 같은 프레임워크를 사용하면 실제 구현체 대신 테스트용 객체를 생성하여 의존성을 대체할 수 있습니다. Spring Context를 실행하지 않고도 필요한 의존 객체를 직접 생성하여 주입할 수 있기 때문에 단위 테스트를 간단하게 작성할 수 있습니다.

```java
@Test
void processTest() {
    SomeDependency dependency = mock(SomeDependency.class);
    SomeRepository repository = mock(SomeRepository.class);

    SomeService service = new SomeService(dependency, repository);

    service.run();
}
```

반면 필드 주입 방식에서는 객체 생성 이후 Spring 컨테이너가 Reflection을 통해 의존성을 주입해야 하기 때문에, 단위 테스트를 작성할 때 Spring Context가 필요한 경우가 많습니다.

---

## 4. 필드 주입을 사용할 수 있는 예외적인 상황

그렇지만 항상 필드 주입을 선택하는 것이 나쁜 것은 아닙니다. 중요한 것은 언제 사용할 수 있는지를 정확히 아는 것입니다.  

---

### 4.1. 선택적(Optional) 의존성

모든 의존성이 반드시 필요한 것은 아닙니다. 선택적 의존성은 생성자 주입과 Optional 등의 방식을 통해 처리할 수도 있지만, 간단한 조건부 의존성 처리 목적이라면 필드 주입 방식이 활용될 수 있습니다.

* 특정 환경에서만 존재하는 빈(Bean)
* 객체가 없어도 기본 기능은 동작해야 하는 경우
* 플러그인 성격의 컴포넌트

```java
@Autowired(required = false)
private SomeOptionalDependency optionalDependency;
```

위의 코드에서 `@Autowired`의 `required` 속성의 값을 `false`로 지정해서 선택적 의존성을 가진 객체의 의존성을 주입하도록 하고 있습니다. 모든 의존성이 선택적이고 객체 생성 시점에 반드시 필요하지 않은 경우에는 필드 주입 방식이 단순한 구현 방법이 될 수 있습니다. 다만 일반적인 비즈니스 로직에서는 생성자 주입을 우선 고려하는 것이 좋습니다.

---

### 4.2. 테스트 클래스

테스트 클래스에서는 Spring Context를 활용해 테스트 환경을 구성하는 경우가 많기 때문에 필드 주입 방식을 사용하는 경우가 있습니다.

```java
@SpringBootTest
class SomeServiceTest {

    @Autowired
    private SomeService someService;
}
```

애플리케이션 운영 코드가 아니기 때문에, 테스트 대상 Bean을 주입받기 위한 목적으로 필드 주입을 사용하는 것은 일반적으로 허용됩니다.

다만, 테스트 대상 클래스인 `SomeService`는 생성자 주입 방식을 유지하는 것이 좋습니다.

---

## 5. Lombok @RequiredArgsConstructor를 활용한 생성자 주입

Spring에서는 생성자 주입을 편리하게 작성하기 위해 Lombok의 `@RequiredArgsConstructor`를 많이 사용합니다.

```java
@Service
@RequiredArgsConstructor
public class SomeService {

    private final SomeDependency someDependency;
    private final SomeRepository someRepository;

}
```

`@RequiredArgsConstructor`는 `final` 필드를 기반으로 생성자를 자동 생성하기 때문에, 생성자 주입의 장점은 유지하면서 코드 작성량을 줄일 수 있습니다. 최근 Spring 프로젝트에서는 생성자를 직접 작성하기보다 Lombok의 `@RequiredArgsConstructor`를 활용해 생성자 주입 방식을 간결하게 구현하는 경우가 많습니다.

---

## 6. 마무리

이번 포스팅을 작성하면서 의존성 주입은 단순히 객체를 편리하게 가져오는 기능이 아니라, 객체 간 관계를 어떻게 설계할 것인지에 대한 중요한 개발 방식이라는 점을 다시 확인할 수 있었습니다.

필드 주입 방식은 간단하게 사용할 수 있다는 장점이 있지만, 객체 생성 과정과 의존 관계가 분리되어 코드 수준에서 객체의 요구사항을 명확하게 표현하기 어렵습니다. 하지만 생성자 주입은 객체가 필요로 하는 의존성을 명확하게 표현하고 안정적인 객체 설계를 가능하게 하는 방식입니다. 

따라서 특별한 이유가 없다면 Spring 애플리케이션에서는 생성자 주입 방식을 기본 전략으로 사용하는 것이 바람직합니다.
