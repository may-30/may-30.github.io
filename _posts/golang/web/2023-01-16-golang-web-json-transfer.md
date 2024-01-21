---
title: '[golang web] json으로 데이터 주고받기'
categories:
  - golang web
tags:
  - [golang, web]
toc: true
toc_sticky: true
date: 2024-01-16
last_modified_at: 2024-01-16
---

## 1. 어떤 정보인가?

golang으로 web 개발 중 json으로 데이터를 주고받는 정보이다.

번외로 "`json.Marshal`은 사용하는데 왜 `json.Unmarshal`은 사용하지 않을까?" 도 담겨 있다.

## 2. 정보 설명

```go
package main

// import 생략

/*
json이 request의 body 값으로 전달된다.
golang에서는 json을 곧장 사용할 수 없기 때문에 json과 유사한 데이터 형식인 struct로 변경하여 사용해주어야 한다.
때문에 "Teddy"라는 이름의 구조체를 선언하였고 request를 통해 같은 속성값들로 들어오는 json 데이터를 조작할 수 있게 된다.
*/
type Teddy struct {
    ProductName string `json:"product_name"`
    CreatedAt time.Time `json:"craeted_at"`
}

// 2-1
// 2-2
func indexHandler(w http.ResponseWriter, r *http.Request) {
}

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("/", indexHandler)

    http.ListenAndServe(":3000", mux)
}
```

공통 소스코드 부분이다.

`indexHandler` 부분에 주석으로 표시된 부분을 숫자에 맞게 보면 된다.

### 2-1. json.NewDecoder.Decode + json.Marshal 방법

```go
// 2-1
func indexHandler(w http.ResponseWriter, r *http.Request) {
    teddy := new(Teddy)

    err := json.NewDecoder(r.Body).Decode(teddy)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "❌ Bad Request: ", err)
        return
    }

    teddy.CreatedAt = time.Now()

    data, _ := json.Marshal(teddy)
    fmt.Fprint(w, string(data))
}
```

유튜브 강의에서 배운 방식이고 내 방식대로 설명을 풀어보겠다.

request body 값으로 들어온 json 데이터를 조작하기 위해서는 golang의 구조체로 변환을 시켜주어야 하며 `Teddy` 라는 구조체를 생성하였다.

json 데이터를 Teddy 구조체로 매핑을 시켜주는 작업을 `Decode`가 하게 된다.

그런데 곧장 Decode를 사용한 것이 아닌 `json.NewDecoder` 뒤에 붙게 된다.

그 이유는 [Decode](https://pkg.go.dev/encoding/json#Decoder.Decode)는 **`"(*json.Decoder).Decode(v any) error"`** 곧바로 사용하지 못하는 성질을 가지고 있다.

그렇다면 `(*json.Decoder)`는 어디서 찾느냐?

바로 [NewDecoder](https://pkg.go.dev/encoding/json#NewDecoder)의 **`"json.NewDecoder(r io.Reader) *json.Decoder"`** 반환값이다.

즉, NewDecoder의 결과값으로 Decode의 선언 조건을 만족하게 할 수 있는 것이다.

NewDecoder의 매개변수인 `io.Reader`는 request의 Body인 r.Body에서 충족시킬 수 있다.

`r.Body`는 타입이 [io.ReadCloser](https://pkg.go.dev/io#ReadCloser)로 Reader를 포함하고 있는 interface이기 때문에 기능을 충분히 수행할 수 있다.


> json.NewDecoder(r io.Reader) *json.Decoder
- NewDecoder returns a new decoder that reads from r.

> (*json.Decoder).Decode(v any) error
- Decode reads the next JSON-encoded value from its input and stores it in the value pointed to by v.

### 2-2. json.Unmarshal + json.Marshal 방법

유튜브 강의를 통해서 접해보고 있는데 `json.NewDecoder.Decode` 방식으로 설명해주는 것에 물 흐르듯이 설명을 듣고 있다가 `json.Marshal` 설명 부분에서 강의를 멈추었다.

**`json.Marshal`은 사용하면서 왜 `json.Unmarshal`은 사용하지 않지?**

대수롭지 않게 넘길 수 있는 부분이었지만 Marshal을 사용했으면 Unmarshal을 사용하는 것이 대칭을 이루어서 예쁘지 않을까 하는 생각이 들게 되어 직접 구현해보았다.

```go
// 2-2
func indexHandler(w http.ResponseWriter, r *http.Request) {
    teddy := new(Teddy)

    req_data, err := io.ReadAll(r.Body)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "😞 Body Null: ", err)
        return
    }

    err = json.Unmarshal(req_data, teddy)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprint(w, "❌ Body Null: ", err)
        return
    }

    teddy.CreatedAt = time.Now()

    data, _ := json.Marshal(teddy)
    fmt.Fprint(w, string(data))
}
```

왜 Marshal - Unmarshal로 구현하지 않고 `json.NewDecoder(r.Body).Decode(teddy)` 로 구현했는지 알 것 같다.

대칭의 아름다움을 찾다가 `io.ReadAll` 같은 로직을 추가해야 하는 번거로움이 있기 때문이다.

json.Unmarshal + json.Marshal 로직을 설명한다.

[Unmarshal](https://pkg.go.dev/encoding/json#Unmarshal)은 **`"Unmarshal(data []byte, v any) error"`**으로 매개변수가 `[]byte` 타입을 가져야 하는데 r.Body는 위에서 언급했던 것처럼 `io.ReadCloser`이다.

코드 상에서 형변환을 해주면 되지 않을까? 하고 접근해보았다.

![스크린샷 2024-01-16 오후 9 02 37](https://github.com/may-30/may-30.github.io/assets/155306250/09ac95da-c562-4ffd-8881-2d0ed8859299)

강타입인 golang은 형변환에 호락호락하지 않다는 것을 배우게 된다...

약간의 꼼수를 사용한 것이 [io.ReadAll](https://pkg.go.dev/io#ReadAll) (구 ioutil.ReadAll) 을 이용한 것이다.

io.ReadAll은 **`"ReadAll(r Reader) ([]byte, error)"`**으로 `[]byte` 타입을 반환값으로 준다.

`io.ReadAll`을 통해서 []byte로 형변환된 `r.Body`는 `json.Unmarshal`을 통해서 Teddy 구조체로 디코딩이 되며 데이터를 조작할 수 있게 된다.

~~역시 잘 알려진 방법에는 다 이유가 있다... ^__^~~