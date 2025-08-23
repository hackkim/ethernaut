# Ethernaut Level 26: Double Entry Point - Writeup

## 문제 개요

이 레벨은 `CryptoVault` 컨트랙트의 `sweepToken` 기능에서 발생하는 Double
Entry Point 취약점을 다룹니다.\
CryptoVault는 **100개의 DET 토큰**과 **100개의 LGT 토큰**을 보유하고
있으며, **DET 토큰은 절대 인출되어서는 안 됩니다.**

**목표**: Detection Bot을 구현하여 Forta 시스템에 등록하고 DET 토큰
유출을 방지

------------------------------------------------------------------------

## 취약점 분석

### 1. 공격 시나리오

1.  `CryptoVault.sweepToken(LGT_ADDRESS)` 호출\
2.  `LegacyToken.transfer()` 호출\
3.  delegate가 설정되어 있어 `DoubleEntryPoint.delegateTransfer()` 호출\
4.  실제로는 **DET 토큰이 CryptoVault에서 유출됨**

### 2. 핵심 취약점

#### LegacyToken의 transfer 함수

``` solidity
function transfer(address to, uint256 value) public override returns (bool) {
    if (address(delegate) == address(0)) {
        return super.transfer(to, value);
    } else {
        return delegate.delegateTransfer(to, value, msg.sender);
    }
}
```

-   `sweepToken(LGT)` 실행 시, `delegateTransfer()`가 호출됨\
-   결국 `CryptoVault`의 DET 토큰이 잘못 전송됨

#### DoubleEntryPoint의 delegateTransfer 함수

``` solidity
function delegateTransfer(
    address to,
    uint256 value,
    address origSender
) public override onlyDelegateFrom fortaNotify returns (bool) {
    _transfer(origSender, to, value);
    return true;
}
```

-   `origSender == CryptoVault` → DET 토큰 전송 발생

------------------------------------------------------------------------

## 보호 메커니즘 분석

`fortaNotify` modifier:

``` solidity
modifier fortaNotify() {
    address detectionBot = address(forta.usersDetectionBots(player));
    uint256 previousValue = forta.botRaisedAlerts(detectionBot);
    
    forta.notify(player, msg.data);
    _;
    
    if(forta.botRaisedAlerts(detectionBot) > previousValue) 
        revert("Alert has been triggered, reverting");
}
```

-   Detection Bot이 alert 발생 시 거래가 revert됨

------------------------------------------------------------------------

## 해결 방법

### Detection Bot 구현

가장 간단한 방법: **모든 delegateTransfer 호출에 대해 alert 발생**

``` solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function raiseAlert(address user) external;
}

contract FakeDetectionBot is IDetectionBot {
    IForta public forta;

    constructor(address fortaAddress) {
        forta = IForta(fortaAddress);
    }

    function handleTransaction(address user, bytes calldata) external override {
        forta.raiseAlert(user);
    }
}
```

------------------------------------------------------------------------

## 배포 및 등록 과정

1.  Remix에서 배포 (constructor 파라미터: Forta 주소)\
2.  브라우저 콘솔에서 등록

``` javascript
const detectionBotAddress = "배포된_주소";
await forta.methods.setDetectionBot(detectionBotAddress).send({from: player});
```

3.  등록 확인

``` javascript
const registeredBot = await forta2.methods.usersDetectionBots(player).call();
console.log("Registered bot:", registeredBot);
```

------------------------------------------------------------------------

## 작동 원리

1.  LGT 토큰 sweep 시도\
2.  `delegateTransfer` 호출\
3.  `fortaNotify` 작동 → Detection Bot 호출\
4.  Detection Bot이 무조건 `raiseAlert()` 실행\
5.  Alert 감지 → 거래 revert\
6.  **DET 토큰 유출 방지 성공**

------------------------------------------------------------------------

## 교훈

-   **토큰 설계 주의**: 여러 진입점(delegate/forwarding)을 허용하는
    패턴의 위험성\
-   **실시간 모니터링 필요**: Forta 같은 보안 시스템의 중요성\
-   **단순함의 힘**: 복잡한 조건보다 단순 차단이 더 효과적일 수 있음

------------------------------------------------------------------------

## 대안적 해결 방법

보다 정교한 Detection Bot:

``` solidity
contract AdvancedDetectionBot is IDetectionBot {
    address private cryptoVault;
    
    constructor(address _cryptoVault, address _forta) {
        cryptoVault = _cryptoVault;
    }
    
    function handleTransaction(address user, bytes calldata msgData) external {
        bytes4 delegateTransferSelector = bytes4(keccak256("delegateTransfer(address,uint256,address)"));
        
        if (msgData.length >= 100) {
            bytes4 selector;
            assembly {
                selector := calldataload(msgData.offset)
            }
            
            if (selector == delegateTransferSelector) {
                (,, address origSender) = abi.decode(msgData[4:], (address, uint256, address));
                
                if (origSender == cryptoVault) {
                    IForta(msg.sender).raiseAlert(user);
                }
            }
        }
    }
}
```

-   `delegateTransfer` 호출 여부, `origSender`가 `CryptoVault`인지까지
    검사 가능

------------------------------------------------------------------------

## 결론

이 레벨은 **DeFi 프로토콜에서 발생할 수 있는 토큰 상호작용 문제**를 잘
보여준다.

### 배운 핵심 교훈

-   토큰의 이중 진입점(Double Entry Point)은 위험하다\
-   Forta 같은 모니터링 시스템은 효과적 방어책\
-   단순 차단 vs 정교한 검증 → 상황에 맞는 설계 필요
