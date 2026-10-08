# 블록체인 상태 머신과 결정성

상태: 초안 · 개념 정리 완료

이 문서는 특정 회사 체인의 비공개 구현을 설명하지 않는다. Tendermint 계열 합의 엔진,
ABCI 애플리케이션, Ethereum/EVM에서 공통으로 이해해야 하는 공개 개념을 정리한다.

## 1. 블록체인은 복제된 결정적 상태 머신이다

블록체인은 여러 노드가 같은 입력을 받아 같은 상태 변화를 계산하고, 그 결과를 합의하는
복제된 결정적 상태 머신이다.

```text
같은 이전 상태 + 같은 순서의 블록 + 같은 규칙 = 같은 다음 상태
```

상태 전이는 다음처럼 표현할 수 있다.

```text
F(Sₙ, block) → Sₙ₊₁
```

- `Sₙ`: n번째 블록을 처리하기 전의 전체 상태
- `block`: 처리할 블록과 트랜잭션 목록
- `F`: 블록을 상태에 적용하는 규칙
- `Sₙ₊₁`: 처리 후의 새로운 상태

예를 들어:

```text
Sₙ = { Alice: 100, Bob: 50 }
block = Alice → Bob : 20
Sₙ₊₁ = { Alice: 80, Bob: 70 }
```

두 노드가 같은 `Sₙ`과 같은 블록을 사용하면 같은 `Sₙ₊₁`을 계산해야 한다. 한 노드만
Alice의 잔액을 90으로 계산하면 이후 상태 root와 합의가 갈라진다.

## 2. 상태·트랜잭션·블록

상태는 계정 잔액, nonce, 컨트랙트 코드와 저장소, 검증자와 voting power, 거버넌스 설정처럼
체인이 다음 블록을 처리하는 데 필요한 현재 값의 집합이다.

트랜잭션은 상태 변경 요청이다.

```text
Alice가 Bob에게 20 전송
검증자에게 지분 위임
스마트 컨트랙트 함수 호출
```

트랜잭션은 서명·nonce·잔액·가스 등을 검사하고 실행해야 실제 상태 변경이 된다.

블록은 순서가 있는 트랜잭션 묶음과 헤더다.

```text
Block N
├─ Header
├─ Transaction 1
├─ Transaction 2
└─ Transaction 3
```

`tx1 → tx2`와 `tx2 → tx1`은 같은 트랜잭션 집합이어도 다른 결과를 만들 수 있다.

## 3. 결정성을 깨뜨리는 대표적인 코드

### 현재 시간

```go
if time.Now().Unix() > deadline {
    reject()
}
```

노드마다 실행 시각이 다를 수 있다. 합의된 블록 헤더의 시간처럼 모든 노드가 공유하는
입력을 사용해야 한다.

### 랜덤 값

```go
value := rand.Intn(100)
```

난수 생성기 상태가 노드마다 다르면 결과도 달라진다. 랜덤성이 필요하면 블록 해시처럼
모든 노드가 동일하게 아는 값을 사용해 결정적으로 계산해야 한다.

### 플랫폼별 결과

운영체제별 파일 경로, 시간 처리, 정수 크기, 시스템 호출 결과를 합의 로직에 직접 사용하면
플랫폼마다 상태가 달라질 수 있다.

### map 순회 순서

Go의 map 순회 순서는 보장되지 않는다.

```go
for address, balance := range balances {
    write(address, balance)
}
```

해시나 직렬화에 순서를 사용한다면 키를 먼저 정렬해야 한다.

```go
keys := sortedKeys(balances)
for _, address := range keys {
    write(address, balances[address])
}
```

### 외부 네트워크

```go
price := fetchPriceFromExchange()
```

노드마다 응답이나 시점이 다를 수 있다. 외부 데이터가 필요하면 오라클이 합의 가능한
트랜잭션이나 검증된 메시지로 먼저 기록해야 한다.

### 부동소수점과 정렬

부동소수점은 십진수를 정확히 표현하지 못할 수 있으므로 토큰 금액은 최소 단위의 정수로
저장하는 것이 일반적이다. 또한 `[A, B, C]`와 `[C, A, B]`는 같은 집합이어도 순서가 있는
해시에서는 다른 입력이므로 키·이벤트·proof 항목의 정렬 규칙을 명시해야 한다.

## 4. ABCI와 상태 전이

ABCI(Application Blockchain Interface)는 Tendermint 합의 엔진과 애플리케이션을 연결한다.

```text
Tendermint: 블록 제안·검증자 투표·합의·블록 확정
       │ ABCI
       ▼
Application: 트랜잭션 검사·실행·상태 변경·커밋 반환
```

### `CheckTx`

아직 합의되지 않은 트랜잭션을 mempool에 넣어도 되는지 사전 검사한다.

```text
서명 형식 · 잔액 · nonce · 가스 · 트랜잭션 형식 검사
```

`CheckTx` 성공은 확정을 뜻하지 않는다. 그 사이 다른 트랜잭션이 잔액이나 nonce를 바꿀 수
있기 때문이다.

### `DeliverTx`

합의된 블록의 트랜잭션을 블록 순서대로 실제 상태에 적용한다.

```text
BeginBlock → DeliverTx(tx1...txN) → EndBlock → Commit
```

`CheckTx`를 통과한 송금도 앞선 트랜잭션이 잔액을 사용하면 `DeliverTx`에서 실패할 수 있다.

### `Commit`

성공한 상태를 영구 저장하고 상태를 대표하는 커밋을 반환한다.

```text
Sₙ + blockₙ → Sₙ₊₁ → Commit → appHash 또는 state root
```

## 5. root 값의 구분

```text
transactionsRoot: 블록의 트랜잭션 목록 커밋
stateRoot:        실행 결과 상태 트리 커밋
appHash:          애플리케이션 전체 상태 대표값
blockHash:        블록 헤더 전체의 식별자
```

트랜잭션 변경은 보통 다음처럼 전파된다.

```text
tx 변경 → transactionsRoot 변경 → 헤더 변경 → blockHash 변경
```

상태 변경은 다음처럼 전파된다.

```text
상태 변경 → state root 또는 여러 상태 root 변경 → appHash 변경
```

`appHash`는 일반적으로 블록 해시가 아니다. 블록 해시는 블록 자체를 식별하고, `appHash`는
블록 적용 이후 애플리케이션 상태를 대표한다.

## 6. 캐시·스냅샷·Commit

트리 전체를 메모리에 복사하지 않고 필요한 값과 변경사항을 임시 상태에 기록한다.

```text
영구 상태 트리 → 필요한 key 조회 → 메모리 캐시
                              ↓
                       변경된 key 기록
                              ↓
                 성공 시 변경된 경로만 Commit
```

```text
영구 상태: Alice = 100
캐시:       Alice = 80
```

실행 중 Alice를 다시 읽으면 캐시의 80을 우선 사용한다. 실행이 실패하면 캐시를 버리거나
snapshot으로 되돌리고, 성공한 변경만 영구 저장한다.

## 7. Merkle proof

Merkle proof는 전체 트리를 전달하지 않고도 특정 값이 root에 포함됐음을 증명한다.

```text
                 root
                /    \
             hAB      hCD
            /  \     /  \
          hA   hB   hC   hD
```

A를 증명하려면 A, 루트까지의 각 단계에 있는 형제 hash, 위치 정보, 비교 기준 root가 필요하다.

```text
hA = hash(A)
hAB = hash(hA || hB)
calculatedRoot = hash(hAB || hCD)
```

`calculatedRoot`가 기준 root와 같으면 A가 해당 트리에 포함됐다고 판단한다. 기준 root는
블록 헤더에서 이미 알고 있다면 proof에 중복해서 넣지 않아도 된다.

## 8. IAVL과 Ethereum trie

IAVL은 정렬된 키를 사용하는 균형 트리와 버전 관리에 초점을 둔다.

```text
Version 10: Alice = 100
Version 11: Alice = 80
```

Ethereum 상태 구조는 계정 trie와 계약 storage trie가 연결된 형태로 이해할 수 있다.

```text
storage leaf 변경
  → storage root 변경
  → 계정의 storageRoot 변경
  → 계정 trie 경로 변경
  → 최종 state root 변경
```

둘 다 변경된 경로의 hash를 다시 계산하고 root로 상태를 대표한다는 공통점이 있다.

## 정리

```text
블록체인 = 복제된 결정적 상태 머신
F(Sₙ, block) → Sₙ₊₁

CheckTx   = mempool에 넣기 전 사전 검사
DeliverTx = 합의된 블록의 트랜잭션 실행
Commit    = 성공한 상태 저장 및 상태 커밋 반환

transactionsRoot = 입력 트랜잭션 목록 커밋
stateRoot/appHash = 실행 결과 상태 커밋
blockHash         = 블록 헤더 식별자
```

## 남은 질문

- Tendermint proposer, prevote, precommit은 정확히 어떤 순서로 진행되는가?
- 검증자 세트와 voting power는 언제 다음 블록에 반영되는가?
- EVM의 가스와 상태 rollback은 어떤 규칙으로 연결되는가?
- Merkle proof의 비포함 증명은 어떻게 표현되는가?

## 참고 자료

- Tendermint Documentation: ABCI 및 합의 개념
- Cosmos SDK Documentation: 상태 저장과 IAVL 개념
- Ethereum Yellow Paper 및 Execution Layer 문서
- Ethereum Developer Documentation: Transactions와 Merkle Patricia Trie
