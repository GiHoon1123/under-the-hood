# Merkle proof와 라이트 클라이언트

상태: 초안 · 공개 구조 기준

## Merkle proof의 목적

라이트 클라이언트가 전체 상태를 저장하지 않고도 특정 값이 상태 root에 포함됐는지 확인할
수 있게 하는 증명이다.

```text
전체 트리 대신
값 + root까지의 형제 hash + 위치 정보
```

## 작은 트리 예시

```text
                 root
                /    \
             hAB      hCD
            /  \     /  \
          hA   hB   hC   hD
```

A를 증명하려면 A, hB, hCD와 각 hash의 왼쪽·오른쪽 위치가 필요하다.

```text
hA = hash(A)
hAB = hash(hA || hB)
calculatedRoot = hash(hAB || hCD)
```

계산한 root가 블록 헤더나 신뢰하는 상태 커밋의 root와 같으면 A가 포함됐다고 판단한다.
root는 이미 알고 있는 기준값이면 proof에 중복해서 보낼 필요가 없다.

## 값이 바뀌면

```text
A → A'
  ↓
leaf 변경 → 부모 hash 변경 → root 변경
```

예전 root와 proof는 새 상태에서 더 이상 통하지 않는다. 그래서 root는 상태의 지문처럼
사용된다.

## Non-membership proof

어떤 key가 없다는 것도 트리 구조로 증명할 수 있다.

```text
key가 있어야 할 경로를 따라감
  ↓
빈 노드 또는 다른 key의 leaf에 도달
  ↓
해당 key가 없음을 증명
```

정확한 proof 형식은 IAVL과 Ethereum trie처럼 트리 구조에 따라 달라진다.

## 라이트 클라이언트

풀 노드는 전체 블록·트랜잭션·상태를 저장하고 직접 실행할 수 있다. 라이트 클라이언트는
신뢰할 수 있는 헤더, 상태 root, 검증자 커밋, 필요한 proof만 유지한다.

```text
풀 노드:
  전체 데이터를 저장하고 직접 계산

라이트 클라이언트:
  작은 커밋과 proof를 받아 직접 검증
```

## 잔액 조회 예시

```text
1. 블록 헤더에서 기준 state root 확보
2. 노드에 Alice의 값과 Merkle proof 요청
3. leaf부터 root까지 hash 재계산
4. 계산한 root와 헤더의 root 비교
```

```text
계산한 root == 기준 root
→ Alice의 값이 해당 상태에 포함됨
```

라이트 클라이언트는 전체 트리를 받지 않았지만 이 값을 검증할 수 있다.

## 왜 헤더가 필요한가

악성 노드가 값과 proof를 함께 조작할 수 있기 때문에 proof만 받아서는 안 된다.

```text
조작된 값 + 조작된 proof → 조작된 root
조작된 root ≠ 신뢰하는 헤더의 root → 거부
```

따라서 proof 검증에는 신뢰할 수 있는 기준 root가 필요하다.

## Tendermint 계열 라이트 검증

Tendermint 계열에서는 헤더와 검증자 커밋도 확인해야 한다.

```text
신뢰하는 헤더
  ↓
검증자 서명과 voting power 확인
  ↓
다음 헤더가 이전 헤더와 연결되는지 확인
  ↓
상태 root/appHash 확보
  ↓
Merkle proof 검증
```

검증자 세트가 바뀌면 새 세트 변경이 이전 세트의 승인으로 정당한지 확인해야 한다.

## 한계

라이트 클라이언트도 처음에는 신뢰할 기준 헤더나 검증자 정보를 확보해야 한다.

```text
올바른 proof + 가짜 기준 root = 잘못된 상태를 믿을 위험
```

따라서 부트스트랩 과정과 헤더 검증이 중요하다.

## 정리

풀 노드는 데이터를 제공하고, 라이트 클라이언트는 신뢰하는 root와 proof를 사용해 데이터가
그 상태에 포함됐는지 직접 검증한다.

## proof에 실제로 필요한 값

“본인, 옆 노드, 루트 바로 아래 노드”라고만 설명하면 작은 4-leaf 예시에서는 맞아 보이지만,
일반적인 표현은 다음과 같다.

```text
증명 대상 key/value
+ leaf에서 root까지 각 단계의 형제 hash 하나씩
+ 각 형제의 왼쪽·오른쪽 위치
+ 비교 기준 root hash
```

루트 바로 아래 노드만 필요한 것이 아니라, 대상 leaf에서 root까지 올라가는 모든 단계의
형제 hash가 필요하다. 기준 root는 이미 블록 헤더로 알고 있으면 proof payload에 중복하지
않을 수 있다.

## 포함 증명과 비포함 증명

포함 증명은 특정 key/value에서 root까지 재계산해 root가 일치하는지 확인한다. 비포함 증명은
key가 있어야 할 경로를 따라가 빈 노드나 다른 key의 leaf에 도달함을 보여준다.

트리 종류마다 branch·extension·leaf 구조가 다르므로 IAVL proof와 Ethereum trie proof의
바이트 형식이 같다고 가정하면 안 된다.

## 라이트 클라이언트의 신뢰 시작점

라이트 클라이언트는 proof만 받는다고 자동으로 안전해지는 것이 아니다. 먼저 신뢰할 헤더나
검증자 세트를 확보해야 한다.

```text
신뢰할 헤더 확보
  → 검증자 커밋과 헤더 연결 확인
  → 상태 root 확보
  → 값과 proof 요청
  → root 재계산
```

가짜 기준 root를 사용하면 올바른 형식의 proof라도 잘못된 상태를 믿게 된다.
