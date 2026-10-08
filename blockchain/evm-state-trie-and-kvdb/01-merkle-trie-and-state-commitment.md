# Ethereum 상태 trie와 상태 커밋

상태: 초안 · 공개 구조 기준

## 계정 상태

Ethereum의 계정 상태는 계정 주소를 key로 하는 trie에 저장된다고 이해할 수 있다. 계정에는
대략 다음 정보가 있다.

```text
nonce
balance
storageRoot
codeHash
```

계정이 스마트 컨트랙트라면 `storageRoot`가 계약 저장소 trie를 가리킨다.

```text
Account Trie
  └─ Contract Account
       └─ storageRoot
            └─ Storage Trie
```

## Patricia trie의 경로

키를 16진수 nibble 경로로 나누어 저장하고, 공통 경로는 압축할 수 있다.

대표적인 노드 종류는 다음과 같다.

```text
Branch: 여러 경로가 갈라지는 지점
Extension: 공통 경로를 압축한 지점
Leaf: 실제 값에 도달하는 마지막 노드
```

전체 상태를 평평한 배열로 저장하는 것이 아니라 key 경로에 따라 노드를 연결한다.

## 값 하나가 바뀌는 과정

계약 storage slot 하나를 바꾼다고 하자.

```text
storage leaf 변경
  ↓
storage trie 경로 hash 변경
  ↓
새 storageRoot
  ↓
계정의 storageRoot 필드 변경
  ↓
account trie 경로 hash 변경
  ↓
최종 state root 변경
```

따라서 작은 storage 변경도 계정 trie의 최종 root까지 전파된다.

## EVM 실행과 임시 상태

EVM은 실행 중 계정과 storage의 변경을 메모리 상태 계층에 기록한다.

```text
영구 상태 trie
      ↓ 읽기
메모리 상태
      ↓ opcode 실행
변경된 계정·storage 추적
      ↓ 성공
Commit
```

실행이 revert되면 snapshot으로 돌아가 해당 호출의 변경사항을 폐기한다.

```text
Snapshot
  ↓
storage 변경
  ↓ revert
Snapshot 시점으로 복구
```

## 계정 trie와 storage trie의 차이

```text
계정 trie:
  주소 → nonce, balance, storageRoot, codeHash

storage trie:
  storage key → storage value
```

모든 계정이 별도의 storage trie를 갖는 것은 아니다. 일반 외부 소유 계정은 코드와 계약
저장소가 없는 계정으로 처리된다.

## KV DB와 trie

trie는 논리적인 상태 구조이고, 실제 노드 bytes는 key-value 데이터베이스에 저장될 수 있다.

```text
trie node hash → encoded trie node
```

KV DB를 직접 조회하는 것과 상태 trie의 논리적 key를 조회하는 것은 같은 말이 아니다.
트리 노드의 저장 key와 사용자 계정 주소·storage key를 구분해야 한다.

## State root

최종 state root는 현재 계정과 계약 storage 상태 전체를 대표한다.

```text
계정·계약 상태
      ↓ trie hash 계산
state root
```

두 노드가 같은 이전 state root와 같은 트랜잭션을 실행했는데 다른 root를 얻으면 실행 결과가
달라진 것이다.

## IAVL과 비교

IAVL은 정렬된 키를 가진 균형 트리와 버전 관리에 초점을 둔다. Ethereum trie는 계정 trie와
storage trie를 연결하고 key 경로를 압축하는 구조다.

```text
IAVL:
  key/value 상태 트리 + 버전

Ethereum trie:
  계정 상태 + 계약별 storage 상태 + state root
```

공통점은 변경된 경로의 hash를 다시 계산하고 root로 무결성을 대표한다는 것이다.

## 정리

상태 변경은 영구 DB에 무작정 바로 쓰는 것이 아니라 실행 중인 상태 계층에 먼저 반영하고,
성공한 결과를 Commit하면서 변경된 trie 경로와 root를 계산한다. 이 구조가 rollback과 상태
무결성 검증을 가능하게 한다.

## 변경사항과 캐시의 관계

EVM 실행 중에는 상태 trie 전체를 메모리에 복사하지 않는다. 접근한 계정과 storage slot을
읽고 변경된 항목을 메모리 상태 계층에서 추적한다.

```text
영구 trie
  ↓ 필요한 계정·slot 조회
메모리 StateDB 계층
  ↓ opcode 실행
변경 목록과 snapshot
  ↓ 성공
trie Commit
```

실패하면 snapshot으로 돌아가 해당 호출의 변경을 버린다. 디스크를 다시 읽어 모든 값을
복구하는 것보다 메모리 rollback 비용이 작고, 부분적으로 저장된 상태도 방지할 수 있다.

## 계정 상태 변경의 전파

외부 소유 계정의 잔액이나 nonce가 바뀌면 계정 leaf와 account trie 경로가 바뀐다. 계약
storage slot이 바뀌면 한 단계가 더 있다.

```text
storage slot
  → storage trie 경로
  → storageRoot
  → 계정의 storageRoot 필드
  → account trie 경로
  → state root
```

그래서 계약의 작은 storage 변경도 최종 state root에 영향을 준다.

## KV DB와 논리적 trie 구분

KV DB는 trie node bytes를 저장하는 물리적 저장 계층이고, trie는 계정과 storage key를
논리적으로 연결하는 상태 구조다.

```text
trie node hash → encoded trie node
```

사용자 계정 주소를 조회하는 것과 DB에서 내부 trie node를 직접 조회하는 것은 다른 작업이다.
