---
title: "Swift - Structured/Unstructured Task"
tags: [Swift]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

## Swift Concurrency - Structured Task vs Unstructured Task

Swift Concurrency에서 Task는 크게 두 가지로 나뉩니다.

- Structured Task

- Unstructured Task

이 둘의 차이를 이해하지 못하면 cancellation 전파, actor 격리, 예상치 못한 동시성 버그를 피하기 어렵습니다.

-----

### 1. Structured Task란?

**Structured Task**는 부모 Task의 생명 주기와 구조 안에 포함된 Task 입니다.

대표적인 예는 다음과 같습니다.

```swift
func load() async {
    async let user = fetchUser()
    async let posts = fetchPosts()

    let result = await (user, posts)
}
```

**특징**

- 부모 Task가 끝나기 전까지 자식 Task는 반드시 완료

- 부모가 취소되면 자식도 자동 취소

- 결과를 반드시 await 해야 함

- Swift Concurrency가 생명주기를 관리

👉 즉, 작업 트리가 명확하게 구조화되어 있습니다.

-----

### 2. Structured Task의 장점

**1️⃣ Cancellation 전파**

```swift
Task {
    async let data = fetchData()
    await data
}
```

- 부모 Task가 cancel -> fetchData()도 자동 cancel

- 별도의 cancellation 코드가 필요 없음

**2️⃣ 메모리 & 생명주기 안정성**

- 끝났는지 모르는 Task

- 어디서 생성됐는지 모르는 Task

이런 Task들이 생기지 않습니다.

**3️⃣ Actor 격리와 궁합이 좋음**

```swift
actor Store {
    func refresh() async {
        async let a = fetchA()
        async let b = fetchB()

        await (a, b)
    }
}
```

- actor 내부 직렬성 유지

- 예상 가능한 실행 흐름

-----

### 3. Unstructured Task란?

**Unstructured Task**는 부모와 생명주기적으로 분리된 독립 Task 입니다.

```swift
Task {
    await sendLog()
}

또는

Task.detached {
    await doSomething()
}
```

**특징**

- 부모 Task와 취소/종료가 연결되지 않음

- 결과를 기다릴 필요 없음

- 생명주기를 개발자가 책임짐

- Fire-and-forget 형태

-----

### 4. Unstructured Task가 필요한 이유

Unstructured Task가 나쁜 것이 아닙니다.

다만 용도가 명확해야 합니다.

**1️⃣ 로깅/분석 이벤트**

```swift
func track() {
    Task {
        await sendAnalytics()
    }
}
```

- 화면 생명주기 무관

- 실패해도 UI에 영향이 없음

**2️⃣ 시스템 이벤트 트리거**

- 앱 종료 직전 저장

- 백그라운드 진입 시 정리 작업

-----

### 5. 가장 위험한 패턴 ❌

**Actor 내부에서 Unstructured Task 생성**

```swift
actor UserStore {
    func refresh() {
        Task {
            await fetchUser()
        }
    }
}
```

**❌ 문제점**

- 생성된 Task는 actor에 묶이지 않음

- actor의 직렬성 보장이 깨짐

- race condition 가능성 증가

👉 이 코드는 겉보기엔 안전해 보이지만, 매우 위험합니다.

-----

### 6. 올바른 대안 ✅

**Structured Task로 만들기**

```swift
actor UserStore {
    func refresh() async {
        await fetchUser()
    }
}
```

**호출부에서 Task 생성**

```swift
Task {
    await userStore.refresh()
}
```

- actor 내부는 async 함수만

- Task 생성 책임은 외부로 이동

-----

### 7. Task.detached는 언제 쓰는가?

```swift
Task.detached {
    await doSomething()
}
```

**특징**

- 부모 Task의 actor context, priority 상속 ❌

- 완전히 독립적인 실행단위

**사용 시점**

- actor / MainActor 격리를 의도적으로 끊어야 할 때

- 라이브러리 내부에서 호출 컨텍스트를 신경 쓰지 않아야 할 때

-----

### 8. Structured vs Unstructured 한 눈에 비교

| 항목 | Structured Task | Unstructured Task |
| :-- | :-- | :-- |
| 생명주기 | 부모에 종속 | 독립 |
| Cancellation 전파 | 자동 | 수동 |
| 결과 대기 | 필수 | 선택 |
| Actor 격리 | 유지됨 | 깨질 수 있음 |
| 대표 예 | `async let`, `withTaskGroup` | `Task { }`, `Task.detached { }` |

-----

### 9. 마무리

Swift Concurrency의 핵심은 **'동시에 실행할 수 있느냐'**가 아니라 **'누가 생명주기를 책임지느냐'**입니다.

- Structured Task -> Swift가 책임

- Unstructured Task -> 개발자가 책임

이 기준을 잡는 순간, cancellation 위치 / actor 설계 / Task 생성 위치가 전부 자연스럽게 정리됩니다.