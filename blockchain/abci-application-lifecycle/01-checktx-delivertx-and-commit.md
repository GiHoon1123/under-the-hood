# ABCI 애플리케이션 생명주기

상태: 초안 · 공개 프로토콜 기준

## 합의 엔진과 애플리케이션의 역할

ABCI는 Tendermint와 애플리케이션을 연결하는 인터페이스다.

```text
Tendermint: 블록 제안·투표·합의
     │ ABCI
     ▼
Application: 트랜잭션 검사·실행·상태 저장
```

Tendermint가 “어느 블록을 채택할지”를 결정하고, 애플리케이션이 “그 블록을 상태에 어떻게
적용할지”를 계산한다.

## 트랜잭션 제출과 `CheckTx`

사용자가 raw transaction을 제출하면 노드는 먼저 사전 검사를 수행한다.

```text
사용자 → RPC → CheckTx → mempool
```

대표적인 검사:

```text
인코딩 형식
서명
chain ID
nonce 또는 sequence
잔액과 수수료
가스 제한
```

`CheckTx` 성공은 확정을 뜻하지 않는다. 두 트랜잭션이 같은 잔액을 기준으로 각각 통과해도,
블록에서 앞선 트랜잭션이 잔액을 사용하면 뒤의 트랜잭션은 실행에 실패할 수 있다.

## 블록 실행 순서

```text
BeginBlock
  ↓
DeliverTx(tx1)
DeliverTx(tx2)
  ↓
EndBlock
  ↓
Commit
```

### BeginBlock

블록 시간, 블록별 가스 한도, 블록 카운터, 이전 상태 등의 실행 컨텍스트를 준비한다.

### DeliverTx

확정된 블록의 트랜잭션을 정해진 순서로 실행한다.

```text
현재 상태 + tx1 → 중간 상태
중간 상태 + tx2 → 다음 중간 상태
```

실행 중에는 상태 캐시나 snapshot을 사용할 수 있다.

```text
실행 성공 → 변경사항 유지
실행 실패 → 해당 변경사항 rollback
```

실패한 트랜잭션도 블록에 포함될 수 있으며, 사용한 가스나 수수료는 체인 규칙에 따라
청구될 수 있다.

### EndBlock

블록 전체 실행이 끝난 뒤 검증자 업데이트, 보상 계산, 만료 상태 정리 같은 블록 단위
후처리를 수행한다.

### Commit

성공한 변경사항을 영구 저장하고 새로운 상태 대표값을 반환한다.

```text
Sₙ + blockₙ → Sₙ₊₁
                  ↓
               Commit
                  ↓
             appHash/state root
```

## `CheckTx`와 `DeliverTx`의 차이

| 단계 | 목적 | 상태의 의미 |
|---|---|---|
| `CheckTx` | mempool에 넣을 수 있는지 검사 | 확정 전, 다른 tx가 먼저 실행될 수 있음 |
| `DeliverTx` | 확정 블록의 tx 실행 | 블록 순서와 현재 상태를 실제로 적용 |
| `Commit` | 결과 영구 저장 | 다음 블록의 기준 상태가 됨 |

## 결정성 주의점

ABCI 메서드는 여러 노드에서 같은 블록에 대해 실행된다. 로컬 시간, 랜덤 값, 외부 HTTP,
정렬되지 않은 map 순회, 플랫폼별 시스템 호출을 실행 결과에 직접 사용하면 안 된다.

```text
같은 블록 + 같은 이전 상태
        ↓ 모든 노드에서 동일한 계산
같은 appHash
```

## 상태 캐시와 rollback

트리 전체를 복사하지 않고 변경된 key를 임시 상태에 기록한다.

```text
영구 상태: Alice = 100
임시 상태: Alice = 80
```

성공하면 Commit에서 영구 저장하고, 실패하면 임시 변경만 버린다. 이 구조는 디스크를
부분적으로 쓴 뒤 다시 되돌리는 비용과 원자성 문제를 줄인다.

## 정리

```text
CheckTx  = 후보 tx 사전 검사
BeginBlock = 블록 컨텍스트 준비
DeliverTx = 순서대로 상태 전이
EndBlock  = 블록 후처리
Commit    = 상태 저장과 커밋 계산
```

## `CheckTx`를 통과해도 `DeliverTx`가 실패할 수 있는 이유

`CheckTx`와 `DeliverTx` 사이에는 합의와 다른 트랜잭션의 실행이 있다.

```text
초기 Alice 잔액: 100
tx1: Alice → Bob 80
tx2: Alice → Carol 50
```

각 트랜잭션이 제출될 때는 잔액 100을 기준으로 사전 검사를 통과할 수 있다. 그러나 블록에서
tx1이 먼저 실행되면 Alice의 잔액은 20이 되고 tx2는 `DeliverTx`에서 잔액 부족으로 실패한다.

따라서 `CheckTx`는 “후보로 받을 수 있는가”를 확인하고, `DeliverTx`는 “현재 블록 순서와
실제 상태에서 실행되는가”를 다시 확인한다.

## 캐시와 rollback

실행 중에는 디스크에 바로 쓰기보다 임시 상태에 기록한다.

```text
영구 상태: Alice = 100
임시 상태: Alice = 80
```

실행이 실패하면 임시 상태를 버리거나 snapshot으로 되돌린다. 디스크에 여러 변경을 부분적으로
기록한 뒤 복구하는 비용과 원자성 문제를 피할 수 있다. 성공한 변경만 `Commit`에서 상태
트리와 영구 저장소에 반영한다.

## 실패한 트랜잭션과 가스

트랜잭션이 실패했다고 반드시 블록에서 제거되는 것은 아니다. 실행 결과는 실패로 기록되고
상태 변경은 rollback되지만, 이미 사용한 계산량과 수수료는 체인 규칙에 따라 청구될 수 있다.

```text
상태 변경: 되돌림 가능
계산 자원: 이미 사용했으므로 비용 청구 가능
```

## 참고 자료

- Tendermint ABCI specification
- Cosmos SDK ABCI application lifecycle documentation
