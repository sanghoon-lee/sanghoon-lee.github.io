---
layout: post
title: "[학습기록] 스프링에서 의존성 주입 방식 비교 : 생성자 주입 vs 필드 주입"
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

스프링에서 의존관계를 자동으로 설정해주는 **의존성 주입(Dependency Injection, DI)**은 매우 중요한 기능으로 객체의 생성과 사용 책임을 분리하는 설계 원칙이기도 합니다.
하지만, 이에 대해서 제대로 공부하기보다는 그냥 "@Autowired를 붙이는 기술” 정도로만 알고 사용하는 개발자들도 의외로 많이 있는 것 같습니다. 

<img class="main-image" src="/assets/images/springboot.png" alt="스프링부트">

### 1. 의존성 주입의 개념

본격적으로 시작하기에 전에 **의존성(Dependency)**이 무엇인지 개념부터 정리해보겠습니다.

```java
public class SomeService {
    private SomeDependency someDependency = new SomeDependency();
}
```

위의 코드에서 SomeService는 SomeDependency 없이는 정상적으로 동작할 수 없습니다. 이처럼 한 객체가 또 다른 객체를 필요로 하는 경우, 의존성이 있는 관계라고 설명할 수 있습니다.

그런데 객체가 의존성이 있는 객체를 직접 생성하고 관리하는 것은 결코 쉬운 일이 아닙니다. 다음과 같은 문제가 발생하기 때문입니다.

1️⃣ 객체 간 결합도 상승

객체를 실질적으로 구현하고 있는 구현체가 코드에 직접 지정되어야 합니다. 그렇기 때문에, 코드를 수정하지 않으면 다른 구현체로 변경할 수가 없습니다. 즉, 변경에 취약한 구조가 되는 것입니다.

2️⃣ 테스트 용이성 감소

테스트를 위한 가상의 Mocking 객체 주입이 사실상 불가능합니다. 외부 시스템(DB, API 등)과 연동해서 동작하는 경우에 가상의 객체를 사용할 수 없어 단위 테스트보다는 통합 테스트 위주로 진행될 수 밖에 없습니다.

3️⃣ 객체 생성의 책임이 분산

객체를 생성하는 기능이 분산되면서 자원 관리(생명주기, 설정 변경)의 통합이 어려워지게 됩니다.

4️⃣ 시스템 안정성 저하

의존성 관계에 있는 객체가 언제, 어떤 상태로 생성되는지 보장하기 어렵습니다. 이는 런타임 오류(NullPointerException 등)의 발생 가능성 증가로 이어지게 됩니다.

이러한 문제를 해결할 수 있도록 스프링에서는 컨테이너가 필요한 객체를 생성햇서 전달하는 **의존성 주입(Dependency Injection)** 기능을 제공하고 있습니다. 따라서, 객체에서 의존성이 있는 객체를 직접 생성하고 관리할 필요가 없습니다. 

### 2. 의존성 주입 방법 : 생성자 주입 vs 필드 주입

스프링에서는 제공되는 의존성 주입 방법은 다음과 같습니다.

* **생성자 주입** 
* **필드 주입** 
* **수정자(setter) 주입**
* **일반 메소드 주입** 

저희 팀에서는 주로 **생성자 주입**과 **필드 주입**을 사용하고 있습니다. 최근 팀에서 관리하고 있는 서비스의 소스코드를 정적분석 도구인 Sonarqube로 분석했는데, 의존성 주입 방법을 부적절하게 사용하고 있는 사례가 다수 확인되었습니다. 대부분 **생성자 주입**과 **필드 주입**의 차이점을 제대로 구분하지 못해서 발생한 것이었습니다.

**생성자 주입과 필드 주입 방법의 차이점**
| 특징 | 생성자 주입 | 필드 주입 |
| --- | --- | --- |
| 주입 시점	| 	객체 생성 시, 생성자에서 의존성 주입됨 | 객체 생성 후, 필드 초기화 시점에 의존성 주입됨 |
| 의존성 관리 |	의존성이 명확하게 주입되고, 생성자에서 초기화됨 | 의존성 주입 전에 변수 사용 시 예외 발생 가능 |
| 불변성 |	final 키워드를 사용하여 불변성 보장 | 불변성 보장되지 않음 |	
| 테스트 용이성	| 생성자 주입을 통해 의존성 주입이 명확하여 테스트가 용이 | 의존성 주입이 명확하지 않아서 테스트가 어려울 수 있음	 | 
| 컴파일 타임 검증	| 컴파일 시점에 의존성 누락 문제를 발견할 수 있음 | 의존성 주입 순서나 누락에 대한 검증이 어려움	 | 

### 3. 필드 주입보다는 생성자 주입이 추천되는 이유

필드 주입 방법을 사용하는 것은 간단합니다. 전역변수를 @Autowired 어노테이션만 붙이서 선언하면 자동으로 의존성이 주입되기 때문입니다. 하지만, 일반적인 상황에서는 필드 주입보다 생성자 주입을 사용하는 것이 더 안정적인 방법으로 알려져있습니다.  

**잘못된 코드**

```java
public class SomeService {
    @Autowired
    private SomeDependency someDependency;

    private String name = someDependency.getName();  // 의존성 주입 전에 사용됨
}
```

위의 코드처럼 필드 주입으로 전역변수 someDependency를 선언하면, 관련된 객체가 생성된 후에 의존성이 주입됩니다. 단, 의존성 주입 순서는 보장되지 않습니다. 그래서 전역변수 name을 선언하는 시점에는 someDependency의 의존성 주입 여부가 불확실합니다. 의존성이 주입되지 못한 상태에서 getName() 메소드가 호출되면서 **NullPointerException**이 발생할 수 있는 가능성이 얼마든지 존재하는 것입니다.

이러한 문제의 발생은 생성자 주입 방식을 사용해서 방지할 수 있습니다.

**수정된 코드**

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

someDependency는 생성자에서 의존성 주입이 이루어지고, 그 후에 getName()메소가 호출되면서 name 변수가 초기화됩니다. 의존성 주입이 완료된 상태이기 때문에, NullPointerException은 발생하지 않습니다. 추가적으로 final 키워드를 사용하여 someDependency와 name을 초기화 후 변경할 수 없게 만듭니다. 이로 인해, 객체가 **불변성(Immutable)**을 가지게 되면서 상태가 변경되지 않음을 보장할 수 있습니다.

생성자 주입 방법을 선택하면, 의존성 주입 절차가 명확하게 드러나는 장점이 있습니다. 그래서 테스트 코드를 작성하는 경우에 필요한 가상의 Mocking 객체를 주입하는 것도 용이합니다.

### 4. 결론

생성자 주입은 의존성 주입이 완료된 상태를 보장하고, 객체의 불변성을 유지하면서 컴파일 타임에 의존성에 문제가 있는지 검증할 수 있는 강력한 방식입니다. 반면, 필드 주입은 의존성이 주입되기 전에 사용될 가능성이 있어 예기치 않은 오류가 발생할 수 있습니다.

일반적인 경우에는 생성자 주입으로 의존성을 주입하는 것이 바람직합니다. 

✔ 기본 원칙
* 비즈니스 로직을 가진 핵심 클래스 → 생성자 주입
* 초기화 순서가 중요 → 생성자 주입
* 불변 객체 → 생성자 주입

그렇지만, 모든 상황에서 필드 주입의 사용이 나쁜 선택이라는 것은 아닙니다. 필드 주입의 사용이 가능한 경우도 얼마든지 있습니다. 중요한 것은 언제 사용할 수 있는지를 정확히 아는 것입니다.

1️⃣ 초기화 시점과 무관한 의존성인 경우

생성자나 필드 초기화 시점에서는 전혀 사용되지 않으면서, 메서드 내부에서만 사용되는 경우에는 필드 주입은 비교적 안전합니다.

```java
@Service
public class SomeService {

    @Autowired
    private SomeDependency someDependency;

    public void process() {
        someDependency.doSomething();
    }
}
```
2️⃣ 선택적(Optional) 의존성인 경우

모든 의존성이 반드시 필요한 것은 아닙니다. 아래의 경우라면,선택적 의존성에 해당됩니다.

* 특정 환경에서만 존재하는 빈(Bean)
* 객체가 없어도 기본 기능은 동작해야 하는 경우
* 플러그인 성격의 컴포넌트

```java
@Autowired(required = false)
private SomeOptionalDependency optionalDependency;
```

위의 코드에서 @Autowired의 required 속성의 값을 false로 지정해서 선택적 의존성을 가진 객체의 의존성을 주입하도록 하고 있습니다. 생성자 주입을 사용해도 되지만, 필수 의존성과 선택 의존성이 섞여 코드의 가독성이 저하되기 떄문에 권장되지는 않습니다.

3️⃣ 테스트 코드(@Test)에서의 주입

테스트 코드에서는 가독성과 생산성이 더 중요해지는 경우가 많습니다. 따라서, 필드 주입을 사용해도 무방합니다.

```java
@SpringBootTest
class SomeServiceTest {

    @Autowired
    private SomeService someService;
}
```

4️⃣ DTO, 설정 클래스, 단순 어댑터 성격의 클래스

비즈니스 로직이 거의 없으면서, 생명주기 관리가 단순한 경우에 생성자 주입을 사용하면 코드가 과도하게 장황해질 수 있습니다.

```java
@Component
public class SomeAdapter {

    @Autowired
    private ExternalClient client;
}
```
