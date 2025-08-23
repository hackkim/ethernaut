# Ethernaut **Level 0** Write‑up

> 튜토리얼 레벨 · 게임의 기본 메커니즘과 웹3 콘솔 사용법을 익히기

---

## 📋 개요
**Ethernaut Level 0**는 게임의 기본 메커니즘을 익히는 튜토리얼 레벨입니다. 웹3 콘솔 사용법과 스마트 컨트랙트 상호작용 방법을 배우는 것이 목적입니다.

---

## 🛠 환경 설정
- **네트워크:** Sepolia 테스트넷  
- **지갑:** MetaMask  
- **도구:** 브라우저 개발자 콘솔

---

## 🔎 취약점 분석
이 레벨은 실제 취약점을 다루지 않고, **게임 플레이 방식과 상호작용 패턴을 학습**하는 데 초점을 둡니다.

---

## ✅ 해결 과정

### 1) 기본 정보 수집
```javascript
await contract.info();
// "You will find what you need in info1()."
```

### 2) 연쇄적 정보 탐색
```javascript
await contract.info1();
// "Try info2(), but with \"hello\" as a parameter."

await contract.info2("hello");
// "The property infoNum holds the number of the next info method to call."

await contract.infoNum();
// "42"

await contract.info42();
// "theMethodName() is the name of the next method."

await contract.theMethodName();
// "The method name is method7123949."

await contract.method7123949();
// "If you know the password, submit it to authenticate()."
```

### 3) 패스워드 획득
```javascript
await contract.password();
// "ethernaut0"
```

### 4) 인증 수행
```javascript
await contract.authenticate("ethernaut0");
// 트랜잭션 전송 및 승인
```

---

## 🧰 핵심 개념

### Web3 콘솔 활용
- `player`: 현재 플레이어 주소 확인  
- `getBalance(player)`: 이더 잔액 조회  
- `help()`: 사용 가능한 헬퍼 함수 목록  
- `ethernaut`: 메인 게임 컨트랙트 객체

### 컨트랙트 상호작용 패턴
- **정보 수집:** ABI를 통한 메서드 탐색  
- **순차 호출:** 각 메서드의 힌트를 따라 진행  
- **트랜잭션 실행:** 상태 변경 함수 호출 시 승인 필요  
- **결과 확인:** 블록체인에서 트랜잭션 처리 대기

---

## 🧩 문제 해결 & 디버깅

### 일반적인 오류
- `contract is not defined`: 인스턴스 로딩이 완료되기 전 접근
- `Transaction failed`: 가스비 부족, 네트워크 문제, 또는 잘못된 파라미터
- `Promise pending`: `await` 누락 또는 Promise 처리 미흡

### 디버깅 팁
```javascript
// 컨트랙트 인스턴스/ABI 확인
contract

// 트랜잭션 상태 확인
web3.eth.getTransactionReceipt("0x...")

// 이벤트 로그 모니터링
contract.getPastEvents("allEvents")
```

---

## 📚 학습 포인트

### 기술적 측면
- 웹3 라이브러리 기본 사용법
- 스마트 컨트랙트 ABI 탐색 방법
- 비동기 JavaScript 처리 (`async`/`await`)
- MetaMask 트랜잭션 승인 플로우

### 보안 관점
- 컨트랙트 메서드 가시성(`public` vs `private`)의 의미
- 온체인 데이터의 **투명성**
- **패스워드 하드코딩의 위험성**: 온체인에서 누구나 읽을 수 있음

---

## 🏁 결론
Level 0는 실제 해킹 기법보다는 **도구 사용법과 온체인 상호작용 흐름**에 집중합니다.  
그러나 **스마트 컨트랙트의 공개적 특성**과 **모든 데이터가 체인에서 접근 가능**하다는 점을 보여주므로, 실제 DApp 개발에서는 **민감한 정보를 컨트랙트에 직접 저장하지 말아야** 합니다.

---

## 🧪 코드 요약 (풀 스크립트)
```javascript
await contract.info();
await contract.info1();
await contract.info2("hello");
await contract.infoNum();
await contract.info42();
await contract.theMethodName();
await contract.method7123949();
await contract.password();
await contract.authenticate("ethernaut0");
```

> 재시도: Claude는 실수를 할 수 있습니다. 응답을 반드시 다시 확인해 주세요.
