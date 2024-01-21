---
title: '[golang web] google oauth2 소셜로그인 개념 및 구현'
categories:
  - golang web
tags:
  - [golang, web, oauth2]
toc: true
toc_sticky: true
date: 2024-01-21
last_modified_at: 2024-01-21
---

## 1. 어떤 정보인가?

많은 웹 사이트, 앱에서 별도 회원가입을 하지 않고 Google, Facebook, Kakao, Naver 등 기존에 가입되어 있는 계정 정보로 서비스를 이용하는 경우를 많이 경험해보았을 것이다.

즉, Oauth는 다른 웹사이트 상의 자신들의 정보에 대해서 웹 사이트나 애플리케이션의 접근 권한을 부여할 수 있는 공통적인 수단이다.

사용자 입장에서는 여러 서비스들을 자주 사용하는 하나의 계정으로 관리할 수 있는 편안함을 제공해준다.

개발자 입장에서는 민감한 사용자 정보를 다루지 않아도 되므로 부담감이 훨씬 줄어든 상태로 사용자 정보를 활용할 수 있다는 간편함을 제공해준다.

## 2. 정보 설명

### 2-1. Oauth 로직

![스크린샷 2024-01-21 14 48 24](https://github.com/may-30/may-30.github.io/assets/155306250/8370754c-15ef-4ecc-9c1a-39c671b3b563)

Oauth의 로직은 위 사진과 같다.

1. 사용자가 이용하려는 웹 서비스에 로그인 요청을 한다.

2. 로그인 요청을 받은 웹 서비스는 서비스 제공자(여기서는 Google) 에게 Oauth 정보를 담아서 요청한다.

3. Oauth 정보가 정확하다면 접근하려는 사용자의 정보를 활용할 수 있는 URL을 제공하는데 이를 Callback URL이라고 한다.

4. 서비스 제공자(Google)은 사용자에게 자신들의 로그인 페이지를 제공한다.

5. 사용자는 제공된 로그인 페이지에 로그인을 한다.

6. 로그인이 성공적으로 되었다면 사용자의 정보를 활용할 수 있는 Callback URL을 서비스 제공자가 웹 서비스에 제공한다.

### 2-2. 코드

```html
<!-- public/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Go OAuth 2.0 Test</title>
</head>
<body>
    <p><a href="./auth/google/login">Google Login</a></p>
</body>
</html>
```

```go
// main.go
package main

// import 문은 생략한다.

// 2. 로그인 요청을 받은 웹 서비스는 서비스 제공자(여기서는 Google) 에게 Oauth 정보를 담아서 요청한다.
var googleOauthConfig = oauth2.Config{
    // 3. Oauth 정보가 정확하다면 접근하려는 사용자의 정보를 활용할 수 있는 URL을 제공하는데 이를 Callback URL이라고 한다.
    RedirectURL: "http://localhost:3000/auth/google/callback",
    ClientID: os.Getenv("GOOGLE_CLIENT_ID"),
    ClientSecret: os.Getenv("GOOGLE_SECRET_KEY"),
    Scopes: []string{"https://www.googleapis.com/auth/userinfo.email"},
    Endpoint: google.Endpoint,
}

// 2. 로그인 요청을 받은 웹 서비스는 서비스 제공자(여기서는 Google) 에게 Oauth 정보를 담아서 요청한다.
func googleLoginHandler(w http.ResponseWriter, r *http.Request) {
    // - 1회성 난수 설정
    state := generateStateOauthCookie(w)

    // 4. 서비스 제공자(Google)은 사용자에게 자신들의 로그인 페이지를 제공한다.
    url := googleOauthConfig.AuthCodeURL(state)

    // 4. 서비스 제공자(Google)은 사용자에게 자신들의 로그인 페이지를 제공한다.
    http.Redirect(w, r, url, http.StatusTemporaryRedirect)
}

// 2. 로그인 요청을 받은 웹 서비스는 서비스 제공자(여기서는 Google) 에게 Oauth 정보를 담아서 요청한다.
func generateStateOauthCookie(w http.ResponseWriter) string {
    // - 만료 시간 설정 (현재 시간으로부터 1일 뒤)
    expiration := time.Now().Add(1 * 24 * time.Hour)

    // - 난수 설정
    b := make([]byte, 16)
    rand.Read(b)
    state := base64.URLEncoding.EncodeToString(b)

    // - cookie 생성
    cookie := &http.Cookie{
        Name: "oauthstate",
        Value: state,
        Expires: expiration,
    }
    http.SetCookie(w, cookie)

    return state
}

// 6. Oauth 정보가 정확하다면 접근하려는 사용자의 정보를 활용할 수 있는 URL을 제공하는데 이를 Callback URL이라고 한다.
func googleAuthCallback(w http.ResponseWriter, r *http.Request) {
    // - cookie에 들어있는 state 정보
    oauthstate, _ := r.Cookie("oauthstate")

    // - cookie에 들어있는 state 정보와 state정보가 일치하는지 판단
    if r.FormValue("state") != oauthstate.Value {
        log.Printf("Invalied google oauth state! cookie: %s, state: %s\n", oauthstate.Value, r.FormValue("state"))
        http.Redirect(w, r, "/", http.StatusTemporaryRedirect)
        return
    }

    // - user info 가져오기
    data, err := getGoogleUserInfo(r.FormValue("code"))
    if err != nil {
        log.Println(err.Error())
        http.Redirect(w, r, "/", http.StatusTemporaryRedirect)
        return
    }

    fmt.Fprint(w, string(data))
}

// 6. Oauth 정보가 정확하다면 접근하려는 사용자의 정보를 활용할 수 있는 URL을 제공하는데 이를 Callback URL이라고 한다.
const googleOauthURLAPI = "https://www.googleapis.com/oauth2/v2/userinfo?access_token="

// 6. Oauth 정보가 정확하다면 접근하려는 사용자의 정보를 활용할 수 있는 URL을 제공하는데 이를 Callback URL이라고 한다.
func getGoogleUserInfo(code string) ([]byte, error) {
    // - token 획득
    token, err := googleOauthConfig.Exchange(context.Background(), code)
    if err != nil {
        return nil, fmt.Errorf("Failed to exchange %s\n", err.Error())
    }

    // - userinfo 획득
    res, err := http.Get(googleOauthURLAPI + token.AccessToken)
    if err != nil {
        return nil, fmt.Errorf("Failed to get userinfo %s\n", err.Error())
    }

    return io.ReadAll(res.Body)
}

func main() {
    mux := pat.New()

    // 2. 로그인 요청을 받은 웹 서비스는 서비스 제공자(여기서는 Google) 에게 Oauth 정보를 담아서 요청한다.
    mux.HandleFunc("/auth/google/login", googleLoginHandler)
    // 6. Oauth 정보가 정확하다면 접근하려는 사용자의 정보를 활용할 수 있는 URL을 제공하는데 이를 Callback URL이라고 한다.
    mux.HandleFunc("/auth/google/callback", googleAuthCallback)

    n := negroni.Classic()
    n.UseHandler(mux)

    http.ListenAndServe(":3000", n)
}
```

### 2-3. 구현 사진

![스크린샷 2024-01-21 15 22 56](https://github.com/may-30/may-30.github.io/assets/155306250/fa55acb6-abf7-43f6-9b37-f98a5b6b7182)

![스크린샷 2024-01-21 15 23 12](https://github.com/may-30/may-30.github.io/assets/155306250/edc51306-db54-4888-b254-d1645e2e7001)

### 2-4. 개선사항

1. 코드에서 ClientID 와 ClientSecret을 환경변수에 등록하여 호출하고 있다.

- 이걸 나중에 파일 형태로 구현해보고 싶다. (`.gitignore` 파일에 등록하여 github에 실리지 않도록 구현해야 함.)

2. Google Oauth를 구현했더니 Kakao Oauth도 구현해보고 싶다.

- 우리나라 사람들은 대부분이 Kakao 계정이 있을 것이라 생각이 들기 때문이다.