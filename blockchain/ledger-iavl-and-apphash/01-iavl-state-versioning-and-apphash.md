# IAVL 상태 버전과 `appHash`

상태: 초안 · 공개 개념 기준

## IAVL이란

IAVL은 Immutable AVL tree다.

- AVL tree: 키 순서를 기준으로 균형을 유지하는 이진 탐색 트리
- Immutable: 커밋된 버전을 직접 덮어쓰기보다 새 버전을 만드는 방식

```text
             root
            /    \
        key A    key C
                  /  \
              key B  key D
```

노드는 자식 hash와 자신의 key/value를 이용해 hash를 계산한다. 하나의 값이 바뀌면 leaf에서
root까지의 경로만 다시 계산하면 된다.

```text
leaf 변경 → 부모 hash 변경 → 상위 hash 변경 → 새 root
```

## 버전

```text
Version 10: Alice = 100
Version 11: Alice = 80
```

Version 10을 보존하고 Version 11을 새로 만들 수 있다. 이 특성은 과거 상태 조회와 Merkle
proof 생성에 유용하다.

## 실행 중 상태와 커밋된 상태

블록 실행 중에는 변경사항을 임시 상태나 mutable tree에 기록한다.

```text
커밋된 Version N
        ↓
작업 중인 상태
        ↓ tx 실행
변경된 key와 트리 경로
        ↓ Commit
커밋된 Version N+1
```

트리 전체를 매번 메모리에 복사하는 것은 아니다. 필요한 key를 읽고 변경된 값과 경로를
추적한 뒤 Commit에서 영구 저장한다.

## 캐시와 읽기 우선순위

```text
영구 상태: Alice = 100
캐시:       Alice = 80
```

실행 중 Alice를 읽을 때는 캐시의 80을 우선 반환해야 한다. 그렇지 않으면 한 트랜잭션 안에서
자신이 방금 변경한 값을 읽지 못하게 된다.

```text
캐시에 값 있음 → 캐시 반환
캐시에 값 없음 → 영구 상태 조회
```

## Snapshot과 rollback

```text
Snapshot 0: Alice = 100
        ↓
임시 변경: Alice = 80
        ↓ 실행 실패
Snapshot 0으로 복구
```

디스크에 매번 쓰고 되돌리는 대신 메모리 변경을 폐기하거나 snapshot으로 되돌리기 때문에
실패 처리가 저렴하고 부분 상태가 디스크에 남는 위험도 줄어든다.

## `appHash`

`appHash`는 애플리케이션이 현재 상태를 대표하도록 계산해 반환하는 커밋 값이다.

```text
블록 실행
  ↓
애플리케이션 상태 변경
  ↓
상태 저장소들의 커밋 계산
  ↓
appHash 반환
```

`appHash`가 항상 하나의 IAVL root와 같아야 하는 것은 아니다. 애플리케이션이 여러 상태
저장소를 관리한다면 여러 root를 정해진 순서로 결합할 수 있다.

```text
accountRoot
validatorRoot
governanceRoot
contractRoot
       ↓ 정해진 결합 규칙
     appHash
```

## `appHash`와 블록 해시

```text
blockHash:
  블록 헤더 전체의 식별자

appHash:
  블록 적용 후 애플리케이션 상태의 대표값
```

블록 헤더에 `appHash`가 포함되면 appHash의 변화가 최종 block hash에도 영향을 줄 수 있지만,
두 값이 같은 개념은 아니다.

## Merkle proof

특정 key/value가 root에 포함됐음을 증명하려면 값과 leaf에서 root까지의 형제 hash가 필요하다.

```text
값 + 형제 hash들 + 위치 정보
        ↓ 재계산
계산된 root == 기준 root?
```

기준 root는 블록 헤더나 신뢰하는 상태 커밋에서 얻는다. 전체 트리를 전달할 필요는 없다.

## 주의할 점

IAVL root, 애플리케이션 appHash, 블록의 transactionsRoot는 각각 다른 대상을 대표한다.

```text
transactionsRoot = 입력 트랜잭션 목록
IAVL root         = 특정 상태 트리
appHash           = 애플리케이션 전체 상태 커밋
blockHash         = 블록 헤더 식별자
```

## 정리

IAVL은 변경된 경로만 다시 계산하면서 버전별 상태를 보존하는 상태 트리다. 애플리케이션은
Commit에서 성공한 변경을 저장하고 하나의 IAVL root 또는 여러 상태 커밋을 조합해 `appHash`를
반환할 수 있다.

## 트리 전체를 캐시에 복사하는 것은 아니다

상태가 수백만 개의 key를 가지고 있어도 트리 전체를 메모리에 올리지 않는다. 필요한 key와
변경된 경로를 읽고 임시 상태에 기록한다.

```text
영구 트리에서 Alice 조회
  ↓
캐시에 Alice = 80 기록
  ↓
Commit 때 Alice leaf부터 root까지 갱신
```

실행 중 같은 Alice를 다시 읽으면 캐시의 80을 읽어야 한다. 영구 트리의 100을 다시 읽으면
트랜잭션 안에서 방금 한 변경이 사라진 것처럼 보인다.

## root가 바뀌는 범위

값 하나가 바뀌면 leaf 하나와 root까지의 조상 경로가 바뀐다.

```text
Alice leaf 변경
  → 부모 hash 변경
  → 상위 hash 변경
  → 새 IAVL root
```

트리 전체를 다시 hash하는 것이 아니라 변경 경로를 다시 계산하므로 큰 상태에서도 효율적이다.

## `appHash`와 다른 root

`transactionsRoot`는 입력 트랜잭션 목록, IAVL root는 특정 상태 트리, `appHash`는 애플리케이션
전체 상태 커밋, `blockHash`는 블록 헤더 식별자다.

```text
여러 상태 root
  → 애플리케이션이 정한 순서로 결합
  → appHash
```

따라서 `appHash = 특정 IAVL root = blockHash`라고 일반화하면 안 된다.

## Merkle proof의 구성

증명 대상 key/value, leaf에서 root까지 각 단계의 형제 hash, 각 형제의 왼쪽·오른쪽 위치,
비교 기준 root가 필요하다. 기준 root가 블록 헤더에 이미 있으면 proof가 중복해서 보내지
않을 수 있다.
