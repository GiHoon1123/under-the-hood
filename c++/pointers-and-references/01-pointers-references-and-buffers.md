# 포인터, 참조, 주소

상태: 기초 정리 완료

## 변수와 주소

```cpp
int value = 10;
```

`value`는 메모리 공간에 저장된 값이고, `&value`는 그 공간의 주소다.

```cpp
int* pointer = &value;
```

`pointer`에는 `10`이 아니라 `value`가 저장된 주소가 들어간다.

```text
value   → 주소 0x1000, 값 10
pointer → 값 0x1000
```

주소는 보통 16진수로 표시한다. `0x`는 뒤의 숫자가 16진수라는 뜻이다. 16진수 한 자리는 2진수 네 자리와 대응하므로 긴 주소를 읽기 쉽다.

## 선언의 `*`와 사용의 `*`

```cpp
int* pointer;
```

선언에서 `*`는 포인터 타입이라는 뜻이다.

```cpp
*pointer = 20;
```

사용에서 `*pointer`는 pointer가 가리키는 주소로 가서 그 값을 사용한다는 뜻이다. 이를 역참조라고 한다.

## `nullptr`

```cpp
int* pointer = nullptr;
```

아무 객체도 가리키지 않는 포인터다. 역참조하기 전에 확인해야 한다.

```cpp
if (pointer != nullptr) {
  std::cout << *pointer;
}
```

## 참조

```cpp
int value = 10;
int& reference = value;
```

참조는 기존 변수의 다른 이름처럼 동작한다.

```cpp
reference = 20;
```

그러면 `value`도 20이 된다. 포인터처럼 주소를 직접 저장하고 `*`로 접근하지 않는다.

```text
포인터: 주소를 저장하고 *로 값에 접근
참조:   같은 객체의 별명처럼 사용
```

## `.`와 `->`

객체를 직접 가지고 있으면 `.`을 사용한다.

```cpp
Request request;
request.result = 10;
```

객체의 주소를 가진 포인터라면 `->`를 사용한다.

```cpp
Request* pointer = &request;
pointer->result = 20;
```

`pointer->result`는 다음과 같은 뜻이다.

```cpp
(*pointer).result
```

즉 포인터가 가리키는 객체에 접근한 뒤 그 객체의 `result` 멤버를 읽는다.

## `uv_buf_t`와 포인터

```cpp
char data[1024];

uv_buf_t buffer;
buffer.base = data;
buffer.len = sizeof(data);
```

`data`가 실제 버퍼이고 `buffer.base`는 그 시작 주소다. `uv_buf_t`는 위치와 길이를 묶은 설명 정보다.

## 비동기 작업과 주소 수명

```cpp
void start() {
  char data[1024];
  start_async_read(data);
}
```

`start()`가 반환된 뒤에도 읽기 작업이 계속되면 `data`가 사라진 주소를 libuv가 사용할 수 있다. 이런 포인터를 댕글링 포인터라고 한다.

```text
버퍼 생성
  ↓
주소를 비동기 작업에 전달
  ↓
함수 반환·버퍼 소멸
  ↓
나중에 작업 완료
  ↓
죽은 주소 사용 위험
```

따라서 요청 객체나 별도 소유자가 완료 시점까지 버퍼를 유지해야 한다.

## Node/libuv에서 읽는 법

```cpp
uv_fs_t* req;
req->result;
req->cb(req);
```

`req`는 `uv_fs_t` 객체의 주소다. `req->result`는 그 객체의 필드에 접근하는 표현이다. `req->cb(req)`는 요청 객체 안에 저장된 콜백 함수를 호출하면서 요청 객체 자신의 주소를 인자로 전달하는 코드다.

## 정리

```text
&value       → value의 주소
int* p       → int를 가리키는 포인터
*p           → p가 가리키는 값
int& r       → 기존 객체의 참조
object.field → 객체 직접 접근
pointer->field → 포인터가 가리키는 객체 접근
```

포인터를 볼 때 주소 자체보다 그 주소가 가리키는 객체의 소유권과 수명을 먼저 확인한다.

## 포인터를 단계별로 읽기

```cpp
int value = 10;
int* pointer = &value;
```

메모리를 단순화하면 다음과 같다.

```text
value
  주소: 0x1000
  값:   10

pointer
  주소: 0x2000
  값:   0x1000
```

`pointer`는 `value`와 다른 변수다. 단지 `value`가 있는 위치를 저장하고 있다.

```cpp
*pointer = 20;
```

이 코드는 pointer 변수의 값을 바꾸는 것이 아니라, pointer가 가리키는 `0x1000`의 값을 바꾼다.

## `.`와 `->`의 관계

```cpp
Request request;
request.result = 10;
```

객체 자체가 있으면 `.`을 사용한다.

```cpp
Request* req = &request;
req->result = 20;
```

객체 주소를 가지고 있으면 `->`를 사용한다. `req->result`는 정확히 `(*req).result`와 같다.

```text
req
  → Request 객체 주소
  → *req로 객체에 접근
  → .result로 멤버 접근
```

JavaScript의 `object.property`와 목적은 비슷하지만, C++은 객체 자체와 객체를 가리키는 포인터를 구분하기 때문에 문법도 구분한다.

## 포인터와 소유권은 별개다

```cpp
int* pointer = &value;
```

이 문장만으로 pointer가 value를 소유한다는 뜻은 아니다. pointer는 주소를 빌려 알고 있을 뿐이고, value의 수명은 다른 규칙으로 결정된다.

```text
주소를 알고 있음 ≠ 객체를 소유함
```

비동기 코드에서는 이 구분이 특히 중요하다. 포인터를 요청 객체에 저장했다면 그 포인터가 가리키는 메모리가 완료 시점까지 살아 있는지 확인해야 한다.

## `req->cb(req)`를 읽는 순서

```cpp
req->cb(req);
```

다음처럼 나누어 생각한다.

```cpp
void (*callback)(Request*) = req->cb;
callback(req);
```

첫 번째 `req`는 콜백 주소를 찾는 데 사용되고, 두 번째 `req`는 콜백 함수에 완료된 요청 상태를 전달하는 인자다. 요청 객체가 자기 자신을 호출하는 것이 아니다.
