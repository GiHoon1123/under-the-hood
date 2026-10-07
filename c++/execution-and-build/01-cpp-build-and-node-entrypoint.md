# C++ 코드가 실행 파일이 되는 과정

상태: 코드 흐름 정리 완료

## 이 글에서 알아볼 것

- C++ 소스가 실행 파일이 되는 과정
- 전처리, 컴파일, 링크의 차이
- 헤더와 구현 파일이 빌드에 참여하는 방식
- Node.js의 `main()`과 빌드 파일이 이 과정에서 맡는 역할

## 기준 코드

- 저장소: `nodejs/node`
- 기준 커밋: `30fa54a3fe`
- 관련 파일: `src/node_main.cc`, `src/node.cc`, `configure.py`, `node.gyp`, `BUILD.gn`

## 전체 흐름

```text
소스 파일
  ↓ 전처리
전처리된 소스
  ↓ 컴파일
목적 파일(.o/.obj)
  ↓ 링크
실행 파일
  ↓
프로세스 시작
```

`.cc` 파일은 운영체제가 바로 실행하는 파일이 아니다. 여러 소스 파일과 라이브러리를 컴파일하고 링크한 결과가 실행 파일이다.

## 전처리

전처리기는 `#include`, `#define`, `#if` 같은 지시문을 컴파일 전에 처리한다.

```cpp
#define BUFFER_SIZE 1024

char buffer[BUFFER_SIZE];
```

컴파일 전에는 `BUFFER_SIZE`가 `1024`로 치환된다. 매크로는 실행 중인 함수가 아니라 컴파일 전 코드 변환 규칙이다.

플랫폼 분기도 이 단계에서 처리된다.

```cpp
#ifdef _WIN32
// Windows 코드
#else
// Unix 계열 코드
#endif
```

따라서 같은 저장소를 빌드해도 운영체제에 따라 컴파일되는 코드가 달라질 수 있다.

## 컴파일

전처리된 각 소스 파일은 독립적으로 목적 파일로 변환된다.

```text
src/node_main.cc → node_main.o
src/node.cc      → node.o
src/env.cc       → env.o
src/node_file.cc → node_file.o
deps/uv/src/*.c  → libuv 목적 파일들
```

목적 파일에는 함수의 기계어 코드와 다른 함수에 대한 연결 정보가 들어 있지만, 아직 모든 함수가 연결된 실행 파일은 아니다.

## 링크

링커는 Node의 목적 파일과 V8·libuv 등의 외부 의존성 목적 파일을 하나의 실행 파일로 연결한다.

```text
Node C++ 목적 파일
  + libuv 목적 파일
  + V8 라이브러리
  + OpenSSL 등
  ↓ 링크
node 실행 파일
```

예를 들어 `src/node_file.cc`가 `uv_fs_read()`를 호출할 수 있는 것은 libuv 헤더에서 선언을 보고 컴파일한 뒤, libuv 구현까지 링크하기 때문이다.

## Node의 실행 진입점

터미널에서 `node app.js`를 실행하면 운영체제는 Node 실행 파일의 진입점인 `main()` 또는 Windows의 `wmain()`을 호출한다.

```text
src/node_main.cc
  ↓
main() / wmain()
  ↓
node::Start()
  ↓
StartInternal()
  ↓
NodeMainInstance
  ↓
V8·libuv·Environment 초기화
```

그 뒤 Node는 내부 JavaScript와 사용자 JavaScript를 실행하고, 초기 실행이 끝나면 이벤트 루프를 시작한다.

## 빌드 설정 파일

```text
configure.py → 운영체제·컴파일러·옵션 확인
node.gyp    → GYP 빌드 입력
BUILD.gn    → GN 빌드 입력
Makefile    → Unix 계열 빌드 보조
vcbuild.bat → Windows 빌드 보조
```

이 파일들은 런타임 API를 실행하는 코드가 아니라 어떤 소스를 어떤 옵션으로 컴파일하고 링크할지 결정한다.

## 처음 예상과 달랐던 점

처음에는 `node_main.cc`를 실행하면 Node가 시작된다고 생각하기 쉽지만, 실제로 실행되는 것은 그 파일 자체가 아니라 전체 소스를 빌드한 `node` 실행 파일이다. `main()`은 그 실행 파일 안에 포함된 진입점이다.

## 정리

```text
Node 소스
  ↓ 전처리·컴파일
목적 파일
  ↓ 링크
node 실행 파일
  ↓
main()/wmain()
  ↓
Node 런타임 초기화
```

## 남은 질문

- 실제 빌드 디렉터리에서 컴파일 명령과 링크 명령을 어떻게 확인하는가?
- `node.gyp`와 `BUILD.gn`은 같은 소스를 어떤 타깃으로 묶는가?
- 플랫폼별 `main()` 구현은 어떤 조건으로 선택되는가?

## 첫 사이클에서 헷갈렸던 부분

### 매크로와 인라인은 같은가

매크로는 전처리기가 컴파일 전에 텍스트를 치환하는 장치다.

```cpp
#define SIZE 10
int values[SIZE];
```

컴파일 전에 `SIZE`가 `10`으로 바뀐다. 타입 검사나 실행 시 호출은 없다. 반면 C++ 함수 인라이닝은 실제 함수가 존재하고, 컴파일러가 호출을 함수 본문으로 펼칠지 결정하는 최적화다.

```cpp
inline int add(int a, int b) {
  return a + b;
}
```

둘 다 호출 비용을 줄이는 것처럼 보일 수 있지만 시점과 주체가 다르다.

```text
매크로      → 전처리기, 텍스트 치환
함수 인라이닝 → 컴파일러, 최적화 판단
```

### 빌드와 실행은 다른 단계다

`src/node_main.cc`를 직접 실행하는 것이 아니다. 이 파일은 다른 Node 소스·V8·libuv와 함께 빌드되어 `node` 실행 파일에 포함된다.

```text
소스 파일을 읽음
  ↓
각 파일을 목적 파일로 컴파일
  ↓
목적 파일과 라이브러리를 링크
  ↓
node 실행 파일 생성
  ↓
운영체제가 main()/wmain() 호출
```

따라서 소스의 `main()`은 프로그램이 시작될 때 호출되는 함수이지만, 그 함수가 실행되려면 먼저 빌드 결과물인 실행 파일이 존재해야 한다.

### Node 시작 흐름과 C++ 빌드 흐름의 연결

빌드 시점에는 `src/node_main.cc`가 여러 목적 파일 중 하나가 되고, 실행 시점에는 그 파일에 정의된 진입점이 호출된다.

```text
빌드 시점:
src/node_main.cc → node_main.o → node 실행 파일

실행 시점:
node 실행 파일 → main()/wmain() → node::Start()
```

이 둘을 섞으면 “어떤 파일이 실행되는가?”라는 질문에 혼동이 생긴다. 실제로 실행되는 것은 소스 파일이 아니라 링크 결과다.
