---
title: "intrinsicContentSize"
tags: [iOS]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

[https://developer.apple.com/documentation/uikit/uiview/intrinsiccontentsize](https://developer.apple.com/documentation/uikit/uiview/intrinsiccontentsize)

```swift
var intrinsicContentSize: CGSize { get }
```

- CustomView의 경우 일반적으로 레이아웃 시스템이 인식하지 못하는 내용이 표시
- intrinsicContentSize를 설정하면 CustomView가 컨텐츠 사이즈에 따라 원하는 크기를 레이아웃 시스템에 전달


intrinsicContentSize를 override하여 size를 유연하게 적용할 수 있다.
```swift
override var intrinsicContentSize: CGSize {
    if UIDevice.current.userInterfaceIdiom == .phone {
        return CGSize(width: 100, height: 100)
    } else {
        return CGSize(width: 200, height: 200)
    }
}
```