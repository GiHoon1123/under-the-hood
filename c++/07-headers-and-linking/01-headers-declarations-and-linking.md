# 헤더, 선언, 정의, 링크

상태: 기초 정리 완료

## 선언과 정의

```cpp
// calculator.h
int add(int a, int b); // 선언
```

```cpp
// calculator.cc
int add(int a, int b) { // 정의
  return a + b;
}
```

선언은 함수의 존재와 형태를 알리고, 정의는 실제 동작을 제공한다.

## `#include`

```cpp
#include "calculator.h"
```

전처리기가 헤더의 내용을 현재 소스에 포함한다. `main.cc`는 `add()`의 선언을 알고 호출 코드를 컴파일할 수 있다. 실제 구현은 링크 단계에서 연결된다.

## 컴파일 단위

```text
main.cc       → main.o
calculator.cc → calculator.o
```

각 `.cc` 파일은 보통 독립적으로 컴파일된다. `main.o`에는 `add()`를 호출한다는 정보가 있고, `calculator.o`에는 실제 구현이 있다.

## 링크

```text
main.o + calculator.o → program
```

링커가 호출과 구현을 연결한다. 구현 파일을 빼면 `undefined reference` 같은 링크 오류가 발생한다.

Node의 빌드에서도 다음이 함께 연결된다.

```text
src/node_file.cc
src/env.cc
deps/uv/src/*.c
deps/v8/src/*.cc
  ↓
각 목적 파일
  ↓
node 실행 파일
```

## 중복 포함 방지

헤더는 여러 소스에서 포함될 수 있으므로 중복 정의를 막는다.

```cpp
#ifndef REQUEST_H
#define REQUEST_H

class Request {};

#endif
```

또는 다음을 사용할 수 있다.

```cpp
#pragma once
```

## 전방 선언

전체 클래스 정의를 몰라도 포인터나 참조를 선언할 수 있다.

```cpp
class Request;

void Process(Request* request);
```

하지만 멤버에 접근하려면 전체 정의가 필요하다.

```cpp
request->result; // Request의 정의 필요
```

## Node 파일 배치

```text
src/node_file.h
  → 클래스·함수 선언

src/node_file.cc
  → 일반 구현·JavaScript 메서드 등록

src/node_file-inl.h
  → 템플릿·inline 구현
```

`node_file.cc`가 `uv_fs_read()`를 호출하는 것은 관련 libuv 헤더 선언을 포함하고, 최종 링크에서 libuv 구현이 함께 연결되기 때문이다.

## `.inl.h`와 inline

템플릿 구현은 사용하는 타입을 컴파일하는 곳에서 보여야 하므로 선언과 가까운 헤더에 두는 경우가 많다. `.inl.h`는 이런 inline·템플릿 구현을 담는 관습적인 이름이다.

`inline`이 있다고 항상 함수가 호출 없이 펼쳐진다는 뜻은 아니다. 실제 인라이닝 최적화는 컴파일러가 결정한다.

## 정리

```text
.h       → 선언과 타입 정보
.cc      → 실제 구현
#include → 선언을 현재 소스에서 사용
컴파일   → 소스별 목적 파일 생성
링크     → 목적 파일·라이브러리 연결
```

코드를 읽을 때 함수의 선언 위치, 구현 위치, 포함 헤더, 빌드 타깃을 함께 확인한다.

## 선언만 있고 구현이 다른 파일에 있는 이유

```cpp
// request.h
class Request {
 public:
  void Resolve();
};
```

```cpp
// request.cc
#include "request.h"

void Request::Resolve() {
  // 실제 구현
}
```

다른 파일은 헤더만 포함하고 `Resolve()`를 호출할 수 있다. 구현을 사용하는 파일마다 복사하지 않기 때문에 빌드 구조가 관리하기 쉬워진다.

## 헤더를 읽을 때의 순서

1. 이 헤더가 선언하는 타입과 함수는 무엇인가?
2. 멤버가 값인지 포인터인지 참조인지 확인한다.
3. 구현이 같은 파일에 있는지 `.cc`나 `.inl.h`에 있는지 찾는다.
4. 사용하는 소스가 빌드 타깃에 포함되는지 `node.gyp`나 `BUILD.gn`에서 확인한다.

## 링크 오류를 읽는 법

```text
컴파일 오류:
  선언이나 문법을 이해하지 못함

링크 오류:
  선언은 봤지만 실제 정의를 연결하지 못함
```

Node 코드를 수정할 때 새 함수 선언만 추가하고 구현 또는 빌드 타깃을 빠뜨리면 컴파일은 통과해도 링크 단계에서 실패할 수 있다.

## 전처리와 헤더의 관계

`#include`는 파일을 실행 중 불러오는 것이 아니다. 전처리 단계에서 헤더의 내용을 현재 소스에 삽입한다. 따라서 헤더 중복 포함 방지와 플랫폼 매크로가 실제 컴파일 결과에 영향을 준다.
