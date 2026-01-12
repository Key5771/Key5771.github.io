---
title: "AppIntent"
tags: [iOS]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

### AppIntent

iOS 16부터 도입된 AppIntent는 Siri, Spotlight, App Shortcut 등 다양한 시스템 진입 경로를 하나의 선언형 API로 통합해주는 프레임워크입니다.

Apple은 AppIntent를 통해 "사용자가 앱을 열지 않아도, 앱의 핵심 기능을 시스템 전반에서 실행할 수 있게" 만드는 것을 목표로 합니다.

-----

### AppIntent의 기본 개념

AppIntent는 크게 다음 역할을 가집니다.

- 사용자가 실행할 수 있는 행동(Action)을 정의

- Siri / Spotlight / App Shortcut에서 호출 가능

- 필요 시 앱을 자동으로 실행(openAppWhenRun)

간단한 예를 들면:

```swift
static SearchProjectIntent: AppIntent {
    static var title: LocalizedStringResource = "홈 화면 열기"
    static var description = IntentDescription("앱의 홈 화면으로 이동합니다")
    static var openAppWhenRun: Bool = false

    func perform() async throws -> some IntentResult {
        // Intent 실행 로직
        return .result()
    }
}
```

주요 구성 요소

| 요소 | 역할 |
| :-- | :-- |
| title | Siri/Shotcut에 표시될 이름 |
| description | Intent 설명 |
| openAppWhenRun | Intent 실행 시 앱 자동 실행 여부 |
| perform() | 실제 실행 로직 |

이렇게 정의된 Intent는:

- Siri 음성 명령

- Spotlight 검색

- App Shortcut

등을 통해 실행될 수 있습니다.

#### openAppWhenRun의 의미

```swift
static var openAppWhenRun: Bool = true
```

이 값이 true 이면:

- Intent 실행 시 앱이 자동으로 실행됩니다.

- Siri나 Shortcut에서 실행해도 앱이 열립니다.

반대로 false이면:

- 앱 UI 없이 백그라운드 작업만 수행할 수도 있습니다.

-----

### AppIntent에 파라미터 추가

Intent는 사용자 입력값을 받을 수 있습니다.

```swift
struct SearchIntent: AppIntent {
    static var title: LocalizedStringResource = "검색"
    static var openAppWhenRun: Bool = true

    @Parameter(title: "검색어")
    var keyword: String

    func perfomr() async throws -> some IntentResult {
        // 키워드 사용
        return .result()
    }
}
```

이렇게 하면:

- Siri에서 "OOO 검색해줘"

- Shortcut에서 검색어 입력

같은 UX를 자연스럽게 제공할 수 있습니다.

-----

### Parameter Summary로 자연스러운 문장 만들기

```swift
static var parameterSummary: some ParameterSummary {
    Summary("검색어 \\(\\.$keyword)로 검색")
}
```

이 설정은 Shortcut 앱에서 "이 단축어가 무슨 일을 하는지"를 사람이 읽기 좋은 문장으로 보여줍니다.

-----

### Intent 실행 결과에 Dialog 제공하기

Siri나 Shortcut 실행 후 사용자에게 음성 / 텍스트 피드백을 주고 싶다면 `ProvidesDialog`를 사용할 수 있습니다.

```swift
func perform() async throws -> some IntentResult & ProvidesDialg {
    return .result(dialog: "검색 결과를 보여드릴게요")
}
```

이렇게 하면

- Siri가 결과를 읽어주거나

- Shortcut에서 실행 결과가 표시됩니다.

----

### App Shortcut에 Intent 노출하기

Intent를 만들었다고 해서 자동으로 App Shortcut에 나타나지 않습니다.

AppShortcutsProvider를 통해 명시적으로 등록해야 합니다.

```swift
struct MyAppShortcuts: AppShortsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: SearchIntent(),
            phrases: [
                "${applicationName}에서 검색",
                "${applicationName} 검색해줘"
            ],
            shortTitle: "검색",
            systemImageName: "magnifyingglass"
        )
    }
}
```

이제 사용자는 Spotlight / Siri 제안 / Shortcut 앱에서 이 Intent를 사용할 수 있습니다.

-----

### AppIntent를 처음 구현할 때 기억할 점

- AppIntent는 UI가 아니라 행동(Action)입니다.

- 어디서 호출되는지는 시스템이 결정합니다.

- 개발자는 Intent의 의미와 결과만 정의하면 됩니다.

-----

### 마무리

AppIntent는 "앱 기능을 시스템과 자연스럽게 연결하기 위한 API" 입니다.