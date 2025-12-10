---
title: "Strict Concurrency"
tags: [iOS]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

이번에는 Tuist에서 잠깐 벗어나 회사에서 Swift 6로 전환하면서 적용하고 있는 Strict Concurrency에 대해 알아보려고 합니다.

### Strict Concurrency

Swift 5.9부터 정식으로 적용되기 시작한 Strict Concurrency(엄격 동시성 모드)는 Swift Concurrency에서 "데이터 경쟁(Data Race)"을 컴파일 단계에서 최대한 제거하기 위한 검증 시스템입니다.

간단히 말해,
> 동시성 환경에서 메모리를 안전하게 다루고 있는지 컴파일러가 강제적으로 체크해주는 모드

### 1. 왜 Strict Concurrency가 등장했을까?

Swift Concurrency(특히, actor, async/await)가 등장하면서 많은 비동기 코드가 안전해졌지만, 기존 코드나 동시성 고려 없이 작성된 타입은 여전히 Data Race 위험을 가질 수 있습니다.

예를 들어,
- 여러 Task가 동시에 같은 클래스의 프로퍼티를 읽고 쓰는 경우
- Sendable을 만족하지 않는 타입을 여러 스레드가 공유하는 경우
- actor isolation 규칙을 어기는 경우

이런 문제들은 런타임에서 드물게 발생해 디버깅이 매우 어렵습니다.
Strict Concurrency는 컴파일 타임에 이런 잠재적 위험을 잡아주는 장치입니다.

-----

### 2. Strict Concurrency가 하는 일

Strict Concurrency는 크게 3가지를 강제합니다.

#### Sendable 체크 강화

여러 스레드에서 접근할 가능성이 있는 타입은 반드시 Sendable을 만족해야 합니다.

```swift
struct MyData: Sendable {
    var value: Int
}
```

불변(immutable) 구조체는 자동으로 Sendable을 만족하지만, 참조 타입(class)이나 가변 속성을 가진 타입은 명시적으로 Sendable을 선언해야 합니다.

#### Actor Isolation 검사 강화

Actor는 내부 상태를 독점적으로 관리합니다.
Strict Concurrency에서는 actor의 상태를 접근할 때 규칙을 어기면 컴파일 에러를 발생시킵니다.

```swift
actor Counter {
    var value = 0
}

let counter = Counter()
print(counter.value)    // ❌ 에러: actor 외부에서 직접 접근 불가
```

반드시 await을 통해 안전하게 접근해야 합니다.

```swift
let v = await counter.value     // OK
```

#### MainActor 및 Nonisolated 규칙 적용 강화

UI 업데이트는 반드시 메인 스레드(MainActor)에서 이루어져야 합니다.
Strict Concurrency는 이런 어노테이션 위반도 에러로 잡아냅니다.

```swift
@MainActor
class ViewModel {
    var text = ""
}

func update(vm: ViewModel) {
    vm.text = "Hello"   // ❌ 에러: actor 외부에서 직접 접근 불가
}
```

이런 경우 함수에도 MainActor를 부여해야 합니다.

```swift
@MainActor
func update(vm: ViewModel) {
    vm.text = "Hello"
}
```

-----

### 3. Strict Concurrency 모드 단계

Swift는 점진적으로 개발자를 압박하지 않기 위해 다양한 수준을 제공합니다.

| 모드 | 설명 |
| :-- | :-- |
| minimal | 기본. 대부분의 체크 느슨함 |
| targeted | 특정 모듈에서만 검사 강화 |
| complete | Swift 6 기본값. 모든 규칙 엄격 적용 |

Swift 6에서는 기본이 complete(strict)이기 때문에 마이그레이션 시 컴파일러 경고, 에러가 폭발하는 경우가 많습니다.

-----

### 4. Strict Concurrency와 @unchecked Sendable

Strict Concurrency 활성화 후 가장 많이 마주치는 오류는 바로 Sendable 관련 에러입니다.
예를 들어, 클래스는 기본적으로 Sendable이 아닙니다.

```swift
class Manager { ... }   // ❌ Sendable 아님
```

그러나 코드에서 class를 여러 Task가 공유해야 하는 상황이라면
- 타입을 완전히 thread-safe하게 만들거나
- 정말 어쩔 수 없는 경우 @unchecked Sendable 사용

```swift
final class Manager: @unchecked Sendable {
    var value: Int
    init(value: Int) { self.value = value }
}
```

이 경우 컴파일러는 Sendable 체크를 건너뛰지만,
개발자가 thread-safety를 직접 보장해야 한다는 의미이므로 매우 주의해야 합니다.

-----

### 5. 기존 프로젝트에서 Strict Concurrency를 적용하며 겪는 문제

기존 Swift 5 프로젝트에서 Strict Concurrency를 적용하면 다음과 같은 현상이 발생합니다.

- class는 자동 Sendable이 아니기 때문에 Sendable 에러가 쏟아지는 경우
    - 구조를 actor로 전환하거나, Sendable 준수 여부를 재검토 해야 합니다.
- `@MainActor` 부재로 인해 UI 코드 관련 에러 발생
    - ViewModel, ObservableObject 등 메인 스레드에서만 접근해야 하는 타입은 MainActor 설정이 필요합니다.
- closure의 escaping/non-escaping 문제
    - `@Sendable` closure 규칙이 강화되어 컴파일 에러가 발생할 수 있습니다.

-----

### 6. Strict Concurrency를 점진 적용하는 전략

##### 모듈 단위로 strict 모드를 적용

- 점진적으로 한 모듈씩 strict 모드로 전환하는 방식이 안전합니다.

##### actor 채택 검토

- Shared mutable state가 존재하면 actor가 가장 안전한 대안입니다.

##### @MainActor 적극 활용

- UI 관련 타입과 메소드에는 확실하게 `@MainActor`를 설정하는 것이 좋습니다.

-----

### 7. 마무리

Strict Concurrency는 Swift 개발자가 데이터 경쟁 문제를 원천적으로 차단하도록 돕는 강력한 도구입니다.
Swift 6에서는 기본 모드가 되기 때문에, 앞으로 iOS 개발자라면 반드시 이해하고 적용해야 하는 개념입니다.

정리하자면:
- 동시성 안정성을 컴파일 단계에서 보장
- actor isolation, Sendable 규칙 강제
- 기존 코드 수정 필요성이 많을 수 있다.
- 하지만 애플 생태계에서는 필수적인 방향