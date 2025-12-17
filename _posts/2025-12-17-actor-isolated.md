---
title: "actor-isolated"
tags: [Swift]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

## Swift Concurrency에서 actor-isolated 이해하기

Swift Concurrency를 공부하다 보면 가장 자주 마주치는 표현 중 하나가 바로 "actor-isolated"라는 개념입니다.

에러 메시지나 경고에서 이런 문장을 본 적이 있을 것입니다.

>Actor-isolated property can not be accessed from a nonisolated context

처음에는 이 문장이 다소 추상적으로 느껴집니다.

하지만 actor-isolated를 정확히 이해하면, 왜 await이 필요한지, 왜 어떤 접근은 허용되지 않는지 자연스럽게 이해할 수 있습니다.

이 글에서는 actor-isolated가 무엇인지, 그리고 Swift가 왜 이 개념을 중심으로 동시성을 설계했는지를 기초부터 정리해보려고 합니다.

-----

### Actor는 왜 필요할까

동시성에서 가장 큰 문제는 데이터 경쟁(data race)입니다.

여러 스레드가 동시에 같은 데이터를 읽고 수정하면, 의도하지 않은 상태가 만들어질 수 있습니다.

Swift의 actor는 이 문제를 해결하기 위한 타입입니다.

```swift
actor Counter {
    var value: Int = 0

    func increment() {
        value += 1
    }
}
```

이 actor는 value라는 상태를 자기 자신만 접근할 수 있도록 보호합니다.

이 보호 규칙을 Swift에서는 격리(Isolation)라고 부릅니다.

-----

### actor-isolated란 무엇일까

actor-isolated란

>actor 내부의 상태(프로퍼티, 메소드)가 해당 actor의 실행 컨텍스트에서만 접근 가능하도록 보호되는 것

actor 내부에 정의된 대부분의 멤버는 기본적으로 actor-isolated 입니다.

```swift
actor UserStore {
    var users: [String] = []

    func add(_ user: String) {
        users.append(user)
    }
}
```

- users

- add(_: )

이 둘은 모두 actor-isolated 상태입니다.

-----

### actor-isolated 접근 규칙

actor-isolated 멤버에는 명확한 접근 규칙이 있습니다.

1. actor 내부에서 접근

**동기 접근 가능**

```swift
actor UserStore {
    var users: [String] = []

    func count() -> Int {
        users.count
    }
}
```

같은 actor 안에서는 안전성이 보장되므로 await 없이 사용 가능합니다.

2. actor 외부에서 접근

**비동기 접근(await) 필요**

```swift
let store = UserStore()

let count = await store.count()
```

이 await은 단순한 문법이 아니라, "이 코드는 actor의 실행 컨텍스트로 넘어간다"는 의미를 가집니다.

-----

### 왜 actor-isolated 접근은 await이 필요할까

actor는 내부적으로 하나의 직렬 실행 큐(serial executor)를 가집니다.

외부에서 actor-isolated 멤버에 접근하면, 

- 현재 실행 컨텍스트를 잠시 중단

- actor의 실행 큐로 작업 전달

- 안전하게 작업 수행

- 결과 반환

이 과정이 필요하기 때문에 Swift는 await을 요구합니다.

즉,

>await = "actor의 보호 영역에 들어간다"

-----

### actor-isolated 프로퍼티 접근 예제

```swift
actor Profile {
    var name: String = "Kim"
}
```

**❌ 잘못된 접근**

```swift
let profile = Profile()
print(profile.name) // 컴파일 에러
```

**✔ 올바른 접근**

```swift
let profile = Profile()
let name = await profile.name
```

이 에러는 Swift가 컴파일 타임에 데이터 경쟁 가능성을 차단하고 있다는 증거입니다.

-----

### actor-isolated 메소드와 side effect

actor-isolated 메소드는 내부 상태를 안전하게 변경할 수 있습니다.

```swift
actor Counter {
    private var value = 0

    func increment() {
        value += 1
    }
}
```

여러 Task가 동시에 increment()를 호출해도,

- 내부적으로는 항상 순차 실행

- value는 안전하게 보호됨

이 점이 actor의 핵심 가치입니다.

-----

### actor-isolated와 @MainActor

`@MainActor`도 사실상 특수한 actor 입니다.

```swift
@MainActor
final class ViewModel {
    var title: String = ""
}
```'

- title은 MainActor-isolated

- 메인 스레드에서만 접근 가능

```swift
Task {
    await viewModel.titie = "Hello"
}
```

즉,

- actor-isolated

- MainActor-isolated

두 개는 같은 개념이며, 어떤 actor에 격리되어 있는가만 다를 뿐입니다.

-----

### actor-isolated와 nonisolated의 대비

| 구분 | actor-isolated | nonisolated |
| :-- | :-- | :-- |
| 접근 방식 | await 필요 | 동기 접근 가능 |
| 보호 여부 | actor가 보호 | 보호 없음 |
| 상태 접근 | 가능 | 불가능 |
| 사용 목적 | mutable state 보호 | 안전한 상수 / 메타 정보 |

`nonisolated`는 "이 멤버는 actor 보호가 필요 없다"는 명시적인 선언이고, 기본 상태는 항상 actor-isolated라는 점이 중요합니다.

-----

### actor-isolated를 이해하면 보이는 것들

actor-isolated 개념을 이해하면 다음이 자연스럽게 연결됩니다.

- 왜 프로퍼티 접근에도 await이 필요한지

- 왜 deinit에서 MainActor 경고가 발생하는지

- 왜 `nonisolated`가 위험한 키워드인지

- Swift가 런타임이 아니라 컴파일 타임에 동시성을 검사하는 이유

-----

### 정리

- actor 내부 멤버는 기본적으로 actor-isolated

- actor-isolated 멤버는 외부에서 접근할 때 await이 필요

- 이는 데이터 경쟁을 원천적으로 차단하기 위한 설계

- `@MainActor`도 같은 isolation 모델 위에 있음

Swift Concurrency의 핵심은 "개발자가 조심하는 것"이 아니라 "**컴파일러가 안전하지 않은 코드를 허용하지 않는 것**"입니다.