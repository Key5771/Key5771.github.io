---
title: "Property Wrapper"
tags: [Swift]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

## Property Wrapper
애플 문서에는 다음과 같이 설명하고 있다.
> When you use a property wrapper, you write the management code once when you define the wrapper,
and then reuse that management code by applying it to multiple properties.

Property Wrapper를 사용하면 wrapper를 정의할 때 한 번만 코드를 정의하고 관리할 수 있다.
그리고 이 wrapper를 여러 프로퍼티에 재활용하여 적용시킬 수 있다.

한 번만 정의하고 재활용을 해야하는 경우는 어떤게 있을까?
예를 들어, 도시와 나라에 대한 정보를 가지고 있는 타입이 있다고 생각해보자.

```swift
struct City {
    private var _name: String = ""

    var cityName: String {
        get {
            return _name.uppercased()
        }
        set {
            _name = newValue
        }
    }

    init(cityName: String) {
        self.cityName = cityName
    }
}

struct Country {
    private var _name: String = ""

    var countryName: String {
        get {
            return _name.uppercased()
        }
        set {
            _name = newValue
        }
    }

    init(countryName: String) {
        self.countryName = countryName
    }
}
```

다음과 같이 다른 2개의 타입이 있지만 코드는 거의 중복으로 이루어지고 있다.
이런 경우 Property Wrapper를 사용할 수 있다.

Property Wrapper를 정의할 때, 구조체 / 열거형 / 클래스에서는 `wrappedValue`라는 프로퍼티를 만들어야 한다.
UpperCase로 변환하는 Property Wrapper를 구현하면 다음과 같다.

```swift
@propertyWrapper
struct UpperCase {
    private var value: String = ""

    var wrappedValue: String {
        get { return value.uppercased() }
        set { value = newValue }
    }

    init(wrappedValue: String) {
        self.wrappedValue = wrappedValue
    }
}
```

다음과 같은 Property Wrapper를 사용하는 방법은 다음과 같다.
```swift
struct City {
    @UpperCase var cityName: String

    init(cityName: String) {
        self.cityName = cityName
    }
}

struct Country {
    @UpperCase var countryName: String

    init(countryName: String) {
        self.countryName = countryName
    }
}
```

이렇게 중복되는 코드들을 한 곳에서 관리할 수 있게 해주는 것이 Property Wrapper라고 소개하고 있다.
