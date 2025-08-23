# 스마트 컨트랙트 감사 취약점 201 - 추가 100가지

## 1. 문서화 및 테스트 문제

### 불완전한 테스트 커버리지
**문제**: 복잡한 DeFi 시스템에서 엣지 케이스와 실패 시나리오 미테스트
```solidity
// 권장사항: 모든 함수와 엣지케이스 커버
describe("Edge Cases", () => {
    it("should handle zero amounts", async () => {
        await expect(contract.deposit(0)).to.be.reverted;
    });
    
    it("should handle maximum values", async () => {
        await expect(contract.deposit(MAX_UINT256)).to.be.reverted;
    });
});
```

### Hook 수신 컨트랙트 문서 부족
**문제**: `withdrawTokenAndCall()` 같은 함수의 Hook 동작 미문서화
**해결책**: Hook 컨트랙트 개발자를 위한 명확한 가이드라인 제공

### 토큰 동작 제한 미문서화
**문제**: 지원하지 않는 토큰 타입 명시 부족
```solidity
/**
 * @notice 지원하지 않는 토큰 타입:
 * - 수수료 부과 토큰 (Fee-on-transfer)
 * - 디플레이셔너리 토큰 (Deflationary)
 * - 리베이싱 토큰 (Rebasing)
 * - 인플레이셔너리 토큰 (Inflationary)
 */
```

## 2. 입력 검증 및 매개변수 문제

### 생성자 입력 검증 부족
**문제**: 생성자에서 주소나 값 유효성 미검증
```solidity
// 취약한 코드
constructor(address _token, uint256 _rate) {
    token = _token;
    rate = _rate;
}

// 안전한 코드
constructor(address _token, uint256 _rate) {
    require(_token != address(0), "Invalid token");
    require(_rate > 0 && _rate <= MAX_RATE, "Invalid rate");
    token = _token;
    rate = _rate;
}
```

### 성숙도 값 검증 누락
**문제**: fyDAI 컨트랙트 배포시 만료일 검증 부족
**해결책**: 과거나 너무 먼 미래 날짜 방지

### 영주소 확인 누락
**문제**: 중요 함수에서 `address(0)` 확인 누락
```solidity
function transferOwnership(address newOwner) external {
    require(newOwner != address(0), "Zero address");
    owner = newOwner;
}
```

## 3. 접근 제어 및 권한 문제

### 로깅 접근 제어 누락
**문제**: 누구나 로거 컨트랙트에 로그 생성 가능
```solidity
modifier onlyAuthorized() {
    require(authorizedCallers[msg.sender], "Unauthorized");
    _;
}

function log(string memory message) external onlyAuthorized {
    emit LogMessage(msg.sender, message);
}
```

### 관리자 권한 과다
**문제**: 단일 계정이 모든 시스템 제어 가능
**해결책**: 
- 역할 기반 접근 제어 (RBAC)
- 다중 서명 지갑
- Timelock 컨트랙트

### KYC 관리자 지연 없는 권한 부여
**문제**: KYC_ADMIN_ROLE 즉시 부여로 자금 동결 위험
**해결책**: TimelockController 사용

## 4. 토큰 및 ERC20 관련 문제

### 반환값 미사용 문제
**문제**: `TokenUtils.withdrawTokens` 반환값 미사용
```solidity
// 취약한 코드
TokenUtils.withdrawTokens(token, amount);

// 안전한 코드
uint256 actualAmount = TokenUtils.withdrawTokens(token, amount);
require(actualAmount == amount, "Withdrawal failed");
```

### 18자리 초과 소수점 가정
**문제**: TOKEN_DECIMALS > 18일 때 언더플로우
```solidity
function adjustDecimals(uint256 amount) public view returns (uint256) {
    if (TOKEN_DECIMALS <= 18) {
        return amount * (10**(18 - TOKEN_DECIMALS));
    } else {
        return amount / (10**(TOKEN_DECIMALS - 18));
    }
}
```

### ERC20 비표준 동작 미대응
**문제**: approve/transfer 프론트러닝 취약점
**해결책**: OpenZeppelin SafeERC20 사용

## 5. 수학 연산 오류

### 보상 기간보다 큰 지속시간
**문제**: `rewardRate = reward/duration`에서 0으로 반올림
```solidity
function notifyRewardAmount(uint256 reward) external {
    require(reward > 0, "Reward must be positive");
    require(duration > 0, "Duration must be positive");
    require(reward >= duration, "Reward too small");
    rewardRate = reward / duration;
}
```

### SafeMath 사용 불충분
**문제**: 중요한 계산에서 SafeMath 누락
```solidity
// Solidity 0.8.0 이전
using SafeMath for uint256;

function calculateTrade(uint256 amount) public pure returns (uint256) {
    return amount.mul(price).div(PRECISION);
}
```

### 블록 연도 근사치 의존
**문제**: 하드코딩된 연간 블록 수로 인한 부정확성
**해결책**: 실제 블록 시간 기반 동적 계산

## 6. 거버넌스 및 관리 문제

### 제안 취소 과도한 권한
**문제**: 승인된 제안도 제안자가 취소 가능
```solidity
function cancel(uint proposalId) external {
    require(proposals[proposalId].proposer == msg.sender, "Only proposer");
    require(
        proposals[proposalId].state == ProposalState.Pending ||
        proposals[proposalId].state == ProposalState.Active,
        "Invalid state"
    );
    proposals[proposalId].canceled = true;
}
```

### 소유자 제거 불가
**문제**: `isOwner` 매핑이 업데이트되지 않음
```solidity
function setOwners(address[] memory newOwners) external {
    // 기존 소유자 제거
    for (uint i = 0; i < currentOwners.length; i++) {
        isOwner[currentOwners[i]] = false;
    }
    
    // 새 소유자 추가
    for (uint i = 0; i < newOwners.length; i++) {
        isOwner[newOwners[i]] = true;
    }
    currentOwners = newOwners;
}
```

### 쿼럼 0 설정 가능
**문제**: 투표 쿼럼을 0으로 설정 가능
```solidity
function setVotingQuorum(uint256 newQuorum) external onlyOwner {
    require(newQuorum > 0, "Quorum must be positive");
    votingQuorum = newQuorum;
}
```

## 7. 가스 및 성능 문제

### 하드코딩된 가스 한도
**문제**: 고정 가스 값으로 인한 미래 호환성 문제
```solidity
// 취약한 코드
recipient.call{gas: 2300}("");

// 개선된 코드 - 동적 가스 계산
uint256 gasToSend = gasleft() * 63 / 64; // EIP-150
recipient.call{gas: gasToSend}("");
```

### 무제한 루프 자원 소모
**문제**: 외부 호출이 포함된 무제한 루프
```solidity
// 취약한 코드
for (uint i = 0; i < tokens.length; i++) {
    IERC20(tokens[i]).transfer(recipient, amounts[i]);
}

// 개선된 코드 - 배치 크기 제한
function batchTransfer(address[] memory tokens, uint256 start, uint256 end) external {
    require(end <= tokens.length && start < end, "Invalid range");
    require(end - start <= MAX_BATCH_SIZE, "Batch too large");
    
    for (uint i = start; i < end; i++) {
        IERC20(tokens[i]).transfer(recipient, amounts[i]);
    }
}
```

### 계정 생성 스팸
**문제**: 계정 생성에 수수료 없어 스팸 가능
**해결책**: 계정 생성 수수료 또는 MAX_NLEVELS >= 32

## 8. 코드 품질 및 모범 사례

### 컴파일러 버전 비고정
**문제**: `^0.6.0` 같은 부동 pragma 사용
```solidity
// 취약한 코드
pragma solidity ^0.6.0;

// 안전한 코드
pragma solidity 0.8.19;
```

### 중복 코드
**문제**: 동일 로직이 여러 파일에 복사
**해결책**: 상속이나 라이브러리 사용

### 설명 없는 어셈블리 블록
**문제**: 복잡한 어셈블리 코드에 주석 없음
```solidity
/**
 * @dev 고정점 연산을 위한 어셈블리 블록
 * x^y 계산에서 가스 효율성을 위해 사용
 */
assembly {
    // 각 명령어에 대한 상세 설명
    let result := exp(x, y)
}
```

### ABIEncoderV2 사용
**문제**: 실험적 기능 사용으로 안정성 위험
**해결책**: 구조체 전달 대신 개별 매개변수 사용

## 9. 프록시 및 업그레이드 문제

### 상속 누락
**문제**: 컨트랙트가 인터페이스를 구현하지만 상속하지 않음
```solidity
// 권장: 명시적 상속
contract MyContract is IMyInterface {
    // 구현
}
```

### 초기화 함수 재진입 가능
**문제**: 초기화 중 외부 호출로 재진입 위험
```solidity
bool private initialized;

modifier initializer() {
    require(!initialized, "Already initialized");
    initialized = true;
    _;
}
```

## 10. 시간 기반 및 오라클 문제

### 체인링크 구버전 API
**문제**: 폐지된 `latestAnswer()` 함수 사용
```solidity
// 구버전 (취약)
int256 price = priceFeed.latestAnswer();

// 신버전 (안전)
(, int256 price, , uint256 timeStamp, ) = priceFeed.latestRoundData();
require(timeStamp > 0, "Round not complete");
```

### 블록 타임스탬프 신뢰성
**문제**: `block.timestamp` 조작 가능성
**해결책**: 타임스탬프 조작 위험성 문서화 및 사용자 경고

## 11. 프론트러닝 및 MEV 문제

### setFrozen 프론트러닝
**문제**: 관리자가 거래 전 컨트랙트 동결 가능
**해결책**: 동결 기간 제한 또는 영구 동결만 허용

### 승인 취소 프론트러닝
**문제**: ERC20 approve의 고전적 프론트러닝 취약점
```solidity
// 안전한 패턴
function safeApprove(IERC20 token, address spender, uint256 amount) internal {
    token.approve(spender, 0);
    token.approve(spender, amount);
}
```

## 12. 재진입 및 상태 관리

### 이벤트 순서 불일치
**문제**: 재진입 시 이벤트 순서 변경
```solidity
// Check-Effects-Interactions 패턴
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount, "Insufficient balance");
    
    balances[msg.sender] -= amount; // Effects
    emit Withdrawal(msg.sender, amount); // Effects
    
    payable(msg.sender).transfer(amount); // Interactions
}
```

### 변수 섀도잉
**문제**: 상속에서 같은 이름 변수 재정의
```solidity
// 기본 컨트랙트
contract Base {
    uint256 public totalSupply;
}

// 파생 컨트랙트 - 섀도잉 방지
contract Derived is Base {
    // uint256 public totalSupply; // 금지!
    uint256 public derivedSupply; // 다른 이름 사용
}
```

## 13. 구성 및 배포 문제

### 하드코딩된 주소
**문제**: 테스트 어려움과 재사용성 저해
```solidity
// 취약한 코드
address constant USDC = 0xA0b86a33E6d...;

// 개선된 코드
address public immutable USDC;

constructor(address _usdc) {
    require(_usdc != address(0), "Invalid USDC address");
    USDC = _usdc;
}
```

### 테스트와 프로덕션 상수 혼재
**문제**: `TEST_MODE` 플래그로 인한 가독성 저하
**해결책**: 별도 환경 설정

## 14. 경제적 및 인센티브 문제

### 청산자 인센티브 부족
**문제**: 청산 과정에서 적절한 보상 없음
```solidity
function liquidate(address borrower) external {
    require(isLiquidatable(borrower), "Not liquidatable");
    
    uint256 collateralAmount = getCollateralAmount(borrower);
    uint256 discountedAmount = collateralAmount * LIQUIDATION_BONUS / 100;
    
    // 청산자에게 할인된 담보 제공
    collateralToken.transfer(msg.sender, discountedAmount);
}
```

### 시장 비지급능력 위험
**문제**: 담보 가치가 부채보다 낮아질 위험
**해결책**: 
- 보수적 담보 비율
- 다양한 자산 포트폴리오
- 비상 시나리오 대응책

### 플래시 론 이자율 조작
**문제**: 플래시 론으로 안정 이자율 조작 가능
**해결책**: 이자율 업데이트 메커니즘 개선 및 모니터링

## 주요 보안 원칙 (추가)

15. **컴파일러 최적화 신중 사용**: 최적화 관련 버그 위험성 고려
16. **이벤트 로깅**: 중요한 작업 후 적절한 이벤트 발생
17. **반환값 처리**: 모든 외부 호출 반환값 확인
18. **경계값 테스트**: 0, 최대값 등 엣지 케이스 처리
19. **토큰 화이트리스트**: 검증된 토큰만 시스템에 허용
20. **문서화**: 모든 가정사항과 제약조건 명확히 문서화

## 품질 관리 도구

- **정적 분석**: Slither, Mythril, MythX
- **테스트 커버리지**: Istanbul, Solidity Coverage  
- **의존성 관리**: npm audit, Snyk
- **코드 포맷팅**: Prettier, Solhint
- **문서 생성**: Solidity Docgen, NatSpec

## 결론

이 추가 100가지 취약점들은 스마트 컨트랙트 개발에서 자주 간과되는 세부사항들과 시스템적 위험들을 포괄합니다. 특히 문서화, 테스트, 코드 품질 등 개발 프로세스 전반에 걸친 모범 사례의 중요성을 강조합니다. 

실제 프로덕션 배포 전에는 이러한 모든 측면을 종합적으로 검토하고, 지속적인 모니터링 및 개선 체계를 구축하는 것이 중요합니다.
