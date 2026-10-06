# Node.js의 비동기 메서드는 어떤 경로로 실행되는가

상태: 코드 확인 완료

이 글은 JavaScript에서 Node.js 비동기 API를 호출한 뒤, Node 내부 JavaScript와 C++ 바인딩을 거쳐 libuv가 작업을 처리하고, 결과가 다시 V8과 JavaScript 콜백 또는 Promise로 돌아오는 과정을 정리한다.

## 기준 코드

- 저장소: nodejs/node
- 기준 커밋: 30fa54a3fe
- 로컬 경로: /Users/dustin/Desktop/common/open-source/node
- 주요 파일:
  - lib/fs.js
  - lib/internal/fs/read/context.js
  - lib/internal/bootstrap/realm.js
  - src/node_binding.cc
  - src/node_file.cc
  - src/api/embed_helpers.cc
  - libuv 파일 시스템 구현

## LoadEnvironment에서 비동기 API를 위한 기반 준비

첫 번째 글에서 본 LoadEnvironment()는 사용자 JavaScript를 실행하기 전에 Environment를 준비한다. 이 단계가 끝나야 이후의 fs.readFile(), setTimeout(), Promise 기반 API가 Node 런타임 안에서 동작할 수 있다.

~~~text
LoadEnvironment()
  ├─ InitializeLibuv()
  ├─ InitializeDiagnostics()
  ├─ embedder preload 저장
  ├─ InitializeCompileCache()
  └─ StartExecution()
~~~

### InitializeLibuv()

src/env.cc의 InitializeLibuv()는 이벤트 루프에서 사용할 Node 내부 핸들을 준비한다.

- 타이머 핸들
- setImmediate 처리를 위한 check와 idle 핸들
- 다른 스레드의 native immediate를 메인 루프로 전달하는 uv_async 핸들
- 이벤트 루프 유휴 시간을 V8 profiler가 구분하기 위한 prepare/check 핸들

이 함수는 핸들을 초기화할 뿐 uv_run()을 호출하지 않는다. 따라서 LoadEnvironment() 중에 이벤트 루프가 이미 실행되는 것은 아니다.

### InitializeDiagnostics()

src/node.cc의 InitializeDiagnostics()는 V8의 진단 기능을 Node Environment에 연결한다. heap profiler의 embedder graph 콜백과 heap limit 관련 처리를 등록하고, 옵션에 따라 uncaught exception stack trace와 Promise hook을 활성화한다.

이 준비는 fs.readFile() 자체를 수행하지 않는다. 비동기 작업이나 오류가 발생했을 때 관찰할 수 있는 진단 경로를 미리 설정하는 것이다.

### Embedder preload

LoadEnvironment()는 선택적으로 embedder preload 콜백을 Environment에 저장한다. 일반적인 node app.js 실행에서는 없을 수 있지만, Node를 다른 C++ 애플리케이션에 임베드하는 경우 내부 JavaScript 실행 전에 임베더가 제공한 초기화 코드를 실행할 수 있게 한다.

### InitializeCompileCache()

src/env.cc의 InitializeCompileCache()는 NODE_COMPILE_CACHE 환경 변수를 확인하고, 조건이 맞으면 JavaScript 컴파일 캐시를 활성화한다. 이는 파일 I/O 이벤트를 등록하는 단계가 아니라 모듈을 읽고 컴파일할 때 사용할 캐시 정책을 준비하는 단계다.

### StartExecution()

src/node.cc의 StartExecution()은 실행 모드에 따라 첫 번째 내부 JavaScript 모듈을 선택한다. 일반적인 node app.js에서는 internal/main/run_main_module로 이어지고, eval·REPL·test runner·worker·watch mode 등에서는 각각 다른 internal/main 모듈이 선택된다.

이 선택이 끝나야 사용자의 JavaScript API 호출이 시작된다.

## 먼저 구분할 네 단계

~~~text
1. JavaScript API 호출
2. libuv에 작업 등록
3. uv_run()이 이벤트 루프를 실행하며 완료 작업 처리
4. Node C++와 V8을 거쳐 JavaScript 결과 전달
~~~

libuv에 작업을 등록하는 것과 uv_run()으로 이벤트 루프를 실행하는 것은 서로 다른 단계다. JavaScript가 실행 중이어도 Node C++ 바인딩을 통해 libuv에 작업을 등록할 수 있다. 그러나 초기 사용자 JavaScript가 끝나지 않으면 Node가 SpinEventLoopInternal()에 도달하지 못해 최초 uv_run()을 호출하지 못할 수 있다.

## 1. 콜백 기반 fs.readFile

~~~js
const fs = require('fs');

fs.readFile('file.txt', (error, data) => {
  if (error) throw error;
  console.log('file read');
});
~~~

~~~text
fs.readFile()
  ↓
lib/fs.js
  ↓
내부 FS read context
  ↓
internalBinding('fs')
  ↓
Node C++ fs binding
  ↓
uv_fs_open / uv_fs_read / uv_fs_close
  ↓
uv_run()
  ↓
libuv 완료 콜백
  ↓
Node C++가 JavaScript 콜백 호출
  ↓
V8
  ↓
사용자 콜백 실행
~~~

## 2. Node 내부 JavaScript 계층

파일: lib/fs.js

fs.readFile()은 사용자가 호출하는 공개 API다. 경로와 옵션을 정리하고, 파일을 열고 읽고 닫는 여러 비동기 단계를 조율한다.

~~~text
파일 열기
  ↓
파일 상태 확인
  ↓
파일 내용 읽기
  ↓
파일 닫기
  ↓
최종 사용자 콜백 호출
~~~

파일: lib/internal/fs/read/context.js

이 모듈은 여러 파일 읽기 단계를 하나의 작업으로 이어 붙인다. fs.readFile()을 호출한 순간 최종 콜백이 바로 실행되는 것이 아니다. 내부 단계가 완료될 때마다 다음 단계로 넘어가고 모든 단계가 끝난 뒤 최종 콜백이 실행된다.

## 3. internalBinding과 Node C++ 바인딩

Node 내부 JavaScript는 internalBinding('fs')를 통해 C++ 구현에 접근한다.

~~~text
lib/fs.js
  ↓
internalBinding('fs')
  ↓
Node C++ native binding
~~~

internalBinding()은 일반 사용자 코드가 임의로 사용하는 공개 API가 아니다. Node 내부 모듈이 C++ 기능을 가져오기 위한 내부 연결 지점이다.

파일: src/node_binding.cc

Node는 내부 바인딩 이름을 받아 native 모듈을 찾아 반환한다. fs 바인딩을 요청하면 파일 시스템 C++ 구현이 연결된다.

파일: src/node_file.cc

이 파일은 파일 시스템 C++ 함수와 JavaScript가 사용할 메서드를 연결한다. 초기화 과정에서 open, read, fstat, close와 같은 함수를 V8 객체에 등록한다.

단순화한 형태:

~~~cpp
SetMethod(target, "read", Read);
~~~

SetMethod는 JavaScript에서 특정 이름으로 함수를 호출할 수 있게 native C++ 함수와 연결하는 역할을 한다.

## 4. C++에서 libuv 작업 등록

Node C++ 파일 시스템 함수는 libuv의 파일 시스템 API를 사용한다.

~~~cpp
uv_fs_read(loop, request, fd, buffers, count, offset, callback);
~~~

이 호출은 함수 안에서 파일 내용을 모두 읽으라는 뜻이 아니다. libuv에 파일 읽기 요청과 완료 콜백을 등록한다는 뜻이다.

~~~text
Node C++
  ↓
uv_fs_read()
  ↓
libuv가 요청 상태 저장
  ↓
운영체제 또는 libuv 스레드풀에서 작업
~~~

작업을 등록하고 반환하면 Node C++ 바인딩도 반환하고 호출한 JavaScript 코드도 다음 줄로 진행할 수 있다.

## 5. uv_run과 작업 완료 처리

Node의 초기 사용자 JavaScript가 끝나면 SpinEventLoopInternal()이 이벤트 루프를 실행한다.

~~~cpp
uv_run(env->event_loop(), UV_RUN_DEFAULT);
~~~

uv_run()은 타이머, 운영체제 이벤트, libuv 작업 완료 상태, 완료된 요청의 C/C++ 콜백을 처리한다.

libuv가 uv_run()을 호출하는 것이 아니라 Node가 uv_run()을 호출한다.

~~~text
Node C++ 런타임
  ↓ uv_run() 호출
libuv C 이벤트 루프
  ↓
작업 완료 확인
~~~

파일 읽기 작업 자체는 libuv 스레드풀이나 운영체제를 통해 먼저 끝날 수 있다. 그러나 Node의 메인 흐름이 완료 결과를 처리하고 JavaScript 콜백을 실행하려면 이벤트 루프가 해당 결과를 처리해야 한다.

## 6.1 libuv가 운영체제를 호출하고 완료를 돌려주는 경로

파일: deps/uv/include/uv.h

uv_fs_read()는 libuv가 제공하는 파일 읽기 API다. Node C++는 이 함수에 이벤트 루프, uv_fs_t 요청 구조체, 파일 디스크립터, 버퍼, offset, 완료 콜백을 전달한다.

~~~text
uv_fs_read()
  ├─ loop: 어느 이벤트 루프에 등록할지
  ├─ req: 파일 작업 상태
  ├─ file: 파일 디스크립터
  ├─ bufs: 결과를 저장할 메모리
  ├─ offset: 읽기 시작 위치
  └─ cb: 작업 완료 콜백
~~~

Unix 구현은 deps/uv/src/unix/fs.c에 있다. uv_fs_read()는 요청을 초기화하고 비동기 콜백이 있으면 uv__work_submit()을 통해 작업을 제출한다.

~~~text
deps/uv/src/unix/fs.c
  ↓
uv_fs_read()
  ↓
uv__work_submit()
  ↓
uv__fs_work()
~~~

uv__fs_work()는 요청의 fs_type을 보고 실제 작업 함수를 선택한다. 읽기 요청이면 uv__fs_read()가 호출된다.

~~~text
uv__fs_work()
  ↓
UV_FS_READ 확인
  ↓
uv__fs_read()
  ↓
read() 또는 pread()
  ↓
운영체제 파일 시스템
~~~

읽기 위치가 음수이면 현재 파일 위치를 사용하는 read() 계열 경로를 타고, 명시적인 offset이 있으면 pread() 계열 경로를 사용할 수 있다. 이 함수들은 JavaScript 함수가 아니라 Unix 운영체제가 제공하는 시스템 호출이다.

작업이 끝난 뒤에는 deps/uv/src/threadpool.c의 uv__work_done()이 완료 큐를 처리한다.

~~~text
파일 작업 완료
  ↓
완료 큐에 결과 등록
  ↓
uv_run()이 이벤트 루프에서 완료 이벤트 처리
  ↓
uv__work_done()
  ↓
w->done(w, status)
  ↓
uv__fs_done()
  ↓
req->cb(req)
~~~

파일 작업에서 req->cb는 Node C++가 AsyncCall()에 넘긴 AfterInteger(), AfterStat() 같은 완료 함수다. 이 지점까지가 libuv가 작업을 끝내고 Node C++로 결과를 돌려주는 과정이다.

## 6.2 libuv 완료 콜백과 JavaScript 콜백

~~~text
파일 읽기 완료
  ↓
libuv가 등록된 완료 콜백 호출
  ↓
Node C++의 FS 완료 처리
~~~

libuv 완료 콜백은 C/C++ 함수이고 사용자가 작성한 readFile 콜백은 JavaScript 함수다.

~~~text
libuv C 콜백
  ↓
Node C++ FSReqCallback 처리
  ↓
V8 MakeCallback 계열 호출
  ↓
사용자 JavaScript 콜백
~~~

uv_run()이 JavaScript 콜백을 직접 호출하는 것이 아니다. libuv가 Node C++ 콜백을 부르고 Node C++가 V8을 호출한다.

## 6.3 Promise 결과가 V8 마이크로태스크로 이어지는 경로

콜백 방식과 Promise 방식은 libuv 완료 지점까지는 비슷하지만, Node C++가 결과를 전달하는 방식이 다르다.

~~~text
콜백 방식
  → FSReqCallback::Resolve()
  → MakeCallback()
  → JavaScript callback

Promise 방식
  → FSReqPromise::Resolve()
  → v8::Promise::Resolver::Resolve()
  → Promise fulfilled
  → V8 Promise reaction 예약
~~~

Promise 요청 클래스의 선언은 src/node_file.h에 있고, 템플릿 구현은 src/node_file-inl.h에 있다.

src/node_file-inl.h의 FSReqPromise::Resolve()는 요청 객체 안에 보관해 둔 v8::Promise::Resolver를 가져와 Resolver::Resolve()를 호출한다.

~~~text
AfterInteger()
  ↓
FSReqPromise::Resolve()
  ↓
v8::Promise::Resolver::Resolve()
  ↓
Promise fulfilled
  ↓
then 또는 await continuation 예약
~~~

Node C++가 직접 마이크로태스크 큐에 await 코드를 넣는 것은 아니다. Promise Resolver를 호출하면 V8이 Promise에 연결된 후속 작업을 Promise reaction으로 예약한다.

FSReqPromise::Resolve() 안에는 InternalCallbackScope도 있다. 이 scope가 닫힐 때 src/api/callback.cc의 InternalCallbackScope::Close()가 실행되고, 조건이 맞으면 다음 코드를 호출한다.

~~~cpp
context->GetMicrotaskQueue()->PerformCheckpoint(isolate);
~~~

이 호출이 V8 마이크로태스크 큐에 등록된 Promise 후속 작업을 실제로 실행하는 지점이다.

~~~text
FSReqPromise::Resolve()
  ↓
V8 Resolver::Resolve()
  ↓
V8이 continuation을 마이크로태스크로 예약
  ↓
InternalCallbackScope::Close()
  ↓
PerformCheckpoint()
  ↓
await 이후 JavaScript 실행
~~~

다만 Node는 마이크로태스크만 처리하는 것이 아니다. InternalCallbackScope::Close()는 tick이 예약되어 있는지도 확인하고, Node의 nextTick 처리와 V8 마이크로태스크 처리를 조정한다. 실제 실행 순서는 현재 callback scope와 tick 상태에 따라 Node가 관리한다.

## 7. 콜백 안에서 새 비동기 작업을 등록하는 경우

~~~js
setTimeout(() => {
  console.log('timer');

  fs.readFile('file.txt', () => {
    console.log('file read');
  });
}, 0);
~~~

fs.readFile()이 별도의 이벤트 루프를 만드는 것은 아니다. 이미 실행 중인 같은 uv_loop_t에 새 작업을 등록한다.

~~~text
Node의 기존 uv_run()
  ↓
타이머 콜백 실행
  ↓
fs.readFile()이 같은 libuv loop에 작업 등록
  ↓
타이머 콜백 종료
  ↓
기존 uv_run()으로 복귀
  ↓
파일 읽기 완료 처리
  ↓
file read JavaScript 콜백 실행
~~~

새로운 uv_run()이 콜백 안에서 자동으로 중첩 호출되는 것이 아니다. 현재 JavaScript 콜백이 반환된 뒤 이미 실행 중인 이벤트 루프가 다음 작업을 처리한다.

## 8. Promise 기반 API

~~~js
const fs = require('fs/promises');

fs.readFile('file.txt').then((data) => {
  console.log('file read');
});
~~~

Promise는 지금 당장 결과가 나오지 않는 작업의 미래 결과를 표현하는 JavaScript 객체다. 별도의 스레드가 아니라 작업 상태와 완료 후 실행할 코드를 연결하는 객체다.

~~~text
작업 시작
  ↓
pending: 아직 결과 없음
  ├─ 성공 → fulfilled
  └─ 실패 → rejected
~~~

- pending: 아직 결과가 결정되지 않음
- fulfilled: 작업이 성공했고 결과를 사용할 수 있음
- rejected: 작업이 오류로 실패함

fulfilled나 rejected로 결정된 Promise를 settled 상태라고 한다. 한 번 settled가 되면 다시 pending으로 돌아가지 않는다.

## 9. Promise가 fulfilled로 바뀌는 시점

~~~text
fs.promises.readFile() 호출
  ↓
Node가 Promise 생성
  ↓
Promise = pending
  ↓
Node가 libuv에 파일 읽기 등록
  ↓
사용자 JavaScript는 다음 코드로 진행하거나 await에서 중단
  ↓
uv_run()이 완료된 파일 읽기 작업 처리
  ↓
Node C++가 결과를 Promise에 전달
  ↓
Promise = fulfilled
  ↓
then 또는 await 후속 코드가 V8 마이크로태스크 큐에 등록
~~~

fulfilled로 바뀌는 순간은 파일 읽기 요청을 등록한 때가 아니다. 운영체제와 libuv 작업이 성공적으로 끝나고 Node C++가 결과를 Promise에 전달한 때다.

## 10. V8 마이크로태스크 큐

then의 함수나 await 뒤의 코드는 일반 libuv 콜백 큐에 직접 들어가지 않는다. Promise가 완료되면 V8의 마이크로태스크 큐에 후속 실행이 등록된다.

~~~text
Promise fulfilled
  ↓
V8 마이크로태스크 큐에 continuation 등록
  ↓
현재 JavaScript 실행 종료
  ↓
마이크로태스크 실행
~~~

~~~js
fs.promises.readFile('file.txt').then(() => {
  console.log('done');
});
~~~

~~~text
libuv 파일 읽기 완료
  ↓
Node C++가 Promise resolve
  ↓
V8이 then 후속 작업을 마이크로태스크 큐에 등록
  ↓
현재 처리 종료
  ↓
then 콜백 실행
~~~

모든 마이크로태스크가 uv_run()을 필요로 하지는 않는다.

~~~js
Promise.resolve().then(() => {
  console.log('done');
});
~~~

이미 완료된 Promise를 사용하므로 libuv 작업이나 새로운 uv_run() 없이 V8 마이크로태스크로 이어진다.

## 11. process.nextTick과 마이크로태스크

Node에는 V8 마이크로태스크 큐와 별도로 process.nextTick() 큐가 있다.

~~~js
process.nextTick(() => {
  console.log('nextTick');
});

Promise.resolve().then(() => {
  console.log('promise');
});
~~~

일반적인 Node 실행 흐름:

~~~text
현재 JavaScript 실행 종료
  ↓
process.nextTick() 큐
  ↓
V8 마이크로태스크 큐
  ↓
다음 libuv 이벤트
~~~

process.nextTick() 큐에는 사용자가 등록한 함수와 Node 내부 후속 작업이 들어갈 수 있다. 파일 I/O 콜백이나 타이머 콜백 자체가 전부 이 큐에 들어가는 것은 아니다.

## 12. 연속된 await

~~~js
async function work() {
  console.log('A');

  await Promise.resolve();
  console.log('B');

  await Promise.resolve();
  console.log('C');
}

work();
console.log('D');
~~~

출력:

~~~text
A
D
B
C
~~~

첫 번째 await에 도달하면 B 이후의 코드가 첫 번째 마이크로태스크로 등록된다. 첫 번째 continuation이 실행되어 두 번째 await에 도달해야 C 이후의 코드가 두 번째 마이크로태스크로 등록된다.

~~~text
work()
  ├─ A 출력
  ├─ await 1에서 중단
  └─ continuation 1 등록

현재 스크립트
  └─ D 출력

마이크로태스크 1
  ├─ B 출력
  ├─ await 2에서 중단
  └─ continuation 2 등록

마이크로태스크 2
  └─ C 출력
~~~

두 번째 continuation이 처음부터 첫 번째 것과 함께 큐에 들어가는 것은 아니다. 두 번째 await에 도달하는 코드가 첫 번째 continuation 안에 있기 때문이다.

## 13. await와 libuv

~~~js
const fs = require('fs/promises');

async function read() {
  console.log('A');

  const first = await fs.readFile('first.txt');
  console.log('B');

  const second = await fs.readFile('second.txt');
  console.log('C');
}

read();
console.log('D');
~~~

~~~text
read() 실행
  ↓
A 출력
  ↓
first.txt 읽기 요청이 libuv에 등록
  ↓
첫 번째 await에서 read() 중단
  ↓
D 출력
  ↓
uv_run()이 first.txt 완료 처리
  ↓
Promise fulfilled
  ↓
첫 번째 await 뒤 코드가 마이크로태스크로 등록
  ↓
B 출력
  ↓
second.txt 읽기 요청이 libuv에 등록
  ↓
두 번째 await에서 다시 중단
  ↓
uv_run()이 second.txt 완료 처리
  ↓
Promise fulfilled
  ↓
두 번째 await 뒤 코드가 마이크로태스크로 등록
  ↓
C 출력
~~~

await 하나마다 uv_run()이 하나씩 호출되는 것은 아니다.

- await Promise.resolve()는 libuv 없이 V8 마이크로태스크만 사용할 수 있다.
- await fs.promises.readFile()은 파일 읽기 완료를 위해 libuv 이벤트 처리가 필요하다.
- UV_RUN_DEFAULT에서는 한 번의 uv_run()이 여러 작업을 처리할 수 있다.
- Node의 외부 SpinEventLoopInternal()이 필요하면 uv_run()을 다시 호출한다.

따라서 await와 uv_run()은 일대일 관계가 아니다. await는 async 함수의 실행을 Promise 완료 뒤로 나누고, libuv 기반 Promise의 완료는 이벤트 루프가 처리한다.

## 14. 이벤트 루프가 이미 실행 중일 때

~~~js
setTimeout(() => {
  console.log('timer');

  fs.readFile('file.txt', () => {
    console.log('file read');
  });
}, 0);
~~~

~~~text
사용자 JavaScript 종료
  ↓
uv_run()
  ↓
타이머 발견
  ↓
timer JavaScript 콜백 실행
  ↓
fs.readFile()이 같은 loop에 작업 등록
  ↓
timer 콜백 종료
  ↓
Node C++로 복귀
  ↓
기존 uv_run()이 다음 이벤트 처리
  ↓
file read 콜백 실행
~~~

새로운 이벤트 루프가 생기거나 uv_run()이 콜백 안에서 중첩 호출되는 것이 아니다.

## 15. JavaScript가 콜백을 막는 경우

~~~js
fs.readFile('file.txt', () => {
  console.log('file read');
});

while (true) {}
~~~

fs.readFile() 요청은 libuv에 등록될 수 있지만 사용자 JavaScript가 반환하지 않아 SpinEventLoopInternal()에 도달하지 못할 수 있다. 그러면 최초 uv_run()이 호출되지 않고 JavaScript 콜백도 실행되지 않는다.

반대로 이벤트 루프 콜백 안에서 while (true)를 실행하면 uv_run()은 이미 실행 중이다. 하지만 JavaScript 콜백이 반환하지 않아 uv_run() 내부 C 코드로 복귀하지 못하고 다음 콜백을 처리할 수 없다.

## 확인한 사실과 정정한 이해

- JavaScript API 호출은 Node 내부 JavaScript와 C++ 바인딩을 거쳐 libuv 요청으로 연결된다.
- libuv 작업 등록과 uv_run() 실행은 별개의 단계다.
- uv_run()은 libuv C 이벤트 루프를 실행하고 Node C++ 완료 콜백을 호출한다.
- Node C++ 완료 콜백이 V8을 호출해 JavaScript 콜백을 실행한다.
- Promise 후속 작업은 V8 마이크로태스크 큐에서 실행된다.
- process.nextTick()은 V8 마이크로태스크와 별도의 Node 큐다.
- 연속된 await는 각 후속 실행을 별도의 continuation으로 나눈다.

처음에는 libuv가 작업을 끝내면 uv_run()을 호출한다고 생각하기 쉬웠다. 실제로는 Node가 uv_run()을 호출하고 실행 중인 libuv 이벤트 루프가 완료된 작업을 확인한다.

또한 V8로 넘어온 모든 함수가 마이크로태스크 큐에 들어가는 것도 아니다. 일반적인 libuv 완료 콜백은 Node C++가 V8을 통해 JavaScript 콜백으로 호출할 수 있다. 그 콜백 안에서 Promise 후속 작업이 생겼을 때 해당 작업이 마이크로태스크가 된다.

## 정리

~~~text
콜백 기반:
JavaScript API
  ↓
Node 내부 JavaScript
  ↓
internalBinding('fs')
  ↓
Node C++ 바인딩
  ↓
libuv 작업 등록
  ↓
Node가 uv_run() 호출
  ↓
libuv 작업 완료
  ↓
Node C++ 완료 콜백
  ↓
V8
  ↓
JavaScript 콜백

Promise 기반:
fs.promises.readFile()
  ↓
Promise pending
  ↓
libuv에 파일 읽기 등록
  ↓
uv_run()이 완료 처리
  ↓
Node C++가 Promise resolve
  ↓
Promise fulfilled
  ↓
V8 마이크로태스크 큐
  ↓
then 또는 await 후속 코드 실행
~~~

핵심은 다음과 같다.

JavaScript가 Node API를 호출하면 Node C++가 libuv에 작업을 등록한다. Node가 uv_run()으로 이벤트 루프를 구동하면 libuv가 완료된 작업의 C/C++ 콜백을 호출하고, Node C++가 V8을 통해 JavaScript 콜백이나 Promise 완료를 연결한다.

## 남은 질문

- FSReqCallback::Resolve()가 V8 MakeCallback()을 호출하는 정확한 인자와 스코프는 무엇인가?
- fs.promises.readFile()은 콜백 기반 fs.readFile()과 내부적으로 어느 부분을 공유하는가?
- Node는 각 native callback 뒤에 process.nextTick()과 V8 마이크로태스크를 정확히 어떤 순서로 비우는가?
- uv_run() 내부의 타이머·pending·poll·check·close 단계와 Node의 콜백 실행 시점은 어떻게 연결되는가?
- UV_RUN_ONCE로 바꿨을 때 SpinEventLoopInternal()과 beforeExit 시점에 어떤 차이가 생기는가?
