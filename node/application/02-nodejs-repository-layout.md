# Node.js 저장소의 디렉터리와 파일 확장자 구조

상태: 초안

이 문서는 Node.js 저장소를 처음 읽을 때 필요한 큰 지도를 만든다. 특정 메서드의 구현을 바로 따라가기 전에 각 디렉터리가 어떤 책임을 갖는지, 같은 이름의 파일이 왜 여러 개 존재하는지, 자주 보이는 확장자가 무엇을 의미하는지 정리한다.

## 기준 코드

- 저장소: nodejs/node
- 기준 커밋: 30fa54a3fe
- 로컬 경로: /Users/dustin/Desktop/common/open-source/node

저장소는 계속 변경되므로 파일의 역할은 위 기준 커밋에서 확인한 내용을 기준으로 한다.

## Node 저장소를 읽는 기본 그림

~~~text
사용자 JavaScript
  ↓
lib/
  ↓
src/
  ↓
deps/
  ↓
운영체제
~~~

- lib/: Node 공개 API와 내부 JavaScript
- src/: Node 런타임의 C/C++ 코드
- deps/: V8, libuv, OpenSSL 등 외부 의존성
- test/: 기능과 회귀 검증
- benchmark/: 성능 측정
- tools/: 빌드·테스트·개발 도구
- doc/: 개발자와 기여자 문서
- typings/: TypeScript 타입 선언

이 구조는 호출 계층과 완전히 일치하는 것은 아니다. 예를 들어 test와 tools는 실행 경로에 참여하기보다 개발 과정에서 코드를 검증하거나 빌드하는 역할을 한다.

## 최상위 디렉터리

### lib/

Node의 JavaScript 구현이 들어 있다.

~~~text
lib/
  ├─ fs.js
  ├─ net.js
  ├─ events.js
  ├─ stream.js
  └─ internal/
~~~

사용자가 호출하는 공개 API와 그 API를 구현하는 내부 모듈이 함께 있다.

~~~text
사용자 fs.readFile()
  ↓
lib/fs.js
  ↓
lib/internal/fs/
  ↓
Node C++ 바인딩
~~~

### src/

Node 런타임의 C/C++ 코드가 들어 있다.

~~~text
src/
  ├─ node_main.cc
  ├─ node.cc
  ├─ node_file.cc
  ├─ node_file.h
  ├─ env.cc
  └─ env.h
~~~

주요 책임은 다음과 같다.

- V8과 Node API 연결
- libuv와 Node 연결
- 파일·네트워크·타이머 native 바인딩
- 프로세스 초기화
- Environment 관리
- C++에서 JavaScript 콜백 호출

### deps/

Node가 사용하는 외부 프로젝트의 소스가 들어 있다.

~~~text
deps/
  ├─ v8/
  ├─ uv/
  ├─ openssl/
  ├─ icu-small/
  └─ zlib/
~~~

Node가 실행될 때 인터넷에서 이 프로젝트를 가져온다는 뜻이 아니다. Node가 특정 버전의 의존성을 소스 트리에 포함하고 빌드에 연결한다는 뜻이다.

~~~text
Node C++
  ├─ V8 호출
  └─ libuv 호출

deps/v8/
  └─ JavaScript 엔진 구현

deps/uv/
  └─ 이벤트 루프와 운영체제 I/O 구현
~~~

### test/

기능이 의도대로 동작하는지 검증하는 테스트가 들어 있다. 테스트 파일은 일반 코드처럼 보여도 테스트 러너의 종료 코드, 출력, fixture, 플랫폼 조건과 함께 읽어야 한다.

### benchmark/

정답 여부를 확인하는 test와 달리 성능을 측정한다.

~~~text
test       → 기능과 회귀 검증
benchmark  → 실행 시간·메모리·처리량 측정
~~~

### tools/

빌드, 테스트, V8 의존성 관리, 스냅샷 생성, 결과 변환 같은 개발 도구가 들어 있다.

예:

~~~text
tools/gyp_node.py
tools/v8/
tools/snapshot/
tools/test.py
~~~

### doc/

사용자 API 문서뿐 아니라 Node 개발자와 기여자를 위한 문서가 들어 있다. 코드를 읽을 때 doc/contributing을 먼저 읽으면 테스트와 빌드의 의도를 이해하는 데 도움이 된다.

### typings/

JavaScript로 구현된 Node API를 TypeScript가 검사할 수 있도록 타입 선언을 제공한다.

~~~text
JavaScript 구현
  +
TypeScript 선언
~~~

구현 파일과 타입 선언 파일은 같은 디렉터리에 있을 필요가 없다.

## 같은 이름의 파일이 여러 개 있는 이유

### 헤더와 구현 분리

~~~text
src/node_file.h
src/node_file.cc
~~~

일반적으로 .h는 함수·클래스·자료형의 선언을 담고 .cc는 실제 구현을 담는다.

~~~cpp
// node_file.h
void ReadFile();
~~~

~~~cpp
// node_file.cc
void ReadFile() {
  // 실제 구현
}
~~~

헤더는 여러 구현 파일에서 포함할 수 있기 때문에 기능의 외부 형태와 내부 구현을 분리할 수 있다.

### 플랫폼별 구현

~~~text
lib/path.js
lib/path/posix.js
lib/path/win32.js
~~~

공통 진입점이 현재 플랫폼을 확인하고 Unix 계열 구현 또는 Windows 구현을 선택하는 구조다.

~~~text
path.js
  ├─ Unix 계열 → path/posix.js
  └─ Windows   → path/win32.js
~~~

따라서 같은 이름이 반복된다고 중복 코드라는 뜻은 아니다. 선언·구현, 플랫폼별 구현, 공개 API·내부 구현, 소스·생성 파일이 서로 나뉜 것일 수 있다.

## 자주 보게 될 확장자

### JavaScript 계열

- .js: Node API와 내부 JavaScript
- .mjs: 명시적인 ECMAScript Module
- .cjs: 명시적인 CommonJS 모듈
- .ts: TypeScript 소스
- .mts: TypeScript ECMAScript Module 계열
- .cts: TypeScript CommonJS 계열

Node 런타임의 핵심이 모두 TypeScript로 작성된 것은 아니다. 파일이 어느 디렉터리에 있고 실제 실행 경로에 포함되는지 함께 확인해야 한다.

### C와 C++

- .c: C 소스
- .cc: C++ 소스
- .cpp: C++ 소스
- .h: C 또는 C++ 헤더
- .hpp: C++ 헤더
- .inc: 다른 소스에 포함되는 코드 조각 또는 선언

Node 코어는 .cc와 .h를 많이 사용하고, deps 아래 외부 프로젝트에서는 .c, .cpp, .hpp 등 다양한 관습을 볼 수 있다.

~~~text
src/
  → Node C++ 코드가 많음

deps/uv/
  → libuv C 코드가 많음
~~~

### 빌드 설정

- .gyp, .gypi: GYP 빌드 정의와 공통 설정
- .gn, .gni: GN 빌드 정의와 재사용 설정
- .mk: Make 기반 설정
- .cmake: CMake 설정
- .py, .pl, .sh, .bat: 빌드·테스트·개발 스크립트

Node의 빌드는 컴파일러 명령 하나로 끝나지 않는다. 플랫폼과 컴파일러에 맞는 빌드 파일을 생성하고, 그 결과가 src와 deps의 소스를 실행 파일로 묶는다.

### 생성·실행 관련 파일

- .snapshot: V8 또는 Node 실행에 사용되는 스냅샷 데이터
- .map: 소스 맵
- .wasm: WebAssembly 바이너리
- .S, .s: 어셈블리 소스
- .ld, .lds: 링커 스크립트

이런 파일은 순수한 JavaScript나 C++ 구현이 아니므로, 발견하면 소스인지 생성 결과인지 먼저 확인해야 한다.

## 빌드 진입점

최상위의 다음 파일들은 런타임 API가 아니라 빌드 과정을 담당한다.

~~~text
configure
configure.py
Makefile
vcbuild.bat
node.gyp
node.gypi
BUILD.gn
~~~

대략적인 흐름은 다음과 같다.

~~~text
configure
  ↓
운영체제·컴파일러·옵션 확인
  ↓
config.mk·config.gypi 등 생성
  ↓
GYP·GN·Make·Visual Studio 빌드
  ↓
Node 실행 파일 생성
~~~

config.mk, config.gypi, config.status, out/처럼 현재 환경에서 생성되는 항목은 직접 작성된 핵심 소스와 구분해야 한다.

## 파일을 읽을 때 먼저 물어볼 것

새 파일을 발견하면 다음 질문을 순서대로 한다.

1. 사람이 직접 관리하는 소스인가?
2. 빌드 과정에서 자동 생성되는가?
3. 외부 저장소에서 가져온 코드인가?
4. 테스트나 벤치마크에서만 사용하는가?
5. 플랫폼별 구현인가?
6. 다른 파일에 포함되는 선언 또는 설정인가?

이 질문을 거치면 낯선 확장자를 단순히 이름으로 외우지 않고 저장소에서의 역할로 이해할 수 있다.

## fs.readFile 경로로 확인한 실제 디렉터리 연결

디렉터리의 역할은 파일 이름만 외우는 것보다 하나의 실제 API를 끝까지 따라가면서 확인하는 편이 쉽다. Promise 기반 파일 읽기를 예로 들면 다음과 같은 계층을 통과한다.

~~~text
lib/fs/promises.js
  ↓
lib/internal/fs/promises.js
  ↓
internalBinding('fs')
  ↓
src/node_file.cc
  ↓
src/node_file-inl.h
  ↓
deps/uv/include/uv.h
  ↓
deps/uv/src/unix/fs.c
  ↓
deps/uv/src/threadpool.c
  ↓
운영체제 파일 시스템
~~~

### lib/fs/promises.js와 lib/internal/fs/promises.js

lib/fs/promises.js는 공개 진입점이다. 실제 Promise 기반 파일 시스템 구현은 lib/internal/fs/promises.js에 있다.

~~~text
require('fs/promises')
  ↓
lib/fs/promises.js
  ↓
lib/internal/fs/promises.js
~~~

lib/internal/fs/promises.js는 internalBinding('fs')로 C++ 바인딩을 가져온 뒤 open, fstat, read, close 같은 작업을 순서대로 호출한다. 따라서 공개 API 파일과 실제 작업 조율 파일이 분리되어 있다.

### src/node_file.h, src/node_file.cc, src/node_file-inl.h

파일 시스템 C++ 바인딩은 세 가지 파일을 함께 봐야 한다.

~~~text
src/node_file.h
  → 클래스·함수 선언과 자료형

src/node_file.cc
  → 일반 함수 구현과 JavaScript 메서드 등록

src/node_file-inl.h
  → 템플릿·inline 구현
~~~

node_file.h의 FSReqBase는 libuv의 uv_fs_t 요청을 Node 객체로 감싼 공통 요청 클래스다. FSReqCallback은 callback API 결과를 전달하고, FSReqPromise는 Promise Resolver를 통해 결과를 전달한다.

node_file.cc에는 다음과 같은 등록 코드가 있다.

~~~cpp
SetMethod(isolate, target, "read", Read);
SetMethod(isolate, target, "open", Open);
SetMethod(isolate, target, "close", Close);
~~~

이 등록으로 internalBinding('fs')가 반환하는 객체의 read, open, close가 C++ 함수와 연결된다.

node_file-inl.h에는 FSReqPromise의 Resolve와 Reject가 있다. 이 파일은 템플릿 구현을 선언부와 가까이 두기 위한 파일이다. Promise 요청을 만들고 V8 Promise Resolver를 호출하는 세부 구현을 여기서 확인할 수 있다.

### deps/uv/include와 deps/uv/src

libuv도 선언과 구현을 나눈다.

~~~text
deps/uv/include/uv.h
  → Node가 호출하는 libuv 공개 함수 선언

deps/uv/src/
  → 함수의 실제 구현
~~~

uv_fs_read()는 uv.h에 선언되어 있고 Unix 구현은 deps/uv/src/unix/fs.c에 있다. Windows에서는 deps/uv/src/win/fs.c의 플랫폼별 구현을 사용한다.

### deps/uv/src/unix/fs.c와 threadpool.c

Unix 파일 읽기의 대표 경로는 다음과 같다.

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

작업이 끝나면 완료 큐를 통해 다음 경로로 돌아온다.

~~~text
uv__work_done()
  ↓
uv__fs_done()
  ↓
Node가 등록한 AfterInteger() 또는 AfterStat()
~~~

deps/uv/src/unix/fs.c는 파일 작업의 Unix 구현을, deps/uv/src/threadpool.c는 작업 제출과 완료 큐 처리를 담당한다. 두 파일은 역할이 다르므로 파일 시스템 호출과 완료 전달을 구분해서 읽어야 한다.

## 현재 문서에서 확인한 호출 지도

~~~text
공개 Promise API
  → lib/fs/promises.js
  → lib/internal/fs/promises.js
  → internalBinding('fs')
  → src/node_file.cc의 Open·Read·Close
  → src/node_file-inl.h의 FSReqPromise
  → AsyncCall()
  → deps/uv/include/uv.h의 uv_fs_*
  → deps/uv/src/unix/fs.c
  → deps/uv/src/threadpool.c
  → 운영체제
~~~

결과가 돌아오는 경로는 반대 방향이다.

~~~text
운영체제
  → libuv 완료 큐
  → uv__work_done()
  → uv__fs_done()
  → Node C++ 완료 함수
  → FSReqPromise::Resolve()
  → V8 Promise Resolver
  → 마이크로태스크
  → await 이후 JavaScript
~~~

## 현재까지 확인한 것

- lib/는 JavaScript 공개 API와 내부 구현을 담는다.
- src/는 Node C/C++ 런타임을 담는다.
- deps/는 V8, libuv 등 외부 의존성을 담는다.
- test/는 기능과 회귀를 검증한다.
- benchmark/는 성능을 측정한다.
- tools/는 빌드와 개발을 돕는다.
- doc/는 사용자·개발자 문서를 담는다.
- typings/는 TypeScript 타입 선언을 담는다.
- .h와 .cc는 선언과 구현을 나누는 대표적인 C++ 파일 쌍이다.
- 같은 이름의 파일은 플랫폼, 계층, 선언·구현, 생성 여부에 따라 나뉠 수 있다.
- .inl.h는 템플릿이나 inline 구현을 선언부와 가까이 두는 데 사용될 수 있다.
- lib/fs/promises.js는 공개 진입점이고 lib/internal/fs/promises.js가 작업 순서를 조율한다.
- src/node_file.cc는 JavaScript 메서드를 C++ 함수에 등록하고, src/node_file-inl.h는 FSReqPromise 같은 템플릿 구현을 담는다.
- deps/uv/include/uv.h는 libuv API 선언, deps/uv/src는 구현이며 unix와 win 디렉터리에서 플랫폼별 코드를 나눈다.
- deps/uv/src/threadpool.c는 파일 작업 제출과 완료 큐 처리를 담당한다.
- 하나의 API를 이해하려면 lib → src → deps/uv/include → deps/uv/src → 운영체제 경로를 함께 봐야 한다.

## 다음에 확인할 질문

- test/fs의 테스트는 lib/fs.js와 어떤 방식으로 매핑되는가?
- node.gyp와 BUILD.gn은 같은 소스를 어떤 방식으로 빌드 타깃에 포함하는가?
- .tq 파일은 V8과 Node 빌드에서 어떤 역할을 하는가?
