---
title: "Swift withCheckedContinuation"
tags: [Swift]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

## Swift withCheckedContinuation 완벽 이해하기

Swift Concurrency(async/await)는 비동기 흐름을 명확하고 안전하게 만들어주지만,
기존의 콜백 기반 API나 Delegate 기반 API는 그대로 async 함수로 사용할 수 없습니다.

이런 기존 비동기 API를 async/await 형태로 래핑(wrap)할 때 사용하는 도구가 바로 `withCheckedContinuation`입니다.

>withCheckedContinuation
콜백 기반 비동기 코드를 Swift async/await로 변환하는 안전한 브릿지
>

### Checked Continuation이란?

Continuation은 "나중에 재개될 코드 조각" 정도로 이해하면 쉽습니다.

withCheckedContinuation은 Swift에게 이렇게 말하는 것과 같습니다.

```bash
"내가 비동기 콜백을 하나 async 함수처럼 바꾸고 싶어.
이 continuation을 특정 시점에 불러서 async 흐름을 재개시켜줘"
```

예를 들어 기존 콜백 코드:

```swift
fetchUser { user in 
    print(user)
}
```

이걸 async/await 형태로 변환하면

```swift
let user = await fetchUser()
```

이 둘 사이의 다리가 바로 `withCheckedContinuation` 입니다.

### 기본 사용 예

기존 콜백 기반 API

```swift
func loadUser(id: String, completion: @escaping (User) -> Void) {
    api.fetch(id: id, completion: completion)
}
```

async/await으로 감싸기

```swift
func loadUser(id: String) async -> User {
    return await withCheckedContinuation { continuation in 
        loadUser(id: id) { user in 
            continuation.resume(returning: user)
        }
    }
}
```

이렇게 변경하면 다음과 같이 사용이 가능합니다.

```swift
let user = await loadUser(id: "1234")
```

### Checked Continuation의 핵심: "검증 가능"

Checked Continuation은 Swift가 continuation 사용을 검증해줍니다.

Swift는 실행 중 다음을 체크합니다.

**resume을 반드시 호출했는지**

호출하지 않으면 async 함수가 영원히 대기 상태가 되어 memory leak이나 deadlock이 발생합니다.

**resume을 여러 번 호출하지 않았는지**

두 번 호출하면 크래시가 발생합니다.

예:
```swift
continuation.resume(returning: user)
continuation.resume(returning: user)    // ❌ 런타임 크래시
```

그렇기 때문에 "Checked"가 버그를 조기에 찾을 수 있어 강력합니다.

### withCheckedContinuation vs withCheckedThrowingContinuation

##### withCheckedContinuation

성공만 있는 케이스

```swift
withCheckedContinuation { continuation in 
    continuation.resume(returning: value)
}
```

##### withCheckedThrowingContinuation

에러를 던질 수 있는 async 함수로 변환할 때

```swift
func fetch() async throws -> String {
    return try await withCheckedThrowingContinuation { continuation in 
        api.call { result in
            switch result {
            case .success(let value):
                continuation.resume(returning: value)
            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
    }
}
```

### Checked vs Unchecked 차이

Swift에는 두 종류가 있습니다.

| 함수 | 특징 |
| :-- | :-- |
| withCheckedContinuation | 안정성 검증 -> resume 누락/중복 사용 체크 O |
| withUnsafeContinuation | 검증 없음 -> resume 실수 시 undefined behavior |

### 왜 continuation이 필요한가?

**async/await은 Swift Concurrency 시스템을 따른다**

하지만 기존 콜백 / delegate는 Swift Concurrency 규칙과 별개입니다.

**bridging이 필요하다**

기존 라이브러리 대부분이 여전히 콜백 기반이며 이를 async 함수로 바꾸기 위해 continuation이 필요합니다.

### 주의해야 할 점

**1. resume은 반드시 한 번만 호출할 것**

중복 resume은 크래시 -> Checked가 잡아줍니다.

**2. escape 하지 않도록 주의**

continuation을 저장해두고 나중에 사용하는 것은 위험합니다.
async 함수의 생명주기와 맞지 않아 장기적으로 문제가 발생할 수 있습니다.

```swift
var stored: CheckedContinuation<User, Never>? // ❌ 잘못된 패턴

func fetchAsync() async -> User {
    return await withCheckedContinuation { continuation in 
        stored = continuation   // 위험
    }
}
```

**3. callback이 메인 스레드에서 실행된다고 가정하면 안됩니다**

필요하면 명시적으로 `@MainActor` 또는 `DispatchQueue.main.async` 사용