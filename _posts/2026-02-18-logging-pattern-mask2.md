---
layout: post
title: "[토이 프로젝트] 스프링 애플리케이션의 로그에서 민감 정보 마스킹: (2) 구현 상세"
date: 2026-02-18
description: 스프링 애플리케이션의 로그에 포함된 민감정보를 암호화하는 토이 프로젝트를 수행하는 과정을 정리한 글입니다. 2/3
image: /assets/images/security.png
categories: 토이프로젝트
---

지난 포스팅에서 출력 문자열을 가공하는 방식으로 로그 마스킹을 구현하기로 결정한 이유에 대해서 기술했습니다. 

이제부터는 이 방식의 구현을 위해 어떻게 Logback의 PatternLayout을 확장했는지 코드를 중심으로 살펴보도록 하겠습니다.

<img class="main-image" src="/assets/images/logback.png" alt="LOGBACK">

---

## 1. Logback의 로그 출력 과정

Logback에서 로그 출력을 처리하는 과정은 다음과 같습니다.

1. 애플리케이션 코드에서 log.info() 메서드 호출
2. ILoggingEvent 생성
3. PatternLayout이 이벤트 내용을 문자열로 변환
4. 변환된 문자열이 Appender를 통해 출력

로그 마스킹을 구현하기 위해서 개입한 지점은 3번과 4번 사이입니다. 로그가 출력되기 직전에 민감 정보를 찾아서 마스킹하는 것이 핵심이기 때문입니다.

---

## 2. MaskingPatternLayout 클래스의 구현: PatternLayout 클래스의 확장

그래서 PatternLayout을 상속받은 **MaskingPatternLayout** 클래스에서 이벤트 내용을 문자열로 변환할 때 호출되는 **doLayout()** 메소드를 오버라이드했습니다. 

**MaskingPatternLayout.java**
```java
public class MaskingPatternLayout extends PatternLayout {
    
    ...
    
    private volatile SensitiveStringSanitizer sanitizer;
    
    ...
    
    @Override
    public String doLayout(ILoggingEvent event) {
        String out = super.doLayout(event);
        if (!enabled)
            return out;

        SensitiveStringSanitizer s = sanitizer;
        // start() 호출 전에 doLayout 메서드가 호출되는 경우에는 원본 로그 그대로 출력(방어 로직 실행)
        if (s == null) 
            return out; 

        return s.sanitize(out);
    }
    
    ...
    
}
```

**super.doLayout(event)** 메서드를 호출해서 로그에 출력할 문자열을 생성하고, SensitiveStringSanitizer 객체의 **sanitize()** 메서드를 통해서 민감 정보를 마스킹합니다. 

이렇게하면 애플리케이션 로직을 수정하지 않고도, 로그 출력 레벨에서 일괄적으로 민감 정보를 통제할 수 있습니다.

개발자의 주의에 의존하지 않고, 구조적으로 문제를 해결하는 방식입니다. 민감 정보의 통제 지점을 애플리케이션 로직이 아닌 로그 출력 계층으로 옮긴 셈입니다.

---

## 3. 주의: Logback의 초기화

Logback은 애플리케이션이 시작될 때 `logback.xml`을 읽고 Layout, Appender, MaskingPatternLayout 등 필요한 객체를 생성합니다. 

일반적으로 Logback의 초기화 절차는 다음과 같습니다.
1. Layout 객체 생성(토이 프로젝트에서는 PatternLayout을 상속받은 MaskingPatternLayout 객체)
2. 설정 값 주입 (rule, enabled 등)
3. Layout 객체의 **start()** 메서드 호출
4. 로그 출력 시작

sanitizer에는 **start()** 메서드 내부에서 생성된 DefaultSensitiveStringSanitizer 객체가 할당됩니다.

**MaskingPatternLayout.java**
```java
public class MaskingPatternLayout extends PatternLayout {
    
    ...
    
    @Override
    public void start() {
        List<DefaultSensitiveStringSanitizer.Rule> rules = new ArrayList<>();
        for (String spec : ruleSpecs) {
            rules.add(RuleSpecParser.parse(spec));
        }

        this.sanitizer = new DefaultSensitiveStringSanitizer(rules, parseTriggers(triggers));
        super.start();
    }
    
    ...
    
}
```

일반적으로는 **start()** 메서드 호출 이후에 로그가 출력됩니다. 

하지만, 초기화 과정 중 예외적인 상황을 대비하여 sanitizer가 null인 경우에는 원본 로그가 그대로 출력하도록 **doLayout()** 메서드에 방어 로직을 추가했습니다.

```java
SensitiveStringSanitizer s = sanitizer;
// start() 호출 전에 doLayout 메서드가 호출되는 경우에는 원본 로그 그대로 출력(방어 로직 실행)
if (s == null) 
    return out; 
```
민감 정보의 마스킹은 보조 기능이지만, 로그 출력 자체는 실패해서는 안 된다고 판단했기 때문입니다. 로그 시스템은 항상 안정적으로 동작해야 합니다.

---

## 4. Trigger 기반 사전 필터링

정규식으로 로그 문자열에서 민감 정보에 해당되는 패턴을 찾아서 마스킹하는 방식은 구조적으로 단순합니다. 

> **"하지만, 모든 로그 출력에 대해서 매번 정규식 검사를 수행해도 괜찮을까요?"** 

INFO 레벨의 로그 출력이 많은 서비스라면 문자열 치환이 누적되면서 성능에 부담이 될 수 있습니다. 그래서 trigger 기반 사전 필터링 기능을 추가했습니다.

예를 들어, 다음과 같이 전화번호를 마스킹하는 정규식이 있다고 가정해보겠습니다.

```
(\b01[016789]-?)\d{3,4}(-?\d{4}\b)
```

이 정규식은 비교적 복잡합니다. 그런데 사실 대부분의 로그에는 전화번호가 포함되어 있지 않습니다. 그래서 매번 정규식 검사를 수행할 필요는 없습니다. 

대신 로그 문자열에 특정 키워드가 포함되어 있는 경우에는 정규식 검사가 수행되도록 했습니다. 

이 키워드는 정확한 검증을 위한 조건이 아니라, 정규식 수행 여부를 빠르게 판단하기 위한 1차 필터입니다.

키워드는 `logback.xml` 파일에 <triggers> 태그에 정의하면 되고, 콤마(,)로 단어가 구분됩니다. 

**logback.xml에 키워드 정의 예시**
```xml
<triggers>010-,@,Bearer </triggers>
```
* 010- → 전화번호 가능성
* @ → 이메일 가능성
* Bearer → 토큰 가능성

지금까지 설명한 Trigger 기반 사전 필터링의 동작 절차는 다음과 같습니다.

1. 로그 문자열 입력
2. Trigger 키워드 확인
3. 키워드가 포함되어 있으면 정규식 검사를 수행, 그렇지 않으면 생략

---

## 5. DefaultSensitiveStringSanitizer 클래스의 구현

Trigger 기반 사전 필터링 기능은 DefaultSensitiveStringSanitizer의 **maybeSensitive()**메서드에 구현했습니다.

**DefaultSensitiveStringSanitizer.java**
```java
public final class DefaultSensitiveStringSanitizer implements SensitiveStringSanitizer{
    
    ...
    
    private boolean maybeSensitive(String input) {
        if (triggers.isEmpty()) {
            return true;
        }

        for (String trigger : triggers) {
            if (input.contains(trigger)) {
                return true;
            }
        }
        return false;
    }
    
    ...
    
}
```
로그 문자열은 DefaultSensitiveStringSanitizer의 **sanitize()** 메서드로 입력됩니다. 여기에서 **maybeSensitive()**가 호출되면서 Trigger 기반 사전 필터링 기능이 작동하게 됩니다. 

Trigger 키워드가 포함되어 있는 경우에만 Rule에 정의된 정규식이 적용됩니다.

**DefaultSensitiveStringSanitizer.java**
```java
public final class DefaultSensitiveStringSanitizer implements SensitiveStringSanitizer{
    
    ...
    
    @Override
    public String sanitize(String input) {
        if (input == null || input.isEmpty())
            return input;

        // triggers가 있으면 "민감 가능성" 빠른 체크로 대부분 로그는 정규식 검사&마스킹 처리를 생략
        if (!maybeSensitive(input))
            return input;

        String out = input;
        for (Rule r : rules) {
            out = r.apply(out);
        }
        return out;
    }

    ...

}
```

**요약**
* Trigger → 빠른 1차 필터
* 정규식 검사 → 정확한 2차 판별

토이 프로젝트에서는 구조를 단순하게 하기 위해서 Trigger 키워드와 하나라도 매칭되면 모든 Rule이 적용되도록 구현했습니다.

---

## 6. 마스킹 규칙(Rule) 정의

마스킹 규칙은 `logback.xml` 파일의 **<rule>** 태그에 정의하면 됩니다. 

**logback.xml 파일 예시**
```xml
<layout class="sanghoon.study.logging.mask.logback.MaskingPatternLayout">

    <enabled>true</enabled>

    <triggers>010-,@</triggers>

    <rule>phone|(\\b01[016789]-?)\\d{3,4}(-?\\d{4}\\b)|$1****$2</rule>
    <rule>email|([A-Za-z0-9._%+-]{2})[A-Za-z0-9._%+-]*(@[A-Za-z0-9.-]+\\.[A-Za-z]{2,})|$1****$2</rule>

</layout>
```

각 **<rule>** 태그는 다음의 형식을 따릅니다.

```
name | regex | replacement
```

* name → 규칙 식별자
* regex → 정규식 패턴
* replacement → 치환 문자열

---

## 7. RuleSpecParser 클래스의 구현: Rule 객체 생성

MaskingPatternLayout의 **start()** 메서드 내부에서 `logback.xml` 파일의 **<rule>** 태그에 정의된 마스킹 규칙을 가져오기 위해서 RuleSpecParser 객체가 사용됩니다.

```java
List<DefaultSensitiveStringSanitizer.Rule> rules = new ArrayList<>();
for (String spec : ruleSpecs) {
    rules.add(RuleSpecParser.parse(spec));
}
```

**ruleSpecs**는 `logback.xml` 파일의 <rule> 태그에 정의된 마스킹 규칙에 해당되는 문자열을 나타냅니다. 

이 문자열을 name,regex,replacement로 분류하는 코드는 `RuleSpecParser`의 **parse()** 메서드에 구현했습니다.

**RuleSpecParser.java**
```java
public final class RuleSpecParser {
    
    ...

    public static DefaultSensitiveStringSanitizer.Rule parse(String spec) {
        
        ...
    
        String[] parts = spec.split("\\|", 3);
        if (parts.length != 3) {
            throw new IllegalArgumentException("invalid rule spec. expected name|regex|replacement. got: " + spec);
        }

        String name = parts[0];
        String regex = parts[1];
        String replacement = parts[2];

        ...

        return new DefaultSensitiveStringSanitizer.Rule(name, Pattern.compile(regex), replacement);
    }

    ...

}
```

이렇게 생성된 Rule 객체는 Pattern으로 컴파일되어 DefaultSensitiveStringSanitizer 객체에 전달됩니다.

---

## 8. 전체 흐름 정리

<img class="main-image" src="/assets/images/security.png" alt="민감 정보">

지금까지 살펴본 코드의 전체 흐름을 한 번 정리해보겠습니다.

1. `logback.xml` 파일에 정의된 마스킹 규칙(Rule)과 Trigger 키워드를 조회
2. Logback 초기화 과정에서 **MaskingPatternLayout.start()** 메서드 호출
3. **RuleSpecParser.parse()** 메서드가 호출되면서 객체 마스킹 규칙 문자열이 Rule 객체로 변환
4. DefaultSensitiveStringSanitizer 객체 생성
5. 로그 출력 시 **MaskingPatternLayout.doLayout()** 메서드 호출
6. Trigger 1차 필터링
7. 정규식 기반 마스킹 수행
8. 최종 문자열 출력

구조는 단순하지만, 설정과 코드가 명확하게 분리되어 있습니다.

코드를 수정하지 않고도 `logback.xml` 파일의 내용만 변경하면 마스킹 정책을 바꿀 수 있다는 점이 장점입니다.

---

## 9. 포스팅 시리즈

* [[토이 프로젝트] 스프링 애플리케이션의 로그에서 민감 정보 마스킹: (1) 순수 문자열 패턴 기반으로 설계](https://sanghoon-lee.github.io/2026/02/16/logging-pattern-mask/)
* [토이 프로젝트] 스프링 애플리케이션의 로그에서 민감 정보 마스킹: (2) 구현 상세
* [[토이 프로젝트] 스프링 애플리케이션의 로그에서 민감 정보 마스킹: (3) 테스트 방법 및 로그 마스킹 구현 방식의 한계](https://sanghoon-lee.github.io/2026/02/20/logging-pattern-mask3/)

---

## 10. 소스 코드 
* [logging-pattern-mask](https://github.com/sanghoon-lee/logging-pattern-mask)

---

#민감정보 #로그 #마스킹 #문자열 #Logback #PatternLayout #토이프로젝트 #필터링 
