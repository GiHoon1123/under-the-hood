# 함수 포인터와 콜백

상태: 기초 정리 완료

## 함수도 주소를 가진다

함수는 메모리에 올라가 실행되므로 함수의 주소도 얻을 수 있다.

```cpp
int add(int a, int b) {
  return a + b;
}

int (*operation)(int, int) = add;
```

`operation`은 정수 두 개를 받고 정수를 반환하는 함수를 가리키는 포인터다.

```cpp
int result = operation(2, 3);
```

`operation`이 가리키는 `add`가 호출된다.

## 콜백

콜백은 다른 함수에 전달해 두었다가 나중에 호출하는 함수다.

```cpp
void run_later(void (*callback)()) {
  callback();
}

void finished() {
  std::cout << "done\n";
}

run_later(finished);
```

```text
finished 함수 주소 전달
  ↓
run_later가 작업 수행
  ↓
callback() 호출
  ↓
finished() 실행
```

## libuv 완료 콜백

libuv 파일 API는 작업이 끝났을 때 호출할 함수를 함께 받는다.

개념적으로:

```cpp
using uv_fs_cb = void (*)(uv_fs_t* request);
```

```cpp
void after_read(uv_fs_t* request) {
  // 읽은 바이트 수와 오류 확인
}

uv_fs_read(loop, request, file, buffers, count, offset, after_read);
```

작업이 끝나면 libuv가 `after_read(request)`를 호출한다.

## `req->cb(req)`

요청 객체가 콜백 함수 주소를 보관한다고 단순화하면:

```cpp
struct Request {
  int result;
  void (*cb)(Request*);
};
```

```cpp
void completed(Request* request) {
  std::cout << request->result;
}

Request request;
request.cb = completed;
Request* req = &request;

req->cb(req);
```

이 표현은 다음과 같다.

```cpp
void (*callback)(Request*) = req->cb;
callback(req);
```

첫 번째 `req`는 어떤 객체의 콜백을 찾을지 정하고, 두 번째 `req`는 완료된 요청의 상태를 콜백에 전달한다. 요청 객체가 자신을 호출하는 것이 아니다.

여러 요청이 같은 콜백을 사용하더라도 인자로 받은 요청 주소를 통해 어느 작업이 끝났는지 알 수 있다.

## Node C++와 연결

```text
Node C++가 AsyncCall()에 완료 함수 전달
  ↓
uv_fs_read()에 콜백 저장
  ↓
작업 완료
  ↓
uv__fs_done()
  ↓
req->cb(req)
  ↓
AfterInteger() 또는 AfterStat()
```

콜백 함수는 작업 결과를 전달하는 통로이고, 요청 객체는 결과·오류·파일 정보 등 상태를 담고 있다.

## 함수 포인터와 메서드의 차이

`cb`는 보통 객체의 멤버 메서드를 자동으로 호출하는 기능이 아니라 C 스타일 함수 포인터다. C 스타일 함수에는 암시적인 `this`가 없으므로 완료된 요청 객체를 명시적으로 인자로 넘긴다.

```text
멤버 함수: this가 현재 객체를 가리킴
함수 포인터: 필요한 객체를 인자로 직접 전달
```

## 정리

함수 포인터는 함수 주소를 저장하고, 콜백은 나중에 호출할 함수를 전달하는 방식이다. libuv는 비동기 작업을 등록할 때 완료 콜백을 저장하고, 작업이 끝나면 해당 콜백에 완료된 요청 객체를 함께 전달한다.

## 왜 요청 객체를 콜백에 다시 전달하는가

비동기 작업은 등록 시점과 완료 시점이 떨어져 있다.

```text
요청 A → a.txt 읽기
요청 B → b.txt 읽기
요청 C → c.txt 읽기
```

세 요청이 같은 완료 함수를 사용한다면 콜백 이름만으로는 어떤 작업이 끝났는지 알 수 없다.

```cpp
void completed(Request* request) {
  // request->result, request->error, request->path 확인
}
```

그래서 완료된 요청의 주소를 함께 전달한다.

```cpp
completed(request);
```

`req->cb(req)`는 “요청 객체가 자신을 호출한다”는 뜻이 아니다. `req` 안에 저장된 외부 함수의 주소를 꺼내고, 그 함수에 완료된 요청 객체를 데이터로 전달하는 것이다.

## 멤버 함수와 C 스타일 콜백

C++ 멤버 함수는 암묵적인 `this`를 받지만, 일반 함수 포인터에는 현재 객체가 자동으로 연결되지 않는다.

```text
멤버 함수:
  this가 현재 객체를 가리킴

C 스타일 콜백:
  필요한 요청 객체를 인자로 직접 전달
```

libuv의 callback 타입은 C API와 호환되어야 하므로 완료 함수에 `uv_fs_t*` 같은 요청 포인터를 명시적으로 넘긴다.

## 콜백은 결과 전달 통로다

```text
작업 등록
  → 완료 함수 주소 저장
  → 작업 수행
  → 완료 시 콜백 호출
  → 요청 객체에서 결과·오류 확인
```

따라서 콜백을 읽을 때는 함수 본문만 보지 말고 다음도 함께 본다.

1. 언제 등록되는가?
2. 어떤 요청 객체에 저장되는가?
3. 어느 스레드 또는 이벤트 루프에서 호출되는가?
4. 인자로 전달된 객체는 누가 소유하는가?
