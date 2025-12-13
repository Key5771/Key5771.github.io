---
title: "Swift isolated deinit"
tags: [Swift]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

## Swift isolated deinit 정리

### Strict Concurrency에서 deinit 경고를 해결하는 가장 안전한 방법

Swift 6 모드에서 Strict Concurrency를 활성화하면, 기존에는 문제없던 deinit 코드에서 경고가 발생하는 경우가 많습니다. 
특히 `@MainActor`가 적용된 UIView, UIViewController 내부에서 리소스를 정리하는 코드가 대표적입니다.

이 글에서는 다음과 같은 기준으로 정리해보려고 합니다.

- 왜 deinit에서 경고가 발생하는지
- isolated deinit이 무엇인지
- `willMove(toWindow:)`로 옮기는 방식과의 차이

### 1. 문제 상황: deinit에서 발생하는 Strict Concurrency 경고

다음과 같은 코드는 UIKit 코드에서 흔히 볼 수 있습니다.

```swift
@MainActor
final class PlayerView: UIView {

    private var player: AVPlayer?
    private var statusObservation: NSKeyValueObservation?

    deinit {
        removeObservers()
        player = nil
    }

    private func removeObservers() {
        statusObservation?.invalidate()
        statusObservation = nil
    }
}
```

Swift 6 모드에서는 아래와 같은 경고가 발생할 수 있습니다.

```bash
Call to main actor-isolated instance method 'removeObservers()' in a synchronous nonisolated context

Main actor-isolated property 'player' can not be mutated from a nonisolated context
```

분명 UIView는 메인 스레드에서만 쓰이는데, 왜 이런 경고가 날까?

### 2. 핵심 원인: deinit은 기본적으로 nonisolated

Swift Concurrency에서 중요한 규칙 중 하나는 다음과 같습니다.

> deinit은 기본적으로 어떤 actor에도 속하지 않는다.(nonisolated)

즉, 클래스에 @MainActor가 있어도, deinit은 actor isolation을 상속하지 않습니다.

그래서 deinit 내부에서 `@MainActor` 메소드 호출, `@MainActor` 프로퍼티 변경을 하면 컴파일러가 데이터 레이스 가능성을 경고합니다.

### 3. 흔한 우회 방법: willMove(toWindow:)

이 문제를 피하기 위한 방법 중 하나는 다음과 같이 변경하는 방법일 수 있습니다.

```swift
@MainActor
final class PlayerView: UIView {

    override func willMove(toWindow newWindow: UIWindow?) {
        super.willMove(toWindow: newWindow)

        if newWindow == nil {
            removeObervers()
            playerView = nil
        }
    }
}
```

이 방식은 `@MainActor` 보장이 확실하고, Strict Concurrency 경고도 발생하지 않습니다.

하지만 의미적인 차이가 있습니다.

**willMove(toWindow:)의 한계**
- 뷰가 화면에서 분리될 때 호출됨
- 객체가 소멸되는 시점이 아님
- 뷰 재사용 / 다시 붙은 경우에도 호출될 수 있음

>즉, "객체 생명 종료 시 정리"라는 의미로는 정확하지 않습니다.

### 해결책: isolated deinit

Swift 5.9부터는 deinit에도 isolation을 명시할 수 있습니다.

```swift
@MainActor
final class PlayerView: UIView {
    
    private var player: AVPlayer?
    private var statusObservation: NSKeyValueObservation?

    isolated deinit {
        removeObservers()
        player = nil
    }
}
```

isolated deinit의 의미

- deinit을 해당 actor에서 실행하도록 명시
- 위 예제에서는 MainActor에서 deinit을 실행
- Strict Concurrency 경고 제거
- deinit의 **의미(객체 소멸 시 정리)** 를 유지

### 5. iOS 분기 버전이 필요 없는 이유

iOS 버전 상관없이 모두 동일하게 동작합니다.
Swift 5.9+ / Swift 6 모드만 만족되면 사용 가능합니다.

### 6. 주의사항: isolated deinit에서 할 수 없는 것

❌ async 작업

```swift
isolated deinit {
    await cleanup() // 컴파일 에러
}
```

- deinit은 반드시 동기 코드로
- actor hopping은 가능하지만 await은 불가

❌ Task 생성으로 책임 떠넘기기

```swift
deinit {
    Task { @MainActor in 
        player = nil
    }
}
```

- 객체 생명 종료와 정리 시점이 분리됨
- 메모리 / 타이밍 버그 위험

### 7. 언제 isolated deinit을 써야 할까?

다음 조건을 만족한다면 추천합니다.

- `@MainActor` 클래스
- deinit에서 UI / Observer / Player / Cancellable 정리 필요
- Strict Concurrency 경고 없이 의미적으로 올바른 해제 시점 유지

### 8. 정리

- deinit은 기본적으로 nonisolated
- Swift 6 Strict Concurrency에서 경고가 발생하는 것은 정상
- `willMove(toWindow:)` 함수는 타이밍을 바꾸는 우회 방법
- isolated deinit은 의미와 안정성을 모두 지키는 해결책