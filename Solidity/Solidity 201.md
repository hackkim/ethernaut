# Solidity 201 고급 개념 정리

## 1. 상속과 다형성

### 다중 상속
- **C3 Linearization**: Python과 유사하게 다이아몬드 문제 해결
- **함수 호출 순서**: 오른쪽에서 왼쪽으로 깊이 우선 탐색
- **다형성**: 가장 파생된 컨트랙트의 함수가 실행됨

### 함수 오버라이딩
```solidity
// 기본 컨트랙트
function foo() public virtual returns (string memory) {
    return "Base";
}

// 파생 컨트랙트
function foo() public override returns (string memory) {
    return "Derived";
}
```

### 컨트랙트 타입
- **Abstract Contract**: 구현되지 않은 함수가 있는 컨트랙트
- **Interface**: 모든 함수가 external이고 구현되지 않음
- **Library**: `DELEGATECALL`로 재사용되는 코드

## 2. 스토리지 레이아웃

### 기본 원칙
- **슬롯 크기**: 각 스토리지 슬롯은 32바이트 (256비트)
- **패킹**: 32바이트보다 작은 여러 값들이 하나의 슬롯에 저장됨
- **순서**: 선언 순서대로 slot 0부터 저장

### 패킹 규칙
```solidity
contract StorageExample {
    uint128 a;  // slot 0 (16 bytes)
    uint128 b;  // slot 0 (16 bytes) - 같은 슬롯에 패킹
    uint256 c;  // slot 1 (32 bytes)
}
```

### 동적 배열과 매핑
- **동적 배열**: `keccak256(slot)`에서 시작하여 데이터 저장
- **매핑**: 키 `k`의 값은 `keccak256(h(k) . slot)`에 저장
- **bytes/string**: 31바이트 이하는 length*2, 이상은 length*2+1

## 3. 메모리 레이아웃

### 예약된 메모리 영역
- **0x00-0x3f**: 해싱용 스크래치 공간
- **0x40-0x5f**: 현재 할당된 메모리 크기 (free memory pointer)
- **0x60-0x7f**: zero slot (동적 배열 초기값용)

### Free Memory Pointer
- **시작 위치**: 0x80 (128바이트)
- **할당**: 새 메모리 객체는 포인터 위치에 할당
- **해제 없음**: 메모리는 트랜잭션 종료시까지 해제되지 않음

## 4. Inline Assembly (Yul)

### 기본 문법
```solidity
assembly {
    let x := 7
    let y := add(x, 3)
    if lt(y, 10) { 
        sstore(0, y) 
    }
}
```

### 변수 접근
- **값 타입**: 직접 사용 가능
- **스토리지 변수**: `x.slot`, `x.offset` 사용
- **메모리/calldata**: 주소 반환

## 5. Solidity 버전별 주요 변경사항

### v0.6.0 주요 변경
- **함수 오버라이딩**: `virtual`/`override` 키워드 필수
- **배열 길이**: 읽기 전용, `push()`/`pop()` 사용
- **receive/fallback**: 명확한 구분
- **try/catch**: 외부 호출 오류 처리

### v0.7.0 주요 변경
- **함수 호출 문법**: `x.f{gas: 10000, value: 2 ether}(arg)`
- **now 폐지**: `block.timestamp` 사용
- **gwei 키워드**: 리터럴 표현 지원

### v0.8.0 주요 변경
- **오버플로우 체크**: 기본적으로 활성화, `unchecked {}` 블록 사용
- **ABI Coder v2**: 기본 활성화
- **결합성**: `a**b**c`는 `a**(b**c)`로 해석
- **Panic vs Revert**: 내부 오류는 `Panic(uint256)` 사용

## 6. 보안 베스트 프랙티스

### 중요한 검사들
```solidity
// Zero address 체크
require(user != address(0), "Zero address");

// tx.origin vs msg.sender 체크
require(msg.sender == tx.origin, "Contract not allowed");

// SafeMath 사용 (0.8.0 이전)
using SafeMath for uint256;
```

## 7. OpenZeppelin 라이브러리

### 토큰 표준
#### ERC20 주요 함수
```solidity
// 기본 함수
totalSupply() → uint256
balanceOf(address account) → uint256
transfer(address recipient, uint256 amount) → bool
approve(address spender, uint256 amount) → bool
transferFrom(address sender, address recipient, uint256 amount) → bool

// 추가 함수
increaseAllowance(address spender, uint256 addedValue) → bool
decreaseAllowance(address spender, uint256 subtractedValue) → bool
```

#### ERC721 (NFT) 주요 함수
```solidity
balanceOf(address owner) → uint256
ownerOf(uint256 tokenId) → address
transferFrom(address from, address to, uint256 tokenId)
safeTransferFrom(address from, address to, uint256 tokenId)
approve(address to, uint256 tokenId)
setApprovalForAll(address operator, bool approved)
```

#### ERC777 (고급 토큰)
- **Hooks**: `tokensReceived`, `tokensToSend`
- **Operators**: 토큰 대신 전송 가능한 주소들
- **ERC20 호환**: 하위 호환성 제공

#### ERC1155 (멀티 토큰)
- **단일 컨트랙트**: 여러 토큰 타입 관리
- **배치 연산**: `safeBatchTransferFrom`, `balanceOfBatch`
- **가스 효율성**: 대량 토큰 관리시 유리

### 접근 제어
#### Ownable
```solidity
modifier onlyOwner() {
    require(owner() == msg.sender, "Not owner");
    _;
}
```

#### AccessControl (RBAC)
```solidity
bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

modifier onlyRole(bytes32 role) {
    require(hasRole(role, msg.sender), "Missing role");
    _;
}
```

### 보안 모듈
- **ReentrancyGuard**: `nonReentrant` modifier
- **Pausable**: 긴급 정지 메커니즘
- **PullPayment**: Pull 결제 패턴

### 암호화 및 서명
- **ECDSA**: 서명 검증 유틸리티
- **MerkleProof**: 머클 트리 증명
- **EIP712**: 구조화된 데이터 서명

### 프록시 패턴
#### 투명 프록시
```solidity
// 관리자만 업그레이드 함수 호출 가능
// 일반 사용자는 구현체 함수만 호출 가능
TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(
    implementation,
    admin,
    initData
);
```

#### 비콘 프록시
```solidity
// 여러 프록시가 하나의 비콘을 참조
UpgradeableBeacon beacon = new UpgradeableBeacon(implementation);
BeaconProxy proxy = new BeaconProxy(beacon, initData);
```

### 유틸리티
- **Multicall**: 여러 함수 호출 배치 처리
- **Create2**: 결정적 주소로 컨트랙트 배포
- **Clones**: EIP-1167 최소 프록시 복제

## 8. DeFi 프로토콜

### WETH (Wrapped Ether)
```solidity
// ETH → WETH
function deposit() public payable {
    balanceOf[msg.sender] += msg.value;
}

// WETH → ETH
function withdraw(uint wad) public {
    require(balanceOf[msg.sender] >= wad);
    balanceOf[msg.sender] -= wad;
    msg.sender.transfer(wad);
}
```

### Uniswap V2
- **AMM 공식**: `x * y = k` (Constant Product)
- **유동성 공급**: LP 토큰으로 지분 증명
- **Flash Swap**: 선불 없이 토큰 인출 후 같은 트랜잭션에서 상환

### Uniswap V3
- **집중 유동성**: 특정 가격 범위에 유동성 집중
- **다중 수수료**: 위험도에 따른 차등 수수료
- **향상된 오라클**: 최대 9일간 TWAP 제공

### Chainlink Price Feeds
```solidity
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

AggregatorV3Interface internal priceFeed;

function getLatestPrice() public view returns (int) {
    (,int price,,,) = priceFeed.latestRoundData();
    return price;
}
```

## 9. 개발 패턴 및 라이브러리

### Dappsys 라이브러리
- **DSProxy**: 코드 실행을 위한 프록시
- **DSMath**: 안전한 산술 연산
- **DSAuth**: 유연한 권한 관리
- **DSRoles**: 역할 기반 접근 제어

### 고정소수점 연산
- **WAD**: 18 decimal precision (10^18)
- **RAY**: 27 decimal precision (10^27)
- **기본 연산**: `wmul`, `wdiv`, `rmul`, `rdiv`

## 10. 가스 최적화 팁

### 스토리지 최적화
```solidity
// Bad - 3 slots
struct BadPacking {
    uint128 a;
    uint256 b;  
    uint128 c;
}

// Good - 2 slots  
struct GoodPacking {
    uint128 a;
    uint128 c;
    uint256 b;
}
```

### 함수 호출 최적화
- **External > Public**: calldata vs memory
- **View/Pure**: 상태 읽기/수정 제한
- **Batch Operations**: 여러 작업을 하나의 트랜잭션으로

이 정리본은 Solidity의 고급 개념들과 실제 DeFi 개발에서 사용되는 주요 라이브러리 및 프로토콜들을 포괄적으로 다룹니다. 각 섹션은 실무에서 바로 활용할 수 있는 코드 예제와 모범 사례를 포함하고 있습니다.
