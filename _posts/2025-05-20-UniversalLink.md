---
title: "Universal Link"
tags: [iOS]
categories: [Study]
layout: single
author_profile: true
classes: wide
---

### 유니버셜 링크(Universal Link)
- iOS에서 딥링크를 구현하는 방법 중 하나로 <b>웹에서 앱을 호출하는 기능</b>이 필요할 때 사용
<br>
`딥링크(DeepLink) : 특정 주소나 값을 입력하면 앱이 실행되거나 앱 내 특정 페이지에 랜딩할 수 있는 링크`

1. URI 스킴 : 앱에 URI 스킴(Scheme) 값을 등록하여 딥링크 사용
2. 앱 링크(App Link) : Android 제공 - 도메인 주소를 이용한 딥링크 사용
3. 유니버셜 링크(Universal Link) : iOS 제공 - 도메인 주소를 이용한 딥링크 사용

<img src="/assets/2025-05-20/image.png" width="100%"/>

#### 유니버셜 링크의 사용
- QR코드, NFC 인식 시 앱으로 연결
- 웹 광고 배너 클릭 시 앱 이동을 통한 앱 홍보

#### 유니버셜 링크의 특징
- IP 및 HTTP에서는 동작하지 않음
- SSL이 있는 백엔드 필요
- 허용 여부를 묻지 않음
- 지원 iOS 버전 : iOS 9 이상

#### 유니버셜 링크 설정 과정
- 웹 서버 작업 : 웹 서버에 AASA(apple-app-site-association) 파일 추가
- 앱 작업 : Associated Domain 추가

------
### 사용 방법
#### 1. 웹 서버에 AASA 파일 추가
Universal Link를 사용하기 위해서는 도메인이 필요
따라서 도메인 정보가 포함된 JSON 파일인 apple-app-site-association(AASA) 파일을 웹 서버에 업로드
- 사용자가 앱 설치 시, 이 파일에 등록된 도메인으로 apple-app-site-association 파일에 대한 요청을 보냄
- 또, 사용자가 이미 앱을 설치했다면 앱을 바로 실행해주는 도메인에서 호스팅

```
주의할점
- Redirection이 없어야 함
- HTTPS 지원
- 128KB 보다 작아야 함
```

#### apple-app-site-association 파일
```json
iOS 13 이상
{
    "applinks": {
        "details": {
            "appIDs": ["<TEAM_DEVELOPER_ID>.<BUNDLE_IDENTIFIER>"],
            "components": [
                { }
            ]
        }
    }
}

iOS 12 이하
{
    "applinks": {
        "apps": [],
        "details": [
            {
                "appID": "<TEAM_DEVELOPER_ID>.<BUNDLE_IDENTIFIER>",
                "paths": ["*"]
            },
            {
                "appID": "<TEAM_DEVELOPER_ID>.<BUNDLE_IDENTIFIER>",
                "paths": ["/files"]
            }
        ]
    }
}
```
- apps : 유니버셜 링크에서는 사용하지 않지만 빈 배열로라도 반드시 명시되어 있어야 함.
- details : 웹사이트에서 핸들링되는 앱들의 목록입니다. 즉 한 웹사이트에서 유니버셜 링크를 사용하는 여러 앱 연동이 가능.
- appId : 앱의 식별 값으로 팀 아이디와 번들 id를 사용. 형식:<TEAM_DEVELOPER_ID>.<BUNDLE_IDENTIFIER>
- paths : 앱에서 지원하는 웹사이트 경로(*: 전 위치 가능)

<img src="/assets/2025-05-20/image2.png" width="100%"/>

#### 2. iOS 앱에 Capability - Associated Domains  추가
유니버셜 링크를 지원하려면, 먼저 앱에서 유니버셜 링크를 허용할 도메인을 추가
Project > Target > Signing&Capabilities> Associated Domains에 Domains 추가

<img src="/assets/2025-05-20/image3.png" width="100%"/>
* 등록이 잘 됐는지 entitlements로 확인
<br><br>

#### 3. 앱에서 동적 링크 URL Handle
유니버셜 링크로 들어오는 요청은 AppDelegate의 `application(_:continue:restoreationHandler:)` 메소드에서 전달받은 NSUserActivity 객체를 사용해 처리 가능

```swift
func application(_ application: UIApplication,
                 continue userActivity: NSUserActivity,
                 restorationHandler: @escaping ([UIUserActivityRestoring]?) -> Void) -> Bool
{
    // Get URL components from the incoming user activity.
    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
        let incomingURL = userActivity.webpageURL,
        let components = NSURLComponents(url: incomingURL, resolvingAgainstBaseURL: true) else {
        return false
    }

    // Check for specific URL components that you need.
    guard let path = components.path,
    let params = components.queryItems else {
        return false
    }    
    print("path = \(path)")

    if let albumName = params.first(where: { $0.name == "albumname" } )?.value,
        let photoIndex = params.first(where: { $0.name == "index" })?.value {


        print("album = \(albumName)")
        print("photoIndex = \(photoIndex)")
        return true
    } else {
        print("Either album name or photo index missing")
        return false
    }
}
```

앱이 Scene에 참여했고, 앱이 실행 중이 아닌 경우, 시스템은 실행 후에 `scene( _:willConnectTo:options:)` 메소드에 대한 범용 링크를 전달하고, 앱이 실행 중이거나 메모리에서 일시 중단된 동안 유니버셜 링크가 탭 되면 `scene( _:continue:)`에 전달
```swift
func scene(_ scene: UIScene, willConnectTo
           session: UISceneSession,
           options connectionOptions: UIScene.ConnectionOptions) {
    
    // Get URL components from the incoming user activity.
    guard let userActivity = connectionOptions.userActivities.first,
        userActivity.activityType == NSUserActivityTypeBrowsingWeb,
        let incomingURL = userActivity.webpageURL,
        let components = NSURLComponents(url: incomingURL, resolvingAgainstBaseURL: true) else {
        return
    }


    // Check for specific URL components that you need.
    guard let path = components.path,
        let params = components.queryItems else {
        return
    }
    print("path = \(path)")


    if let albumName = params.first(where: { $0.name == "albumname" })?.value,
        let photoIndex = params.first(where: { $0.name == "index" })?.value {
        
        print("album = \(albumName)")
        print("photoIndex = \(photoIndex)")
    } else {
        print("Either album name or photo index missing")
    }
}
```

> **Warning**
> 경고
> 유니버설 링크는 앱에 잠재적인 공격 경로를 제공하므로 모든 URL 매개변수의 유효성을 검사하고 잘못된 URL은 삭제해야한다. 
> 또한, 사용자 데이터를 위험에 빠뜨리지 않는 작업으로만 사용 가능한 작업을 제한해야 한다. 
> 예를 들어, 유니버설 링크가 콘텐츠를 직접 삭제하거나 사용자에 대한 민감한 정보에 접근하도록 허용 X
> URL 처리 코드를 테스트할 때는 테스트 케이스에 잘못된 형식의 URL이 포함되어 있는지 확인.