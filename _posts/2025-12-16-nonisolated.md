---
title: "nonisolated"
tags: [Swift]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

## Swift Concurrency에서 nonisolated 이해하기

Swift Concurrency를 사용하다 보면 `@MainActor`, `actor`, `isolated` 같은 키워드를 자주 접하게 됩니다.

그 중에서도 `nonisolated`는 비교적 사용 빈도는 낮지만, 등장하는 순간 이해하지 못하면 코드가 갑자기 어려워지는 키워드입니다.

이 글에서는 `nonisolated`가 왜 존재하는지, 그리고 언제, 어떤 의미로 사용되는지를 기초부터 정리해보려고 합니다.

-----

### Actor와 격리(Isolation) 개념 먼저 이해하기

Swift에서 `actor`는 데이터 경쟁(data race)을 방지하기 위한 타입입니다.

`actor` 내부의 상태는 기본적으로 하나의 실행 컨텍스트에서만 접근 가능하도록 보호됩니다.

```swift
actor Counter {
    var value: Int = 0

    func increment() {
        value += 1
    }
}
```

이 때 `actor` 내부의 프로퍼티와 메소드는 기본적으로 `actor`에 의해 격리(isolated)됩니다.

- `actor` 외부에서 접근 -> `await` 필요

- `actor` 내부에서 접근 -> 안전하게 보장됨

이 "격리" 개념이 `nonisolated`를 이해하는 핵심입니다.

-----

### nonisolated란 

`nonisolated`란 말 그대로 격리되지 않은 상태를 의미합니다.

즉,

>actor 또는 @MainActor 타입 안에 정의되어 있지만, 해당 actor의 격리 규칙을 따르지 않는 멤버

```swift
actor UserManager {
    nonisolated let identifier: String = "user_manager"
}
```

위 코드에서 `identifier`는 actor에 속해 있지만, actor의 실행 컨텍스트와는 완전히 무관하게 접근 가능합니다.

-----

### 왜 nonisolated가 필요한가

1. 동기 접근이 필요한 값

actor 내부의 모든 멤버가 항상 비동기 접근을 요구한다면 불편한 경우가 있습니다.

```swift
actor Logger {
    let category: String
}
```

이 경우 `category`에 접근하려면 외부에서 항상 `await` 키워드가 필요합니다.
하지만 이 값이 불변(let)이고, 동기적으로 접근해도 안전하다면 굳이 actor 격리를 탈 필요가 없습니다.

```swift
actor Logger {
    nonisolated let category: String
}
```

이렇게 하면 어디서든 `await` 없이 접근 가능합니다.

2. 프로토콜 요구사항 충족

가장 자주 `nonisolated`가 등장하는 케이스입니다.

```swift
protocol Identifiable {
    var id: String { get }
}
```

actor가 이 프로토콜을 채택하면 문제가 발생합니다.

```swift
actor User: Identifiable {
    let id: String
}
```

컴파일 에러
> Actor-isolated property ‘id’ cannot be used to satisfy nonisolated protocol requirement

이유는 간단합니다.

- 프로토콜 요구사항은 기본적으로 nonisolated

- actor의 프로퍼티는 기본적으로 isolated

이 불일치를 해결하기 위해 `nonisolated`를 사용합니다.

```swift
actor User: Identifiable {
    nonisolated let id: String
}
```

------

### nonisolated 메소드

`nonisolated`는 프러퍼티 뿐만 아니라 메소드에도 사용할 수 있습니다.

```swift
actor Cache {
    nonisolated func description() -> String {
        "Cache"
    }
}
```

**중요한 제약**

`nonisolated` 메소드는 actor 내부 상태에 접근할 수 없습니다.

```swift
actor Cache {
    var count: Int = 0

    nonisolated func printCount() {
        print(count)    // ❌ 컴파일 에러
    }
}
```

이유는 명확합니다.
`nonisolated`는 actor 보호를 받지 않기 때문에,
actor 내부 상태에 접근하는 순간 데이터 레이스 가능성이 생깁니다.

-----

### @MainActor와 nonisolated

`@MainActor`에서도 동일한 개념이 적용됩니다.

```swift
@MainActor
final class PlayerViewModel {
    nonisolated let identifier = "player_vm"
}
```

- 클래스 전체는 MainActor에 격리

- identifier는 메인 스레드와 무관하게 접근 가능

이 패턴은 다음과 같은 경우에 유용합니다.

- 로깅용 상수

- 디버그 식별자

- 동기 접근이 필요한 메타 정보

-----

### nonisolated와 Sendable의 관계

`nonisolated` 멤버는 동시 접근 가능하다는 의미를 내포합니다.
따라서 보통 다음 조건을 만족해야 안전합니다.

- let 상수

- 값 타입(struct, enum)

- 내부 상태를 변경하지 않는 함수

이 때문에 `nonisolated`는 Sendable 개념과도 밀접합니다.

> "이 값은 어떤 실행 컨텍스트에서 접근해도 안전한가?"

이 질문에 확신할 수 있을 때만 `nonisolated`를 사용해야 합니다.

-----

### 언제 nonisolated를 피해야 할까?

다음과 같은 경우에는 `nonisolated`를 사용하면 안됩니다.

- mutable 상태(var)

- actor 내부 상태를 읽거나 쓰는 메소드

- 동기 접근 시 thread-safety를 보장할 수 없는 경우

`nonisolated`는 편의 기능이 아니라, 명시적인 책임 선언에 가깝습니다.

-----

### 정리

- `nonisolated`는 `actor` / `@Mainactor`의 격리에서 벗어난 멤버를 선언합니다.

- 동기 접근이 안전한 경우에만 사용해야 합니다.

- 프로토콜 채택 시 가장 자주 등장합니다.

- 잘못 사용하면 Concurrency 안전성을 직접 깨는 결과가 됩니다.

Swift Concurrency는 "컴파일러가 대신 고민해주는 모델"에 가깝습니다.
`nonisolated`는 그 보호막을 의도적으로 벗어나는 선택이기 때문에,
왜 필요한지 명확히 이해하고 사용하는 것이 중요합니다.