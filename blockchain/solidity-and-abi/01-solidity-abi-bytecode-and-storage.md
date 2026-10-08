# Solidity, ABI, bytecode와 상태 저장

상태: 초안 · 공개 Ethereum 구조 기준

## Solidity에서 EVM까지

EVM은 Solidity 소스를 직접 실행하지 않는다.

```text
Solidity 소스 → 컴파일 → creation/runtime bytecode → 배포 → EVM opcode 실행
```

## 컨트랙트 배포

배포 트랜잭션은 일반 송금과 달리 `to`가 없고 `data`에 creation bytecode가 들어간다.
creation bytecode는 constructor를 실행한 뒤 배포 후 사용할 runtime bytecode를 반환한다.

```text
배포 tx → creation bytecode 실행 → constructor 실행 → runtime bytecode 반환
```

constructor 코드는 배포 시 한 번 실행되고 이후 함수 호출에는 runtime bytecode만 사용된다.

## 계정과 코드 저장

컨트랙트 계정은 대략 `nonce`, `balance`, `storageRoot`, `codeHash`를 가진다. runtime bytecode
전체가 계정 leaf에 직접 들어가는 것이 아니라 `codeHash`가 코드 저장소의 바이트코드를 가리킨다.

```text
Account Trie → codeHash → Code Database → runtime bytecode
Account Trie → storageRoot → Storage Trie → slot/value
```

## ABI

ABI(Application Binary Interface)는 함수, 인자, 반환값, 이벤트를 bytes로 표현하는 규칙이다.

예를 들어 `transfer(address,uint256)` 함수의 ABI는 함수 이름·인자 타입·반환 타입을 설명하고,
클라이언트는 이 정보를 이용해 calldata와 반환값을 인코딩·디코딩한다.

## Function selector

함수 시그니처를 문자열로 만들고 Keccak-256을 계산한 뒤 앞 4바이트를 selector로 사용한다.

```text
transfer(address,uint256) → Keccak-256 → 앞 4바이트 selector
```

## Calldata와 bytecode의 구분

runtime bytecode는 EVM이 실행하는 프로그램이고 calldata는 selector와 ABI 인자를 담은 입력이다.

EVM이 runtime bytecode를 프로그램 카운터 0부터 실행하면 bytecode 안의 dispatch 로직이
`CALLDATALOAD(0)`으로 calldata의 앞부분을 읽고 selector를 비교한다.

```text
runtime bytecode 실행
  → calldata 앞 4바이트 추출
  → selector 비교
  → 함수 코드 위치로 jump
```

함수 구현은 bytecode의 중간 어느 위치에든 있을 수 있다. dispatch 코드가 해당 `JUMPDEST`로
이동시키기 때문이다. selector가 맞지 않으면 fallback, receive 또는 revert 경로로 갈 수 있다.

## ABI 인자 인코딩

ABI는 값을 32바이트 word 단위로 정렬한다. 주소가 20바이트여도 ABI word에서는 32바이트로
확장되고 `uint256`도 32바이트로 표현된다. 문자열·배열 같은 동적 타입은 offset, length,
data 영역으로 나뉜다.

```text
고정 영역 → 동적 데이터 위치(offset)
동적 영역 → length + 실제 bytes
```

## Opcode와 상태

`increment()`는 개념적으로 storage slot 읽기, 1 더하기, 다시 저장을 수행한다. 대표적인
opcode는 `PUSH`, `ADD`, `JUMP`, `SLOAD`, `SSTORE`, `CALL`, `RETURN`, `LOG`다.

`SLOAD`는 storage 상태를 읽고 `SSTORE`는 변경을 기록한다. 실행 중에는 메모리 캐시나
StateDB를 우선 사용하고, 성공한 결과만 Commit에서 storage trie에 반영한다.

## 이벤트

Solidity의 `Transfer(address indexed from, address indexed to, uint256 amount)` 같은 이벤트는
storage가 아니라 외부 검색을 위한 log다. `topics`에는 이벤트 식별자와 indexed 값이 들어가고,
indexed가 아닌 값은 `data`에 들어간다.

## 상태 root 변화

storage slot 하나가 바뀌면 `storage leaf → storageRoot → 계정의 storageRoot 필드 → account
trie 경로 → state root` 순서로 변경이 전파된다.

## 정리

컨트랙트 코드는 `codeHash`를 통해 조회되는 runtime bytecode이고, 상태는 `storageRoot` 아래에
저장된다. 함수 호출은 calldata selector를 runtime bytecode의 dispatch 로직이 해석하고,
해당 위치의 opcode가 상태를 읽고 변경한다.
