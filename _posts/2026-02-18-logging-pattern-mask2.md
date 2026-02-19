---
layout: post
title: "[토이 프로젝트] 스프링 애플리케이션의 로그에서 민감 정보 마스킹: (2) 구현 상세"
date: 2026-02-18
categories: 토이프로젝트
---

지난 포스팅에서 Logback에서 PatternLayout을 확장하여 출력 문자열을 가공하는 방식을 선택하게 된 과정을 설명했습니다. 

Logback에서 로그 출력을 처리하는 과정은 다음과 같습니다.

1. 애플리케이션 코드에서 log.info() 메서드 호출
2. ILoggingEvent 생성
3. PatternLayout이 이벤트 내용을 문자열로 변환
4. 변환된 문자열이 Appender를 통해 출력

토이 프로젝트에서 로그 마스킹 기능을 구현하기 위해서 개입한 지점은 3번과 4번 사이입니다. 로그가 출력되기 직전에 민감 정보를 찾아서 마스킹하도록 구현하는 것이 핵심이기 때문입니다.

## PatternLayout 클래스의 확장

그래서 `PatternLayout`를 상속받은 `MaskingPatternLayout` 클래스에서 이벤트 내용을 문자열로 변환할 때 호출되는 **doLayout()** 메소드를 오버라이드했습니다. 

**MaskingPatternLayout.java**
```java
@Override
public String doLayout(ILoggingEvent event) {
    String out = super.doLayout(event);
    if (!enabled)
        return out;

    SensitiveStringSanitizer s = sanitizer;
    if (s == null) // start() 호출 전에 doLayout 메서드가 호출되는 경우에는 원본 로그 그대로 출력(방어 로직 실행)
        return out; 

    return s.sanitize(out);
}
```
super.doLayout(event) 메서드를 호출해서 로그에 출력할 문자열을 생성하고, `SensitiveStringSanitizer` 객체의 **sanitize()** 메서드를 통해서 민감 정보를 마스킹합니다. 이렇게 함으로써 애플리케이션 로직을 수정하지 않고도, 로그 출력 레벨에서 일괄적으로 민감 정보를 통제할 수 있습니다.

개발자의 주의에 의존하지 않고, 구조적으로 문제를 해결하는 방식입니다. 민감 정보의 통제 지점을 애플리케이션 로직이 아닌 로그 출력 계층으로 옮긴 셈입니다.

## 주의 : Logback의 초기화

Logback은 애플리케이션이 시작될 때 설정 파일(`logback.xml`)을 읽고 Layout, Appender, MaskingPatternLayout 등 필요한 객체를 생성합니다. 

일반적으로 Logback의 초기화 절차는 다음과 같습니다.
1. Layout 객체 생성(토이 프로젝트에서는 PatternLayout을 상속받은 `MaskingPatternLayout` 객체)
2. 설정 값 주입 (rule, enabled 등)
3. Layout 객체의 start() 메서드 호출
4. 로그 출력 시작

sanitizer에는 start() 메서드 내부에서 생성된 객체가 할당됩니다.

**MaskingPatternLayout.java**
```java
@Override
public void start() {
    List<DefaultSensitiveStringSanitizer.Rule> rules = new ArrayList<>();
    for (String spec : ruleSpecs) {
        rules.add(RuleSpecParser.parse(spec));
    }

    this.sanitizer = new DefaultSensitiveStringSanitizer(rules, parseTriggers(triggers));
    super.start();
    }
```

일반적으로는 초기화 이후에 로그가 출력되지만, 예외적인 상황을 대비하여 sanitizer가 null인 경우에는 원본 로그를 그대로 출력하도록 **doLayout()** 메서드에 방어 로직을 추가했습니다.

```java
    SensitiveStringSanitizer s = sanitizer;
    if (s == null) // start() 호출 전에 doLayout 메서드가 호출되는 경우에는 원본 로그 그대로 출력(방어 로직 실행)
        return out; 
```
민감 정보의 마스킹은 보조 기능이지만, 로그 출력 자체는 실패해서는 안 된다고 판단했기 때문입니다.
로그 시스템은 항상 안정적으로 동작해야 합니다.

## trigger 기반 사전 필터링

정규식으로 로그 문자열에서 민감 정보에 해당되는 패턴을 찾아서 마스킹하는 방식은 구조적으로 단순합니다. 

하지만, **모든 로그 출력에 대해서 매번 정규식 검사를 수행해도 괜찮을까요?** INFO 레벨의 로그 출력이 많은 서비스라면 문자열 치환이 누적되면서 성능에 부담이 될 수 있습니다. 그래서 trigger 기반 사전 필터링 기능을 추가했습니다.

예를 들어, 다음과 같이 전화번호를 마스킹하는 정규식이 있다고 가정해보겠습니다.

> (\b01[016789]-?)\d{3,4}(-?\d{4}\b)

이 정규식은 비교적 복잡합니다. 그런데 사실 대부분의 로그에는 전화번호가 포함되어 있지 않습니다. 그래서 매번 정규식 검사를 수행할 필요는 없습니다. 대신 로그 문자열에 특정 키워드가 포함되어 있는 경우에는 정규식 검사가 수행되도록 했습니다.

키워드는 `logback.xml` 파일에 <trigger></trigger> 태그 사이에 나열하면 되고, 콤마(,)로 단어를 구분합니다. 

**예시) logback.xml에 키워드 정의**
```xml
<triggers>010-,@,Bearer </triggers>
```
* 010- → 전화번호 가능성
* @ → 이메일 가능성
* Bearer → 토큰 가능성

지금까지 설명한 Trigger 기반 사전 필터링의 동작 절차는 다음과 같습니다.
1. 로그 문자열을 받는다.
2. trigger 키워드가 포함되어 있는지 먼저 검사한다.
3. 키워드가 포함되어 있다면 정규식 검사를 수행한다. 만약, 키워드가 포함되어 있지 않다면 정규식 검사 수행은 생략된다.

**요약**
> * Trigger → 빠른 1차 필터
> * 정규식 검사 → 정확한 2차 판별

## DefaultSensitiveStringSanitizer 클래스의 구현

**DefaultSensitiveStringSanitizer.java**
```java
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
```
sanitize() 메서드에서 로그 메시지를 가공하기 maybeSensitive()를 호출해서 특정 문자열의 포함 여부를 확인합니다.

**DefaultSensitiveStringSanitizer.java**
```java
@Override
public String sanitize(String input) {
    if (input == null || input.isEmpty())
        return input;

    // triggers가 있으면 "민감 가능성" 빠른 체크로 대부분 로그는 스킵
    if (!maybeSensitive(input))
        return input;

    String out = input;
    for (Rule r : rules) {
        out = r.apply(out);
    }
    return out;
}
```


# 포스팅 시리즈

* [[토이 프로젝트] 스프링 애플리케이션의 로그에서 민감 정보 마스킹: (1) 순수 문자열 패턴 기반으로 설계](https://sanghoon-lee.github.io/2026/02/16/logging-pattern-mask/)
* [토이 프로젝트] 스프링 애플리케이션의 로그에서 민감 정보 마스킹: (2) 구현 상세

## 참고 : 소스 코드 
* [logging-pattern-mask](https://github.com/sanghoon-lee/logging-pattern-mask)