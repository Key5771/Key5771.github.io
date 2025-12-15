---
title: "URLSession"
tags: [iOS]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

## 다시 정리하는 URLSession 기초

iOS에서 네트워크 통신을 구현할 때는 대부분은 Alamofire 같은 라이브러리를 사용합니다. 저 역시 실무에서 Alamofire를 주로 사용해왔습니다.

하지만 어느 순간부터 URLSession에 대해 개념적으로 명확하게 설명하기 어렵다는 느낌이 들었고, 기초적인 사용법조차 공식 문서를 다시 확인해야 하는 상황이 잦아졌습니다.

Alamofire는 결국 URLSession 위에서 동작합니다. URLSession의 구조와 역할을 제대로 이해하지 못하면, 네트워크 레이어를 커스터마이징 해야 하거나 문제가 발생했을 때 근본적인 원인을 파악하기 어렵습니다.

그래서 이 글에서는 라이브러리 사용 전에 꼭 알고 있어야 할 URLSession의 기초 개념과 기본 사용법을 정리해보려고 합니다.

-----

### URLSession이란

URLSession이란 iOS에서 HTTP 기반 네트워크 통신을 담당하는 핵심 API입니다.
서버와 데이터를 주고받는 과정에서 다음과 같은 역할을 수행합니다.

- 요청(Request) 전송

- 서버 응답(Response) 수신

- 에러 처리

- 캐시, 쿠키, 인증 관리

URLSession은 단순히 네트워크 요청을 보내는 객체가 아니라, 네트워크 작업을 관리하는 세션(Session) 개념을 중심으로 설계되어 있습니다.

-----

### URLSession의 기본 구성 요소

URLSession을 이해하려면 함께 사용되는 타입들을 먼저 살펴볼 필요가 있습니다.

#### URLRequest

서버로 보낼 요청을 표현하는 객체입니다.

- 요청 URL

- HTTP Method

- Header

- Body

```swift
var request = URLRequest(url: URL(string: "https://example.com")!)
request.httpMethod = "GET"
```

#### URLResponse / HTTPURLResponse

서버로부터 받은 응답을 나타냅니다.
HTTP 통신의 경우 대부분 HTTPURLResponse로 캐스팅해서 사용합니다.

```swift
if let response = response as? HTTPURLResponse {
    print(response.statusCode)
}
```

#### URLSession

실제 네트워크 작업을 수행하는 객체입니다.
URLSession은 설정(Configuration)을 기반으로 생성됩니다.

-----

### URLSessionConfiguration

URLSessionConfiguration은 URLSession의 동작 방식을 결정합니다.
대표적으로 세 가지 기본 설정이 제공됩니다.

#### default

- 캐시, 쿠키, 인증 정보 저장

- 가장 일반적인 설정

```swift
let session = URLSession(configuration: .default)
```

#### ephemeral

- 캐시, 쿠키, 인증 정보를 저장하지 않음

- 로그인, 민감한 요청에 적합

```swift
let session = URLSession(configuration: .ephemeral)
```

#### background

- 앱이 종료되어도 네트워크 작업 지속

- 대용량 파일 다운로드에 사용

-----

### URLSessionTask

URLSession은 실제 작업을 URLSessionTask라는 단위로 수행합니다.

- `URLSessionDataTask`

- `URLSessionUploadTask`

- `URLSessionDownloadTask`

이 중 가장 많이 사용하는 것은 `URLSessionDataTask` 입니다.

-----

### 기본적인 데이터 요청(Completion Handler 방식)

가장 기본적인 사용 방법은 `dataTask(with:completionHandler:)`를 사용하는 것입니다.

```swift
let task = session.dataTask(with: request) { data, response, error in 
    if let error = error {
        print("Error: \(error)")
        return
    }

    guard let data = data else { return }
    print("Response Data: \(data)")
}

task.resume()
```

여기서 중요한 점은 task를 생성한 뒤 반드시 `resume()`을 호출해야 요청이 시작된다는 점입니다.
`resume()`을 호출하지 않으면 네트워크 요청은 실행되지 않습니다.

-----

### 응답 처리 시 주의할 점

#### HTTP 상태 코드 확인

네트워크 요청이 성공적으로 끝났더라도 HTTP 상태 코드는 반드시 확인해야 합니다.

```swift
guard let httpResponse = response as? HTTPURLResponse,
      (200..<300).contains(httpResponse.statusCode) else {
        return
}
```

#### 메인 스레드

CompletionHandler는 메인 스레드를 보장하지 않습니다.
UI 업데이트가 필요하다면 명시적으로 메인 스레드로 전환해야 합니다.

```swift
DispatchQueue.main.async {
    // UI update
}
```

-----

### async/await 기반 URLSession

iOS 15부터는 async/await을 사용하는 API가 제공됩니다.

```swift
let (data, response) = try await session.data(for: request)
```

이 방식은 다음과 같은 장점이 있습니다.

- 비동기 흐름이 직관적

- 에러 처리가 명확

- 중첩된 콜백 제거

HTTP 응답 검증도 훨씬 간결해집니다.

```swift
guard let httpResponse = response as? HTTPURLResponse,
      (200..<300).contains(httpResponse.statusCode) else {
        throw NetworkError.invalidResponse
}
```

-----

### Alamofire를 쓰더라도 URLSession을 알아야 하는 이유

Alamofire는 URLSession을 감싸서 편의 기능을 제공한는 라이브러리입니다.
하지만 다음과 같은 상황에서는 URLSession에 대한 이해가 반드시 필요합니다.

- 네트워크 레이어 커스터마이징

- 비동기 / 동시성 이슈 디버깅

- 성능 또는 캐시 문제 분석

- 라이브러리 의존성을 줄여야 하는 경우

URLSession의 동작 원리를 이해하면, 라이브러리를 사용할 때도 어떤 설정이 내부적으로 적용되는지 예측할 수 있습니다.

-----

### 마무리

URLSession은 iOS 네트워크 통신의 가장 기본이 되는 API 입니다.
라이브러리를 사용하고 있다면 더욱 더, 그 기반이 되는 URLSession을 한 번쯤은 정리해 볼 필요가 있습니댜.
