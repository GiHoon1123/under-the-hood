# `fs.promises.readFile()`을 Node와 libuv 코드로 연결하기

상태: 전체 흐름 정리 완료

## 기준 코드

- 저장소: `nodejs/node`
- 기준 커밋: `30fa54a3fe`
- Node 파일: `lib/fs/promises.js`, `lib/internal/fs/promises.js`, `src/node_file.cc`, `src/node_file.h`, `src/node_file-inl.h`
- libuv 파일: `deps/uv/include/uv.h`, `deps/uv/src/unix/fs.c`, `deps/uv/src/threadpool.c`

## 시작점

```js
const data = await fs.promises.readFile('file.txt');
```

이 한 줄은 JavaScript, Node 내부 JavaScript, C++ 바인딩, libuv, 운영체제, V8 Promise 처리를 모두 통과한다.

## 공개 API에서 내부 JavaScript로

```text
lib/fs/promises.js
  ↓
lib/internal/fs/promises.js
```

공개 파일은 사용자에게 제공되는 진입점이고, 내부 파일은 파일을 열고 상태를 확인하고 읽고 닫는 여러 작업을 조율한다.

내부 구현은 `internalBinding('fs')`로 native 바인딩을 가져온다.

```js
const binding = internalBinding('fs');
```

## C++ 바인딩 등록

`src/node_file.cc`에는 다음과 같은 등록이 있다.

```cpp
SetMethod(isolate, target, "read", Read);
SetMethod(isolate, target, "open", Open);
SetMethod(isolate, target, "close", Close);
```

이 등록으로 JavaScript의 `binding.read()`가 C++의 `Read()`를 호출한다.

## 요청 객체와 버퍼

`Read()`는 인자를 검증하고 `FSReqPromise` 같은 요청 객체와 버퍼를 준비한다.

```text
FSReqPromise
  ├─ uv_fs_t 요청 상태
  ├─ Promise Resolver
  ├─ 버퍼 정보
  └─ 완료 결과
```

요청 객체와 버퍼는 파일 작업이 완료될 때까지 살아 있어야 한다.

## Node C++에서 libuv로

```text
Read()
  ↓
AsyncCall(..., uv_fs_read, ...)
  ↓
uv_fs_read()
```

`uv_fs_read()` 선언은 `deps/uv/include/uv.h`에 있고 Unix 구현은 `deps/uv/src/unix/fs.c`에 있다.

```text
uv_fs_read()
  ↓
uv__work_submit()
  ↓
uv__fs_work()
  ↓
uv__fs_read()
  ↓
read() 또는 pread()
```

파일 작업이 스레드풀에서 처리되는 경우 worker thread가 운영체제 파일 함수를 실행한다. Promise를 사용한다고 worker thread에서 JavaScript가 실행되는 것은 아니다.

## 완료 처리

작업이 끝나면 libuv는 완료 큐를 통해 메인 이벤트 루프로 결과를 전달한다.

```text
작업 완료
  ↓
uv__work_done()
  ↓
uv__fs_done()
  ↓
req->cb(req)
  ↓
Node C++ 완료 함수
```

`req->cb(req)`는 요청 객체에 저장된 완료 함수를 호출하면서 완료된 요청 객체를 인자로 전달한다. 콜백은 요청의 결과·오류·읽은 바이트 수를 확인할 수 있다.

## Promise 결과와 V8

Promise 방식에서는 Node C++ 완료 함수가 `FSReqPromise::Resolve()`를 호출한다.

```text
AfterInteger()
  ↓
FSReqPromise::Resolve()
  ↓
v8::Promise::Resolver::Resolve()
  ↓
Promise fulfilled
  ↓
V8이 then/await continuation 예약
```

Node C++가 await 코드를 직접 마이크로태스크 큐에 넣는 것은 아니다. Promise Resolver를 호출하면 V8이 연결된 후속 작업을 예약한다.

`InternalCallbackScope`가 닫힐 때 `src/api/callback.cc`의 `InternalCallbackScope::Close()`가 조건에 따라 다음을 수행한다.

```cpp
context->GetMicrotaskQueue()->PerformCheckpoint(isolate);
```

그 결과 V8 마이크로태스크 큐에 있던 Promise 후속 코드가 실행되고 `await` 다음 줄로 돌아온다.

## 전체 그림

```text
JavaScript fs.promises.readFile()
  ↓
lib/internal/fs/promises.js
  ↓
internalBinding('fs')
  ↓
src/node_file.cc의 Read()
  ↓
FSReqPromise / AsyncCall()
  ↓
uv_fs_read()
  ↓
uv__work_submit()
  ↓
libuv worker thread
  ↓
read() / pread()
  ↓
운영체제 파일 시스템
  ↓
uv__work_done()
  ↓
uv__fs_done()
  ↓
AfterInteger()
  ↓
FSReqPromise::Resolve()
  ↓
V8 Promise Resolver
  ↓
마이크로태스크
  ↓
await 이후 JavaScript
```

## callback API와의 차이

libuv 완료 지점까지는 callback API와 Promise API가 비슷하다. 차이는 Node C++가 결과를 전달하는 방식이다.

```text
callback API
  → FSReqCallback::Resolve()
  → MakeCallback()
  → JavaScript callback

Promise API
  → FSReqPromise::Resolve()
  → Promise Resolver
  → V8 microtask
  → await continuation
```

## 첫 번째 사이클에서 확인한 사실

- Promise는 작업 실행 방식이 아니라 결과 전달 추상화다.
- 네트워크 소켓은 보통 이벤트 감시를 사용하고 파일 작업은 스레드풀을 사용할 수 있다.
- `uv_run()`은 작업마다 새로 호출되는 함수가 아니라 이벤트 루프를 구동하는 진입점이다.
- 이벤트 루프가 poll에서 잠들어 있을 때 운영체제 이벤트가 대기를 깨운다.
- JavaScript callback은 일반적으로 libuv worker thread가 아니라 메인 스레드에서 실행된다.

## 남은 질문

- `fs.promises.readFile()`이 `open`, `fstat`, 여러 `read`, `close`를 어떤 순서로 조율하는가?
- `InternalCallbackScope::Close()`에서 nextTick과 V8 microtask의 정확한 순서는 무엇인가?
- Windows에서 동일한 파일 경로가 `deps/uv/src/win/fs.c`와 어떤 차이를 보이는가?

## `uv_run()`은 어디에 끼어드는가

JavaScript가 파일 읽기를 등록하는 순간 `uv_run()`이 파일 함수마다 새로 호출되는 것은 아니다.

```text
초기 JavaScript 실행
  ↓
fs.promises.readFile()이 libuv 요청 등록
  ↓
사용자 JavaScript가 반환
  ↓
Node의 SpinEventLoopInternal()
  ↓
uv_run()
  ↓
libuv 완료 큐와 이벤트 처리
```

이벤트 루프가 이미 실행 중이라면 callback 안에서 등록한 새 작업도 같은 loop에 추가된다.

```text
uv_run() 실행 중
  ↓
timer callback에서 fs.readFile() 등록
  ↓
timer callback 반환
  ↓
기존 uv_run()이 다음 이벤트 처리
```

## 파일 작업과 네트워크 작업 비교

같은 Promise API라도 아래 계층의 실행 방식은 다를 수 있다.

```text
fs.promises.readFile()
  → libuv worker thread에서 파일 작업 가능

socket.read()
  → non-blocking 소켓을 epoll/kqueue/IOCP에 등록
```

네트워크 소켓은 보통 OS가 준비 상태를 알려주면 이벤트 루프가 콜백을 실행한다. DNS `lookup`처럼 운영체제의 동기 함수를 감싸야 하는 일부 작업은 스레드풀을 사용할 수 있다.

## callback API와 Promise API를 혼동하지 않기

libuv 완료 지점까지는 둘이 비슷하다.

```text
libuv 작업 완료
  ↓
Node C++ 완료 함수
```

그 뒤 결과 연결 방식이 다르다.

```text
callback API
  → MakeCallback()
  → JavaScript callback 직접 호출

Promise API
  → Promise Resolver::Resolve()
  → V8 reaction 예약
  → microtask checkpoint
  → await continuation
```

따라서 V8로 넘어온 모든 함수가 마이크로태스크가 되는 것은 아니다. Promise 후속 작업이 마이크로태스크이고, 일반 callback은 Node C++의 callback 호출 경로를 통해 실행될 수 있다.

## 첫 사이클의 학습 결론

```text
Node JavaScript
  ↓ 공개·내부 JavaScript
Node C++ 바인딩
  ↓ 요청 객체·버퍼·콜백
libuv
  ↓ 스레드풀 또는 OS 이벤트 감시
운영체제
  ↓ 결과
libuv 완료 처리
  ↓
Node C++
  ↓ callback 또는 Promise resolve
V8
  ↓
JavaScript 후속 코드
```

처음에는 `uv_run()`이 작업마다 호출되고, libuv가 직접 JavaScript 콜스택을 감시한다고 생각하기 쉬웠다. 실제로는 Node가 이벤트 루프를 구동하고, libuv가 OS 이벤트·완료 큐를 처리하며, Node C++가 V8을 호출한다.
