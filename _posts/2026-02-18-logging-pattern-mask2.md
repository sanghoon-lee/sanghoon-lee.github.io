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
3. PatternLayout가 이벤트 내용을 문자열로 변환
4. 변환된 문자열이 Appender를 통해 출력

토이 프로젝트에서 로그 마스킹 기능을 구현하기 위해서 개입한 지점은 3번과 4번 사이입니다. 로그가 출력되기 직전에 민감 정보를 찾아서 마스킹하도록 구현하는 것이 핵심이기 때문입니다.

## PatternLayout 클래스의 확장

그래서 `PatternLayout`를 상속받은 `MaskingPatternLayout` 클래스에서 이벤트 내용을 문자열로 변환할 때 호출되는 **doLayout()** 메소드를 오버라이드했습니다. 

```java
@Override
    public String doLayout(ILoggingEvent event) {
        String out = super.doLayout(event);
        if (!enabled)
            return out;

        SensitiveStringSanitizer s = sanitizer;
        if (s == null)
            return out; // start() 전 방어

        return s.sanitize(out);
    }
```
super.doLayout(event) 메서드를 호출해서 로그에 출력할 문자열을 생성하고, `SensitiveStringSanitizer` 객체의 **sanitize()** 메서드를 통해서 민감 정보를 마스킹합니다. 이렇게 함으로써 애플리케이션 로직을 수정하지 않고도, 로그 출력 레벨에서 일괄적으로 민감 정보를 통제할 수 있습니다.

개발자의 주의에 의존하지 않고, 구조적으로 문제를 해결하는 방식입니다. 민감 정보의 통제 지점을 애플리케이션 로직이 아닌 로그 출력 계층으로 옮긴 셈입니다.

# 포스팅 시리즈

* [[토이 프로젝트] 스프링 애플리케이션의 로그에서 민감 정보 마스킹: (1) 순수 문자열 패턴 기반으로 설계](https://sanghoon-lee.github.io/2026/02/16/logging-pattern-mask/)
* [토이 프로젝트] 스프링 애플리케이션의 로그에서 민감 정보 마스킹: (2) 구현 상세

## 참고 : 소스 코드 
* [logging-pattern-mask](https://github.com/sanghoon-lee/logging-pattern-mask)