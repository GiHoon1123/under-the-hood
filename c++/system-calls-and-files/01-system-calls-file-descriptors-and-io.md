# 시스템 콜, 파일 디스크립터, 파일 I/O

상태: 기초 정리 완료

## 시스템 콜

시스템 콜은 사용자 프로그램이 운영체제 커널에 기능을 요청하는 통로다.

```text
Node JavaScript
  ↓
Node C++
  ↓
libuv
  ↓ 시스템 콜
운영체제 커널
  ↓
파일 시스템·디스크·네트워크
```

일반 프로그램은 디스크나 장치를 직접 제어하지 않고 운영체제에 요청한다.

## 파일 디스크립터

운영체제는 열린 파일과 소켓을 숫자로 식별한다.

```cpp
int fd = open("file.txt", O_RDONLY);
```

`fd`는 파일 이름 자체가 아니라 운영체제가 관리하는 열린 파일 정보의 번호다.

```text
0 → stdin
1 → stdout
2 → stderr
3 이상 → 파일·소켓 등
```

소켓과 파이프도 운영체제에 따라 파일 디스크립터와 유사한 방식으로 다뤄진다.

## `read()`와 버퍼

```cpp
ssize_t read(int fd, void* buffer, size_t count);
```

뜻은 `fd`가 가리키는 대상에서 최대 `count` 바이트를 읽어 `buffer` 주소에 쓰라는 것이다.

```cpp
char data[1024];
ssize_t result = read(fd, data, sizeof(data));
```

`data`는 실제 결과가 들어갈 메모리 버퍼이고, `read()`의 반환값은 실제 읽은 바이트 수 또는 오류다.

## `read()`와 `pread()`

```text
read()
  → 현재 파일 위치에서 읽고 위치가 이동할 수 있음

pread()
  → 지정한 offset에서 읽고 현재 파일 위치와 분리됨
```

libuv의 `uv_fs_read()`는 읽기 offset에 따라 운영체제의 적절한 경로를 사용한다.

## 오류와 `errno`

```cpp
ssize_t result = read(fd, data, sizeof(data));

if (result < 0) {
  // errno에 원인이 들어 있을 수 있음
}
```

대표적인 오류는 다음과 같다.

```text
ENOENT → 파일이 없음
EACCES → 권한 없음
EBADF  → 잘못된 파일 디스크립터
EINTR  → 신호로 작업 중단
```

Node는 운영체제 오류를 libuv 오류 코드와 Node JavaScript `Error`로 변환한다.

```text
커널 오류
  ↓
libuv 오류 코드
  ↓
Node C++ 변환
  ↓
JavaScript Error
```

## Node/libuv 파일 읽기 경로

```text
fs.promises.readFile()
  ↓
lib/internal/fs/promises.js
  ↓
internalBinding('fs')
  ↓
src/node_file.cc
  ↓
uv_fs_read()
  ↓
deps/uv/src/unix/fs.c
  ↓
read() / pread()
  ↓
운영체제
```

결과는 완료 큐와 Node C++ 완료 함수를 통해 반대 방향으로 돌아온다.

## 네트워크와 파일 디스크립터

네트워크 소켓도 운영체제의 비동기 I/O 대상으로 등록할 수 있다.

```text
socket()
  ↓
소켓 디스크립터
  ↓
read()/write() 또는 recv()/send()
```

다만 파일 작업은 스레드풀을 사용하는 경우가 많고, 소켓은 `epoll`, `kqueue`, Windows IOCP 같은 이벤트 감시 경로를 사용하는 등 세부 방식이 다르다.

## 정리

시스템 콜은 Node가 운영체제에 도달하는 경계다. 파일 디스크립터는 열린 파일·소켓을 식별하고, 버퍼는 커널이 결과를 써 넣을 메모리 공간이다.

## 사용자 영역과 커널 영역

Node, V8, libuv의 일반 코드는 사용자 영역에서 실행된다. `read()`나 `kevent()` 같은 시스템 콜을 호출하면 CPU가 커널 영역으로 진입하고, 커널이 권한 있는 파일 시스템이나 장치 작업을 수행한다.

```text
사용자 영역
  └─ Node / V8 / libuv
       ↓ 시스템 콜
커널 영역
  └─ 파일 시스템·네트워크·장치 드라이버
```

커널은 작업 결과를 사용자 영역의 버퍼에 기록하고 호출자에게 바이트 수나 오류를 반환한다.

## 파일 읽기와 버퍼의 관계

```cpp
char data[1024];
ssize_t bytes = read(fd, data, sizeof(data));
```

여기서 `data`는 운영체제가 결과를 써 넣을 실제 메모리 공간이고 `fd`는 열린 파일을 가리키는 번호다. `read()`의 반환값이 5라면 앞의 5바이트가 유효하다는 뜻이다. 버퍼 전체가 항상 채워진다는 뜻은 아니다.

## `read()`와 `pread()`의 차이

파일에는 현재 위치가 있을 수 있다.

```text
read(fd, buffer, length)
  → 현재 위치에서 읽고 위치가 이동할 수 있음

pread(fd, buffer, length, offset)
  → 지정한 offset에서 읽고 현재 위치와 분리
```

libuv의 `uv_fs_read()`는 요청의 offset과 플랫폼 구현에 따라 적절한 시스템 호출 경로를 선택한다.

## 네트워크 소켓도 운영체제 객체다

소켓은 파일과 같은 일반 데이터 파일은 아니지만 운영체제가 관리하는 핸들이며 Unix에서는 파일 디스크립터로 표현되는 경우가 많다.

```text
socket()
  ↓
소켓 디스크립터
  ↓
운영체제 네트워크 스택
  ↓
epoll/kqueue 또는 IOCP 이벤트
```

그래서 libuv는 소켓을 이벤트 루프에 등록하고, 읽기·쓰기 준비 이벤트를 Node C++에 전달할 수 있다.
