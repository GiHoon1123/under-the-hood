# Node.js는 시작될 때 어떤 코드를 실행하는가

상태: 코드 확인 완료

이 글은 node app.js를 실행했을 때 운영체제가 Node 실행 파일을 시작한 뒤, Node의 C++ 런타임이 V8과 libuv를 준비하고 사용자 JavaScript를 실행한 다음 이벤트 루프에 들어가기까지의 흐름을 정리한다.

## 기준 코드

- 저장소: nodejs/node
- 기준 커밋: 30fa54a3fe
- 로컬 경로: /Users/dustin/Desktop/common/open-source/node
- 주요 파일:
  - src/node_main.cc
  - src/node.cc
  - src/node_main_instance.cc
  - src/api/environment.cc
  - src/api/embed_helpers.cc

코드는 변경될 수 있으므로 이 글은 위 기준 커밋에서 확인한 흐름을 기준으로 한다.

## 전체 흐름

~~~text
운영체제가 node 실행 파일 시작
  ↓
src/node_main.cc의 main() 또는 wmain()
  ↓
node::Start(argc, argv)
  ↓
StartInternal(argc, argv)
  ↓
프로세스 전역 상태와 명령줄 인자 초기화
  ↓
libuv 기본 이벤트 루프 준비
  ↓
NodeMainInstance 생성
  ↓
V8 Isolate와 Node Environment 생성
  ↓
LoadEnvironment()
  ↓
Node 내부 JavaScript와 사용자 JavaScript 실행
  ↓
SpinEventLoopInternal()
  ↓
uv_run()
~~~

여기서 가장 중요한 구분은 다음 두 가지다.

~~~text
libuv에 비동기 작업을 등록하는 것
  ≠
uv_run()으로 이벤트 루프를 실제로 실행하는 것
~~~

## 1. 운영체제와 프로그램 진입점

### Unix 계열의 main

파일: src/node_main.cc

~~~cpp
int main(int argc, char* argv[]) {
  return node::Start(argc, argv);
}
~~~

main()은 Node가 JavaScript를 실행하기 전에 운영체제가 호출하는 C/C++ 프로그램 진입점이다. Node가 main()을 호출하는 것이 아니라, 운영체제가 프로세스를 만들고 실행 파일의 시작 규칙에 따라 main()으로 제어를 넘긴다.

### argc와 argv

~~~bash
node app.js --port 3000
~~~

개념적으로 인자는 다음처럼 전달된다.

~~~text
argc = 4

argv[0] = node 실행 파일 경로
argv[1] = app.js
argv[2] = --port
argv[3] = 3000
~~~

argc는 인자의 개수이고 argv는 각 인자를 가리키는 문자열 배열이다. 일반적으로 argv[0]은 실행 파일의 경로 또는 이름이다.

char* argv[]에서 char*는 문자 데이터의 주소를 가리키는 포인터다. 여기서는 C 포인터 문법 전체보다 argv[i]가 i번째 명령줄 인자 문자열을 가리킨다는 점이 중요하다.

### Windows의 wmain

Windows에서는 진입점이 wmain()으로 구성된다.

실제 구현은 Windows 인자를 UTF-8 형태로 변환하고, 변환된 배열을 Unix 계열과 같은 Node 시작 경로로 넘긴다. 개념적으로는 다음과 같이 볼 수 있다.

~~~cpp
int wmain(int argc, wchar_t* wargv[]) {
  char** argv = ConvertWindowsArguments(wargv);
  return node::Start(argc, argv);
}
~~~

위 코드는 실제 함수 이름을 그대로 옮긴 것이 아니라 변환 단계를 보여주는 단순화한 예시다.

Unix의 main()이 char* 문자열 배열을 받는 것과 달리 Windows의 wmain()은 wchar_t* 배열을 받는다. Windows 명령줄 인자가 넓은 문자 형태로 전달되기 때문이다.

따라서 Node는 Windows 진입점에서 받은 인자를 내부에서 사용할 문자열 형식으로 변환한다.

- Unix 계열: main(int, char**)
- Windows: wmain(int, wchar_t**)

플랫폼은 운영체제와 그 운영체제의 실행 파일, 문자열, ABI 규칙을 묶어서 말한다. 이 변환은 이벤트 루프를 실행하는 과정이 아니라 운영체제가 전달한 입력을 Node의 공통 내부 형식으로 맞추는 초기 처리다.

## 2. node::Start와 StartInternal

파일: src/node.cc

~~~cpp
int Start(int argc, char** argv) {
  return static_cast<int>(StartInternal(argc, argv));
}
~~~

Start()는 Node 실행의 바깥 진입 함수이고 실제 프로세스 초기화는 StartInternal()에서 진행된다.

~~~text
main(argc, argv)
  ↓
node::Start(argc, argv)
  ↓
StartInternal(argc, argv)
~~~

이 시점의 argv는 운영체제가 전달한 명령줄 인자 배열이다. 이후 Node는 Node 옵션, 실행할 스크립트, 사용자에게 전달할 인자를 구분한다.

그 결과 사용자가 JavaScript에서 보는 process.argv는 main()의 argv 배열을 그대로 노출한 값이 아니다. Node가 실행 파일 인자와 Node 옵션을 해석한 뒤, JavaScript 프로그램이 사용할 형태로 구성한 결과다. 정확한 구성 과정은 실행 모드와 옵션에 따라 달라지므로 process.argv 자체의 생성 경로는 별도 질문으로 남긴다.

## 3. StartInternal에서 런타임 기반 준비

~~~text
StartInternal()
  ↓
uv_setup_args()
  ↓
InitializeOncePerProcessInternal()
  ↓
uv_default_loop()
  ↓
NodeMainInstance 생성
  ↓
NodeMainInstance::Run()
~~~

uv_setup_args()는 libuv 쪽 인자와 프로세스 이름 관련 초기화에 관여한다. 이 함수는 이벤트 루프를 실행하지 않는다.

InitializeOncePerProcessInternal()은 CLI 옵션, 프로세스 전역 상태, 진단과 경고, V8 플랫폼 초기화에 필요한 준비, 실행 모드에 따른 인자 분리 등을 담당한다.

Node는 기본 libuv 이벤트 루프를 가져온다.

~~~cpp
uv_loop_t* loop = uv_default_loop();
~~~

uv_loop_t는 libuv 이벤트 루프 상태를 담는 C 구조체다. 이 시점은 이벤트 루프 객체를 준비하는 단계이지 uv_run()으로 이벤트 처리를 시작하는 단계가 아니다.

~~~text
이벤트 루프 준비
  ≠
이벤트 루프 실행
~~~

## 4. NodeMainInstance, V8, libuv

파일: src/node_main_instance.cc

NodeMainInstance는 Node 런타임 하나를 실행하기 위한 주요 C++ 객체다.

~~~text
NodeMainInstance
  ├─ V8 Isolate
  ├─ libuv event loop
  ├─ V8 Platform
  ├─ allocator
  └─ Node Environment
~~~

V8 Isolate는 JavaScript 실행 공간이다. JavaScript 객체, 함수, 예외, 가비지 컬렉션 등 V8 실행에 필요한 상태가 Isolate와 연결된다.

NodeMainInstance 생성 과정에서는 NewIsolate()로 Isolate를 만들고 CreateIsolateData()로 Isolate, libuv 이벤트 루프, V8 Platform, 메모리 할당기와 Node 스냅샷 정보를 연결한다.

~~~text
JavaScript
  ↕
V8
  ↕
Node C++
  ↕
libuv
  ↕
운영체제
~~~

Node C++는 V8과 libuv를 연결하는 중간 계층이다.

## 5. NodeMainInstance::Run과 JavaScript 실행

~~~text
NodeMainInstance::Run()
  ↓
LoadEnvironment()
  ↓
StartExecution()
  ↓
Node 내부 JavaScript 실행
  ↓
사용자 JavaScript 실행
  ↓
SpinEventLoopInternal()
~~~

LoadEnvironment()는 사용자 파일만 읽는 함수가 아니다. Node 내부 JavaScript 환경을 준비한 뒤 실행 모드에 맞는 내부 진입 모듈을 시작한다. 기준 커밋의 구현은 다음 네 가지 준비와 실행 선택을 순서대로 수행한다.

~~~text
LoadEnvironment()
  ├─ env->InitializeLibuv()
  ├─ env->InitializeDiagnostics()
  ├─ embedder preload 저장
  ├─ env->InitializeCompileCache()
  └─ StartExecution()
~~~

### InitializeLibuv()

파일: src/env.cc

InitializeLibuv()는 Environment가 사용할 libuv 핸들과 Node 내부 작업 전달 경로를 만든다. 실제 구현에서는 다음과 같은 준비가 일어난다.

- Node 내부 타이머 핸들 초기화
- setImmediate 처리를 위한 check와 idle 핸들 초기화
- native immediate 작업을 전달하기 위한 uv_async 핸들 초기화
- V8 CPU profiler가 이벤트 루프의 유휴 시간을 구분할 수 있도록 prepare/check 핸들 준비
- 이미 다른 스레드에서 등록된 native immediate가 있으면 이벤트 루프를 깨우도록 uv_async_send 호출
- Environment의 libuv 핸들 초기화 상태 표시

여기서 타이머와 check 핸들을 만들었다고 바로 이벤트 루프가 실행되는 것은 아니다. 핸들은 이벤트 루프가 사용할 구조로 등록되고, 실제 처리는 나중에 uv_run()이 호출된 뒤 진행된다. 일부 내부 핸들은 uv_unref로 참조 카운트에서 제외되므로, 그 핸들만 존재한다고 Node 프로세스가 계속 살아 있지는 않는다.

### InitializeDiagnostics()

파일: src/node.cc

InitializeDiagnostics()는 JavaScript 실행과 메모리 상태를 관찰하기 위한 진단 기능을 V8에 연결한다.

- V8 heap profiler에 Node의 embedder graph를 구성하는 콜백 등록
- heap limit 근처에서 스냅샷을 만들기 위한 콜백 준비
- trace_uncaught 옵션이 켜져 있으면 처리되지 않은 예외의 stack trace 캡처 활성화
- trace_promises 옵션이 켜져 있으면 V8 Promise hook 등록

이 함수는 파일 I/O나 이벤트 루프 작업을 시작하지 않는다. 나중에 오류, Promise, 힙 상태를 관찰할 수 있도록 V8의 진단 지점을 설정하는 단계다.

### 임베더 preload 등록

실제 LoadEnvironment()에는 위 네 함수 외에 preload 콜백을 저장하는 단계도 있다.

~~~cpp
if (preload) {
  env->set_embedder_preload(std::move(preload));
}
~~~

Node를 단순히 node app.js로 실행하는 일반 경로에서는 이 콜백이 없을 수 있다. Node를 다른 애플리케이션에 임베드하거나 특수 실행 환경에서 사용하는 경우, Node가 내부 실행을 시작하기 전에 임베더가 제공한 preload 작업을 보관해 두는 단계다.

### InitializeCompileCache()

파일: src/env.cc

InitializeCompileCache()는 환경 변수에 따라 JavaScript 컴파일 캐시를 사용할지 결정한다.

1. NODE_COMPILE_CACHE 환경 변수에서 캐시 디렉터리를 읽는다.
2. 값이 없으면 아무것도 하지 않고 반환한다.
3. NODE_COMPILE_CACHE_PORTABLE이 1이면 상대 경로 기반의 portable 옵션을 선택한다.
4. 조건이 맞으면 EnableCompileCache()를 호출해 캐시를 활성화한다.

캐시의 목적은 JavaScript 소스를 매번 처음부터 같은 방식으로 컴파일하는 비용을 줄이는 것이다. 이 단계는 이벤트 루프에 비동기 작업을 등록하는 것이 아니라, 이후 모듈 로딩과 컴파일에서 사용할 실행 환경을 설정하는 단계다. NODE_DISABLE_COMPILE_CACHE가 설정되어 있으면 활성화 요청이 거부될 수 있다.

### StartExecution()

파일: src/node.cc

StartExecution()은 지금까지 준비한 Environment에서 어떤 내부 JavaScript를 먼저 실행할지 선택한다. 일반적인 node app.js, node -e, REPL, 테스트 실행 모드 등에 따라 경로가 달라진다.

대표적인 선택은 다음과 같다.

- worker context: internal/main/worker_thread
- inspect 인자: internal/main/inspect
- 도움말 옵션: internal/main/print_help
- eval 옵션: internal/main/eval_string
- 문법 검사: internal/main/check_syntax
- test runner: internal/main/test_runner
- watch mode: internal/main/watch_mode
- 일반적인 스크립트 경로: internal/main/run_main_module
- 표준 입력 또는 REPL: internal/main/repl 또는 internal/main/eval_stdin

일반적인 node app.js에서는 internal/main/run_main_module로 이어진다. 이 함수는 Realm의 ExecuteBootstrapper를 통해 Node 내부 JavaScript를 실행하고, 그 과정에서 사용자가 지정한 app.js를 로드한다.

~~~text
Node C++ 호출
  ↓
V8이 Node 내부 JavaScript 실행
  ↓
V8이 사용자 JavaScript 실행
  ↓
사용자 JavaScript 반환
  ↓
Node C++로 제어권 복귀
~~~

## 6. 사용자 JavaScript와 첫 이벤트 루프

~~~js
console.log('start');

setTimeout(() => {
  console.log('timer');
}, 0);

console.log('end');
~~~

~~~text
V8이 app.js 실행
  ↓
console.log('start')
  ↓
setTimeout()이 타이머 작업 등록
  ↓
console.log('end')
  ↓
app.js의 현재 실행 종료
  ↓
Node C++ 런타임으로 복귀
~~~

setTimeout()을 호출했다고 JavaScript 코드 안에서 uv_run()이 직접 호출되는 것은 아니다. Node API가 타이머를 libuv에 등록하고 반환한 뒤, 현재 JavaScript 실행이 끝나면 Node의 실행 흐름이 이벤트 루프 단계로 이동한다.

## 7. SpinEventLoopInternal과 uv_run

파일: src/api/embed_helpers.cc

~~~cpp
do {
  uv_run(env->event_loop(), UV_RUN_DEFAULT);
  platform->DrainTasks(isolate);
  more = uv_loop_alive(env->event_loop());

  if (!more) {
    EmitProcessBeforeExit(env);
    more = uv_loop_alive(env->event_loop());
  }
} while (more);

EmitProcessExitInternal(env);
~~~

uv_run()은 libuv 이벤트 루프를 실제로 구동하는 C 함수다.

~~~c
int uv_run(uv_loop_t* loop, uv_run_mode mode);
~~~

타이머, 운영체제 이벤트, 파일·네트워크 작업의 완료, 완료된 요청의 C/C++ 콜백을 처리한다. JavaScript 콜백을 직접 실행하는 것은 아니다.

~~~text
uv_run()
  ↓
libuv가 완료 이벤트 확인
  ↓
Node C++ 완료 콜백 호출
  ↓
Node C++가 V8 호출
  ↓
V8이 JavaScript 콜백 실행
~~~

UV_RUN_DEFAULT는 이벤트 루프에 살아 있는 작업이 없어질 때까지 실행하는 모드다. UV_RUN_ONCE라면 한 번의 루프 반복 뒤 반환할 수 있지만, Node 바깥의 SpinEventLoopInternal()이 다시 uv_run()을 호출할 수 있다.

### uv_run 이후 libuv가 작업을 완료하는 경로

uv_run()이 실행되었다고 운영체제의 파일 함수가 바로 Node C++에서 호출되는 것은 아니다. libuv는 요청을 등록하고, 작업 종류에 맞는 내부 함수와 완료 큐를 거친다.

파일 읽기 기준의 Unix 경로:

~~~text
uv_run()
  ↓
libuv가 완료 큐와 이벤트를 처리
  ↓
uv__work_done()
  ↓
uv__fs_done()
  ↓
Node가 등록한 완료 콜백
~~~

관련 파일:

- deps/uv/include/uv.h: uv_fs_read() 공개 선언
- deps/uv/src/unix/fs.c: Unix 파일 시스템 구현
- deps/uv/src/threadpool.c: 작업 제출과 완료 큐 처리

uv_fs_read()는 uv_fs_t에 파일 디스크립터, 버퍼, offset, 완료 콜백을 기록한다. 비동기 콜백이 있으면 uv__work_submit()을 통해 uv__fs_work와 uv__fs_done을 작업으로 등록한다.

~~~text
uv_fs_read()
  ↓
uv__work_submit()
  ↓
uv__fs_work()
  ↓
uv__fs_read()
  ↓
운영체제 read() 또는 pread()
~~~

작업이 끝나면 libuv는 loop의 완료 큐를 깨우고, uv_run()이 그 큐를 처리할 때 uv__work_done()을 호출한다. uv__work_done()은 완료된 작업의 done 함수를 호출하고, 파일 작업에서는 그 함수가 uv__fs_done()이다.

~~~text
uv__work_done()
  ↓
w->done(w, status)
  ↓
uv__fs_done()
  ↓
req->cb(req)
~~~

Node가 req->cb로 넘긴 함수는 AfterInteger(), AfterStat() 같은 Node C++ 완료 함수다. 따라서 libuv의 C 작업 완료와 Node C++의 결과 전달이 이 지점에서 연결된다.

## 8. uv_loop_alive와 종료

uv_run()이 반환된 뒤 Node는 이벤트 루프에 아직 작업이 남아 있는지 확인한다. 활성 타이머, 열린 서버, 처리 중인 I/O 요청 등이 있으면 이벤트 루프는 살아 있다고 볼 수 있다.

작업이 없으면 Node는 바로 프로세스를 끝내지 않고 beforeExit을 처리한다.

~~~js
process.on('beforeExit', () => {
  console.log('before exit');
});
~~~

beforeExit에서 새 비동기 작업을 등록하면 이벤트 루프가 다시 살아날 수 있다.

~~~text
uv_run()
  ↓
작업 없음
  ↓
beforeExit
  ├─ 새 작업 있음 → 다시 uv_run()
  └─ 새 작업 없음 → exit
~~~

## 9. 싱글 스레드라는 말의 의미

초기 사용자 JavaScript와 메인 이벤트 루프는 같은 메인 실행 흐름에서 처리된다.

~~~text
사용자 JavaScript
  ↓
Node C++
  ↓
uv_run()
  ↓
Node C++ 완료 콜백
  ↓
V8 JavaScript 콜백
  ↓
다시 Node C++와 libuv
~~~

libuv 내부 스레드풀이나 V8 백그라운드 작업이 별도 스레드를 사용할 수는 있지만, JavaScript 콜백을 실행하는 메인 흐름은 하나다.

최초 JavaScript에서 다음처럼 멈추면:

~~~js
fs.readFile('file.txt', () => {
  console.log('file read');
});

while (true) {}
~~~

fs.readFile() 요청은 libuv에 등록될 수 있지만 사용자 JavaScript가 반환하지 않아 SpinEventLoopInternal()에 도달하지 못할 수 있다. 그러면 최초 uv_run()이 호출되지 않고 JavaScript 콜백도 실행되지 않는다.

반대로 이벤트 루프 콜백 안에서 멈추면 uv_run()이 이미 실행 중이지만 JavaScript 콜백이 반환하지 않아 uv_run() 내부 C 코드로 복귀하지 못한다.

## 확인한 사실과 정정한 이해

- src/node_main.cc의 main() 또는 wmain()이 Node 실행의 C/C++ 진입점이다.
- node::Start()가 StartInternal()로 초기화 흐름을 넘긴다.
- NodeMainInstance가 V8 Isolate와 Node 실행 환경을 준비한다.
- LoadEnvironment() 이후 Node 내부 JavaScript와 사용자 JavaScript가 실행된다.
- 사용자 JavaScript 초기 실행이 끝난 뒤 SpinEventLoopInternal()이 uv_run()을 호출한다.
- uv_run()은 libuv 이벤트 루프를 구동하고 Node C++ 콜백을 호출한다.

처음에는 이벤트 루프가 JavaScript 콜스택을 직접 감시한다고 생각했지만, 실제로는 Node C++가 V8에 JavaScript 콜백을 호출하고 그 콜백이 반환되어 C++ 이벤트 루프로 제어권이 돌아왔을 때 다음 이벤트를 처리한다.

## 정리

~~~text
운영체제
  ↓
main()/wmain()
  ↓ argc, argv
node::Start()
  ↓
StartInternal()
  ↓
libuv loop와 프로세스 상태 준비
  ↓
NodeMainInstance
  ↓
V8 Isolate와 Environment
  ↓
LoadEnvironment()
  ↓
사용자 JavaScript 실행
  ↓
SpinEventLoopInternal()
  ↓
uv_run()
  ↓
libuv 이벤트 처리
  ↓
Node C++ 콜백
  ↓
V8 JavaScript 콜백
  ↓
작업이 없어질 때까지 반복
  ↓
beforeExit / exit
~~~

## 남은 질문

- process.argv는 초기 argc·argv에서 정확히 어떤 변환을 거쳐 만들어지는가?
- internal/main/run_main_module은 사용자 모듈을 어떤 loader 경로로 실행하는가?
- uv_run() 안에서 타이머, pending callback, poll, check, close 단계는 어떤 순서로 진행되는가?
- Node C++의 MakeCallback()은 V8 호출과 nextTick·마이크로태스크 처리에 어떻게 관여하는가?
- Node Worker Thread가 추가되면 이벤트 루프와 Isolate는 어떻게 분리되는가?
