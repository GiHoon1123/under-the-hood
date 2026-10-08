# 스마트 컨트랙트 보안: 재진입, Flash loan, Front-running

상태: 초안 · 공개 Ethereum 보안 개념 기준

## 1. 보안의 기본 관점

스마트 컨트랙트는 배포 후 코드가 쉽게 바뀌지 않고 자산을 직접 보관할 수 있다. 공격자는
코드를 수정하는 것이 아니라 이미 존재하는 실행 경로와 상태 순서를 이용한다.

```text
버그 또는 취약한 가정
  ↓
공격자 트랜잭션
  ↓
EVM이 규칙대로 실행
  ↓
자산 이동 또는 상태 파괴
```

## 2. Reentrancy

외부 호출 중 공격자 컨트랙트가 원래 함수를 다시 호출하는 취약점이다.

```text
withdraw 호출
  ↓
외부 call
  ↓
공격자 fallback이 withdraw 재호출
  ↓
잔액이 아직 초기화되지 않음
  ↓
중복 출금
```

위험한 순서는 외부 호출 후 상태를 바꾸는 것이다. 일반적인 Checks-Effects-Interactions
패턴은 검사, 내부 상태 변경, 외부 호출 순서로 작성한다.

```text
검사 → balances[msg.sender] = 0 → 외부 call
```

재진입 방지 락을 사용할 수도 있지만, 외부 호출 전에 상태를 일관된 상태로 만드는 것이
기본이다.

## 3. Access control

`setOwner`나 `withdrawTreasury` 같은 이름만으로 권한이 생기지 않는다. `msg.sender`와
허용된 역할을 실제로 확인해야 한다.

```text
관리자 함수
  ↓
권한 검사
  ↓
상태 변경
```

## 4. Flash loan

Flash loan은 담보 없이 한 트랜잭션 안에서 자산을 빌리고 같은 트랜잭션 안에서 원금과
수수료를 갚는 기능이다.

```text
대출 → 콜백 실행 → 여러 프로토콜 호출 → 상환
```

상환하지 못하면 전체 트랜잭션이 revert되어 대출 프로토콜은 원금을 잃지 않는다. 문제는
공격자가 평소에는 가질 수 없는 큰 자본을 잠시 사용해 가격·유동성·담보 비율을 조작할 수
있다는 점이다.

## 5. Flash loan 가격 조작 예시

어떤 대출 프로토콜이 DEX의 현재 잔액 비율만 가격으로 사용한다고 하자.

```text
1. Flash loan으로 TokenA 대량 확보
2. DEX에서 TokenA 대량 매도
3. DEX의 가격 비율 왜곡
4. 대출 프로토콜이 조작된 가격 조회
5. 담보 가치가 부풀려진 상태로 대출
6. 차익 확보
7. Flash loan 상환
```

Flash loan 자체가 버그는 아니다. 순간 가격 하나를 신뢰하는 오라클과 조합될 때 취약점이
생긴다.

## 6. Flash loan 방어

```text
TWAP로 일정 시간 평균 가격 사용
여러 오라클 또는 여러 시장 가격 비교
비정상 가격 변화율 차단
담보 비율과 대출 한도 제한
재진입 방지
```

현재 가격 하나만 사용하면 한 트랜잭션에서 조작될 수 있지만, 시간 평균과 여러 공급원을
사용하면 조작 비용과 난도가 올라간다.

## 7. Front-running

사용자의 트랜잭션이 공개 mempool에 있는 동안 공격자가 내용을 보고 더 높은 우선순위로
자신의 트랜잭션을 먼저 넣는 공격이다.

```text
사용자 tx가 mempool 대기
  ↓
공격자가 calldata와 주문 내용 확인
  ↓
더 높은 수수료로 선행 tx 제출
  ↓
공격자 tx가 먼저 실행
```

## 8. Sandwich attack

DEX 거래를 공격자 거래 두 개가 감싸는 형태다.

```text
1. 공격자 매수
2. 사용자 매수
3. 공격자 매도
```

사용자의 큰 매수 때문에 가격이 올라가면 공격자는 앞에서 싸게 사고 뒤에서 비싸게 판다.
공격자에게 큰 자본이 필요할 수 있지만, Flash loan으로 자본을 빌릴 수도 있다.

## 9. Slippage

사용자는 최소 수령량 또는 최대 지불량을 지정해야 한다.

```text
예상 가격: 100
최대 허용 가격: 105
```

실제 가격이 105보다 나쁘면 거래를 revert한다. slippage를 너무 낮추면 정상 거래도 실패하고,
너무 높이면 sandwich 공격에 노출된다.

## 10. Front-running 방어

```text
slippage 제한
deadline 설정
private transaction relay
commit-reveal
batch auction
```

commit-reveal은 먼저 주문의 hash만 제출하고 나중에 원문과 secret을 공개한다.

```text
commit: hash(order + secret)
reveal: order + secret
검증: 같은 hash인지 확인
```

Private relay는 일반 공개 mempool을 거치지 않고 proposer에게 직접 전달한다. Batch auction은
일정 기간의 주문을 모아 순서 경쟁의 이점을 줄인다.

## 11. 기타 주요 취약점

```text
오라클:
  조작되거나 오래된 가격

정수 범위:
  overflow·underflow

가스 DoS:
  커지는 배열을 한 번에 순회

storage collision:
  proxy와 implementation의 슬롯 충돌

서명 replay:
  chain ID·nonce·domain이 없는 오프체인 서명 재사용
```

## 12. 점검 순서

```text
외부 호출이 있는가?
상태를 외부 호출 전에 일관되게 바꾸는가?
권한 검사가 모든 관리자 경로에 있는가?
가격이 단일 순간값에 의존하는가?
slippage와 deadline이 있는가?
반복문이 가스 한도를 넘을 수 있는가?
서명에 chain ID·nonce·domain이 포함되는가?
proxy storage layout이 호환되는가?
```

## 정리

Flash loan은 대규모 자본을 잠시 제공하는 기능이고, Front-running은 공개된 거래와 순서를
악용하는 공격이다. 둘은 함께 사용될 수 있다.

```text
공개 거래 관찰 → Flash loan으로 자본 확보 → 가격 조작
→ 피해자 거래 앞뒤에 배치 → 차익 확보 → 대출 상환
```
