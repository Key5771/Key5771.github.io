---
title: "Swift - Task Cancellation"
tags: [Swift]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

# Swift Task Cancellation

## Task.isCancelled vs Task.checnCancellation

Swift Concurrency에서 Task는 취소 가능한 작업입니다.
하지만 많은 경우 "취소를 어떻게 체크해야 하는지", "어디서 해야 하는지"가 명확하지 않아 실수를 하게 됩니다.

저 역시 코딩 중에 잘못된 위치에 cancellation 체크를 넣는 실수를 했고, 그 경험을 계기로 개념을 다시 정리하게 되었습니다.

-----

### Task.cancel()은 무엇을 하는가

```swift
let task = Task {
    await loadData()
}

task.cancel()
```

`cancel()`은 Task를 즉시 중단시키지 않습니다.

- `cancel()`은 단순히 **취소 플래그를 세팅**하는 것

- Task는 직접 그 상태를 확인해야만 중단됩니다.

이 방식은 **Cooperative Cancellation**이라고 불립니다.

> Swift는 Task를 강제로 kill 하지 않는다 -> 안전하고 예측 가능한 종료를 위해서

-----

### Task.isCancelled - 상태 확인용

```swift
Task {
    if Task.isCancelled {
        return
    }
    await loadData()
}
```

**특징**

- 현재 Task가 취소 요청을 받았는지 Bool 값으로 확인

- 아무 일도 자동으로 일어나지 않음

- 흐름 제어흘 직접 작성해야 함

**언제쓰면 좋을까?**

- 반복 작업

- 부분적으로 무시 가능한 작업

- 취소되었으면 그냥 조용히 끝내도 되는 경우

```swift
for item in items {
    if Task.isCancelled { break }
    await process(item)
}
```

-----

### Task.checkCancellation() - 즉시 중단용

```swift
Task {
    try Task.checkCancellation()
    await loadData()
}
```

**특징**

- Task가 취소되었다면 즉시 CancellationError throw

- 반드시 try/catch 문맥에서 사용

- 흐름을 명확하게 끊어줌

```swift
Task {
    do {
        try Task.checkCancellation()
        await loadData()
    } catch is CancellationError {
        // 취소에 대한 작업 정리
    }
}
```

**언제 쓰면 좋을까?**

- 이 작업이 취소되면 절대 진행되면 안 될 때

- 네트워크 요청, 상태 변경, 사이트 이펙트 직전

- "취소 = 실패"로 간주되는 작업

-----

### 가장 많이 하는 실수

**아무 의미 없는 곳에서 취소 체크하기**

```swift
Task {
    await loadData()
    Task.checkCancellation()    // 이미 끝난 후
}
```

이 코드는 취소 체크의 의미가 거의 없습니다.

- 이미 expensive 작업은 다 수행됨

- cancellation의 목적(불필요한 작업 방지)을 달성하지 못함

👉 취소 체크는 '작업 시작 전' 또는 '반복 중'에 있어야 합니다

-----

### 올바른 위치 감각

**1️⃣ 작업 시작 직전**

```swift
Task {
    do {
        try Task.checkCancellation()    // 작업 시작 전
        let data = await loadData()
    } catch {
        // ...
    }
}
```

- 이미 취소된 Task라면 expensive한 작업을 시작할 이유가 없음 -> 작업 진입 차단용 체크

**2️⃣ await 이후, side effect 직전(중요)**

```swift
Task {
    do {
        try Task.checkCancellation()

        let data = await loadData()

        try Task.checkCancellation()    // 상태 변경 / UI 업데이트 직전 확인
        self.items = data
    }
}
```

await 이후에는 다음이 가능해집니다.

- Task가 중단된 사이에 cancel 요청 수신

- 화면 이탈, Task 교체, ViewModel 해제 등

이 상태에서 side effect(`@Published`, UI 변경 등)가 발생하면 논리적으로 잘못된 상태 변경이 될 수 있습니다.

👉 side effect 보호용 체크

**3️⃣반복 루프 내부**

```swift
for item in items {
    if Task.isCancelled { break }
    await process(item)
}
```

**4️⃣ Actor 내부 long-running 작업**

```swift
actor DataStore {
    func refresh() async throws {
        try Task.checkCancellation()
        await fetchRemoteData()
        try Task.checkCancellation()
        await save()
    }
}
```

-----

### 정리

| 구분 | Task.isCancelled | Task.checkCancellation |
| :-- | :-- | :-- |
| 반환 | Bool | throw |
| 제어 방식 | 수동 분기 | 즉시 중단 |
| 사용 목적 | "계속할까?" | "여기서 끝내!" |
| 흐름 명확성 | 낮음 | 매우 높음 |

👉 명확한 중단이 필요하면 checkCancellation()
👉 부드러운 종료면 isCancelled
