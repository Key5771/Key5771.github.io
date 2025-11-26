---
title: "UIStackView"
tags: [iOS]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

### UIStackView
열 또는 행에 뷰 집합을 배치하기 위한 간소화된 인터페이스

StackView를 사용하면 AutoLayout의 기능을 활용하여 Device의 방향, 화면의 크기 및 사용 가능한 공간의 변경 사항에 동적으로 적용할 수 있는 사용자 인터페이스를 만들 수 있다. 
StackView는 정렬된 subviews 속성에 있는 모든 View의 레이아웃을 관리한다. 
이러한 StackView는 정렬된 하위 View 배열의 순서에 따라 StackView의 축을 따라 정렬된다. 
정확한 레이아웃은 StackView의 axis, distribution, alignment, spacing 및 기타 특성에 따라 달라진다.

<img src="/assets/2025-04-09/image.png" width="70%"/>


#### Axis
- 이 속성을 정렬된 View의 방향을 결정.
- NSLayoutConstraint.Axis.vertical 값은 View를 열로 정의
- NSLayoutConstraint.Axis.horizontal 값은 View를 행으로 정의
- 기본 값은 NSLayoutConstraint.Axis.horizontal


#### Distribution
- 이 속성은 StackView가 축을 따라 정렬된 View를 배치하는 방법을 결정
- 기본 값은 UIStackView.Distribution.fill


##### fill
- 정렬된 View가 StackView 내에 맞지 않으면 Compression resistance 우선순위에 따라 View가 축소
- 정렬된 View가 StackView 내에 맞지 않으면 Hugging 우선순위에 따라 View가 확장
- 모호한 경우 StackView는 정렬된 하위 View 배열의 인덱스를 기준으로 정렬된 View의 크기를 조정

<img src="/assets/2025-04-09/image-1.png" width="70%"/>

##### fillEqually
- StackView의 축을 따라 모두 동일한 크기가 되도록 View 크기가 조정된다.

<img src="/assets/2025-04-09/image-2.png" width="70%"/>


##### fillProportionally
- View의 크기는 StackView 축을 따라 고유한 내용 크기에 따라 비례적으로 조정

<img src="/assets/2025-04-09/image-3.png" width="70%"/>

##### equalSpacing
- 정렬된 View가 StackView를 채우지 않으면 View 사이의 간격이 균일하게 적용
- 정렬된 View가 StackView 내에 맞지 않으면 Compression resistance 우선순위에 따라 View가 축소
- 모호한 경우 StackView는 정렬된 하위 View 배열의 인덱스를 기준으로 View를 축소

<img src="/assets/2025-04-09/image-4.png" width="70%"/>

##### equalCentering
- 정렬된 View가 StackView 내에 맞지 않으면 간격 특성에 의해 정의된 최소 간격에 도달할 때까지 간격을 줄인다.
- 그래도 View가 맞지 않으면 StackView는 Compression resistance 우선순위에 따라 정렬된 View를 축소
- 모호한 경우 StackView는 정렬된 하위 View 배열의 인덱스를 기준으로 View를 축소

<img src="/assets/2025-04-09/image-5.png" width="70%"/>


#### Alignment
- 이 속성은 StackView가 정렬된 View를 축에 수직으로 배치하는 방법을 결정
- 기본 값은 UIStackView.Alignment.fill

##### fill

<img src="/assets/2025-04-09/image-6.png" width="70%"/>


##### center

<img src="/assets/2025-04-09/image-7.png" width="70%"/>

##### leading
- 이 값은 Horizontal StackView 스택의 맨 위 정렬과 동일

<img src="/assets/2025-04-09/image-8.png" width="70%"/>

##### trailing
- 이 값은 Horizontal StackView의 맨 아래 정렬과 동일

<img src="/assets/2025-04-09/image-9.png" width="70%"/>

##### firstBaseline

<img src="/assets/2025-04-09/image-10.png" width="70%"/>

##### lastBaseline

<img src="/assets/2025-04-09/image-11.png" width="70%"/>

##### static var top: UIStackView.Alignment { get }
- 이 값은 Vertical StackView에서 UIStackView.Alignment.leading과 동일

<img src="/assets/2025-04-09/image-12.png" width="70%"/>

##### static var bottom: UIStackView.Alignment { get }
- 이 값은 Vertical StackView에서 UIStackView.Alignment.trailing과 동일

<img src="/assets/2025-04-09/image-13.png" width="70%"/>


#### Spacing
- 이 속성은 UIStackView에 대해 정렬된 View 사이의 간격을 정의
- 기본 값은 0.0

##### UIStackView | Apple Developer Documentation
[https://developer.apple.com/documentation/uikit/uistackview](https://developer.apple.com/documentation/uikit/uistackview)

