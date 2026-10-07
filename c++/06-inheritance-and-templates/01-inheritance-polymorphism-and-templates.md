# 상속, 다형성, 템플릿

상태: 기초 정리 완료

## 상속

상속은 부모 클래스의 공통 기능을 자식 클래스가 물려받는 구조다.

```cpp
class Request {
 public:
  int result;
};

class FileRequest : public Request {
 public:
  void ReadFile();
};
```

`FileRequest`는 `Request`의 멤버를 사용할 수 있다.

```text
Request
  ├─ result
  └─ 공통 기능

FileRequest
  └─ 파일 요청 전용 기능
```

Node의 요청 객체도 공통 상태를 부모에 두고 callback·Promise별 결과 전달을 자식에서 나눌 수 있다.

## 다형성

부모 포인터로 자식의 구현을 호출하려면 가상 함수를 사용한다.

```cpp
class Request {
 public:
  virtual void Resolve() {
    std::cout << "base\n";
  }
};

class PromiseRequest : public Request {
 public:
  void Resolve() override {
    std::cout << "promise\n";
  }
};

PromiseRequest promise_request;
Request* request = &promise_request;
request->Resolve(); // promise
```

공통 인터페이스를 유지하면서 실제 객체의 동작을 선택할 수 있다.

## 템플릿

템플릿은 타입을 매개변수로 받아 컴파일 시점에 타입별 코드를 만드는 문법이다.

```cpp
template <typename T>
T Add(T a, T b) {
  return a + b;
}

Add<int>(1, 2);
Add<double>(1.5, 2.5);
```

개념적으로 컴파일러가 `int` 버전과 `double` 버전을 생성한다고 볼 수 있다. 실행 중 타입을 검사하는 방식이 아니라 컴파일 시 타입이 결정된다.

## 클래스 템플릿

```cpp
template <typename T>
class Holder {
 public:
  T value;
};

Holder<int> number;
Holder<const char*> text;
```

같은 래퍼 구조에 서로 다른 타입을 넣을 수 있다.

## Node 코드와 `ReqWrap<T>`

Node 내부의 요청 래퍼는 여러 libuv 요청 타입을 공통 구조로 감싸기 위해 템플릿을 사용한다.

```cpp
template <typename T>
class ReqWrap {
 public:
  T req;
};
```

```text
ReqWrap<uv_fs_t>   → uv_fs_t 요청을 담는 래퍼
ReqWrap<uv_work_t> → uv_work_t 요청을 담는 래퍼
```

래퍼의 공통 로직은 재사용하고 실제 요청 타입만 바꿀 수 있다.

## `AsyncCall()`을 읽는 관점

파일 API에는 `uv_fs_open`, `uv_fs_read`, `uv_fs_close`, `uv_fs_stat`처럼 서로 다른 함수가 있다. 인자와 결과는 다르지만 요청 생성·등록·완료 처리라는 큰 흐름은 공통이다.

```text
요청 생성
  ↓
libuv 함수 등록
  ↓
완료 콜백 연결
  ↓
결과 전달
```

Node의 `AsyncCall()` 템플릿과 오버로드는 이 반복 흐름을 여러 함수에 재사용하기 위한 구조로 이해할 수 있다.

## 상속과 템플릿의 차이

```text
상속:
"이 객체는 Request의 한 종류다"

템플릿:
"이 코드에 어떤 타입을 넣을지 컴파일 시 정하자"
```

상속은 공통 인터페이스와 실행 시 다형성에 초점이 있고, 템플릿은 컴파일 시 타입별 코드 재사용에 초점이 있다.

## 정리

Node 코드를 볼 때 `class A : public B`는 공통 상태와 특수 동작의 관계를, `Something<T>`는 `T`에 실제로 어떤 타입이 들어가는지를 확인한다.

## 상속과 함수 포인터를 구분하기

상속은 객체 타입 사이의 관계를 표현한다.

```text
PromiseRequest는 Request의 한 종류
```

함수 포인터는 어떤 함수를 나중에 호출할지 저장한다.

```text
Request 객체의 cb → 완료 함수 주소
```

따라서 `FSReqPromise`가 `FSReqBase`를 상속한다는 것과 `req->cb`가 완료 함수를 가리킨다는 것은 서로 다른 층의 개념이다.

```text
상속:
  객체 구조와 공통 기능

함수 포인터:
  실행할 함수 선택
```

## 템플릿은 실행 중 제네릭 타입을 확인하는 기능이 아니다

```cpp
template <typename T>
T Add(T a, T b) {
  return a + b;
}
```

`Add<int>`와 `Add<double>`은 컴파일 과정에서 타입에 맞는 코드로 만들어진다. JavaScript처럼 실행 중 하나의 함수가 임의 타입을 검사하는 모델과 다르다.

## 템플릿을 읽는 실제 순서

```cpp
ReqWrap<uv_fs_t> wrap;
```

이 코드를 보면:

1. `ReqWrap`이 클래스 템플릿인지 확인한다.
2. `T`에 `uv_fs_t`가 들어간다고 치환한다.
3. 내부 멤버와 메서드의 타입을 다시 계산한다.
4. 이 인스턴스가 어떤 Node·libuv 요청을 감싸는지 확인한다.

이렇게 읽으면 꺾쇠괄호 문법을 추상적인 기호로 외우지 않아도 된다.
