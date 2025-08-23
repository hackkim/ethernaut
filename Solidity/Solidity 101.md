# Solidity 101 핵심 정리

## 1. Solidity 소개

**Solidity는 이더리움 스마트 컨트랙트를 위한 고급 프로그래밍 언어입니다.**

- **개발자**: Gavin Wood 제안 (2014), Christian Reitwiessner, Alex Beregszaszi 등이 개발
- **영향받은 언어**: C++ (문법, OOP), Python (modifier, 다중상속), JavaScript (초기 함수 스코핑)
- **특징**: 정적 타입, 상속 지원, 라이브러리, 복잡한 사용자 정의 타입

## 2. 파일 구조 및 Pragma 지시문

### 파일 레이아웃 순서
1. SPDX 라이센스 식별자
2. Pragma 지시문
3. Import 구문
4. 컨트랙트 내부 순서: 상태변수 → 이벤트 → modifier → constructor → 함수

### Pragma 종류
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.3;  // 버전 지정
pragma abicoder v2;       // ABI 인코더 버전
pragma experimental SMTChecker;  // 실험적 기능
```

## 3. 핵심 언어 구성요소

### 컨트랙트 구조
- **State Variables**: 블록체인에 영구 저장되는 변수
- **Functions**: 실행 가능한 코드 단위
- **Modifiers**: 함수 동작을 수정하는 선언적 방법
- **Events**: 블록체인 로그에 저장되는 이벤트
- **Constructor**: 컨트랙트 생성 시 한 번만 실행

### 특수 함수
```solidity
receive() external payable { }  // 이더 수신 함수
fallback() external [payable] { }  // 기본 대체 함수
```

## 4. 데이터 타입

### Value Types (값 타입)
- **Boolean**: `bool` (true/false)
- **Integers**: `uint8`~`uint256`, `int8`~`int256`
- **Address**: `address`, `address payable`
- **Fixed-size bytes**: `bytes1`~`bytes32`
- **Enum**: 사용자 정의 열거형

### Reference Types (참조 타입)
- **Arrays**: 정적/동적 배열
- **Strings & Bytes**: `string`, `bytes`
- **Structs**: 사용자 정의 구조체
- **Mappings**: `mapping(keyType => valueType)`

### 데이터 위치
- **storage**: 상태변수 저장소 (영구)
- **memory**: 함수 실행 중 임시 메모리
- **calldata**: 외부 함수 매개변수 (수정 불가)

## 5. 함수 및 가시성

### 가시성 지정자
- **public**: 내부/외부 모두 호출 가능
- **external**: 외부에서만 호출 가능
- **internal**: 현재 컨트랙트와 상속받은 컨트랙트에서만
- **private**: 현재 컨트랙트에서만

### 상태 변경 지정자
- **view**: 상태를 읽기만 가능
- **pure**: 상태 읽기/쓰기 모두 불가
- **payable**: 이더 수신 가능

### Function Modifier 예제
```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "Not owner");
    _;  // 함수 본문 실행 지점
}
```

## 6. 고급 기능

### 상속 및 다형성
- 다중 상속 지원 (C3 linearization)
- `super` 키워드로 부모 함수 호출
- 함수 오버로딩 지원

### 라이브러리 및 인터페이스
- **Library**: 재사용 가능한 코드 모음
- **Interface**: 컨트랙트 명세 정의

### 어셈블리
- Inline Assembly를 통한 저수준 접근 가능

## 7. 에러 처리

### 에러 처리 함수
```solidity
require(condition, "Error message");  // 입력 검증
assert(condition);                    // 내부 에러 체크
revert("Error message");              // 명시적 되돌리기
```

### Try/Catch
```solidity
try externalContract.method() returns (uint result) {
    // 성공 시 실행
} catch Error(string memory reason) {
    // require/revert 에러 처리
} catch Panic(uint errorCode) {
    // assert/panic 에러 처리
} catch {
    // 기타 에러 처리
}
```

## 8. 내장 함수 및 글로벌 변수

### 블록체인 정보
```solidity
block.timestamp    // 현재 블록 타임스탬프
block.number      // 현재 블록 번호
msg.sender        // 함수 호출자
msg.value         // 전송된 이더 양
tx.origin         // 트랜잭션 발신자
```

### 암호화 함수
```solidity
keccak256(bytes)     // Keccak-256 해시
sha256(bytes)        // SHA-256 해시
ecrecover(hash, v, r, s)  // 서명 복원
```

### ABI 인코딩/디코딩
```solidity
abi.encode(...)           // ABI 인코딩
abi.decode(data, (...))   // ABI 디코딩
abi.encodeWithSelector()  // 셀렉터와 함께 인코딩
```

## 9. 프로그래밍 스타일 가이드

### 코드 레이아웃
- **들여쓰기**: 4개 공백 사용
- **최대 줄 길이**: 79자 (또는 99자)
- **인코딩**: UTF-8 또는 ASCII
- **Import**: 파일 최상단에 위치

### 명명 규칙
- **컨트랙트/라이브러리**: CapWords (예: `SimpleToken`)
- **함수**: mixedCase (예: `getBalance`)
- **변수**: mixedCase (예: `totalSupply`)
- **상수**: UPPER_CASE_WITH_UNDERSCORES (예: `MAX_BLOCKS`)
- **modifier**: mixedCase (예: `onlyOwner`)

### 함수 순서
1. constructor
2. receive function
3. fallback function
4. external functions
5. public functions
6. internal functions
7. private functions

## 10. 보안 고려사항

### 주의할 점
- **재진입 공격**: 외부 호출 전 상태 변경 완료
- **정수 오버플로우**: 0.8.0부터 기본 체크됨
- **가스 한계**: `transfer`/`send`는 2300 가스 제한
- **타임스탬프 조작**: `block.timestamp` 의존성 최소화

### 베스트 프랙티스
- 입력 검증에 `require` 사용
- 내부 에러 체크에 `assert` 사용
- 명확한 에러 메시지 제공
- NatSpec 주석으로 문서화
