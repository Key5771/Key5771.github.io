---
title: "Swift 6에서 달라진 Existential 동작과 Implicitly Opened Existentials"
tags: [Swift]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

## Swift 6에서 달라진 Existential 동작과 Implicitly Opened Existentials

Swift 6로 전환하면서, 기존에 문제없이 사용되던 코드에서 예상치 못한 크래시를 겪었습니다.

원인은 의외로 단순했습니다.

>분면 프로토콜 타입(any Protocol)으로 사용하고 있다고 생각했는데, 
컴파일러가 이를 concrete 타입으로 열어(open) 버리고 있었다.

이 글에서는 [Swift Evolution Proposal SE-0352: Implicitly Opened Existentials](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0352-implicit-open-existentials.md#suppressing-explicit-opening-with-as-any-p--as-any-p)가 무엇인지, 그리고 Swift 6에서 왜 이 변화가 실제 크래시로 이어질 수 있는지를 정리해봅니다.

-----

### Existential 타입이란

Swift에서 프로토콜 타입으로 사용할 때 우리는 흔히 이렇게 작성합니다.

```swift
let service: any LoginService
```

이 때 `any LoginService`는 existential type 입니다.

- "어떤 구체 타입인지는 모르지만"

- "LoginService를 만족하는 타입이다"

기존에는 이 existential을 그 자체로 하나의 박스처럼 다루는 경우가 많았습니다.

-----

### Existential을 여는(Open) 것의 의미

Swift 컴파일러는 때로 existential을 열어서(open) 그 안에 들어 있는 구체 타입(concrete type)으로 취급합니다.

```swift
func handler<T: LoginService>(_ service: T) { }
```

이 함수에 `any LoginService`를 전달하면,
컴파일러는 내부적으로 이렇게 생각합니다.

>"아, 이 순간에는 실제 타입 T가 필요하네.
그럼 existential을 열어서 concrete 타입으로 바꿔야겠다."

이 과정을 existential opening이라고 부릅니다.

-----

### Swift 5까지의 동작

Swift 5에서는 existential opening이 비교적 보수적으로 일어났습니다.

- 명확하게 generic context가 필요한 경우에만 opening

- 그 외에는 existential을 그대로 유지

그래서 많은 코드에서 **"프로토콜 타입으로 쓰고 있다"** 라는 가정이 깨지지 않았습니다.

-----

### Swift 6에서 달라진 점(SE-0352)

SE-0352의 핵심은 다음 문장으로 요약할 수 있습니다.

>Swift 6에서는 implicit opening이 더 적극적으로 발생한다.

즉,

- 이전에는 existential로 유지되던 코드가

- Swift 6에서는 자동으로 concrete 타입으로 열릴 수 있습니다.

**예제**

```swift
func f1<T: P>(_: T) { }
func f1<T>(_: T) { }

func test(p: any P) {
    f1(p)
}
```

Swift 6에서는 여기서 p가 암묵적으로 open되어 T: P 버전이 선택될 수 있습니다.

-----

### 왜 이게 문제가 될까?

문제는 의도하지 않은 concrete 타입 의존성이 생긴다는 점입니다.

**실제로 자주 발생하는 시나리오**

- DI 컨테이터에서 any Protocol로 관리

- 런타임에는 여러 구현체가 섞여 있음

- 그런데 컴파일러가 existential을 열어

- 특정 concrete 타입 기준으로 코드가 생성

그 결과:

- 잘못된 타입 캐스팅

- 예상하지 못한 코드 경로 실행

- 런타임 크래시

**"프로토콜로 추상화했다"** 는 전제가 깨지는 순간입니다.

-----

### Suppressing Implicit Opening

SE-0352에서는 이 문제를 해결하기 위한 명시적 억제 방법을 제공합니다.

**as any P 캐스팅**

```swift
func test(p: any P) {
    f1(p as any P)
}
```

이렇게 작성하면 컴파일러에게 명확하게 말하는 것과 같습니다.

>"이 값은 열지 말고, existential로 그대로 사용해라!"

즉, implicit opening을 억제(suppress)합니다.

-----

### 중요한 포인트: 괄호 하나의 차이

문서에서 특히 주의해야 할 부분이 있습니다.

```swift
f1(p as any P)  // opening 억제
f1((p as any P))    // ❌ 다시 opening 발생
```

괄호 하나만 추가해도 컴파일러의 판단이 달라집니다.
이는 Swift의 타입 추론 규칙과 연관된 매우 미묘한 차이입니다.

-----

### Swift 6 전환 시 주의할 점

Swift 6로 마이그레이션하면서 다음과 같은 코드는 특히 주의해야 합니다.

- any Protocol을 generic 함수에 전달하는 코드

- DI / Service Locator 패턴

- existential을 그냥 박스처럼 사용하던 코드

이 경우,

- 의도치 않은 existential opening이 발생할 수 있고

- concrete 타입 기준으로 동작이 변경될 수 있습니다.

-----

### 언제 suppression이 필요할까

다음 질문을 스스로에게 던져보면 판단이 쉬워집니다.

>"이 값이 concrete 타입으로 취급되어도 괜찮을까?"

- YES -> suppression 불필요

- NO -> `as any P`로 명시적 억제 필요

특히 추상화 계층의 경계에서는 suppression을 고려하는 것이 안전합니다.

-----

### 정리

- Existential은 내부에 concrete 타입을 숨긴 박스

- Swift 6에서는 이 박스를 더 적극적으로 open

- 이는 성능과 타입 안정성 측면에서는 개선

- 추상화 관점에서는 의도치 않은 동작 변화를 만들 수 있음

- `as any P`는 **"열지 말라"** 는 명확한 표현

Swift 6의 변화는 단순한 문법 변경이 아니라, 타입 시스템이 더 정직해진 결과라고 볼 수 있습니다.

하지만 그만큼 우리가 **이 코드는 정말 existential로 써도 되는걸까?** 를 이전보다 더 명확히 고민해야 하는 시점이기도 합니다.