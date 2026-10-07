# 스레드, 이벤트 루프, 동시성

상태: 기초 정리 완료

## 프로세스와 스레드

프로세스는 실행 중인 프로그램이고, 스레드는 프로세스 안에서 실제 코드를 실행하는 흐름이다.

```text
Node 프로세스
  ├─ JavaScript 메인 스레드
  ├─ libuv worker thread pool
  └─ 필요하면 Worker Thread
```

Node의 일반 JavaScript 코드는 메인 스레드에서 실행된다. 그렇다고 모든 운영체제 작업이 메인 스레드에서 직접 실행된다는 뜻은 아니다.

## 비동기와 병렬

비동기는 작업을 맡기고 나중에 결과를 받는 방식이고, 병렬은 여러 실행 흐름이 실제로 동시에 실행되는 것이다. 네트워크 소켓은 운영체제 이벤트 감시로 처리될 수 있지만, 파일 작업은 libuv 스레드풀을 사용하는 경우가 많다.

```text
메인 스레드       → 작업 등록
libuv worker      → 파일 작업
메인 이벤트 루프  → 완료 콜백
```

Promise를 사용한다고 작업이 자동으로 새 스레드에서 실행되는 것은 아니다. Promise는 결과를 JavaScript에 전달하는 표현 방식이다.

## libuv 스레드풀

```text
uv__work_submit()
  ↓
작업 큐
  ↓
worker thread
  ↓
uv__fs_work()
  ↓
운영체제 파일 작업
```

작업이 끝나면 결과는 완료 큐를 거쳐 메인 이벤트 루프로 돌아온다.

```text
worker thread 완료
  ↓
완료 큐
  ↓
uv__work_done()
  ↓
uv__fs_done()
  ↓
Node C++ 완료 콜백
```

JavaScript 콜백은 일반적으로 worker thread에서 실행되지 않는다. 메인 스레드가 결과를 받아 V8을 통해 실행한다.

## 경쟁 조건

여러 스레드가 같은 값을 동시에 수정하면 연산이 섞일 수 있다.

```cpp
int count = 0;
// 두 스레드가 동시에 count++
```

`count++`는 읽기·더하기·쓰기의 여러 단계다. 두 스레드가 같은 이전 값을 읽으면 기대한 2가 아니라 1이 저장될 수 있다.

## mutex와 atomic

mutex는 한 번에 하나의 스레드만 보호 구역에 들어가게 한다.

```cpp
std::lock_guard<std::mutex> lock(mutex);
count = count + 1;
```

`lock_guard`는 RAII로 잠금을 획득하고 범위를 벗어날 때 해제한다.

단순한 값의 원자적 갱신에는 `std::atomic`을 사용할 수 있다.

```cpp
std::atomic<int> count = 0;
count++;
```

atomic이 여러 변수 사이의 복잡한 불변식을 모두 보호해주는 것은 아니다. 그런 경우에는 mutex나 더 큰 자료구조 설계가 필요하다.

## Node의 메인 스레드와 libuv

```text
JavaScript
  → Node C++
  → libuv 작업 등록
  → worker thread 또는 OS 이벤트
  → 메인 이벤트 루프
  → JavaScript 콜백
```

메인 스레드의 JavaScript 콜백이 `while (true)`처럼 반환하지 않으면 worker 작업이 끝났어도 완료 콜백을 처리하지 못한다. 이벤트 루프로 제어권이 돌아오지 않기 때문이다.

## 정리

동시성 코드를 읽을 때는 다음을 확인한다.

1. 어느 스레드에서 실행되는가?
2. 어떤 데이터가 공유되는가?
3. 요청 객체는 완료 시점까지 살아 있는가?
4. 잠금이나 atomic이 필요한가?
5. 완료 처리는 어느 스레드에서 JavaScript로 돌아오는가?

## 이벤트 루프가 잠들고 깨어나는 상태

이벤트 루프가 작업을 처리할 때는 메인 스레드가 C++ 또는 JavaScript 코드를 실행한다.

```text
[깨어 있음]
타이머·완료 큐·소켓 이벤트 처리
```

처리할 일이 없으면 libuv는 운영체제의 대기 시스템 호출로 들어간다.

```text
Linux  → epoll_wait()
macOS → kevent()
Windows → IOCP 대기
```

이 상태를 “이벤트 루프가 잠들었다”고 표현한다. `sleep(5)`처럼 무조건 시간을 기다리는 것이 아니라, 이벤트가 발생하거나 타임아웃이 되면 즉시 반환하는 대기다.

```text
uv_run()
  ↓
uv__io_poll()
  ↓
OS 대기
  ↓ 이벤트 발생
poll 반환
  ↓
libuv 콜백 처리
```

## `uv_run()`과 콜백의 관계

작업이 끝날 때마다 새로운 `uv_run()`을 호출하는 것은 아니다. Node가 `uv_run()`을 시작하면 그 안의 반복이 poll에서 대기하고, 운영체제 이벤트나 완료 큐가 대기를 깨운다.

```text
Node가 uv_run() 호출
  ↓
이미 실행 중인 이벤트 루프가 poll 대기
  ↓
이벤트 발생
  ↓
같은 uv_run() 흐름이 콜백 처리
```

JavaScript callback 안에서 `fs.readFile()`을 다시 호출해도 새 이벤트 루프가 만들어지는 것이 아니다. 현재 loop에 새 작업이 등록되고 기존 `uv_run()`이 다음 반복에서 처리한다.

## JavaScript가 이벤트 루프를 막는 경우

```js
fs.readFile('file.txt', () => {
  console.log('done');
});

while (true) {}
```

작업 등록 자체는 가능하지만 사용자 JavaScript가 반환하지 않으면 Node가 `SpinEventLoopInternal()`과 `uv_run()`으로 제어권을 넘기지 못할 수 있다. 이미 이벤트 루프 callback 안에서 무한 루프를 실행한 경우에도 C++ 이벤트 루프로 돌아오지 못한다.

```text
OS 작업이 끝났음
  ≠
메인 스레드가 완료 콜백을 처리할 수 있음
```

## 네트워크와 스레드풀 구분

TCP 소켓은 보통 `epoll`, `kqueue`, IOCP에 등록되어 이벤트 루프가 준비 상태를 기다린다. 파일 작업처럼 모든 네트워크 작업이 worker thread에서 실행되는 것은 아니다. 다만 DNS의 일부 경로처럼 네트워크 관련 기능도 스레드풀을 사용할 수 있다.
