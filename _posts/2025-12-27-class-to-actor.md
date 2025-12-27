---
title: "기존 class를 actor로 변경하며 겪은 것들"
tags: [Swift]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

### Swift 6 Concurrency 전환 과정에서의 실제 고민과 주의 사항

Swift 6로 전환하면서 가장 크게 체감한 변화 중 하나는 **"기존에 아무 문제 없이 쓰이던 class들이 더 이상 안전하지 않다고 경고"** 한다는 점이었습니다.

Sendable 경고, MainActor-isolated 접근 에러, 그리고 "이 코드는 동시성 환경에서 안전하지 않다"는 컴파일러의 메시지 등이 있습니다.

그 과정에서 자연스럽게 마주한 선택지가 바로 actor로의 전환이었습니다.
하지만 class를 actor로 바꾸는 일은 단순히 키워드 하나를 바꾸는 작업이 아니었습니다.

이 글에서는 기존 class를 actor로 변경하면서 실제로 겪었던 변화, 그리고 전환 과정에서 반드시 고려해야 할 점들을 정리해보려고 합니다.

-----

### 왜 class를 actor로 바꾸게 되었을까

기존의 많은 class들은 다음과 같은 역할을 하고 있었습니다.

- 앱 전역에서 공유되는 상태 관리

- 캐시, 세션, 토큰, 설정 값 보관

- 여러 비동기 작업에서 동시에 접근

이런 class들은 겉보기에는 잘 동작했지만,
Swift 6의 Strict Concurrency 환경에서는 다음과 같은 경고가 발생하기 시작했습니다.

- 여러 Task에서 동시에 접근 가능한 mutable 상태

- actor 또는 MainActor 경계를 무시한 접근

- Sendable 하지 않은 타입의 전달

즉, 동작은 하고 있었지만 안전하다고 보장할 수 없는 코드였던 것입니다.

actor는 이런 문제를 해결하기 위한 Swift의 답입니다.

-----

### 첫 번째 단계: class를 actor로 바꾸기

가장 단순한 시작은 이것입니다.

```swift
final class CacheManager {
    var storage: [String: String] = [:]

    func set(_ value: String, for key: String) {
        storage[key] = value
    }

    func get(_ key: String) -> String? {
        storage[key]
    }
}
```

이 코드를 actor로 바꾸면:

```swift
actor CacheManager {
    var storage: [String: String] = [:]

    func set(_ value: String, for key: String) {
        storage[key] = value
    }

    func get(_ key: String) -> String? {
        storage[key]
    }
}
```

여기까지는 매우 간단합니다.
하지만 진짜 변화는 이 actor를 사용하는 코드에서 시작됩니다.

-----

### actor로 바꾸면 가장 먼저 깨지는 것들

actor 내부의 프로퍼티와 메소드는 기본적으로 actor-isolated 입니다.

즉, 외부에서 접근할 때는 반드시 await 키워드가 필요합니다.

```swift
cacheManager.set("token", for: "auth")  // ❌
await cacheManager.set("token", for: "auth")    // ✅
```

>actor 전환은 "내부 구현 변경" 뿐만 아니라 API 사용 방식 자체가 바뀌는 작업

-----

### 외부 호출부를 수정하는 대표적인 패턴들

#### 1. 호출부까지 async/await로 전파하기

```swift
func updateToken() async {
    await cacheManager.set("token", for: "auth")
}
```

가장 명확하고 안전한 방식입니다.
다만 기존 동기 API가 많을 경우 변경 범위가 커집니다.

-----

#### 2.기존 동기 API를 유지하고 Task로 감싸기

```swift
func updateToken() {
    Task {
        await cacheManager.set("token", for: "auth")
    }
}
```

이 방식은 마이그레이션 초기에 자주 사용하게 됩니다.
하지만 의미가 달라진다는 점을 반드시 인지해야 합니다.

- 함수는 즉시 반환

- 실제 작업은 비동기로 나중에 실행

즉, "이 함수가 끝나면 상태가 변경되어 있다"는 보장이 사라지게 됩니다.

-----

#### 3. UI 전용 상태라면 @MainActor가 더 적절한 경우

모든 상태가 메인 스레드에서 다뤄져야 한다면, 굳이 actor를 쓰는 것이 오히려 복잡할 수 있습니다.

```swift
@MainActor
final class ViewModel {
    var state: State
}
```

이 경우 await은 필요하지만 스레드 모델이 명확해지고 UI 코드에서도 바로 사용할 수 있습니다.

-----

### actor 전환 시 반드시 주의해야 할 포인트

#### 1. actor는 재진입(reentrancy)이 가능하다.

actor는 동시에 두 개의 작업을 실행하지는 않지만, await 지점에서는 현재 작업이 중단되며 그 사이에 다른 작업이 actor에 들어와 실행될 수 있습니다.

>await은 actor reentrancy를 허용하는 지점입니다.

```swift
func update() async {
    count += 1
    await someAsyncWork()
    count += 1
}
```

위 코드에서 await 사이에 다른 Task가 실행되면 count의 값은 예상과 달라질 수 있습니다.

해결 전략:

- await 전후에 의존하는 상태는 로컬로 복사

- 상태 변경은 가능한 짧게 유지

-----

#### 2. actor 내부에 클로저/콜백을 저장할 때 Sendable 이슈

actor는 동시성 경계를 넘나들 수 있기 때문에 내부에 저장하는 값들도 안전해야 합니다.

- CompletionHandler

- Delegate

- Timer Callback

이런 것들은 Swift 6에서 자주 Sendable 경고를 발생시킵니다.

해결 전략:

- 외부로 노출되는 클로저는 `@Sendable` 고려

- actor 내부 상태로 참조 타입을 들고 있지 않은지 점검

-----

#### 3. 기존 동기 API의 "계약"이 바뀐다.

class에서는:

```swift
manager.update()
print(manager.state)       // 업데이트 이후 상태
```

actor + Task 패턴에서는 이 보장이 깨질 수 있습니다.

```swift
manager.update()
print(manager.state)       // 아직 업데이트 전일 수도 있음
```

actor 전환은 단순한 thread-safe 개선이 아니라, API의 의미를 바꾸는 작업이라는 점을 명확히 인식해야 합니다.

-----

#### 4. Singleton class -> actor 전환은 특히 신중해야 합니다.

```swift
static let shared = Manager()
```

이 패턴 자체는 actor에서도 가능하지만,

- 호출부마다 await이 전파되고

- 모든 책임이 하나의 actor로 몰릴 수 있습니다.

이 과정에서 오히려 역할 분리의 필요성이 더 또렷해지게 됩니다.

-----

### class -> actor 전환을 통해 얻은 것

- 동시 접근에 대한 명확한 안전성

- 컴파일 타임에 잡히는 문제들

- "이 상태는 어디에서 접근 가능한가?"를 강제로 생각하게 만드는 구조

actor는 만든 도구가 아닙니다.
하지만 공유 상태를 다루는 class라면, Swift 6 환경에서는 가장 명확한 선택지 중 하나입니다.

-----

### 마무리

기존 class를 actor로 바꾸는 작업은 생각보다 많은 결정을 요구합니다.

- API의 의미를 바꿀 것인가

- 호출부를 async로 전파할 것인가

- MainActor가 더 적절하지는 않은가

- 이 상태는 정말 공유되어야 하는가

Swift 6의 Concurrency 모델은 **"조심해서 쓰세요"** 가 아니라 **"안전하지 않으면 컴파일조차 되지 않게 하겠다"** 는 방향에 가깝습니다.

actor 전환은 그 흐름 속에서 코드의 책임과 경계를 다시 설계하게 만드는 계기가 될 수 있습니다.