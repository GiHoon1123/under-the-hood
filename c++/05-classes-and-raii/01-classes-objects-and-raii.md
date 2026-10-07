# 클래스, 객체, 생성자, 소멸자와 RAII

상태: 기초 정리 완료

## 클래스와 객체

클래스는 객체의 설계도이고 객체는 그 설계도로 만든 실제 값이다.

```cpp
class Request {
 public:
  int result;

  void PrintResult() {
    std::cout << result;
  }
};

Request request;
```

`Request`는 타입이고 `request`는 실제 객체다. 클래스 안의 변수와 함수는 멤버라고 부른다.

## `this`

멤버 함수 안의 `this`는 현재 객체를 가리키는 포인터다.

```cpp
class Request {
 public:
  int result;

  void SetResult(int result) {
    this->result = result;
  }
};
```

```cpp
request.SetResult(10);
```

호출된 객체의 주소가 `this`가 되고, `this->result`는 현재 객체의 멤버를 의미한다.

## 생성자와 소멸자

생성자는 객체가 만들어질 때 실행되고, 소멸자는 객체가 수명을 다할 때 실행된다.

```cpp
class Request {
 public:
  Request() : result(0) {}

  ~Request() {
    // 정리 작업
  }

 private:
  int result;
};
```

```text
객체 생성
  ↓
생성자 실행
  ↓
객체 사용
  ↓
수명 종료
  ↓
소멸자 실행
```

## RAII

RAII는 자원 획득을 생성자에, 자원 해제를 소멸자에 연결하는 방식이다. 자원은 메모리뿐 아니라 파일 디스크립터, 소켓, mutex 잠금도 포함한다.

```cpp
class File {
 public:
  File() {
    fd_ = open("file.txt", O_RDONLY);
  }

  ~File() {
    close(fd_);
  }

 private:
  int fd_;
};
```

```cpp
void read_file() {
  File file;
} // 여기서 자동으로 close()
```

함수 중간에 `return`이나 예외가 생겨도 객체가 정상적으로 소멸하면 정리 코드가 실행된다.

## 비동기 요청 객체

비동기 요청은 함수가 반환된 뒤에도 살아 있어야 한다.

```text
요청 객체 생성
  ↓
libuv 작업 등록
  ↓
함수 반환
  ↓
작업 완료
  ↓
완료 콜백
  ↓
요청 객체 정리
```

Node의 `FSReqBase`, `FSReqCallback`, `FSReqPromise`는 이런 요청 상태를 표현하는 객체다.

```text
FSReqBase
  ├─ 공통 uv_fs_t 요청 상태
  └─ 공통 완료 처리

FSReqCallback
  └─ JavaScript callback 전달

FSReqPromise
  └─ Promise resolve/reject 전달
```

## 객체와 포인터

```cpp
Request request;
request.result = 10;

Request* pointer = &request;
pointer->result = 20;
```

객체 자체는 `.`, 객체 주소를 가진 포인터는 `->`로 멤버에 접근한다.

## 정리

클래스는 상태와 동작을 묶고, 생성자와 소멸자는 객체 수명에 맞춰 초기화·정리를 수행한다. Node의 비동기 요청 객체는 작업이 끝날 때까지 상태를 유지해야 하므로 객체 수명과 소유권을 함께 읽어야 한다.

## `req->cb(req)`와 객체 상태

요청 객체는 단순히 콜백 주소만 저장하는 것이 아니다.

```text
Request 객체
  ├─ 작업 종류
  ├─ 결과값
  ├─ 오류 정보
  ├─ 버퍼·파일 정보
  └─ 완료 콜백
```

작업이 끝나면 libuv는 콜백 함수만 호출하는 것이 아니라, 완료된 요청 객체의 주소를 같이 전달한다. 콜백은 그 객체에서 어떤 작업이 끝났고 성공했는지 확인한다.

```cpp
void completed(Request* req) {
  if (req->result < 0) {
    // 오류 처리
  }
}
```

## RAII와 비동기 수명은 같은 문제인가

RAII는 객체가 범위를 벗어날 때 자원을 정리하는 강력한 방식이지만, 비동기 작업에서는 범위를 너무 빨리 벗어나지 않도록 객체의 소유자를 설계해야 한다.

```text
함수 지역 객체가 작업 완료 전에 소멸
  → 비동기 콜백이 잘못된 주소를 사용할 수 있음
```

따라서 Node의 요청 래퍼는 작업 등록 후에도 살아 있고, 완료 처리와 정리 시점이 연결되어 있다. “소멸자가 있으니 안전하다”가 아니라 “소멸자가 언제 실행되는가”까지 확인해야 한다.

## 스마트 포인터로 이어지는 질문

첫 사이클에서는 직접 `new`와 `delete`를 깊게 다루지 않았다. 다음 사이클에서는 `std::unique_ptr`, `std::shared_ptr`, 이동과 소유권 이전을 함께 확인해야 한다.
