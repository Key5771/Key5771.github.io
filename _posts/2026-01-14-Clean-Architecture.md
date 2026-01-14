---
title: "iOS Clean Architecture"
tags: [CS]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

### iOS 클린 아키텍처 정리

Repository / UseCase / ViewModel / View의 역할과 경계

클린아키텍처를 iOS에 적용할 때 가장 많이 헷갈리는 부분은 아래의 내용이라고 생각합니다.

> 이 로직은 ViewModel일까?
> 아니면 UseCase 일까?
> Repository는 어디까지 해야하지?

이 글의 목표는 하나입니다.

> 각 레이어가 '무엇을 책임지고, 무엇을 절대 하지 말아야 하는지'를 코드로 명확히 이해하는 것

-----

#### 전체 구조 한눈에 보기

```
View
 ↓
ViewModel
 ↓
UseCase
 ↓
Repository
 ↓
Data Source (API / DB)
```

핵심 원칙은 단 하나입니다.

의존성은 항상 아래로만 흐릅니다. (UI -> Domain -> Data)

-----

#### 1. Repository: "데이터를 어디서 가져올 것인가"

Repository는 '데이터 접근 방법'을 책임집니다.

- API에서 오든

- DB에서 오든

- 캐시에서 오든

👉 사용하는 쪽은 몰라도 됩니다.

**Repository가 하는 일**

- 네트워크 / DB 호출

- 데이터 소스 선택

- DTO -> Entity 변환

**Repository가 하면 안 되는 일 ❌**

- 사용자 상태 판단

- 정책 / 조건 분기

- UI를 위한 가공

**Repository 예제**

```swift
protocol UserRepository {
    func fetchUser() async throws -> User
}
```

```swift
final class UserRepositoryImpl: UserRepository {
    func fetchUser() async throws -> User {
        // API 호출이라고 가정
        return User(
            id: "1",
            name: "Kim",
            isBlocked: false,
            isPremium: true
        )
    }
}
```

👉 Repository는 “그냥 데이터만” 줍니다.

-----

#### 2. UseCase: "이 앱에서 무엇이 가능한가"

UseCase는 앱의 '행동'과 '비즈니스 규칙'을 표현합니다.

- 여러 Repository 조합

- 조건 판단

- 정책 결정

👉 UI 없이도 의미가 있는 로직

**UseCase가 다루는 질문**

- 이 사용자는 이 행동을 할 수 있는가

- 이 요청은 허용되는가

- 어떤 결과를 반환해야 하는가

**UseCase 예제**

```swift
protocol FetchAvailableUserUseCase {
    func execute() async throws -> User
}
```

```swift
final class FetchAvailableUserUseCaseImpl: FetchAvailableUserUseCase {
    private let userRepository: UserRepository

    init(userRepository: UserRespository) {
        self.userRepository = userRepository
    }

    func execute() async throws -> User {
        let user = try await userRepository.fetchUser()
        
        // ✅ 비즈니스 정책
        guard !user.isBlocked else {
            throw UserError.blocked
        }

        return user
    }
}
```

```swift
enum UserError: Error {
    case blocked
}
```

👉 이래서 UseCase에 있어야 한다.

- "차단된 사용자는 사용할 수 없다"

- 이건 UI와 무관한 앱 규칙

-----

#### 3. ViewModel: "어떻게 보일 것인가"

ViewModel은 도메인 결과를 UI가 쓰기 쉬운 상태로 변환합니다.

- UseCase 호출

- 결과 -> ViewState 변환

- UI 이벤트 처리

**ViewModel이 처리해도 되는 것 ✅**

- 사용자 정보에 따른 UI 분기

- 버튼 활성화 여부

- 뱃지 / 문구 표시 여부

👉 프레젠테이션 로직

**ViewModel 예제**

```swift
struct ProfileViewState {
    let userName: String
    let showPremiumBadge: Bool
}
```

```swift
@MainActor
final class ProfileViewModel {
    private let fetchUserUseCase: FetchAvailableUserUseCase

    init(fetchUserUseCase: FetchAvailableUserUseCase) {
        self.fetchUserUseCase = fetchUserUseCase
    }

    var state: ProfileViewState?

    func load() async {
        do {
            let user = try await fetchUserUseCase.execute()

            // ✅ UI 표현 로직
            state = ProfileViewState(
                userName = user.name,
                showPremiumBadge: user.isPremium
            )
        } catch UserError.blocked {
            // UI에서 에러 처리
            print("차단된 사용자")
        } catch {
            print("알 수 없는 에러")
        }
    }
}
```

👉 이래서 ViewModel 책임

- isPremium -> "뱃지를 보여줄까?"

- 이건 UI 표현 규칙

-----

#### 4. View: "그린다"

View는 상태를 받아서 화면에 그리는 것만 합니다.

- 로직 ❌

- 판단 ❌

- 상태 변경 ❌

**View(SwiftUI 예제)**

```swift
struct ProfileView: View {

    @StateObject var viewModel: ProfileViewModel

    var body: some View {
        VStack {
            if let state = viewModel.state {
                Text(state.userName)

                if state.showPremiumBadge {
                    Text("✨ Premium")
                }
            }
        }
        .task {
            await viewModel.load()
        }
    }
}
```

👉 View는 조건을 판단하지 않습니다.

👉 이미 ViewModel이 만들어준 상태만 사용합니다.

-----

#### 이 구조의 핵심 판단 기준 정리

아래 질문의 결과에 따라 분류할 수 있습니다.

> ❓ 이 로직이 UI 없이도 의미가 있는가?

- YES -> UseCase

- NO -> ViewModel

**같은 데이터, 다른 위치 예시**

```swift
user.isPremium
```

| 사용 목적 | 위치 |
| :-- | :-- |
| 프리미엄 뱃지 표시 | ViewModel |
| 프리미엄만 결제 가능 | UseCase |

👉 데이터가 아니라 '의미'가 기준

-----

#### 한 장으로 정리

| 레이어 | 책임 | 하지 말아야 할 것 |
| :-- | :-- | :-- |
| View | 그리기 | 판단, 로직 |
| ViewModel | UI 상태 변환 | 비즈니스 정책 |
| UseCase | 행동 / 정책 | UI 고려 |
| Repository | 데이터 접근 | 정책 / 판단 |

**한 줄 요약**

> Repository는 데이터를 가져오고,
> UseCase는 가능 여부를 결정하고,
> ViewModel은 어떻게 보일지 만들고,
> View는 그린다.

이 기준만 지킨다면 클린 아키텍처는 어렵지 않고, 오히려 코드가 말이 됩니다.