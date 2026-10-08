# Ethereum 트랜잭션 인코딩·서명·검증

상태: 초안 · EIP-1559 타입 트랜잭션 기준

## 1. 트랜잭션 객체

예시 필드는 `chainId`, `nonce`, `maxPriorityFeePerGas`, `maxFeePerGas`, `gasLimit`, `to`,
`value`, `data`, `accessList`다. 개발자는 보통 라이브러리로 객체를 만들고 서명한다.

## 2. 서명 전 인코딩

EIP-1559 typed transaction은 타입 바이트 `0x02`로 시작한다.

```text
0x02 || RLP([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas,
             gasLimit, to, value, data, accessList])
```

`||`는 바이트 연결이다. 이 단계에는 아직 `yParity`, `r`, `s`가 없다.

## 3. 헥사 문자열

바이트를 RPC로 보내기 위해 헥사 문자열로 표현한다.

```text
bytes: 02 f8 6a ...
hex:   0x02f86a...
```

헥사 표현은 암호화가 아니다. 단지 바이트를 텍스트로 표시하는 방식이다.

## 4. 서명은 무엇을 하는가

서명 전 bytes를 Keccak-256으로 해시한다.

```text
signingBytes → Keccak-256 → messageHash
```

Ethereum은 secp256k1 계열 ECDSA를 사용한다. 개인키를 `d`, 기준점을 `G`, 임시값을 `k`,
메시지 해시를 정수 `z`라고 하면 개념적인 계산은 다음과 같다.

```text
R = k × G
r = Rₓ mod n
s = k⁻¹ × (z + r × d) mod n
```

최종 서명은 대략 `(yParity, r, s)`다. `privateKey + messageHash`는 덧셈이 아니라 이
알고리즘에 두 입력을 넣는다는 축약 표현이다.

서명은 내용을 숨기지 않는다. 개인키 소유자가 정확한 메시지를 승인했고, 서명 후 메시지가
바뀌지 않았음을 증명한다.

## 5. 서명된 raw transaction

서명값을 필드에 추가해 다시 인코딩한다.

```text
0x02 || RLP([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas,
             gasLimit, to, value, data, accessList, yParity, r, s])
```

이 결과를 헥사 문자열로 만들어 `eth_sendRawTransaction`에 전달한다. 제출하는 것은 hash만이
아니다. 원래 필드와 서명값이 모두 raw transaction 안에 있다.

## 6. 노드가 역순으로 검사하는 과정

노드는 hash를 역산하지 않는다. raw bytes를 디코딩하고 같은 signing bytes를 재구성한다.

```text
hex string → hex decode → type 0x02 확인 → RLP decode
```

그 결과로 트랜잭션 필드와 `yParity/r/s`를 얻는다. 서명 필드를 제외한 필드를 다시 같은
규칙으로 인코딩하고 Keccak-256을 계산한다.

```text
받은 필드 → 동일한 인코딩 → signingBytes → messageHash
```

노드는 `messageHash`, `r`, `s`, `yParity`로 공개키나 송신자 주소를 복원한다.

```text
messageHash + signature → 공개키 복원 → Ethereum address 계산
```

## 7. 서명 이후의 상태 검증

서명이 유효해도 실행이 성공한다는 뜻은 아니다.

```text
chainId·nonce·잔액·최대 수수료·gasLimit·트랜잭션 형식과 주소 길이 검사
```

```text
서명 유효 + 잔액 부족 → 실행 거부
서명 유효 + nonce 불일치 → 실행 거부 또는 대기
```

## 8. Nonce와 replay 방지

현재 nonce가 7이면 보통 nonce 7인 다음 트랜잭션을 처리한다. 처리 후 현재 nonce는 8이
된다. 이미 사용한 nonce 7을 다시 제출하면 오래된 트랜잭션으로 거부된다.

`chainId`도 서명 대상에 포함되므로 체인 A에서 만든 서명을 체인 B에서 재사용할 수 없다.

## 9. 가스와 실행

`gasLimit`은 최대 실행량이고 `gasUsed`는 실제 사용량이다.

```text
fee = gasUsed × 유효 gas price
```

계약 실행이 revert되면 상태 변경은 되돌아갈 수 있지만 이미 사용한 가스는 청구될 수 있다.

## 전체 흐름

```text
트랜잭션 객체 → typed RLP 인코딩 → signing hash 계산 → 개인키 서명
→ 서명값 추가 → raw hex transaction → RPC 제출 → 노드 디코딩
→ signing hash 재계산 → 송신자 복원·서명 검증 → nonce·잔액·가스 검사
→ 블록 포함 후 EVM 실행
```

## 서명이 실제로 계산되는 과정

`privateKey + messageHash`는 실제 덧셈이 아니다. 개인키를 정수 `d`, 곡선의 기준점을 `G`,
메시지 hash를 정수 `z`, 서명마다 새로 사용하는 임시값을 `k`라고 하면 secp256k1 ECDSA는
개념적으로 다음을 계산한다.

```text
R = k × G
r = Rₓ mod n
s = k⁻¹ × (z + r × d) mod n
```

서명은 `(r, s)`와 공개키 복원에 필요한 `yParity`로 표현된다. 임시값 `k`를 재사용하면
두 서명의 관계로 개인키가 노출될 수 있으므로 직접 암호 알고리즘을 구현하지 않고 검증된
라이브러리를 사용해야 한다.

## 노드가 서명을 검증하는 과정

노드는 private key를 받지 않는다.

```text
raw hex transaction
  → hex decode
  → type 0x02 확인
  → RLP decode
  → 서명 필드 분리
  → 나머지 필드로 signing bytes 재구성
  → Keccak-256 재계산
  → (r, s, yParity)로 공개키·주소 복원
```

노드는 hash를 역산해서 원문을 얻는 것이 아니다. raw transaction 안에 원래 필드가 들어
있기 때문에 디코딩하고, 같은 필드로 hash를 다시 계산해 서명을 검증한다.

## 서명 검증과 상태 검증의 차이

서명이 유효하다는 것은 Alice가 정확한 거래 내용을 승인했다는 뜻이지, 거래가 실행 가능하다는
뜻은 아니다.

```text
서명 검증: Alice의 개인키로 승인했는가?
상태 검증: 잔액·nonce·가스 조건을 만족하는가?
실행 검증: 계약 코드가 revert하지 않는가?
```

예를 들어 Alice가 잔액 1 ETH인데 100 ETH 송금에 서명하는 것은 가능하다. 서명은 유효하지만
잔액 검증에서 거부된다.

## RLP와 헥사 문자열

RLP 인코딩은 필드 순서와 바이트 표현을 고정한다. `amount`, `nonce` 순서가 바뀌거나 숫자를
문자열과 정수로 다르게 인코딩하면 signing hash와 서명이 달라진다. 헥사 문자열의 `0x`는
암호화가 아니라 바이트를 표시하는 접두사다.
