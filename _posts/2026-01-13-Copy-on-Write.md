---
title: "Swift - Copy on Write"
tags: [Swift]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

### Copy-on-Write(CoW)란

Swift에서 Array, Dictionary, String은 모두 struct(값 타입)입니다.

여기서

> "값 타입이면 복사가 될텐데, 배열이 커지면 성능이 너무 안좋아지지 않을까?"

이 질문에 대한 답이 바로 Copy-on-Write(CoW)입니다.

-----

### Copy-on-Write

말 그대로 "읽을 때는 공유하고, 수정할 때만 복사한다" 는 메모리 최적화 전략입니다.

즉,

- 값을 전달할 때는 실제 복사 ❌

- 수정하려고 할 때만 복사 ⭕️

값 타입의 안정성과 참조 타입 수준의 성능을 동시에 얻을 수 있습니다.

-----

### CoW가 필요한 이유

만약 CoW가 없다면:

```swift
let a = [1, 2, 3, 4, 5]
let b = a
```

이 시점에서 a와 b는 완전히 다른 메모리를 차지해야 합니다.

즉, 배열이 커질수록 메모리 낭비 + 성능 저하

-----

### CoW의 기본 동작 흐름

```swift
var a = [1, 2, 3]
var b = a       // 이 시점에는 복사 안 함

b.append(4)     // 이 때 실제 복사 발생
```

단계별로 보면

1. a와 b는 **같은 메모리를 공유**

2. b를 수정하려는 순간

3. Swift가 참조 수를 확인

4. 공유 중이면 -> 복사

5. b는 새로운 메모리 사용

즉, a는 전혀 영향을 받지 않습니다.

-----

### CoW는 어떻게 구현될까?

Swift의 표준 라이브러리는 내부적으로 다음 과정을 거칩니다.

1. 참조 수(reference count) 확인

2. 유일 참조(unique reference) 확인

3. 유일하지 않으면 복사

개념적으로는 이런 흐름입니다.

```swift
if !isUniquelyReferenced {
    copy()
}
mutate()
```

-----

### CoW가 적용되는 대표적인 타입들

Swift 표준 라이브러리에서 다음 타입들은 CoW를 사용합니다.

- Array

- Dictionary

- Set

- String

- Data

겉보기에는 값 타입이지만, 내부적으로는 힙 메모리를 공유합니다.

-----

### CoW가 성능에 미치는 영향

👍 장점

- 값 전달 비용 매우 낮음

- 불필요한 복사 방지

- 값 타입의 예측 가능성 유지

👎 주의할 점

- 쓰기 시점에 복사 비용 집중

- 대용량 데이터 + 잦은 수정 -> 비용 커짐

"언제 복사가 일어나는지"를 이해하는게 중요합니다.

-----

### CoW와 let / var

아주 중요한 포인트가 있습니다.

```swift
let a = [1, 2, 3]
let b = a

b.append(4)     // ❌ 컴파일 에러
```

- `let`으로 선언된 값은 수정 불가

- 따라서 CoW도 발생하지 않습니다.

```swift
var b = a
b.append(4) // ⭕️ 이때 COW 발생
```

-----

### 실무에서 조심해야 할 패턴

❌ 반복적인 복사 유발

```swift
for _ in 0..<1000 {
    array = array + [1]
}
```

- 매번 새 배열 생성

- CoW 효과 거의 없음

✅ 올바른 사용

```swift
array.reserveCapacity(1000)
for _ in 0..<1000 {
    array.append(1)
}
```

- 복사 최소화

-----

### CoW와 멀티스레드

CoW는 스레드 안전을 자동으로 보장하지 않습니다.

- 읽기 : 비교적 안전

- 쓰기 : 동기화 필요

CoW는 성능 최적화이지 동시성 해결책은 아닙니다.

-----

### SwiftUI와 CoW

SwiftUI에서 View가 struct 임에도 성능 문제가 적은 이유 중 하나가 바로 CoW입니다.

- View 자체는 값

- 내부 데이터는 필요할 때만 복사

- 빠른 재계산 가능

------

### 한 줄 요약

- CoW는 지연 복사 전략

- 읽을 때는 공유, 쓸 때만 복사

- Swift의 값 타입 성능을 지탱하는 핵심 기술

- 성능과 안정성의 균형점

> "Copy-on-Write는 Swift가 값 타입을 기본으로 선택할 수 있게 만든 비결입니다."