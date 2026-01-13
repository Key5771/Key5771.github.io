---
title: "iOS - 메모리 관리 방법 알아보기"
tags: [iOS, Swift]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

### iOS에서 메모리를 잘 관리하는 방법들

iOS 앱 개발에서 메모리 관리는 성능, 안정성, 사용자 경험을 모두 좌우하는 핵심 요소입니다.

특히 iOS는

- 메모리가 제한적이고

- Swap을 적극적으로 사용하지 않으며

- 메모리가 부족하면 앱을 강제 종료합니다.

-----

#### 1. ARC(Automatic Reference Counting)를 정확히 이해하기

iOS 메모리 관리의 핵심은 ARC입니다.

ARC는

- 참조 카운트를 자동으로 관리하고

- 참조 수가 0이 되면 객체를 해제합니다.

하지만 순환 참조는 자동으로 해결하지 못합니다.

**순환 참조(Cycle)를 피하자**

대표적인 예

```swift
class A {
    var b: B?
}

class B {
    var a: A?
}
```

- a와 b가 서로를 강하게 참조하므로 메모리에서 해제 ❌

해결 방법

```swift
class B {
    weak var a: A?
}
```

- weak: 참조는 하되 소유하지 않음

- unowned: 반드시 존재한다고 확신할 때만 사용

-----

#### 2. 클로저에서 self 캡처에 주의하기

클로저는 순환 참조의 가장 흔한 원인입니다.

대표적인 예

```swift
viewModel.onUpdate = {
    self.updateUI()
}
```

안전한 패턴

```swift
viewModel.onUpdate = { [weak self] in 
    self?.updateUI()
}
```

특히 다음 상황에서 반드시 점검해야 합니다.

- 네트워크 콜백

- 타이머

- 애니메이션 completion

- Combine sink

-----

#### 3. ViewController의 생명 주기 관리

ViewController는 메모리 누수의 중심이 되기 쉽습니다.

**반드시 확인할 것**

- deinit이 호출되는지

- dismiss / pop 이후에도 살아있지 않은지

```swift
deinit {
    print("Deallocated")
}
```

`deinit`이 호출되지 않으면 강한 참조가 어딘가 남아 있다는 뜻입니다.

-----

#### 4. Delegate는 weak이 기본

Delegate 패턴을 쓸 때는 항상 `weak`을 기본값으로 생각해야 합니다.

```swift
weak var delegate: SomeDelegate?
```

- Delegate는 소유 관계 ❌

- 통신 관계 ⭕️

------

#### 5. 값 타입(struct) 적극 활용하기

Swift는 값 타입 중심 언어입니다.

- struct는 ARC 대상 ❌

- 참조 공유 ❌

- 예측 가능성 ⭕️

------

#### 6. Copy-on-Write 이해하고 사용하기

Array, Dictionary, String은 값 타입이지만, CoW로 최적화되어 있습니다.

```swift
var a = largeArray
var b = a   // 복사 ❌

b.append(1) // 이 시점에만 복사 ⭕️
```

주의할 점

- 대용량 데이터 + 잦은 수정 -> 복사 비용 증가

- 필요하면 `reserveCapacity` 사용

-----

#### 7. 이미지 메모리 관리에 신경 쓰기

이미지는 iOS 앱에서 메모리를 가장 많이 잡아먹는 요소 중 하나입니다.

**좋은 습관들**

- 필요 이상으로 큰 이미지 로딩 ❌

- `UIImage(named: )` 남용 ❌ (캐시됨)

- 화면 크기에 맞는 리사이징

- 썸네일과 원본 분리

👉 이미지 한 장이 수 MB를 차지할 수 있습니다.

-----

#### 8. 메모리 경고 대응

iOS는 메모리가 부족하면 `didReceiveMemoryWarning`을 보냅니다.

```swift
override func didReceiveMemoryWarning() {
    super.didReceiveMemoryWarning()
    cache.removeAllObjects()
}
```

👉 캐시, 임시 데이터 정리는 이 시점에

-----

#### 9. 싱글톤(Singleton) 남용하지 않기

Singleton은 앱 수명 동안 살아있습니다.

**안전한 사용 기준**

- 상태 최소화

- 가능하면 읽기 전용

- 테스트 가능하게 설계

-----

#### 10. Instruments로 직접 확인하기

이론만으로는 부족합니다.

반드시 도구로 확인해야 합니다.

**필수 Instruments**

- Leaks: 메모리 누수

- Allocations: 객체 생성/해제 추적

- Memory Graph Debugger: 참조 관계 시각화

-----

#### iOS 메모리 관리 체크리스트

- deinit이 잘 호출되는가?

- 클로저에서 self 캡처는 안전한가?

- delegate는 weak인가?

- 불필요한 Singleton은 없는가?

- 이미지 크기는 적절한가?

- Instruments로 확인했는가?

------

### 한 줄 요약

- iOS는 메모리에 매우 민감한 플랫폼

- ARC를 믿되, 이해하고 써야 합니다.

- 대부분의 메모리 문제는

    - 순환 참조

    - 클로저

    - 이미지

> 메모리 관리는 나중에 고치는 문제가 아니라 처음부터 설계해야 할 문제입니다.