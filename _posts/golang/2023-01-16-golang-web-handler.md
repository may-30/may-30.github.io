---
title: 'web handler 생성 방법 3가지'
categories:
  - golang
tags:
  - [golang, web]
toc: true
toc_sticky: true
date: 2024-01-16
last_modified_at: 2024-01-16
---

## 1. 어떤 정보인가?

golang을 사용해서 web 개발을 위해 강의를 듣고 있는 중 handler를 선언하는 방법이 3가지가 있어 정리하려는 글이다.

## 2. 정보 설명

아마도 여러 가지의 정보들이 더 있겠지만 내가 수강했던 강의에서는 3가지의 web handler 선언 방법이 있었다.

```go
package main

// import는 생략

// 2-2-1
// 2-3-1

func main() {
    // 2-1
    // 2-2-2
    // 2-3-2
    http.ListenAndServe(":3000", nil)
}
```

전체적인 소스코드이며 표시되어 있는 부분에 각 번호에 맞는 소스코드로 치환해주면 된다.

### 2-1. handler를 function형태로 직접 등록하는 방법

```go
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "Hello Index")
})
```

Handler가 들어갈 자리에 익명 함수를 사용하여 바로 등록하는 방법이다.

사용하기엔 간편할 수 있지만 나중에 유지보수 측면에서는 각 path 마다 관리하기 어렵겠다라는 생각이 들었다.

### 2-2. handler를 instance형태로 등록하는 방법

```go
// 2-2-1
type fooHandler struct{}
func (f *fooHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "Hello Foo")
}

// 2-2-2
http.Handle("/foo", &fooHandler{})
```

이 방식은 3가지 중 제일 복잡한 방식이다.

그럼에도 불구하고 주의깊게 보아야 할 부분은 instance와 interface 때문이다.

[Handle](https://pkg.go.dev/net/http#Handle)은 링크처럼 `pattern string, handler Handler`를 매개변수로 받는 메서드이다.

특히 [Handler](https://pkg.go.dev/net/http#Handler)는 `ServeHTTP`라는 함수를 선언하면 데이터 타입이 `Handler`로 속해버리는 interface이다.

따라서 `2-2-2`에서 string 데이터 타입의 pattern은 "/foo"이고 Handler 데이터 타입의 handler는 "fooHandler 구조체의 메서드이자 Handler interface의 ServeHTTP"를 가리키게 된다.

### 2-3. handler를 외부 function형태로 분리하여 등록하는 방법

```go
// 2-3-1
func barHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "Hello Bar")
}

// 2-3-2
http.HandleFunc("/bar", barHandler)
```

난 3가지 중 마지막 방법인 **handler를 외부 function형태로 분리하여 등록하는 방법**이 제일 마음에 들었다.

제일 직관적이고 분리 후 선언하는 모습도 제일 깔끔했다.