# 스마트 컨트랙트 감사 취약점 101가지

## 1. 토큰 처리 관련 취약점

### ERC20 호환성 문제
**문제**: 일부 ERC20 토큰은 실패시 `false`를 반환하고, 일부는 revert함
```solidity
// 취약한 코드
token.transfer(to, amount);

// 안전한 코드
require(token.transfer(to, amount), "Transfer failed");
// 또는 OpenZeppelin SafeERC20 사용
```

**심각도**: Medium-High  
**해결책**: `require()` 문이나 SafeERC20 라이브러리 사용

### 18자리 이상 소수점 토큰
**문제**: 18자리를 초과하는 소수점을 가진 토큰(예: YAMv2 24자리)
**해결책**: 토큰의 decimals 값을 동적으로 처리

### 수수료 부과 토큰
**문제**: USDT와 같이 전송시 수수료를 부과할 수 있는 토큰
**해결책**: 실제 받은 양과 전송 양을 비교하는 검증 로직 추가

### 컨트랙트 존재 확인 누락
**문제**: `safeTransfer`에서 컨트랙트 존재 여부 미확인
```solidity
// 취약한 코드 - 존재하지 않는 컨트랙트도 성공으로 처리
token.call(abi.encodeWithSignature("transfer(address,uint256)", to, amount));

// 안전한 코드
require(isContract(token), "Token contract does not exist");
```

## 2. 재진입(Reentrancy) 공격

### 플래시 론 재진입
**문제**: 플래시 론 실행 중 권한이 유지되어 악의적 함수 호출 가능
**해결책**: ReentrancyGuard 사용

### MetaSwap 재진입
**문제**: `swap()` 함수에서 재진입 공격 가능
```solidity
// 안전한 패턴
modifier nonReentrant() {
    require(!locked, "Reentrant call");
    locked = true;
    _;
    locked = false;
}
```

### ETH 전송 시 재진입
**문제**: `transfer()` 또는 `call()` 시 재진입 공격
**해결책**: Check-Effects-Interactions 패턴 준수

## 3. 접근 제어 문제

### 부적절한 권한 검증
**문제**: 중요한 함수에 접근 제어 누락
```solidity
// 취약한 코드
function setPrice(uint newPrice) external {
    price = newPrice;
}

// 안전한 코드
function setPrice(uint newPrice) external onlyOwner {
    price = newPrice;
}
```

### 초기화 함수 프론트러닝
**문제**: `initialize()` 함수가 프론트러닝에 취약
**해결책**: Factory 패턴 사용하거나 배포 스크립트 강화

### 관리자 권한 남용
**문제**: 관리자가 사용자 자산을 임의로 이동 가능
**해결책**: 시간 잠금(timelock) 메커니즘 도입

## 4. 수학 연산 오류

### 정수 오버플로우/언더플로우
**문제**: Solidity 0.8.0 이전 버전에서 오버플로우 미확인
```solidity
// Solidity 0.8.0 이전
using SafeMath for uint256;

// Solidity 0.8.0 이후
unchecked {
    // 의도적으로 오버플로우 허용시에만 사용
}
```

### 잘못된 계산 순서
**문제**: `rewardPerTokenStored`가 분수의 분자로 잘못 사용됨
```solidity
// 취약한 코드
result = (balance * rewardPerTokenStored) / totalSupply;

// 올바른 코드  
result = balance * (rewardPerTokenStored / totalSupply);
```

### 0으로 나누기
**문제**: 분모가 0인 경우 확인 누락
```solidity
function divide(uint a, uint b) public pure returns (uint) {
    require(b > 0, "Division by zero");
    return a / b;
}
```

## 5. 오라클 조작

### 원자적 프론트러닝 공격
**문제**: 오라클 업데이트를 샌드위치 공격으로 이용
**해결책**: 한 트랜잭션에서 구 가격 사용과 오라클 업데이트 금지

### 체인링크 구버전 API 사용
**문제**: 더 이상 사용되지 않는 `latestAnswer()` 함수 사용
**해결책**: 최신 AggregatorV3Interface 사용

### 고정 가격 가정
**문제**: USDC = 1 USD로 하드코딩
**해결책**: 실제 오라클 데이터 사용

## 6. 거버넌스 취약점

### 제안 취소 불가
**문제**: 큐에 대기 중인 거버넌스 제안을 취소할 방법 없음
**해결책**: `cancelTransaction` 함수 추가

### 관리자 권한 탈취
**문제**: 일반 제안으로도 관리자 변경 가능
**해결책**: 관리자 변경은 특별한 프로세스로만 허용

### Sybil 공격
**문제**: 다중 계정으로 쿼럼 우회 가능
**해결책**: 토큰 가중 투표로 쿼럼 계산

## 7. 프록시 및 업그레이드 문제

### 스토리지 레이아웃 충돌
**문제**: 업그레이드 시 스토리지 변수 순서 변경으로 데이터 손상
**해결책**: 스토리지 gap 예약 및 변수 순서 유지

### 초기화되지 않은 구현체
**문제**: 프록시가 구현체의 상태 변수에 접근 불가
**해결책**: 구현체에서 상수 사용하거나 초기화 함수 제공

### 업그레이드 안전성 미준수
**문제**: OpenZeppelin의 일반 라이브러리를 업그레이드 가능 컨트랙트에서 사용
**해결책**: `@openzeppelin/contracts-ethereum-package` 사용

## 8. 유동성 풀 및 AMM 문제

### 무한 승인 도용
**문제**: 라우터 컨트랙트에 대한 승인을 악용하여 토큰 도용
**해결책**: `msg.sender`에서만 토큰 전송

### 가격 조작
**문제**: 유동성 0인 상태에서 임의 가격 설정 가능
**해결책**: 최소 유동성 요구사항 설정

### 슬리피지 확인 오류
**문제**: 잘못된 반환값으로 슬리피지 확인
**해결책**: 올바른 변수로 최소 수령량 확인

## 9. 시간 및 상태 관리

### 초기화 전 실행 가능
**문제**: 타이머 설정 전에도 함수 실행 가능
**해결책**: 초기화 상태 확인 추가

### 만료된 옵션 거래 가능
**문제**: 만료되거나 중단된 옵션도 거래 가능
**해결책**: 전송 전 상태 확인

### 기간 간격 문제
**문제**: 보상 기간 사이의 간격으로 인한 추가 보상 발생
**해결책**: 연속적인 기간 시작 강제

## 10. 이벤트 및 모니터링

### 민감한 작업 후 이벤트 누락
**문제**: 중요한 상태 변경 후 이벤트 미발생
```solidity
function updateCriticalValue(uint newValue) external onlyOwner {
    criticalValue = newValue;
    emit CriticalValueUpdated(newValue); // 이벤트 추가 필요
}
```

### 잘못된 이벤트 데이터
**문제**: 상태 변경 전 구 데이터로 이벤트 발생
**해결책**: 상태 변경 후 이벤트 발생

## 11. 가스 및 성능 문제

### 무제한 반복문
**문제**: 배열 크기에 제한 없는 반복문으로 가스 한도 초과
```solidity
// 취약한 코드
for (uint i = 0; i < tokens.length; i++) {
    // 처리 로직
}

// 안전한 코드
uint end = Math.min(start + maxBatch, tokens.length);
for (uint i = start; i < end; i++) {
    // 처리 로직
}
```

### 외부 호출 실패로 전체 배치 실패
**문제**: 배치 처리 중 하나 실패시 전체 실패
**해결책**: NoThrow 변형 함수 구현

## 12. 검증 및 입력 처리

### 입력 검증 누락
**문제**: 함수 매개변수 유효성 검사 부족
```solidity
function setParameters(uint[] memory values) external {
    require(values.length > 0, "Empty array");
    require(values[0] > minValue, "Value too small");
    // 추가 검증...
}
```

### 반환값 확인 누락
**문제**: 외부 호출의 반환값 미확인
**해결책**: 모든 외부 호출 반환값 검증

## 13. 암호화 및 서명

### 서명 재사용 공격
**문제**: 하드포크 시 서명 재사용 가능
**해결책**: 서명에 chainID 포함

### ECDSA 복구 실패 처리
**문제**: `ecrecover` 실패시 `address(0)` 반환 미처리
**해결책**: 결과가 `address(0)`인지 확인

## 14. 경제적 인센티브 문제

### 제로 수수료 우회
**문제**: 가스 가격 0으로 수수료 회피
**해결책**: 최소 수수료 설정

### 일시적 스테이킹으로 이자 획득
**문제**: 30분 윈도우 이용한 순간 스테이킹
**해결책**: 이자율 업데이트 주기 조정

### 프론트러닝 인센티브
**문제**: 마켓 메이커의 프론트러닝 비용 절감
**해결책**: 수수료 구조 조정

## 주요 보안 원칙

1. **Check-Effects-Interactions 패턴**: 외부 호출 전 상태 변경 완료
2. **입력 검증**: 모든 매개변수 유효성 검사
3. **반환값 확인**: 모든 외부 호출 결과 검증
4. **재진입 방지**: ReentrancyGuard 사용
5. **권한 관리**: 최소 권한 원칙 적용
6. **타임락**: 민감한 작업에 지연 시간 도입
7. **이벤트 로깅**: 중요한 상태 변경시 이벤트 발생
8. **업그레이드 안전성**: 스토리지 레이아웃 보호
9. **오라클 보안**: 가격 조작 방지 메커니즘
10. **가스 최적화**: DoS 공격 방지를 위한 가스 제한

## 감사 도구 및 권장사항

- **정적 분석**: Slither, MythX
- **동적 분석**: Echidna, Manticore  
- **CI/CD 통합**: Crytic 플랫폼
- **라이브러리 사용**: OpenZeppelin Contracts
- **테스트**: 엣지 케이스 포함 포괄적 테스트
- **문서화**: 보안 고려사항 명확히 문서화
